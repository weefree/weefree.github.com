---
title:  "微信支付 Android 开发整理"
date:   2016-12-21 11:34:23
categories: [Android]
tags: [Android]
---

### 官方文档
[微信支付开发文档](https://pay.weixin.qq.com/wiki/doc/api/app/app.php?chapter=8_3)

### 开始
微信支付流程和支付宝类似，只是请求参数为`MD5`签名验证

### 软件环境
服务端：`Nodejs` `Express框架`
客户端： `Android` `Rxjava` `Retrofit`
### 配置并获取用户id和key
申请开通微信支付账号，
登录[商户平台](https://pay.weixin.qq.com/index.php/core/home/login?return_url=%2F)可以找到以下内容，后面需要：
* 微信支付Appid : 'wxappid'
* 商户id : 'mch_id'
* 参数签名密钥 : `apikey`

### 服务端生成支付参数
微信支付参数需要先请求微信服务器获取`prepay_id`,
流程是：构建参数并签名(无`prepay_id`)->请求微信服务器获取prepay_id->再构建参数并签名(有`prepay_id`),返回给客户端

构建参数并签名(无`prepay_id`)

``` javascript
 var params = {};
  params.appid = WxAppId;
  params.mch_id = MchId;
  params.nonce_str =  getOutTradeNo();
  params.body =  "测试支付-0.01元";
  params.out_trade_no = getOutTradeNo();
  params.total_fee =  "1";
  params.spbill_create_ip =  "127.0.0.1";
  params.notify_url =  notify_url;
  params.trade_type =  "APP";

  params.sign = signWxParams(params,apikey);

  var xml = objectToXml(params);
  
```

``` javascript
function signWxParams(params,apiKey) {
    var keys = Object.keys(params).sort();
    var prestr = [];
    keys.forEach(function (e) {
        if (e !='sign') {
            prestr.push(e+'='+params[e]);
        }
    });
    prestr = prestr.join('&');
    prestr = prestr+"&key="+apiKey;
    return crypto.createHash('md5',"utf8").update(prestr).digest('hex').toUpperCase();
};
```

``` javascript
function objectToXml(object) {
    var builder = new xml2js.Builder({rootName:"xml",cdata:true});
    return builder.buildObject(object);
}
```

请求微信服务器获取prepay_id

``` javascript
 request({
    url: "https://api.mch.weixin.qq.com/pay/unifiedorder",
    method: 'POST',
    body:xml
  }, function(err, response, body){
  
    xmlToObject(body,function (error, result) {
    
      //再构建参数并签名(有`prepay_id`),返回给客户端
      var prepayResult = {};
      prepayResult.appid			= result['appid'];
      prepayResult.partnerid		= result['mch_id'];
      prepayResult.prepayid		= result['prepay_id'];
      prepayResult.noncestr		= result['nonce_str'];
      prepayResult.timestamp		= new Date().getTime()+"";
      prepayResult.package	= "Sign=WXPay";

      prepayResult.sign = utils.signWxParams(prepayResult);
      prepayResult.packagevalue = "Sign=WXPay";
      res.end(JSON.stringify(prepayResult));
    });
  });
```


上面只是核心代码，测试项目见开源项目
注意一下几点：
1.Nodejs Md5字符串加密时不会自动处理编码，需要手动指定utf8 : crypto.createHash('md5',"utf8"),如果不指定，中文签名会有问题
2.上面的签名密钥使用从商户平台获取的apikey
3.getOutTradeNo()是生产商户订单号，需要唯一，长度小于64，可以使用md5(timestemp+random+userid)算法

### 客户端支付
为了保护用户私钥的安全性，客户端用服务端生成的支付参数调用支付，这样客户端结构就比较简单了，请求服务端拿到支付参数，调用支付就可以了，下面使用`Retrofit`网络框架，配合`Rxjava`做异步处理

获取支付参数

``` java
    //为了逻辑简单，此处没有传递商品和用户信息，真实环境此处可以带过去商品和用户信息，用于生成支付参数
     NetClient.getApi().getWxpayParams()
                .subscribeOn(Schedulers.newThread())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Subscriber<RespBody.WXPayParamsResp>() {
                    @Override
                    public void onCompleted() {
                    }

                    @Override
                    public void onError(Throwable e) {
                        ProgressDialogFragment.hide(getFragmentManager());
                        Toast.makeText(MainActivity.this,"获取支付参数失败",Toast.LENGTH_SHORT).show();
                    }

                    @Override
                    public void onNext(RespBody.WXPayParamsResp wxPayParams) {
                        ProgressDialogFragment.hide(getFragmentManager());
                        if(wxPayParams==null||wxPayParams.data==null){
                            Toast.makeText(MainActivity.this,"参数错误",Toast.LENGTH_SHORT).show();
                            return;
                        }
                        if(!wxPayParams.data.isParamOk()){
                            Toast.makeText(MainActivity.this,"参数错误或不完整",Toast.LENGTH_SHORT).show();
                            return;
                        }
                        if(wxPayParams.code == 0){
                            WXPay.getInstance().pay(wxPayParams.data, MainActivity.this);
                        }else{
                            Toast.makeText(MainActivity.this,wxPayParams.msg,Toast.LENGTH_SHORT).show();
                        }

                    }
                });
```

调用支付

``` java
        PayReq req = new PayReq();
        req.appId			= wxpayParams.appid;
        req.partnerId		= wxpayParams.partnerid;
        req.prepayId		= wxpayParams.prepayid;
        req.nonceStr		= wxpayParams.noncestr;
        req.timeStamp		= wxpayParams.timestamp;
        req.packageValue	= wxpayParams.packagevalue;
        req.sign			= wxpayParams.sign;

        api.sendReq(req);
```

支付后的同步结果会在 .wxapi.WXPayEntryActivity 返回，注意此处 wxapi 这个包需要放在项目包名命名的包下，为了降低 WXPayEntryActivity 和项目包名的依赖，可以用 activity-alias 简单处理一下：

``` xml
       <activity
            android:name="com.box.pay.WXPayCallbackActivity"
            android:exported="true"
            android:launchMode="singleTop"
            android:theme="@android:style/Theme.NoDisplay"  />
        <activity-alias
            android:name="[应用包名].wxapi.WXPayEntryActivity"
            android:exported="true"
            android:targetActivity="com.box.pay.WXPayCallbackActivity" />
```

### 服务端验证并处理支付回调
支付成功后，微信支付会通过异步接口返回支付结果。回调地址即上文的`notify_url`,验证过程核心代码如下

``` javascript
      if(signWxParams(result,apiKey)==result['sign']){
          //todo 验证成功，执行后续操作

          var result = {};
          result.return_code = "SUCCESS";
          result.return_msg = "OK";
          res.end(objectToXml(result));
      }else {
        //验证失败
        res.end("fail");
      }

```

``` javascript
  function signWxParams(params,apiKey) {
    var keys = Object.keys(params).sort();
    var prestr = [];
    keys.forEach(function (e) {
        if (e !='sign') {
            prestr.push(e+'='+params[e]);
        }
    });
    prestr = prestr.join('&');
    prestr = prestr+"&key="+apiKey;
    return crypto.createHash('md5',"utf8").update(prestr).digest('hex').toUpperCase();
};
```

验证成功后，服务端可以根据`out_trade_no`找到这条交易，完成发货等后续处理。

### Demo下载
[Android客户端](https://github.com/weefree/android-alipay-wxpay)
[Nodejs服务端](https://github.com/weefree/express-alipay-wxpay)
