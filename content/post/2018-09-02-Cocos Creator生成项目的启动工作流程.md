---
title: Cocos Creator生成项目的启动工作流程
categories:
  - Cocos Creator
date: 2018-09-02 10:46:11
updated: 2018-09-02 10:46:11
tags: 
  - Cocos Creator
---
最近才有机会来看一下这个东西，虽然非常的喜欢Lua，但是现有的项目还是懒得去转换成Lua了，毕竟 Cocos Creator 还是很好用的至少简单，所以来理一理其工作流程。

<!--more-->

# js 绑定

Cocos Creator 使用的是 Cocos2d-X 引擎的 js 绑定，开发语言也是 js 了。这里顺带提一下，关于用 Lua 还是 js 绑定的问题。主要的方式如下：**引擎开启一个 脚本运行（ Lua 是 Lua State，JavaScript 用的是 V8 等等），然后把 C++ 写的代码，注入到这个引擎内。这样，引擎内就可以以调用注入函数的形式，调用底层代码。**

而对于我们的用户而言，所看到的据，我们所编写的 Js/Lua 脚本，居然能够产生就跟原生代码一样的效果。

# 安卓的启动

我们用 Cocos Creator 打包好的安卓项目内，与通常的安卓项目没有什么太大的差异，不过是利用了一些 Jni 技术来加载底层代码。但我们现在只关注启动的流程。

启动的 Activity 是一个叫做 AppActivity 的东西，在其 `onCreate()` 函数内进行了 SDK 的初始化：

```java
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // Workaround in https://stackoverflow.com/questions/16283079/re-launch-of-activity-on-home-button-but-only-the-first-time/16447508
        if (!isTaskRoot()) {
            // Android launched another instance of the root activity into an existing task
            //  so just quietly finish and go away, dropping the user back into the activity
            //  at the top of the stack (ie: the last state of this task)
            // Don't need to finish it again since it's finished in super.onCreate .
            return;
        }
        // DO OTHER INITIALIZATION BELOW
        
        SDKWrapper.getInstance().init(this);
    }
```
SDKWrapper 也是由项目自动生成的类，我们可以看一下其内调用到的函数：

```java
	public void init(Context context) {
		if (PACKAGE_AS) {
			try {
				mClass.getMethod("init", Context.class).invoke(mClass, context);
			} catch (Exception e) {
				e.printStackTrace();
			}
			SDKWrapper.nativeLoadAllPlugins();
		}
		
	}
	private static native void nativeLoadAllPlugins();

```

对于 Jni 技术不是很了解，但是我只想看一下其主要过程就行了。最终都会执行到引擎的 C++ 类： `AppDelegate.cpp`。

在其中的一个方法内就能看到，加载初始化的代码：

```c++
bool AppDelegate::applicationDidFinishLaunching()
{
#if CC_TARGET_PLATFORM == CC_PLATFORM_IOS && PACKAGE_AS
    SDKManager::getInstance()->loadAllPlugins();
#endif
    // initialize director
    auto director = Director::getInstance();
    auto glview = director->getOpenGLView();
    if(!glview) {
#if(CC_TARGET_PLATFORM == CC_PLATFORM_WP8) || (CC_TARGET_PLATFORM == CC_PLATFORM_WINRT)
        glview = GLViewImpl::create("SCMJ");
#else
        glview = GLViewImpl::createWithRect("SCMJ", cocos2d::Rect(0,0,900,640));
#endif
        director->setOpenGLView(glview);
    }
    
    // set FPS. the default value is 1.0/60 if you don't call this
    director->setAnimationInterval(1.0 / 60);

    ScriptingCore* sc = ScriptingCore::getInstance();
    ScriptEngineManager::getInstance()->setScriptEngine(sc);

    se::ScriptEngine* se = se::ScriptEngine::getInstance();

    jsb_set_xxtea_key("0d948dcc-c014-46");
    jsb_init_file_operation_delegate();

#if defined(COCOS2D_DEBUG) && (COCOS2D_DEBUG > 0)
    // Enable debugger here
    jsb_enable_debugger("0.0.0.0", 5086);
#endif

    se->setExceptionCallback([](const char* location, const char* message, const char* stack){
        // Send exception information to server like Tencent Bugly.

    });

    jsb_register_all_modules();

#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID || CC_TARGET_PLATFORM == CC_PLATFORM_IOS) && PACKAGE_AS
    se->addRegisterCallback(register_all_anysdk_framework);
    se->addRegisterCallback(register_all_anysdk_manual);
#endif

    se->start();

    jsb_run_script("main.js");

    return true;
}
```

