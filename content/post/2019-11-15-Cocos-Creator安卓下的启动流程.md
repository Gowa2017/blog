---
title: Cocos-Creator安卓下的启动流程
categories:
  - Cocos Creator
date: 2019-11-15 01:18:15
updated: 2019-11-29 09:35:15
tags: 
  - Cocos Creator
---
事实上， cocos-2dx 和 cocos creator 启动过程大同小异，不过使用的绑定不一致罢了。

<!--more-->

# AppActivity

对于我们的安卓应用而言，是可以找到其入口的 Activity 的，而 CCC 的就是 AppActivity。

其是 Cocos2dxActivity 的一个子类。

比较猥琐的是，没有使用  xml 进行布局，而是使用了动态布局，构造一个 FrameLayout 然后再挂上 GLSurfaceView 上去，就显示出来了。

我们来看过程。

## AppActivity.onCreate

这个代码比较简单，实际上就相当于简单的掉了一下父类的 onCreate 方法就完事：

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

## Cocos2dxActivity.onCreate

大部分工作都是在这个父类里面完成的。

```java
    @Override
    protected void onCreate(final Bundle savedInstanceState) {
        Log.d(TAG, "Cocos2dxActivity onCreate: " + this + ", savedInstanceState: " + savedInstanceState);
        super.onCreate(savedInstanceState);

        // Utils 工具只是调用一下 setSystemUiVisibility 这个方法，来进行全屏
        Utils.setActivity(this);

        // Workaround in https://stackoverflow.com/questions/16283079/re-launch-of-activity-on-home-button-but-only-the-first-time/16447508
        if (!isTaskRoot()) {
            // Android launched another instance of the root activity into an existing task
            //  so just quietly finish and go away, dropping the user back into the activity
            //  at the top of the stack (ie: the last state of this task)
            finish();
            Log.w(TAG, "[Workaround] Ignore the activity started from icon!");
            return;
        }

        Utils.hideVirtualButton();

        Cocos2dxHelper.registerBatteryLevelReceiver(this);
        
        // 加载 so 库
        onLoadNativeLibraries();

        sContext = this;
        // 接收显示对话框的 handler
        this.mHandler = new Cocos2dxHandler(this);

        // 辅助类，什么功能，遇到再说
        Cocos2dxHelper.init(this);
        
        // 画布渲染
        CanvasRenderingContext2DImpl.init(this);

        // 从 native 层内获取 GL 上下文属性信息
        this.mGLContextAttrs = getGLContextAttrs();
        
        // 初始化 UI
        this.init();

        if (mVideoHelper == null) {
            mVideoHelper = new Cocos2dxVideoHelper(this, mFrameLayout);
        }

        if(mWebViewHelper == null){
            mWebViewHelper = new Cocos2dxWebViewHelper(mFrameLayout);
        }

        Window window = this.getWindow();
        window.setSoftInputMode(WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE);
        this.setVolumeControlStream(AudioManager.STREAM_MUSIC);
    }
	
```

## Cocos2dxActivity.init()

这个方法会把 Activity 的内容显示出来，设置其布局。
代码中，就是构造一个 FrameLayout ，添加一个 Cocos2dxGLSurfaceView，为 Cocos2dxGLSurfaceView 设置 Render，再添加一个 Cocos2dxEditBox 就完事。

```java
    public void init() {
        ViewGroup.LayoutParams frameLayoutParams =
                new ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT,
                                           ViewGroup.LayoutParams.MATCH_PARENT);
        mFrameLayout = new RelativeLayout(this);
        mFrameLayout.setLayoutParams(frameLayoutParams);

        Cocos2dxRenderer renderer = this.addSurfaceView();
        this.addDebugInfo(renderer);

        // Should create EditBox after adding SurfaceView, or EditBox will be hidden by SurfaceView.
        mEditBox = new Cocos2dxEditBox(this, mFrameLayout);

        // Set frame layout as the content view
        setContentView(mFrameLayout);
    }
```

设置 Render 。

```java
    private Cocos2dxRenderer addSurfaceView() {
        this.mGLSurfaceView = this.onCreateView();
        this.mGLSurfaceView.setPreserveEGLContextOnPause(true);
        // Should set to transparent, or it will hide EditText
        // https://stackoverflow.com/questions/2978290/androids-edittext-is-hidden-when-the-virtual-keyboard-is-shown-and-a-surfacevie
        mGLSurfaceView.setBackgroundColor(Color.TRANSPARENT);
        // Switch to supported OpenGL (ARGB888) mode on emulator
        if (isAndroidEmulator())
            this.mGLSurfaceView.setEGLConfigChooser(8, 8, 8, 8, 16, 0);

        Cocos2dxRenderer renderer = new Cocos2dxRenderer();
        this.mGLSurfaceView.setCocos2dxRenderer(renderer);

        mFrameLayout.addView(this.mGLSurfaceView);

        return renderer;
    }
```

