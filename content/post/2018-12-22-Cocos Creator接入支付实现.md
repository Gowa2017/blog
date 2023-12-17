---
title: Cocos Creator接入支付实现
categories:
  - Cocos Creator
date: 2018-12-22 21:25:18
updated: 2018-12-22 21:25:18
tags: 
  - Cocos Creator
---
最近搞了个小游戏，需要接入第三方支付。本来在 H5 上是可以用 uri 直接拉起 app 的，但是原生的不行，所以需要手动在代码内接入一下。

# 思考

根据一般支付的流程，不外乎在下单后从服务器拿到一个支付相关的 ID， 把这个 ID 传递给手机上的支付 APP，其在支付成功后回调我们定义的函数就OK了。

但这有几个问题，Cocos Creator 是使用 JS 开发的，其可以通过反射的形式调用原生 JAVA 的代码 [JAVA 原生反射机制](https://docs.cocos.com/creator/manual/zh/advanced-topics/java-reflection.html)。理想的情况是，支付成功后，　 APP 调用我们的原生代码，原生代码再调用 我们定义的 JS 函数，来进行通知。

但我仔细思考了一下就会有个很明显的问题，如果在支付场景，出现意外情况，比如说断线，进行了操作后重至一个新场景的话，比如我们要拉起通知框会不会拉不起来？

最终我选择了一个比较简单的方法。 JS -> 原生代码 -> 支付APP -> 原生代码改变支付状态 -> JS 轮询获取结果。

# 原生代码设计

我们设计一个给 JS 调用的方法，设计一个是否完成支付的代码，及最终支付结果状态的代码。

```java
    private static int payResult; // 表示支付结果，0 失败， 1成功
    private static boolean isPay; // 表示是否开始支付
    
   // 给JS调用已知道是否支付成功
    public static boolean isPaySuccess(){
        Log.d(TAG, "isPaySuccess: " + isPay);
        return isPay;
    }

// JS 调用查看是否完成支付
    public static int getPayResult(){
        Log.d(TAG, "getPayResult: " + payResult);
        return payResult;
    }
    
    // 给JS调用，拉起支付APP
    public static void doMolPay(String uri){
        payResult  = 0;
        isPay = false;
        PayParamsV0 paramsV0 = PayParamsV0.builder()
                .uri(uri)
                .build();
        Log.d(TAG, "doMolPay: " + paramsV0.toString());
        MolPay.builder()
                .setParams(paramsV0)
                .setContext(getContext())
                .setOnPayListener(new OnPayListener() {
                    @Override
                    public void onStart() {
                        Log.d(TAG, "onStart: ");
                    }

                    @Override
                    public void onSuccess() {
                        payResult = 1;
                        Log.d(TAG, "onSuccess: ");

                    }

                    @Override
                    public void onError(int i) {
                        payResult = 0;
                        Log.d(TAG, "onError: " + i);
                    }

                    @Override
                    public void onComplete() {
                        Log.d(TAG, "onComplete: ");
                        isPay = true;
                    }
                })
                .start();
    }    
    
}
```

上面的代码逻辑很简单，支付成功 设置  isPay = 1， 支付错误 设置 isPay = 0， 完成的话，表示 isPay 支付逻辑完成。

# JS 内的代码实现：

```js
var s = this;
// 开始调用原生代码支付
jsb.reflection.callStaticMethod("org/cocos2dx/javascript/AppActivity", "doMolPay", "(Ljava/lang/String;)V", uri);
// 开始定时任务，0.5秒查询一下是否支付完成
s.schedule(function(){
 var isPay = jsb.reflection.callStaticMethod("org/cocos2dx/javascript/AppActivity", "isPaySuccess","()Z");
 // 支付完成获取结果
 if(isPay){
     s.unscheduleAllCallbacks();
     var payResult = jsb.reflection.callStaticMethod("org/cocos2dx/javascript/AppActivity","getPayResult","()I");
     if (payResult === 1){
     			// 这些代码自定义的一些代码，来显示提示框
             s.com_MessageBox.active = !0,
             s.bg_Black.active = !0,
             s.com_MessageBox.getChildByName("lb_Tips").getComponent("cc.Label").string = "支付成功"
             s.netWork.socket.emit("getCoin");
     } else {
             s.com_MessageBox.active = !0,
             s.bg_Black.active = !0,
             s.com_MessageBox.getChildByName("lb_Tips").getComponent("cc.Label").string = "支付失败"
     }
 }
},0.5)
                                        


```