简单的解释据：

1. 初始化 openGL 视图。
2. 初始化脚本核心引擎。
3. 注入所有模块。
4. 启动脚本引擎
5. 执行脚本 *main.js*。

这样就将控制权转交给了脚本引擎中的 *main.js*。

# main.js

每个项目都会生成一个 *main.js* 文件。我是也 link 方式生成的项目，所以位于 build/jsb-link/main.js 。这个脚本，才会进行加载我们用 Cocos Creator 建立的项目内容。

```js
   if (window.jsb) {
        require('src/settings.js');
        require('src/jsb_polyfill.js');
        boot();
        return;
    }
```

上面这段代码，会在加载了我们的设置内容，统一接口文件后，开始进行启动操作。

这 boot() 函数，我们可以看到，进入初始场景（ settings 设置），加载项目相关的 js 然后启动游戏的代码：

```js
            // load scene
            cc.director.loadScene(launchScene, null,
                function () {
                    if (cc.sys.isBrowser) {
                        // show canvas
                        canvas.style.visibility = '';
                        var div = document.getElementById('GameDiv');
                        if (div) {
                            div.style.backgroundImage = '';
                        }
                    }
                    cc.loader.onProgress = null;
                    console.log('Success to load scene: ' + launchScene);
                }
            );
        };

        // jsList
        var jsList = settings.jsList;

        if (!false) {
            var bundledScript = settings.debug ? 'src/project.dev.js' : 'src/project.js';
            if (jsList) {
                jsList = jsList.map(function (x) {
                    return 'src/' + x;
                });
                jsList.push(bundledScript);
            }
            else {
                jsList = [bundledScript];
            }
        }

        // anysdk scripts
        if (cc.sys.isNative && cc.sys.isMobile) {
//            jsList = jsList.concat(['src/anysdk/jsb_anysdk.js', 'src/anysdk/jsb_anysdk_constants.js']);
        }

        var option = {
            //width: width,
            //height: height,
            id: 'GameCanvas',
            scenes: settings.scenes,
            debugMode: settings.debug ? cc.DebugMode.INFO : cc.DebugMode.ERROR,
            showFPS: (!false && !false) && settings.debug,
            frameRate: 60,
            jsList: jsList,
            groupList: settings.groupList,
            collisionMatrix: settings.collisionMatrix,
            renderMode: 0
        }

        cc.game.run(option, onStart);
```

当加载完我们项目相关的 js 后，就会把这些参数，传递给给 引擎的 game 对象 run 方法。启动游戏了


# jsb\_register\_all_modules 将相关的底层接口注册到js引擎

```c++
bool jsb_register_all_modules()
{
    se::ScriptEngine* se = se::ScriptEngine::getInstance();

    se->addBeforeInitHook([](){
        JSBClassType::init();
    });

    se->addBeforeCleanupHook([se](){
        se->garbageCollect();
        PoolManager::getInstance()->getCurrentPool()->clear();
        se->garbageCollect();
        PoolManager::getInstance()->getCurrentPool()->clear();
    });

    se->addRegisterCallback(jsb_register_global_variables);

    se->addRegisterCallback(run_prepare_script);

    se->addRegisterCallback(register_all_cocos2dx);
    se->addRegisterCallback(jsb_register_Node_manual);
    se->addRegisterCallback(register_all_cocos2dx_manual);
    se->addRegisterCallback(JSB_register_opengl);

...}
```
在这个函数中，首先获取一个 ScriptEngine(se) 的实例，然后就会进行一系列的加载操作。暂时我们不用关心一些与我们可能会使用到的东西无关的内容。我们关注一下，对于导出的 js 接口是怎么样导入的。

