# 如何接入微信公众平台

在微信6.0.2版本以前，h5页面是可以自定分享内容的，之后的版本就收回去了。而在很多活动开发中，我们经常需要用到自定义分享内容。比如设置分享的图片，分享的链接，分享后的回调通知等，这些功能对产品运营和营销有很大的帮助。今天主要和大家一起分享如何接入微信公众平台。

## [接入指南](http://mp.weixin.qq.com/wiki/17/2d4265491f12608cd170a95559800f2d.html)

1. 填写服务器配置(开发者中心->配置项->服务器配置)
  + ![配置项](https://mmbiz.qlogo.cn/mmbiz/E7ia3F4UicMx8k28bBHO9FMYsNxicX7BGB5eB2Wxt8dnEoicEDibVx4BzDFbiaevB6EL1yQXGBFbLZz65BiatdLzOEhjQ/0?wx_fmt=png)
  + ![服务器配置](https://mmbiz.qlogo.cn/mmbiz/E7ia3F4UicMx8k28bBHO9FMYsNxicX7BGB5pASnqHb9CWuY7C1tm8hSDFyDR0SUls9W4qwib3cw0deNmmeE5qSzGXA/0?wx_fmt=png)
  + 当配置好相关信息的时候，记得开启服务器配置（开启之后，微信后台自动消息回复就会失效，如果需要智能回复用户信息，就只能由开发自己开发实现了，比如关注回复内容，与用户互动发送内容等，开发者可以实现一个机器人来自动回复），下图为开启服务配置截图
  + ![开启服务配置](https://mmbiz.qlogo.cn/mmbiz/E7ia3F4UicMx8k28bBHO9FMYsNxicX7BGB5bibf4picuNXLaT2oIebO8Oc7YaZrYhqErWicCtlDCSZBCTy4pZ4rXZRRQ/0?wx_fmt=png)

2. 验证服务器地址的有效性(验证开发者服务器是否可用，同时校验消息来自微信服务器，确保通讯是安全的)
```js
var token = 'test_weixin';
function checkSignature(signature,timestamp,nonce,token){
  var crypto = require('crypto');
  var tmpArr = [token,timestamp,nonce];
  tmpArr.sort();
  var tmpStr = tmpArr.join('');
  var shasum = crypto.createHash('sha1');
  shasum.update(tmpStr);
  var shaResult = shasum.digest('hex');
  if(shaResult === signature){
  	return true;
  }
  return false;
}
//响应微信服务器(假定我们填写服务器配置时填写的http://test.navy.com/weixin，在nodejs中处理这条路由的代码如下)
weixin:function(req,res){
  var signature = req.param('signature');
  var timestamp = req.param('timestamp');
  var nonce = req.param('nonce');
  var echostr = req.param('echostr');
  if(checkSignature(signature,timestamp,nonce,token)){
    res.send(echostr);
  }else{
    res.send('invalid request');
  }
}
```

接入指南结束

**依据接口文档实现业务逻辑(下面我们以接入微信JS-SDK为Demo)**

## 接入微信JS-SDK

### 在使用微信JS-SDK前，必须先生成权限验证的签名：signature，具体步骤如下

#### 1. 获取access_token
> http请求方式: GET
> https://api.weixin.qq.com/cgi-bin/token?grant_type=client_credential&appid=APPID&secret=APPSECRET

#### 2. 通过access_token获取jsapi_ticket（jsapi_ticket是公众号用于调用微信JS接口的临时票据。所以我们需要调用微信JS接口时，必须要有这个）
#### 3. 生成JS-SDK权限验证的签名：signature
算法见：[生成JS-SDK权限验证的签名算法](http://mp.weixin.qq.com/wiki/7/aaa137b55fb2e0456bf8dd9148dd613f.html)
```js
function makeSignature(jsapi_ticket,noncestr,timestamp,url){
  var crypto = require('crypto');
	var tmpStr = "jsapi_ticket="+jsapi_ticket+"&noncestr="+noncestr+"&timestamp="+timestamp+"&url="+url;
	var shasum = crypto.createHash('sha1');
	shasum.update(tmpStr);
	var shaResult = shasum.digest('hex');
	console.log('signature:'+shaResult);
	return shaResult;
}
```
### 接入步骤

#### 1. 绑定域名
  + 设置JS接口安全域名后，公众号开发者可在该域名下调用微信开放的JS接口。（域名必须是一级或一级以上域名，须通过ICP备案的验证）
  + ![绑定域名](https://mmbiz.qlogo.cn/mmbiz/E7ia3F4UicMx8k28bBHO9FMYsNxicX7BGB5CbbiaFryicPZPCd6YMJPQEtx8z8XwVWQqCnT83YtQgONp2eEHkgs5BxA/0?wx_fmt=png)
  + ![填写域名](https://mmbiz.qlogo.cn/mmbiz/E7ia3F4UicMx8k28bBHO9FMYsNxicX7BGB5Q0Po2ntZODU2f7Vd1AaZFiaRSrxsvkyFfqZ3FOeGu8zrB4ncVfJfPMQ/0?wx_fmt=png)
 
#### 2. 引入JS文件(在需要调用JS接口的页面引入JS文件:[JS文件地址](http://res.wx.qq.com/open/js/jweixin-1.0.0.js)，可下载到自己服务器，然后引入)

#### 3. 通过config接口注入权限验证配置
```js
wx.config({
  debug: true, // 开启调试模式,调用的所有api的返回值会在客户端alert出来，若要查看传入的参数，可以在pc端打开，参数信息会通过log打出，仅在pc端时才会打印。
  appId: '', // 必填，公众号的唯一标识
  timestamp: , // 必填，生成签名的时间戳
  nonceStr: '', // 必填，生成签名的随机串
  signature: '',// 必填，签名，见[附录1](http://mp.weixin.qq.com/wiki/7/aaa137b55fb2e0456bf8dd9148dd613f.html#.E6.AD.A5.E9.AA.A4.E4.BA.8C.EF.BC.9A.E5.BC.95.E5.85.A5JS.E6.96.87.E4.BB.B6)
  jsApiList: [] // 必填，需要使用的JS接口列表，所有JS接口列表见[附录2](http://mp.weixin.qq.com/wiki/7/aaa137b55fb2e0456bf8dd9148dd613f.html#.E6.AD.A5.E9.AA.A4.E4.BA.8C.EF.BC.9A.E5.BC.95.E5.85.A5JS.E6.96.87.E4.BB.B6)
});
```
注：config中的参数也可以通过后端返回，然后在前端设置。

#### 4. 通过ready接口处理成功验证
```js
wx.ready(function(){
  // config信息验证后会执行ready方法，所有接口调用都必须在config接口获得结果之后，config是一个客户端的异步操作，所以如果需要在页面加载时就调用相关接口，则须把相关接口放在ready函数中调用来确保正确执行。对于用户触发时才调用的接口，则可以直接调用，不需要放在ready函数中。
});
```

#### 5. 通过error接口处理失败验证
```js
wx.error(function(res){
  // config信息验证失败会执行error函数，如签名过期导致验证失败，具体错误信息可以打开config的debug模式查看，也可以在返回的res参数中查看，对于SPA可以在这里更新签名。

});
```
上面五步做完之后，下面我们使用其中的API：**网页授权获取用户基本信息**为例说明如何使用JS-SDK的API

### 网页授权获取用户基本信息
**注意在使用该接口时需要设置接口的调用域名，见下图**
1. ![设置接口的调用域名](https://mmbiz.qlogo.cn/mmbiz/E7ia3F4UicMx8k28bBHO9FMYsNxicX7BGB506AhK19vW7a3wnpbPAcKEtHiawt5QCCHfVVM3XiaHYcHJSqWa4ibX4gng/0?wx_fmt=png)
2. ![设置接口的调用域名](https://mmbiz.qlogo.cn/mmbiz/E7ia3F4UicMx8k28bBHO9FMYsNxicX7BGB5Ep9MShkOTrzicFA9epFkNzPd7CrdfONNd2jicT76hzmoLBBp1JfSOgtg/0?wx_fmt=png)

上面设置完毕后，我们就可以开始网页授权获取用户基本信息了。

1. 用户同意授权，获取code（在我们的页面引导用户到微信授权页）
> [授权链接](https://open.weixin.qq.com/connect/oauth2/authorize?appid=APPID&redirect_uri=REDIRECT_URI&response_type=code&scope=SCOPE&state=STATE#wechat_redirect)
> 参数说明
> ![参数说明](https://mmbiz.qlogo.cn/mmbiz/E7ia3F4UicMx8k28bBHO9FMYsNxicX7BGB5IXy2XAMdc1CPHfYBenRW5gLtkdN63mEXe8JIkTibbD02HWRclv4caYA/0?wx_fmt=png)
> 授权效果图
> ![授权效果图](https://mmbiz.qlogo.cn/mmbiz/E7ia3F4UicMx8k28bBHO9FMYsNxicX7BGB5amo0qBCo7aLEpEMjvs7heiazibtRqta3db9MPuF47Bkf9sMOvKkr9HoA/0?wx_fmt=png)
> 当用户同意授权后我们就获取了微信返回的code。

2. 通过code换取网页授权access_token
> 获取code后，请求以下链接获取access_token：https://api.weixin.qq.com/sns/oauth2/access_token?appid=APPID&secret=SECRET&code=CODE&grant_type=authorization_code

3. 拉取用户信息(需scope为 snsapi_userinfo)
> http：GET（请使用https协议）
> https://api.weixin.qq.com/sns/userinfo?access_token=ACCESS_TOKEN&openid=OPENID&lang=zh_CN

关于如何接入微信开放平台的讲解就到这里，这里有一个实战例子供大家参考：[微信开放平台接入Demo](https://github.com/navyxie/wechat-develop-code)

**同时，在不接入微信API的情况下设置自定义分享内容（图片、链接、标题），有需要可以看这里[https://github.com/navyxie/weixin_js](https://github.com/navyxie/weixin_js)**
