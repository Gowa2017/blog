---
title: Cocos Creator热更新例子的研究
categories:
  - Cocos Creator
date: 2019-01-05 22:18:43
updated: 2019-01-05 22:18:43
tags: 
  - Cocos Creator
---
来源于官方的例子，主要是看一下是怎么实现的。官方文档有一定的描述，但很科幻，也没有集成在插件内。官方的文档描述地址：[热更新范例教程](https://docs.cocos.com/creator/manual/zh/advanced-topics/hot-update.html)

<!--more-->

# 基本原理

对于不同版本的文件级差异，AssetsManager 中使用 Manifest 文件来进行版本比对。本地和远端的 Manifest 文件分别标示了本地和远端的当前版本包含的文件列表和文件版本，这样就可以通过比对每个文件的版本来确定需要更新的文件列表。

Manifest 文件中包含以下几个重要信息：

1. 远程资源包的根路径
2. 远程 Manifest 文件地址
3. 远程 Version 文件地址（非必需）
4. 主版本号
5. 文件列表：以文件路径来索引，包含文件版本信息，一般推荐用文件的 md5 校验码来作为版本号
6. 搜索路径列表

其中 Version 文件内容是 Manifest 文件内容的一部分，不包含文件列表。由于 Manifest 文件可能比较大，每次检查更新的时候都完整下载的话可能影响体验，所以开发者可以额外提供一个非常小的 Version 文件。AssetsManager 会首先检查 Version 文件提供的主版本号来判断是否需要继续下载 Manifest 文件并更新。

# 例子实现

`scripts/module/HotUpdate.js` 是实现逻辑的组件，其挂在了 Canvas 上。组件拥有的属性如下：

```js
    properties: {
        panel: UpdatePanel, // 进度条
        manifestUrl: {      // 版本库文件
            type: cc.Asset,
            default: null
        },
        updateUI: cc.Node, //更新窗口
        _updating: false,
        _canRetry: false,
        _storagePath: ''
    }
```

界面基本如下：
![](../res/cocos-creator-hot-update.png)

# AssetsManager

事实上热更新就是因为 AssetsManager 支持而实现的了。关于这个资源管理器的接口，是绑定到 jsb 的，文档上我没看到，但是官方有提示源代码有能看到：[AssetsManagerEx.h](https://github.com/cocos-creator/cocos2d-x-lite/blob/develop/extensions/assets-manager/AssetsManagerEx.h)

# 初始化
在 onLoad 函数里面完成：

在这个初始化中，先得出一个存储路径 *this_storagePath*，一个版本比较函数 *this_versionCompareHandle*，然后就初始化了 AssetsManager，初始化的时候，将 manifest 路径设置为空。 接着，为 AssetsManager 设置了一个回调 `setVerifyCallback`。

```js
    onLoad: function () {
        // Hot update is only available in Native build
        if (!cc.sys.isNative) {
            return;
        }
        this._storagePath = ((jsb.fileUtils ? jsb.fileUtils.getWritablePath() : '/') + 'blackjack-remote-asset');
        cc.log('Storage path for remote asset : ' + this._storagePath);

        // Setup your own version compare handler, versionA and B is versions in string
        // if the return value greater than 0, versionA is greater than B,
        // if the return value equals 0, versionA equals to B,
        // if the return value smaller than 0, versionA is smaller than B.
        this.versionCompareHandle = function (versionA, versionB) {
            cc.log("JS Custom Version Compare: version A is " + versionA + ', version B is ' + versionB);
            var vA = versionA.split('.');
            var vB = versionB.split('.');
            for (var i = 0; i < vA.length; ++i) {
                var a = parseInt(vA[i]);
                var b = parseInt(vB[i] || 0);
                if (a === b) {
                    continue;
                }
                else {
                    return a - b;
                }
            }
            if (vB.length > vA.length) {
                return -1;
            }
            else {
                return 0;
            }
        };

        // Init with empty manifest url for testing custom manifest
        this._am = new jsb.AssetsManager('', this._storagePath, this.versionCompareHandle);

        var panel = this.panel;
        // Setup the verification callback, but we don't have md5 check function yet, so only print some message
        // Return true if the verification passed, otherwise return false
        this._am.setVerifyCallback(function (path, asset) {
            // When asset is compressed, we don't need to check its md5, because zip file have been deleted.
            var compressed = asset.compressed;
            // Retrieve the correct md5 value.
            var expectedMD5 = asset.md5;
            // asset.path is relative path and path is absolute.
            var relativePath = asset.path;
            // The size of asset file, but this value could be absent.
            var size = asset.size;
            if (compressed) {
                panel.info.string = "Verification passed : " + relativePath;
                return true;
            }
            else {
                panel.info.string = "Verification passed : " + relativePath + ' (' + expectedMD5 + ')';
                return true;
            }
        });

        this.panel.info.string = 'Hot update is ready, please check or directly update.';

        if (cc.sys.os === cc.sys.OS_ANDROID) {
            // Some Android device may slow down the download process when concurrent tasks is too much.
            // The value may not be accurate, please do more test and find what's most suitable for your game.
            this._am.setMaxConcurrentTask(2);
            this.panel.info.string = "Max concurrent tasks count have been limited to 2";
        }
        
        this.panel.fileProgress.progress = 0;
        this.panel.byteProgress.progress = 0;
    },

```

当我们点击检查更新的时候，就会调用对应的事件响应函数：

函数中，先是载入了本地 manifest 文件，接着，就开始检查更新，更新完毕，会回调 我们的 `checkCb()` 回调。


```js
    checkUpdate: function () {
        if (this._updating) {
            this.panel.info.string = 'Checking or updating ...';
            return;
        }
        if (this._am.getState() === jsb.AssetsManager.State.UNINITED) {
            // Resolve md5 url
            var url = this.manifestUrl.nativeUrl;
            if (cc.loader.md5Pipe) {
                url = cc.loader.md5Pipe.transformURL(url);
            }
            this._am.loadLocalManifest(url);
        }
        if (!this._am.getLocalManifest() || !this._am.getLocalManifest().isLoaded()) {
            this.panel.info.string = 'Failed to load local manifest ...';
            return;
        }
        this._am.setEventCallback(this.checkCb.bind(this));

        this._am.checkUpdate();
        this._updating = true;
    },

```

OK，在回调中就可以根据检测的结果，来确定是否需要进行更新了：

```js
    checkCb: function (event) {
        cc.log('Code: ' + event.getEventCode());
        switch (event.getEventCode())
        {
            case jsb.EventAssetsManager.ERROR_NO_LOCAL_MANIFEST:
                this.panel.info.string = "No local manifest file found, hot update skipped.";
                break;
            case jsb.EventAssetsManager.ERROR_DOWNLOAD_MANIFEST:
            case jsb.EventAssetsManager.ERROR_PARSE_MANIFEST:
                this.panel.info.string = "Fail to download manifest file, hot update skipped.";
                break;
            case jsb.EventAssetsManager.ALREADY_UP_TO_DATE:
                this.panel.info.string = "Already up to date with the latest remote version.";
                break;
            case jsb.EventAssetsManager.NEW_VERSION_FOUND:
                this.panel.info.string = 'New version found, please try to update.';
                this.panel.checkBtn.active = false;
                this.panel.fileProgress.progress = 0;
                this.panel.byteProgress.progress = 0;
                break;
            default:
                return;
        }
        
        this._am.setEventCallback(null);
        this._checkListener = null;
        this._updating = false;
    },

```

接着我们就可以点击更新按钮进行更新了，具体的代码，其实都在里面了。


```js
    hotUpdate: function () {
        if (this._am && !this._updating) {
            this._am.setEventCallback(this.updateCb.bind(this));

            if (this._am.getState() === jsb.AssetsManager.State.UNINITED) {
                // Resolve md5 url
                var url = this.manifestUrl.nativeUrl;
                if (cc.loader.md5Pipe) {
                    url = cc.loader.md5Pipe.transformURL(url);
                }
                this._am.loadLocalManifest(url);
            }

            this._failCount = 0;
            this._am.update();
            this.panel.updateBtn.active = false;
            this._updating = true;
        }
    },

```