##     se->addRegisterCallback(run_prepare_script);
在引擎启动前，会预先的准备一个 jsb 环境，这个环境的建立，是使用 js 脚本编写的，存在于 *jsb_prepare.js* 中。

在脚本  *jsb_prepare.js* 中，定义了命名空间 **cc, jsb** 定义了 cc 中的一些方法，如 `cc.clone()`；定义了 *cc.Class* ，一个用来构造类的基类及其相关方法。以及更多内容。更详细的内容可以查看源码。

##     se->addRegisterCallback(register_all_cocos2dx);

注册很多很多函数了。

```c++
bool register_all_cocos2dx(se::Object* obj)
{
    // Get the ns
    se::Value nsVal;
    if (!obj->getProperty("cc", &nsVal))
    {
        se::HandleObject jsobj(se::Object::createPlainObject());
        nsVal.setObject(jsobj);
        obj->setProperty("cc", nsVal);
    }
    se::Object* ns = nsVal.toObject();

    js_register_cocos2dx_Acceleration(ns);
    js_register_cocos2dx_Action(ns);
    js_register_cocos2dx_FiniteTimeAction(ns);
    js_register_cocos2dx_ActionInstant(ns);
    js_register_cocos2dx_Hide(ns);
    js_register_cocos2dx_TMXObjectGroupInfo(ns);
    js_register_cocos2dx_Node(ns);
    js_register_cocos2dx_ParticleSystem(ns);
    js_register_cocos2dx_ParticleSystemQuad(ns);
    js_register_cocos2dx_ParticleSpiral(ns);
    js_register_cocos2dx_ActionInterval(ns);
    js_register_cocos2dx_MoveBy(ns);
    js_register_cocos2dx_MoveTo(ns);
    ....
 }
```

可以看到，这首先会把从 se 对象获取 *cc* 属性，如果获取的 *cc* 属性为空，就会新建一个。 同时，很直白的把这个属性的值，称呼为 ns ，也就是命名空间的意思。

接着，就会一个个的注入各个模块了。我们来看一下 Director 模块。

```c++
bool js_register_cocos2dx_Director(se::Object* obj)
{
    auto cls = se::Class::create("Director", obj, nullptr, nullptr);

    cls->defineFunction("pause", _SE(js_cocos2dx_Director_pause));
    cls->defineFunction("isPurgeDirectorInNextLoop", _SE(js_cocos2dx_Director_isPurgeDirectorInNextLoop));
    cls->defineFunction("setEventDispatcher", _SE(js_cocos2dx_Director_setEventDispatcher));
    cls->defineFunction("setContentScaleFactor", _SE(js_cocos2dx_Director_setContentScaleFactor));
    cls->defineFunction("getDeltaTime", _SE(js_cocos2dx_Director_getDeltaTime));
    cls->defineFunction("getContentScaleFactor", _SE(js_cocos2dx_Director_getContentScaleFactor));
    cls->defineFunction("getWinSizeInPixels", _SE(js_cocos2dx_Director_getWinSizeInPixels));
...
}
```

首先会 cc 的对象建立一个类，然后对类定义各个函数映射。就是这么简单的过程。

## se->addRegisterCallback(run_boot_script);

注入完成后，就会运行我们的启动脚本 *jsb_boot.js*。 这个函数会定义  *cc.sys* 空间，加载 *jsb.js* 脚本。已经定义一些单例的类。详细的内容还是需要自己看脚本哈。

```js
_initSys();

//+++++++++++++++++++++++++something about CCGame end+++++++++++++++++++++++++++++

jsb.urlRegExp = new RegExp("^(?:https?|ftp)://\\S*$", "i");

cc._engineLoaded = false;

(function (config) {
    require("script/jsb.js");
    cc._engineLoaded = true;
    console.log(cc.ENGINE_VERSION);
})();

```

## jsb.js

这个脚本才是把我们各种常用的函数给加载起来的。