```java
    public Cocos2dxGLSurfaceView onCreateView() {
        Cocos2dxGLSurfaceView glSurfaceView = new Cocos2dxGLSurfaceView(this);
        //this line is need on some device if we specify an alpha bits
        if(this.mGLContextAttrs[3] > 0) glSurfaceView.getHolder().setFormat(PixelFormat.TRANSLUCENT);

        Cocos2dxEGLConfigChooser chooser = new Cocos2dxEGLConfigChooser(this.mGLContextAttrs);
        glSurfaceView.setEGLConfigChooser(chooser);

        return glSurfaceView;
    }

```

## Cocos2dxRenderer

这个类实现了三个回调，在 Cocos2dxGLSurfaceView 创建好后，有改变，或绘制帧的时候会被回调。

```java
// Called when the surface is created or recreated.
public void onSurfaceCreated(javax.microedition.khronos.opengles.GL10 gl, javax.microedition.khronos.egl.EGLConfig config);

// Called when the surface changed size.
public void onSurfaceChanged(javax.microedition.khronos.opengles.GL10 gl, int width, int height);
public void onDrawFrame(javax.microedition.khronos.opengles.GL10 gl);

```

### onSurfaceCreated

当 Cocos2dxGLSurfaceView 建立后，Cocos2dxRenderer 收到回调，就会在 native 进行初始化。
```java
    // ===========================================================
    // Methods for/from SuperClass/Interfaces
    // ===========================================================

    @Override
    public void onSurfaceCreated(final GL10 GL10, final EGLConfig EGLConfig) {
        mNativeInitCompleted = false;
        Cocos2dxRenderer.nativeInit(this.mScreenWidth, this.mScreenHeight, mDefaultResourcePath);
        mOldNanoTime = System.nanoTime();
        this.mLastTickInNanoSeconds = System.nanoTime();
        mNativeInitCompleted = true;
        if (mGameEngineInitializedListener != null) {
            Cocos2dxHelper.getActivity().runOnUiThread(new Runnable() {
                @Override
                public void run() {
                    mGameEngineInitializedListener.onGameEngineInitialized();
                }
            });
        }
    }
```
### nativeInit

在 native 层，就会 调用 cocos_android_app_init 来建立一个 AppDelegate，这个在平台间都是通用的。可以说在 JNI 中对 AppDelegate 进行了一个调用（这玩意就是 native 游戏引擎的一个代理类）

```cpp
	/*****************************************************
	 * Cocos2dxRenderer native functions implementation.
	 *****************************************************/

    JNIEXPORT void JNICALL JNI_RENDER(nativeInit)(JNIEnv*  env, jobject thiz, jint w, jint h, jstring jDefaultResourcePath)
    {
        g_width = w;
        g_height = h;
        
        g_app = cocos_android_app_init(env, w, h);

        g_isGameFinished = false;
        ccInvalidateStateCache();
        // 获取默认资源路径，但是这个值其实传过来的是一个空字符串。 
        std::string defaultResourcePath = JniHelper::jstring2string(jDefaultResourcePath);
        LOGD("nativeInit: %d, %d, %s", w, h, defaultResourcePath.c_str());
        
        // 为空不会设置资源路径的欸
        if (!defaultResourcePath.empty())
            FileUtils::getInstance()->setDefaultResourceRootPath(defaultResourcePath);

        // 好了，现在开始获取脚本引擎了。
        se::ScriptEngine* se = se::ScriptEngine::getInstance();
        // 添加一个注册回调函数
        se->addRegisterCallback(setCanvasCallback);
        
        // 注册事件分发器
        EventDispatcher::init();
        
        // 启动 native 应用了
        g_app->start();
        g_isStarted = true;
    }

```

