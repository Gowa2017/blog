---
title: Android利用Input类实现模拟输入
categories:
  - Android
date: 2019-12-02 15:11:18
updated: 2019-12-02 15:11:18
tags: 
  - Android
  - Android Input
---

在文章 {% post_link Android中Input类模拟输入的实现 Android中Input类模拟输入的实现 %} 我们查看了官方提供的 **com.android.commands.input.INPUT** 类来进行模拟输入的实现过程。但是有一个很猥琐的地方就是，这个 Input 类我们是无法访问的，其被定义为非 SDK 类，我们用反射的方法来获取这个类的实现的时候，会收到警告。

<!--more-->

```java
        Class inputS = Class.forName("com.android.commands.input.INPUT");

```
具体原因在这 [这里](https://developer.android.com/distribute/best-practices/develop/restrictions-non-sdk-interfaces)

因为这个类虽然存在于源代码内，但并未打包在 SDK 内，并不是打算提供开发者用的。

> Using Private APIs  Using reflection to access hidden/private Android APIs is not safe; it will often not work on devices from other vendors, and it may suddenly stop working (if the API is removed) or crash spectacularly (if the API behavior changes, since there are no guarantees for compatibility).  Issue id: PrivateApi

所以我们尝试把其源代码搬到我们自己的项目中来吧。

# InputManager.getInstance

我们的 Input 需要使用  InputManager 来进行模拟的输入，但是，我们却是无法获取其实例的，因为其 getInstance 方法，是 @hide 注解了的。所以我们现在却可以通过反射的时候来获取。

```java
    public Input() {
        try {
            Class inputManager = Class.forName("android.hardware.input.InputManager");
            Method getInstance = inputManager.getDeclaredMethod("getInstance");
            mInputManager = (InputManager) getInstance.invoke(null);
            inject = inputManager.getDeclaredMethod("injectInputEvent", InputEvent.class, Integer.TYPE);
            inject.setAccessible(true);
        } catch (Exception e){
            e.printStackTrace();
        }
    }
```
设置 InputManager 里面的所有方法都是 @hide 的。关于如何使用了 Hide 的 API，[这里有一些方法](https://juejin.im/post/5d4249fbf265da03d42f836b)。

其实我们这样做，还是出现了一些警告：

>43: Reflective access to getInstance, which is not part of the public SDK and therefore likely to change in future Android releases
>45: Reflective access to injectInputEvent, which is not part of the public SDK and therefore likely to change in future Android releases

意思就是通过反射调用非公开的 SDK 的内容，在将来的安卓版本中可能会变更，这个就暂时不用害怕了。

这样我们就成功的拿到了 InputManager 了。

# 权限

文本的输入不需要特权。而对于触摸的事件在 native 层会检测是否是本进程，或者是 root 权限，需要root 后才能进行。


# 测试代码

```java
        final Input input = new Input();
        tv.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                input.sendText(InputDevice.SOURCE_KEYBOARD,"hello");
                Executor executor = Executors.newSingleThreadExecutor();
                executor.execute(new Runnable() {
                    @Override
                    public void run() {
                        input.sendTap(InputDevice.SOURCE_TOUCHSCREEN,2,2);
                    }
                });
            }
        });

```

## 问题

结果执行的时候，输入文字没有问题，对于触摸的时候，提示：

```
Caused by: java.lang.SecurityException: Injecting to another application requires INJECT_EVENTS permission
```

及时我申请了ROOT 也会这样，为什么呢？因为签名问题。

我们在 **AndroidManifest.xml** 内加上权限需求：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    android:sharedUserId="android.uid.system"
    package="com.example.myapplication">
    <uses-permission android:name="android.permission.INJECT_EVENTS"/>
....
```

会提示我们：

```
Using system app permission  
Permissions with the protection level signature, privileged or signatureOrSystem are only granted to system apps. If an app is a regular non-system app, it will never be able to use these permissions.  
Issue id: ProtectedPermissions
```

只有系统 APP 才能拥有这些权限，意思只有系统 APP 才能获取这些敏感的权限。

# 系统签名文件

我们需要拿系统的签名信息，来生成签名文件。具体来说就是：

platform.x509.pem platform.pk8 来进行直接或者间接的签名。

## 1. 直接签名

这个需要用到 signapk.jar 这么一个 jar 包，这是官方源码内会提供的工具。在 `aosp/prebuilts/sdk/tools/lib` 内会有这个工具，不过这个工具的依赖于几个库。

我们直接执行命令

```sh
java -jar ./signapk.jar platform.x509.pem platform.pk8 app-debug.apk app-debug-signed.apk
```

会提示我们找不到库：

```sh
Caused by: java.lang.IllegalArgumentException: Failed to load any of the given libraries: [conscrypt_openjdk_jni-osx-x86_64, conscrypt_openjdk_jni]
```

这是因为我们没有相应的 conscrypt_openjdk_jni-osx-x86_64 动态库。我们可以从源码内复制过来：

- Linux  aosp/prebuilts/sdk/tools/linux/lib64/libconscrypt_openjdk_jni.so
- macOS  aosp/prebuilts/sdk/tools/darwin/lib64/libconscrypt_openjdk_jni.dylib

复制到目录下执行先前的命令就OK了。

专门写了个小脚本来干这个活：

```sh
#!/bin/bash
OUTDIR=`pwd`
if [ $# -lt 1 ]; then
    echo "USAGE: sign apkfile"
    exit 1
fi
OUTAPK=${1%%.apk}-signed.apk

if [ ! -f $1 ]; then
    echo "$1 文件不存在"
    exit 1
fi
cp $1 ~/bin/
cd ~/bin
java -jar ./signapk.jar platform.x509.pem platform.pk8 $1 $OUTAPK
cp $OUTAPK $OUTDIR
rm -f $1

```



## 2. 转换为 keystore 进行

我们上面的例子中，platform.x509.pem 是公钥，而 platform.pk8 是私钥。我们可以通过将其导入到 keystore 来进行。

其需要有几步来进行操作：



```sh
# Convert PK8 to PEM KEY 将 PK8 私钥转换成 转成 pem 格式
openssl pkcs8 -inform DER -nocrypt -in platform.pk8 -out key
# Bundle CERT and KEY 导出为 pks12 
openssl pkcs12 -export -in platform.x509.pem -inkey key -out p12 -password pass:android -name sys
# Import P12 in Keystore
keytool -importkeystore -deststorepass android -destkeystore syskeystore -srckeystore p12 -srcstoretype PKCS12 -srcstorepass android 
rm -f key p12
```

这样在我们的 syskeystore 中就会有 sys 条目了，密码是 android。如果我们不指定 -destkeystore 那么会默认存在 `~/.keystore` 中。

然后在我们的 Android 中使用这个 keysotre。



```groovy
    signingConfigs {
        release {
            storeFile file(System.getenv("HOME") + File.separator + ".keystore")
            storePassword 'android'
            keyAlias 'sys'
            keyPassword 'android'
        }
    }

```



OK 没问题能使用了。



# Input 源码

```java
public class Input {
    private static final String TAG = "Input";
    private static final String INVALID_ARGUMENTS = "Error: Invalid arguments for command: ";
    private InputManager mInputManager;
    private Method inject;

    private static final Map<String, Integer> SOURCES = new HashMap<String, Integer>() {{
        put("keyboard", InputDevice.SOURCE_KEYBOARD);
        put("dpad", InputDevice.SOURCE_DPAD);
        put("gamepad", InputDevice.SOURCE_GAMEPAD);
        put("touchscreen", InputDevice.SOURCE_TOUCHSCREEN);
        put("mouse", InputDevice.SOURCE_MOUSE);
        put("stylus", InputDevice.SOURCE_STYLUS);
        put("trackball", InputDevice.SOURCE_TRACKBALL);
        put("touchpad", InputDevice.SOURCE_TOUCHPAD);
        put("touchnavigation", InputDevice.SOURCE_TOUCH_NAVIGATION);
        put("joystick", InputDevice.SOURCE_JOYSTICK);
    }};

    public Input() {
        try {
            Class inputManager = Class.forName("android.hardware.input.InputManager");
            Method getInstance = inputManager.getDeclaredMethod("getInstance");
            mInputManager = (InputManager) getInstance.invoke(null);
            inject = inputManager.getDeclaredMethod("injectInputEvent", InputEvent.class, Integer.TYPE);
            inject.setAccessible(true);
        } catch (Exception e){
            e.printStackTrace();
        }
    }


    /**
     * Command-line entry point.
     *
     * @param args The command-line arguments
     */
    public static void main(String[] args) {
        (new Input()).run(args);
    }

    private void run(String[] args) {
        if (args.length < 1) {
            showUsage();
            return;
        }

        int index = 0;
        String command = args[index];
        int inputSource = InputDevice.SOURCE_UNKNOWN;
        if (SOURCES.containsKey(command)) {
            inputSource = SOURCES.get(command);
            index++;
            command = args[index];
        }
        final int length = args.length - index;

        try {
            if (command.equals("text")) {
                if (length == 2) {
                    inputSource = getSource(inputSource, InputDevice.SOURCE_KEYBOARD);
                    sendText(inputSource, args[index+1]);
                    return;
                }
            } else if (command.equals("keyevent")) {
                if (length >= 2) {
                    final boolean longpress = "--longpress".equals(args[index + 1]);
                    final int start = longpress ? index + 2 : index + 1;
                    inputSource = getSource(inputSource, InputDevice.SOURCE_KEYBOARD);
                    if (args.length > start) {
                        for (int i = start; i < args.length; i++) {
                            int keyCode = KeyEvent.keyCodeFromString(args[i]);
                            if (keyCode == KeyEvent.KEYCODE_UNKNOWN) {
                                keyCode = KeyEvent.keyCodeFromString("KEYCODE_" + args[i]);
                            }
                            sendKeyEvent(inputSource, keyCode, longpress);
                        }
                        return;
                    }
                }
            } else if (command.equals("tap")) {
                if (length == 3) {
                    inputSource = getSource(inputSource, InputDevice.SOURCE_TOUCHSCREEN);
                    sendTap(inputSource, Float.parseFloat(args[index+1]),
                            Float.parseFloat(args[index+2]));
                    return;
                }
            } else if (command.equals("swipe")) {
                int duration = -1;
                inputSource = getSource(inputSource, InputDevice.SOURCE_TOUCHSCREEN);
                switch (length) {
                    case 6:
                        duration = Integer.parseInt(args[index+5]);
                    case 5:
                        sendSwipe(inputSource,
                                Float.parseFloat(args[index+1]), Float.parseFloat(args[index+2]),
                                Float.parseFloat(args[index+3]), Float.parseFloat(args[index+4]),
                                duration);
                        return;
                }
            } else if (command.equals("draganddrop")) {
                int duration = -1;
                inputSource = getSource(inputSource, InputDevice.SOURCE_TOUCHSCREEN);
                switch (length) {
                    case 6:
                        duration = Integer.parseInt(args[index+5]);
                    case 5:
                        sendDragAndDrop(inputSource,
                                Float.parseFloat(args[index+1]), Float.parseFloat(args[index+2]),
                                Float.parseFloat(args[index+3]), Float.parseFloat(args[index+4]),
                                duration);
                        return;
                }
            } else if (command.equals("press")) {
                inputSource = getSource(inputSource, InputDevice.SOURCE_TRACKBALL);
                if (length == 1) {
                    sendTap(inputSource, 0.0f, 0.0f);
                    return;
                }
            } else if (command.equals("roll")) {
                inputSource = getSource(inputSource, InputDevice.SOURCE_TRACKBALL);
                if (length == 3) {
                    sendMove(inputSource, Float.parseFloat(args[index+1]),
                            Float.parseFloat(args[index+2]));
                    return;
                }
            } else {
                System.err.println("Error: Unknown command: " + command);
                showUsage();
                return;
            }
        } catch (NumberFormatException ex) {
        }
        System.err.println(INVALID_ARGUMENTS + command);
        showUsage();
    }

    /**
     * Convert the characters of string text into key event's and send to
     * device.
     *
     * @param text is a string of characters you want to input to the device.
     */
    public void sendText(int source, String text) {

        StringBuffer buff = new StringBuffer(text);

        boolean escapeFlag = false;
        for (int i=0; i<buff.length(); i++) {
            if (escapeFlag) {
                escapeFlag = false;
                if (buff.charAt(i) == 's') {
                    buff.setCharAt(i, ' ');
                    buff.deleteCharAt(--i);
                }
            }
            if (buff.charAt(i) == '%') {
                escapeFlag = true;
            }
        }

        char[] chars = buff.toString().toCharArray();

        KeyCharacterMap kcm = KeyCharacterMap.load(KeyCharacterMap.VIRTUAL_KEYBOARD);
        KeyEvent[] events = kcm.getEvents(chars);
        for(int i = 0; i < events.length; i++) {
            KeyEvent e = events[i];
            if (source != e.getSource()) {
                e.setSource(source);
            }
            injectKeyEvent(e);
        }
    }

    private void sendKeyEvent(int inputSource, int keyCode, boolean longpress) {
        long now = SystemClock.uptimeMillis();
        injectKeyEvent(new KeyEvent(now, now, KeyEvent.ACTION_DOWN, keyCode, 0, 0,
                KeyCharacterMap.VIRTUAL_KEYBOARD, 0, 0, inputSource));
        if (longpress) {
            injectKeyEvent(new KeyEvent(now, now, KeyEvent.ACTION_DOWN, keyCode, 1, 0,
                    KeyCharacterMap.VIRTUAL_KEYBOARD, 0, KeyEvent.FLAG_LONG_PRESS,
                    inputSource));
        }
        injectKeyEvent(new KeyEvent(now, now, KeyEvent.ACTION_UP, keyCode, 0, 0,
                KeyCharacterMap.VIRTUAL_KEYBOARD, 0, 0, inputSource));
    }

    public void sendTap(int inputSource, float x, float y) {
        long now = SystemClock.uptimeMillis();
        injectMotionEvent(inputSource, MotionEvent.ACTION_DOWN, now, x, y, 1.0f);
        injectMotionEvent(inputSource, MotionEvent.ACTION_UP, now, x, y, 0.0f);
    }

    private void sendSwipe(int inputSource, float x1, float y1, float x2, float y2, int duration) {
        if (duration < 0) {
            duration = 300;
        }
        long now = SystemClock.uptimeMillis();
        injectMotionEvent(inputSource, MotionEvent.ACTION_DOWN, now, x1, y1, 1.0f);
        long startTime = now;
        long endTime = startTime + duration;
        while (now < endTime) {
            long elapsedTime = now - startTime;
            float alpha = (float) elapsedTime / duration;
            injectMotionEvent(inputSource, MotionEvent.ACTION_MOVE, now, lerp(x1, x2, alpha),
                    lerp(y1, y2, alpha), 1.0f);
            now = SystemClock.uptimeMillis();
        }
        injectMotionEvent(inputSource, MotionEvent.ACTION_UP, now, x2, y2, 0.0f);
    }

    private void sendDragAndDrop(int inputSource, float x1, float y1, float x2, float y2,
                                 int dragDuration) {
        if (dragDuration < 0) {
            dragDuration = 300;
        }
        long now = SystemClock.uptimeMillis();
        injectMotionEvent(inputSource, MotionEvent.ACTION_DOWN, now, x1, y1, 1.0f);
        try {
            Thread.sleep(ViewConfiguration.getLongPressTimeout());
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        now = SystemClock.uptimeMillis();
        long startTime = now;
        long endTime = startTime + dragDuration;
        while (now < endTime) {
            long elapsedTime = now - startTime;
            float alpha = (float) elapsedTime / dragDuration;
            injectMotionEvent(inputSource, MotionEvent.ACTION_MOVE, now, lerp(x1, x2, alpha),
                    lerp(y1, y2, alpha), 1.0f);
            now = SystemClock.uptimeMillis();
        }
        injectMotionEvent(inputSource, MotionEvent.ACTION_UP, now, x2, y2, 0.0f);
    }

    /**
     * Sends a simple zero-pressure move event.
     *
     * @param inputSource the InputDevice.SOURCE_* sending the input event
     * @param dx change in x coordinate due to move
     * @param dy change in y coordinate due to move
     */
    private void sendMove(int inputSource, float dx, float dy) {
        long now = SystemClock.uptimeMillis();
        injectMotionEvent(inputSource, MotionEvent.ACTION_MOVE, now, dx, dy, 0.0f);
    }

    public void injectKeyEvent(KeyEvent event) {
        Log.i(TAG, "injectKeyEvent: " + event);
        try {
            inject.invoke(mInputManager, event, 0);
        }catch (Exception e){
            e.printStackTrace();
        }
    }

    private int getInputDeviceId(int inputSource) {
        final int DEFAULT_DEVICE_ID = 0;
        int[] devIds = InputDevice.getDeviceIds();
        for (int devId : devIds) {
            InputDevice inputDev = InputDevice.getDevice(devId);
            if (inputDev.supportsSource(inputSource)) {
                return devId;
            }
        }
        return DEFAULT_DEVICE_ID;
    }

    /**
     * Builds a MotionEvent and injects it into the event stream.
     *
     * @param inputSource the InputDevice.SOURCE_* sending the input event
     * @param action the MotionEvent.ACTION_* for the event
     * @param when the value of SystemClock.uptimeMillis() at which the event happened
     * @param x x coordinate of event
     * @param y y coordinate of event
     * @param pressure pressure of event
     */
    private void injectMotionEvent(int inputSource, int action, long when, float x, float y, float pressure) {
        final float DEFAULT_SIZE = 1.0f;
        final int DEFAULT_META_STATE = 0;
        final float DEFAULT_PRECISION_X = 1.0f;
        final float DEFAULT_PRECISION_Y = 1.0f;
        final int DEFAULT_EDGE_FLAGS = 0;
        MotionEvent event = MotionEvent.obtain(when, when, action, x, y, pressure, DEFAULT_SIZE,
                DEFAULT_META_STATE, DEFAULT_PRECISION_X, DEFAULT_PRECISION_Y,
                getInputDeviceId(inputSource), DEFAULT_EDGE_FLAGS);
        event.setSource(inputSource);
        Log.i(TAG, "injectMotionEvent: " + event);
        try {
            inject.invoke(mInputManager, event, 2);
        }catch (Exception e){
            e.printStackTrace();
        }
    }

    private static final float lerp(float a, float b, float alpha) {
        return (b - a) * alpha + a;
    }

    private static final int getSource(int inputSource, int defaultSource) {
        return inputSource == InputDevice.SOURCE_UNKNOWN ? defaultSource : inputSource;
    }

    private void showUsage() {
        System.err.println("Usage: input [<source>] <command> [<arg>...]");
        System.err.println();
        System.err.println("The sources are: ");
        for (String src : SOURCES.keySet()) {
            System.err.println("      " + src);
        }
        System.err.println();
        System.err.println("The commands and default sources are:");
        System.err.println("      text <string> (Default: touchscreen)");
        System.err.println("      keyevent [--longpress] <key code number or name> ..."
                + " (Default: keyboard)");
        System.err.println("      tap <x> <y> (Default: touchscreen)");
        System.err.println("      swipe <x1> <y1> <x2> <y2> [duration(ms)]"
                + " (Default: touchscreen)");
        System.err.println("      draganddrop <x1> <y1> <x2> <y2> [duration(ms)]"
                + " (Default: touchscreen)");
        System.err.println("      press (Default: trackball)");
        System.err.println("      roll <dx> <dy> (Default: trackball)");
    }
}

```