```js
// JavaScript Bindings helper file
//

// DO NOT ALTER THE ORDER
require('script/jsb_cocos2d.js');
require('script/jsb_common.js');
require('script/jsb_property_impls.js');
require('script/jsb_property_apis.js');
require('script/jsb_create_apis.js');
require('script/extension/jsb_cocos2d_extension.js');

if (window.ccui) {
    require('script/ccui/jsb_cocos2d_ui.js');
    require('script/ccui/jsb_ccui_property_impls.js');
    require('script/ccui/jsb_ccui_property_apis.js');
    require('script/ccui/jsb_ccui_create_apis.js');
}

require('script/jsb_opengl_constants.js');
require('script/jsb_opengl.js');

if (window.sp) {
    require('script/jsb_spine.js');
}

if (window.dragonBones) {
    require('script/jsb_dragonbones.js');
}

require("script/jsb_audioengine.js");
require("script/jsb_cocosanalytics.js");

```

在此之后，才正式进入我们的项目的 *main.js* 启动环节。

# Cocos Creator 专有内容

我一直很纳闷，在 *main.js* 内用 `cc.game.run()` 启动游戏，到底是在哪里实现的这个方法，一直没有找到。后面谷歌良久，才发现，这是在 Cocos Creator 实现的。

在 [Cocos Creator Code](https://github.com/cocos-creator/engine/tree/44d068bea8120146521ec334827cb5b67a7d9b8f/cocos2d/core) 目录下，有几个提供给 Cocos Creator 使用的 js 模块。

我们以 [cc.game](https://github.com/cocos-creator/engine/blob/44d068bea8120146521ec334827cb5b67a7d9b8f/cocos2d/core/CCGame.js) 为例来看一下。

最后一句 `cc.game = module.exports = game;` 表明了，将这个模块导出到了 cc.game 。

# cc.game.run()

对于我们启动游戏的逻辑 `cc.game.run()`，其代码定义如下：

```js
    run: function (config, onStart) {
        this._initConfig(config);
        this.onStart = onStart;
        this.prepare(game.onStart && game.onStart.bind(game));
    }
```

其中  onStart() 是我们自己定义的（也是系统生成的，但我们可以进行修改）。我们把配置信息传递过来后 game 就会保留这些信息。然后运行 prepare() 方法。

# prepare(cb)

```js
 prepare (cb) {
        // Already prepared
        if (this._prepared) {
            if (cb) cb();
            return;
        }

        // Load game scripts
        let jsList = this.config.jsList;
        if (jsList && jsList.length > 0) {
            var self = this;
            cc.loader.load(jsList, function (err) {
                if (err) throw new Error(JSON.stringify(err));
                self._prepareFinished(cb);
            });
        }
        else {
            this._prepareFinished(cb);
        }
    }
```

这个很简单了，其实就是把我们配置好的 jsList 全部通过 loader 加载进来。然后就完成了。

之后，执行 准备结束函数 

## _prepareFinished(cb)


```js
    _prepareFinished (cb) {
        this._prepared = true;

        // Init engine
        this._initEngine();
        // Log engine version
        console.log('Cocos Creator v' + cc.ENGINE_VERSION);

        this._setAnimFrame();
        this._runMainLoop();

        this.emit(this.EVENT_GAME_INITED);

        if (cb) cb();
    }
```

### _initEngine()

_initEngine() 只是开启渲染这么一个作用：

```js
    _initEngine () {
        if (this._rendererInitialized) {
            return;
        }

        this._initRenderer();

        if (!CC_EDITOR) {
            this._initEvents();
        }

        this.emit(this.EVENT_ENGINE_INITED);
    }
```

#### _initRenderer
这个方法干的事情可就多了。具体我也不是很会解释了。对图形不是很懂。

```js
 _initRenderer () {
        // Avoid setup to be called twice.
        if (this._rendererInitialized) return;

        let el = this.config.id,
            width, height,
            localCanvas, localContainer,
            isWeChatGame = cc.sys.platform === cc.sys.WECHAT_GAME,
            isQQPlay = cc.sys.platform === cc.sys.QQ_PLAY;

        if (isWeChatGame || CC_JSB) {
            this.container = localContainer = document.createElement("DIV");
            this.frame = localContainer.parentNode === document.body ? document.documentElement : localContainer.parentNode;
            if (cc.sys.browserType === cc.sys.BROWSER_TYPE_WECHAT_GAME_SUB) {
                localCanvas = wx.getSharedCanvas();
            }
            else if (CC_JSB) {
                localCanvas = window.__cccanvas;
            }
            else {
                localCanvas = canvas;
            }
            this.canvas = localCanvas;
        }
        else if (isQQPlay) {
            this.container = cc.container = document.createElement("DIV");
            this.frame = document.documentElement;
            this.canvas = localCanvas = canvas;
        }
        else {
            var element = (el instanceof HTMLElement) ? el : (document.querySelector(el) || document.querySelector('#' + el));

            if (element.tagName === "CANVAS") {
                width = element.width;
                height = element.height;

                //it is already a canvas, we wrap it around with a div
                this.canvas = localCanvas = element;
                this.container = localContainer = document.createElement("DIV");
                if (localCanvas.parentNode)
                    localCanvas.parentNode.insertBefore(localContainer, localCanvas);
            } else {
                //we must make a new canvas and place into this element
                if (element.tagName !== "DIV") {
                    cc.warnID(3819);
                }
                width = element.clientWidth;
                height = element.clientHeight;
                this.canvas = localCanvas = document.createElement("CANVAS");
                this.container = localContainer = document.createElement("DIV");
                element.appendChild(localContainer);
            }
            localContainer.setAttribute('id', 'Cocos2dGameContainer');
            localContainer.appendChild(localCanvas);
            this.frame = (localContainer.parentNode === document.body) ? document.documentElement : localContainer.parentNode;

            function addClass (element, name) {
                var hasClass = (' ' + element.className + ' ').indexOf(' ' + name + ' ') > -1;
                if (!hasClass) {
                    if (element.className) {
                        element.className += " ";
                    }
                    element.className += name;
                }
            }
            addClass(localCanvas, "gameCanvas");
            localCanvas.setAttribute("width", width || 480);
            localCanvas.setAttribute("height", height || 320);
            localCanvas.setAttribute("tabindex", 99);
        }

        this._determineRenderType();
        // WebGL context created successfully
        if (this.renderType === this.RENDER_TYPE_WEBGL) {
            var opts = {
                'stencil': true,
                // MSAA is causing serious performance dropdown on some browsers.
                'antialias': cc.macro.ENABLE_WEBGL_ANTIALIAS,
                'alpha': cc.macro.ENABLE_TRANSPARENT_CANVAS
            };
            if (isWeChatGame) {
                opts['preserveDrawingBuffer'] = true;
            }
            renderer.initWebGL(localCanvas, opts);
            this._renderContext = renderer.device._gl;
            
            // Enable dynamic atlas manager by default
            if (!cc.macro.CLEANUP_IMAGE_CACHE) {
                cc.dynamicAtlasManager.enabled = true;
            }
        }
        if (!this._renderContext) {
            this.renderType = this.RENDER_TYPE_CANVAS;
            // Could be ignored by module settings
            renderer.initCanvas(localCanvas);
            this._renderContext = renderer.device._ctx;
        }
        cc.renderer = renderer;

        this.canvas.oncontextmenu = function () {
            if (!cc._isContextMenuEnable) return false;
        };

        this._rendererInitialized = true;
    }
```
### _setAnimFrame

这玩意看起来是设置一下帧频率的？

```js
_setAnimFrame: function () {
        this._lastTime = new Date();
        var frameRate = game.config.frameRate;
        this._frameTime = 1000 / frameRate;

        if (CC_JSB) {
            jsb.setPreferredFramesPerSecond(frameRate);
            window.requestAnimFrame = window.requestAnimationFrame;
            window.cancelAnimFrame = window.cancelAnimationFrame;
        }
        else {
            if (frameRate !== 60 && frameRate !== 30) {
                window.requestAnimFrame = this._stTime;
                window.cancelAnimFrame = this._ctTime;
            }
            else {
                window.requestAnimFrame = window.requestAnimationFrame ||
                window.webkitRequestAnimationFrame ||
                window.mozRequestAnimationFrame ||
                window.oRequestAnimationFrame ||
                window.msRequestAnimationFrame ||
                this._stTime;
                window.cancelAnimFrame = window.cancelAnimationFrame ||
                window.cancelRequestAnimationFrame ||
                window.msCancelRequestAnimationFrame ||
                window.mozCancelRequestAnimationFrame ||
                window.oCancelRequestAnimationFrame ||
                window.webkitCancelRequestAnimationFrame ||
                window.msCancelAnimationFrame ||
                window.mozCancelAnimationFrame ||
                window.webkitCancelAnimationFrame ||
                window.oCancelAnimationFrame ||
                this._ctTime;
            }
        }
    },
    _stTime: function(callback){
        var currTime = new Date().getTime();
        var timeToCall = Math.max(0, game._frameTime - (currTime - game._lastTime));
        var id = window.setTimeout(function() { callback(); },
            timeToCall);
        game._lastTime = currTime + timeToCall;
        return id;
    },
    _ctTime: function(id){
        window.clearTimeout(id);
    },
```

### _runMainLoop

```js
    _runMainLoop: function () {
        var self = this, callback, config = self.config,
            director = cc.director,
            skip = true, frameRate = config.frameRate;

        debug.setDisplayStats(config.showFPS);

        callback = function () {
            if (!self._paused) {
                self._intervalId = window.requestAnimFrame(callback);
                if (frameRate === 30) {
                    if (skip = !skip) {
                        return;
                    }
                }
                director.mainLoop();
            }
        };

        self._intervalId = window.requestAnimFrame(callback);
        self._paused = false;
    }
```

等这些执行完毕后，才会执行什么开启的函数。

# main.js
我们生成项目的 **main.js** 会先加载配置文件，加载 *jsb_polyfill.js* 文件：

```js
    if (window.jsb) {
        require('src/settings.js');
        require('src/jsb_polyfill.js');
        boot();
        return;
    }
```

然后启动 boot 函数：

```js
    function boot () {
			// 这个settings 在 settings.js 里面获取
        var settings = window._CCSettings;
        window._CCSettings = undefined;

			//  在非debug的时候执行
        if ( !settings.debug ) {
            var uuids = settings.uuids;

            var rawAssets = settings.rawAssets;
            var assetTypes = settings.assetTypes;
            var realRawAssets = settings.rawAssets = {};
            for (var mount in rawAssets) {
                var entries = rawAssets[mount];
                var realEntries = realRawAssets[mount] = {};
                for (var id in entries) {
                    var entry = entries[id];
                    var type = entry[1];
                    // retrieve minified raw asset
                    if (typeof type === 'number') {
                        entry[1] = assetTypes[type];
                    }
                    // retrieve uuid
                    realEntries[uuids[id] || id] = entry;
                }
            }

            var scenes = settings.scenes;
            for (var i = 0; i < scenes.length; ++i) {
                var scene = scenes[i];
                if (typeof scene.uuid === 'number') {
                    scene.uuid = uuids[scene.uuid];
                }
            }

            var packedAssets = settings.packedAssets;
            for (var packId in packedAssets) {
                var packedIds = packedAssets[packId];
                for (var j = 0; j < packedIds.length; ++j) {
                    if (typeof packedIds[j] === 'number') {
                        packedIds[j] = uuids[packedIds[j]];
                    }
                }
            }
        }

        // init engine
        var canvas;

		// 在浏览器下，是利用了一个特定的画布
        if (cc.sys.isBrowser) {
            canvas = document.getElementById('GameCanvas');
        }
        

        if (false) {
            var ORIENTATIONS = {
                'portrait': 1,
                'landscape left': 2,
                'landscape right': 3
            };
            BK.Director.screenMode = ORIENTATIONS[settings.orientation];
            initAdapter();
        }

        function setLoadingDisplay () {
            // Loading splash scene
            var splash = document.getElementById('splash');
            var progressBar = splash.querySelector('.progress-bar span');
            cc.loader.onProgress = function (completedCount, totalCount, item) {
                var percent = 100 * completedCount / totalCount;
                if (progressBar) {
                    progressBar.style.width = percent.toFixed(2) + '%';
                }
            };
            splash.style.display = 'block';
            progressBar.style.width = '0%';

            cc.director.once(cc.Director.EVENT_AFTER_SCENE_LAUNCH, function () {
                splash.style.display = 'none';
            });
        }

        var onStart = function () {
            if (false) {
                BK.Script.loadlib();
            }

            cc.view.resizeWithBrowserSize(true);

            if (!false && !false) {
                // UC browser on many android devices have performance issue with retina display
                if (cc.sys.os !== cc.sys.OS_ANDROID || cc.sys.browserType !== cc.sys.BROWSER_TYPE_UC) {
                    cc.view.enableRetina(true);
                }
                if (cc.sys.isBrowser) {
                    setLoadingDisplay();
                }

                if (cc.sys.isMobile) {
                    if (settings.orientation === 'landscape') {
                        cc.view.setOrientation(cc.macro.ORIENTATION_LANDSCAPE);
                    }
                    else if (settings.orientation === 'portrait') {
                        cc.view.setOrientation(cc.macro.ORIENTATION_PORTRAIT);
                    }
                    cc.view.enableAutoFullScreen([
                        cc.sys.BROWSER_TYPE_BAIDU,
                        cc.sys.BROWSER_TYPE_WECHAT,
                        cc.sys.BROWSER_TYPE_MOBILE_QQ,
                        cc.sys.BROWSER_TYPE_MIUI,
                    ].indexOf(cc.sys.browserType) < 0);
                }

                // Limit downloading max concurrent task to 2,
                // more tasks simultaneously may cause performance draw back on some android system / brwosers.
                // You can adjust the number based on your own test result, you have to set it before any loading process to take effect.
                if (cc.sys.isBrowser && cc.sys.os === cc.sys.OS_ANDROID) {
                    cc.macro.DOWNLOAD_MAX_CONCURRENT = 2;
                }
            }

            // init assets
            cc.AssetLibrary.init({
                libraryPath: 'res/import',
                rawAssetsBase: 'res/raw-',
                rawAssets: settings.rawAssets,
                packedAssets: settings.packedAssets,
                md5AssetsMap: settings.md5AssetsMap
            });

            if (false) {
                cc.Pipeline.Downloader.PackDownloader._doPreload("WECHAT_SUBDOMAIN", settings.WECHAT_SUBDOMAIN_DATA);
            }

            var launchScene = settings.launchScene;

            // load scene
            cc.director.loadScene(launchScene, null,
                function () {
                    if (cc.sys.isBrowser) {
                        // show canvas
                        canvas.style.visibility = '';
                        var div = document.getElementById('GameDiv');
                        if (div) {
                            div.style.backgroundImage = '';
                        }
                    }
                    cc.loader.onProgress = null;
                    console.log('Success to load scene: ' + launchScene);
                }
            );
        };

        // jsList
        var jsList = settings.jsList;
			// 这个就用来加载工程内的 js 了
        if (!false) {
            var bundledScript = settings.debug ? 'src/project.dev.js' : 'src/project.js';
            if (jsList) {
                jsList = jsList.map(function (x) {
                    return 'src/' + x;
                });
                jsList.push(bundledScript);
            }
            else {
                jsList = [bundledScript];
            }
        }

        // anysdk scripts
        if (cc.sys.isNative && cc.sys.isMobile) {
//            jsList = jsList.concat(['src/anysdk/jsb_anysdk.js', 'src/anysdk/jsb_anysdk_constants.js']);
        }
        
		// 游戏运行选项
        var option = {
            //width: width,
            //height: height,
            id: 'GameCanvas',
            scenes: settings.scenes,
            debugMode: settings.debug ? cc.DebugMode.INFO : cc.DebugMode.ERROR,
            showFPS: (!false && !false) && settings.debug,
            frameRate: 60,
            jsList: jsList,
            groupList: settings.groupList,
            collisionMatrix: settings.collisionMatrix,
            renderMode: 0
        }

        cc.game.run(option, onStart);
    }
```