```cpp
Application *cocos_android_app_init(JNIEnv *env, int width, int height)
{
    LOGD("cocos_android_app_init");
    auto app = new AppDelegate(width, height);
    return app;
}

```
```java
    @Override
    public void onSurfaceChanged(final GL10 GL10, final int width, final int height) {
        Cocos2dxRenderer.nativeOnSurfaceChanged(width, height);
    }
    @Override
    public void onDrawFrame(final GL10 gl) {
        if (mNeedToPause)
            return;

        if (mNeedShowFPS) {
            /////////////////////////////////////////////////////////////////////
            //IDEA: show FPS in Android Text control rather than outputing log.
            ++mFrameCount;
            long nowFpsTime = System.nanoTime();
            long fpsTimeInterval = nowFpsTime - mOldNanoTime;
            if (fpsTimeInterval > 1000000000L) {
                double frameRate = 1000000000.0 * mFrameCount / fpsTimeInterval;
                Cocos2dxHelper.OnGameInfoUpdatedListener listener = Cocos2dxHelper.getOnGameInfoUpdatedListener();
                if (listener != null) {
                    listener.onFPSUpdated((float) frameRate);
                }
                mFrameCount = 0;
                mOldNanoTime = System.nanoTime();
            }
            /////////////////////////////////////////////////////////////////////
        }
        /*
         * No need to use algorithm in default(60 FPS) situation,
         * since onDrawFrame() was called by system 60 times per second by default.
         */
        if (sAnimationInterval <= INTERVAL_60_FPS) {
            Cocos2dxRenderer.nativeRender();
        } else {
            final long now = System.nanoTime();
            final long interval = now - this.mLastTickInNanoSeconds;

            if (interval < Cocos2dxRenderer.sAnimationInterval) {
                try {
                    Thread.sleep((Cocos2dxRenderer.sAnimationInterval - interval) / Cocos2dxRenderer.NANOSECONDSPERMICROSECOND);
                } catch (final Exception e) {
                }
            }
            /*
             * Render time MUST be counted in, or the FPS will slower than appointed.
            */
            this.mLastTickInNanoSeconds = System.nanoTime();
            Cocos2dxRenderer.nativeRender();
        }
    }

```

# AppDelegate.applicationDidFinishLaunching

在 Application 的 start 方法会，会启动应用代理类。

```cpp
bool AppDelegate::applicationDidFinishLaunching()
{
    se::ScriptEngine *se = se::ScriptEngine::getInstance();
    
    jsb_set_xxtea_key("017a798f-8b93-40");
    // 解密 js 文件
    jsb_init_file_operation_delegate();
    
#if defined(COCOS2D_DEBUG) && (COCOS2D_DEBUG > 0)
    // Enable debugger here
    jsb_enable_debugger("0.0.0.0", 6086, false);
#endif
    
    se->setExceptionCallback([](const char *location, const char *message, const char *stack) {
        // Send exception information to server like Tencent Bugly.
    });
    
    // 注册所有的 js 绑定到 ScriptEngine 内
    jsb_register_all_modules();
    
    // 启动脚本引擎
    se->start();
    
    se::AutoHandleScope hs;
    // 运行脚本
    jsb_run_script("jsb-adapter/jsb-builtin.js");
    // 运行入口脚本
    jsb_run_script("main.js");
    
    se->addAfterCleanupHook([]() {
        JSBClassType::destroy();
    });
    
    return true;
}

```
# ScriptEngine::start()

这里面会将所有进行已经添加了的回调函数进行调用。

```cpp
    bool ScriptEngine::start()
    {
        if (!init())
            return false;

        se::AutoHandleScope hs;

        // debugger
        if (isDebuggerEnabled())
        {
#if SE_ENABLE_INSPECTOR
            // V8 inspector stuff, most code are taken from NodeJS.
            _isolateData = node::CreateIsolateData(_isolate, uv_default_loop());
            _env = node::CreateEnvironment(_isolateData, _context.Get(_isolate), 0, nullptr, 0, nullptr);

            node::DebugOptions options;
            options.set_wait_for_connect(_isWaitForConnect);// the program will be hung up until debug attach if _isWaitForConnect = true
            options.set_inspector_enabled(true);
            options.set_port((int)_debuggerServerPort);
            options.set_host_name(_debuggerServerAddr.c_str());
            bool ok = _env->inspector_agent()->Start(_platform, "", options);
            assert(ok);
#endif
        }
        //
        bool ok = false;
        _startTime = std::chrono::steady_clock::now();

        for (auto cb : _registerCallbackArray)
        {
            ok = cb(_globalObj);
            assert(ok);
            if (!ok)
                break;
        }

        // After ScriptEngine is started, _registerCallbackArray isn't needed. Therefore, clear it here.
        _registerCallbackArray.clear();

        return ok;
    }
```

这样，我们使用的 js 环境就能调用 所有 cocos 引擎里面的导出函数了。



# 总结

其总体的启动过程就如下了：

1. Cocos2dxActivity.onCreate 会先加载 so 库，然后用代码的形式构造一个 RelativeLayout。
2. 构造 Cocos2dxGLSurfaceView，然后将其添加的 RelativeLayout。
3. Cocos2dxGLSurfaceView 需要一个 Cocos2dxRenderer 之后都会由它往 Cocos2dxGLSurfaceView 上绘制内容。
4. Cocos2dxGLSurfaceView 建立在屏幕上完成后就会调用 Cocos2dxRenderer 的 回调函数，进行 nativeInit 处理。在这个过程就会建立 AppDelegate，初始化引擎，然后执行脚本逻辑。
