---
title:  "支付宝 Android 开发整理"
date:   2016-12-21 11:34:23
categories: [Android]
tags: [Android]
---

`支付宝` Android 开发整理
### 官方文档
[支付宝1.0接口说明](https://doc.open.alipay.com/docs/doc.htm?spm=a219a.7629140.0.0.KXebEA&treeId=58&articleId=103584&docType=1)
[微信支付开发文档](https://pay.weixin.qq.com/wiki/doc/api/app/app.php?chapter=8_3)
### 开始
最近项目需要微信和支付宝支付，虽然以前开发过多次，但是每次还是会遇到各种问题，支付宝和微信支付一直在升级，文档结构也变得混乱。本文档整理了开发流程和遇到的一些问题及技巧。
支付宝和微信 app 支付流程是类似的，类似的下面步骤：
1.应用下单，应用服务端生成支付参数
2.客户端通过支付参数支付，完成付款
3.支付宝服务端回调应用服务端，完成购买流程
4.客户端刷新

不同的是：
1.支付宝支付参数使用`RSA`非对称签名验证，而微信使用`MD5`签名验证
2.支付宝下单流程不需要请求支付宝服务器，只需要应用服务器构造参数签名即可；微信支付需要请求微信支付服务器，获取`prepay_id`再构造支付参数。

本文只写支付宝支付内容，微信支付见下一篇

### 软件环境
服务端：`Nodejs` `Express框架`
客户端： `Android` `Rxjava` `Retrofit`
### 配置并获取用户id和key
申请支付宝商家账号，签约移动支付，生成用户`RSA`密钥([生成说明](https://doc.open.alipay.com/doc2/detail?treeId=58&articleId=103242&docType=1))，公钥配置到支付宝后台，申请后能拿到以下内容，后面需要：
* 合作者身份PID : 'partnerid'
* 商户id : 'seller_id'
* 支付宝公钥（alipay_rsa_public_key.pem）
* 商户私钥（my_rsa_private_key.pem）

### 服务端生成支付参数
支付宝参数生成不需要访问支付宝服务器，参数拼接签名即可，核心代码如下

``` javascript
var alpayParams = {
    partner : 'partnerid',
    seller_id : 'seller_id',
    out_trade_no : getOutTradeNo(),
    subject : subject,
    body : body,
    total_fee : total_fee,
    notify_url : 'notify_url',
    service : "mobile.securitypay.pay",
    payment_type : "1",
    _input_charset : "utf-8",
    it_b_pay : "30m",
    return_url : "m.alipay.com"
};

var keys = Object.keys(alpayParams).sort();
var prestr = [];
keys.forEach(function (e) {
  if (e !='sign') {
    prestr.push(e+'="'+alpayParams[e]+'"');
  }
});
prestr = prestr.join('&');

var signer=crypto.createSign("RSA-SHA1");
signer.update(prestr);
var sign=signer.sign(fs.readFileSync('./pem/my_private_key.pem').toString(),"base64");
var encodedSign = encodeURIComponent(sign);
var result = prestr+'&sign="'+encodedSign+'"&sign_type="RSA"';
console.log(result);
```
上面只是核心代码，测试项目见github https://github.com/weefree/express-alipay-wxpay
注意一下几点：
1.上面的签名密钥使用用户的私钥
2.getOutTradeNo()是生产商户订单号，需要唯一，长度小于64，可以使用md5(timestemp+random+userid)算法

### 客户端支付
为了保护用户私钥的安全性，客户端用服务端生成的支付参数调用支付，这样客户端结构就比较简单了，请求服务端拿到支付参数，调用支付就可以了，下面使用`Retrofit`网络框架，配合`Rxjava`做异步处理

获取支付参数
``` java
    //为了逻辑简单，此处没有传递商品和用户信息，真实环境此处可以带过去商品和用户信息，用于生成支付参数
    NetClient.getApi().getAlipayParams()
                .subscribeOn(Schedulers.newThread())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Subscriber<AlipayParams>() {
                    @Override
                    public void onCompleted() {
                    }

                    @Override
                    public void onError(Throwable e) {
                        Toast.makeText(MainActivity.this,"获取支付参数失败",Toast.LENGTH_SHORT).show();
                    }

                    @Override
                    public void onNext(AlipayParams alipayParams) {
                        if(alipayParams.code==0){
                            Alipay.getInstance().pay(MainActivity.this, alipayParams.data, MainActivity.this);
                        }else{
                            Toast.makeText(MainActivity.this,alipayParams.msg,Toast.LENGTH_SHORT).show();
                        }
                    }
                });
```
调用支付
``` java
 /**
     * 9000	订单支付成功
     * 8000	正在处理中，支付结果未知（有可能已经支付成功），请查询商户订单列表中订单的支付状态
     * 4000	订单支付失败
     * 5000	重复请求
     * 6001	用户中途取消
     * 6002	网络连接出错
     * 6004	支付结果未知（有可能已经支付成功），请查询商户订单列表中订单的支付状态
     * 其它	其它支付错误
     */
    public void pay(final Activity activity, final String orderInfo, final OnPayFinishListener listener){

        Observable.create(new Observable.OnSubscribe<AlipayResult>() {
            @Override
            public void call(Subscriber<? super AlipayResult> subscriber) {
                PayTask alipay = new PayTask(activity);
                AlipayResult payResult = new AlipayResult(alipay.pay(orderInfo, true));
                subscriber.onNext(payResult);
                subscriber.onCompleted();
            }
        }).subscribeOn(Schedulers.newThread())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Action1<AlipayResult>() {
                    @Override
                    public void call(AlipayResult result) {
                        if(listener==null)return;
                        if(TextUtils.equals(result.resultStatus,"9000")){
                            listener.onSuccess();
                        }else if(TextUtils.equals(result.resultStatus,"6001")){
                            listener.onCancel();
                        }else{
                            listener.onFail(result.resultStatus);
                        }
                    }
                });
    }
```
### 服务端验证并处理支付回调
支付成功后，支付宝会通过同步接口和异步接口返回支付结果，由于同步结果还是需要再服务端验证（客户端不能有用户私钥），此处为了支付流程简单，支付结果完全依赖服务端回调。支付宝回调地址即上文的`notify_url`,验证过程核心代码如下
``` java
tils.verifyAlipayCallback = function (body,alipayPublicKey) {
    try {
        var keys = Object.keys(body).sort();
        var prestr = [];
        keys.forEach(function (e) {
            if (e !='sign' && e !='sign_type') {
                prestr.push(e+'='+body[e]);
            }
        });
        prestr = prestr.join('&');
        prestr = new Buffer(prestr);
        var sign = new Buffer(body['sign'],'base64');
        return crypto.createVerify('RSA-SHA1').update(prestr).verify(alipayPublicKey, body['sign'],'base64');
    }catch (error){
        console.error(error);
        return false;
    }

};
```
上面传递的`alipayPublicKey`即支付宝公钥，具体实现见开源代码
验证成功后，服务端可以根据`out_trade_no`找到这条交易，完成发货等后续处理。
### Demo下载
[Android客户端](https://github.com/weefree/android-alipay-wxpay)
[Nodejs服务端](https://github.com/weefree/express-alipay-wxpay)
