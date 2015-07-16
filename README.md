# 如何接入微信公众平台

## 接入微信公众平台

> 1.填写服务器配置
> 2.验证服务器是否可用

### 1.填写服务器配置

在微信公众平台管理后台点击“开发者中心”，填写有效的服务器验证`URL`，`Token`和`EncodingAESKey`（自定义或者随机生成），包括选择“消息加解密模式”。当你确定“提交”之后，微信服务器将发送GET请求到该URL链接...

### 2.验证服务器是否可用

在上面发送GET请求之前，其实应该在服务器端部署好验证的代码。根据文档的 **加密/校验流程**要求，部署代码如下（nodejs版）：

    var crypto = require('crypto');
    var token = 'value of the token';

    var checkSignature = function(signature, timestamp, nonce) {
      var tmpArr = [token, timestamp, nonce];
      tmpArr.sort();                           // 1.将token、timestamp、nonce三个参数进行字典序排序
      var tmpStr = tmpArr.join('');            // 2.将三个参数字符串拼接成一个字符串tmpStr    
      var shasum = crypto.createHash('sha1');  
      shasum.update(tmpStr);              
      var shaResult = shasum.digest('hex');    // 3.字符串tmpStr进行sha1加密
      if(shaResult === signature){             // 4.加密后的字符串与signature对比，确定来源于微信
          return true;
      }
      return false;
    }
    
    module.exports = {
        weixin : function (req, res) {
        var signature = req.param('signature');
        var timestamp = req.param('timestamp');
        var nonce = req.param('nonce');
        var echostr = req.param('echostr');
        
        if (checkSignature(signature, timestamp, nonce)) {
            res.send(echostr);   // 确认来源是微信，并把echostr返回给微信服务器。
        } else {
            res.json(200, { code : -1, msg : 'You aren't wechat server !'});
        }
        
        ...
    }
    
例如，请求的URL是`/xxx/weixin`，然后调用的`controller`是`weixin`，这样就可以对微信服务器发送GET请求的进行响应了，换句话说，微信服务器可以验证我们的服务器是否可用了。

接入微信公众平台开发，建议看一下微信文档的[接入指南](http://mp.weixin.qq.com/wiki/17/2d4265491f12608cd170a95559800f2d.html)。

## 网页授权获取用户基本信息

> 1.设置 **回调域名**。
> 2.设置网页授权的方式。

###  1.设置 **回调域名**

在“开发者中心”页面中的“接口权限表” ——> “网页授权获取用户基本信息”设置 **回调域名**，例如：www.kaolalicai.cn。

**注： 这是一个小细节（坑），所以记得检查或者配置，别被坑了。**

### 2.设置网页授权的方式

摘抄文档：

> 1、以snsapi_base为scope发起的网页授权，是用来获取进入页面的用户的openid的，并且是静默授权并自动跳转到回调页的。用户感知的就是直接进入了回调页（往往是业务页面）
> 2、以snsapi_userinfo为scope发起的网页授权，是用来获取用户的基本信息的。但这种授权需要用户手动同意，并且由于用户同意过，所以无须关注，就可在授权后获取该用户的基本信息。

因为我们需要获取用户的基本信息，因此使用`snsapi_userinfo`进行网页授权。

## 获得用户授权

> 1.这里使用两个node的模块：[wechat-api](https://github.com/node-webot/wechat-api) 和 [wechat-oauth](https://github.com/node-webot/wechat-oauth)。
> 2.在服务器端完成授权。

举个代码栗子：
    
    var config = {
      token: "test",
      appid: 'test',
      encodingAESKey: 'test',
      secret:'test'
    };
    var client = new OAuth(config.appid, config.secret);
    var api = new API(config.appid, config.secret);

    module.exports = {
        weixin : function (req, res) {
            ...
        },
        
        wechatAuth : function (req, res) {    // 主要做授权的处理
            var redirectUrl = 'this is a redirectUrl';  // 这个就是通过认证的回调地址（可以理解为目标地址）
            var url = client.getAuthorizeURL(redirectUrl, 'ok', 'snsapi_userinfo');  // 还记得网页授权的两种方式吗，参数“snsapi_userinfo”表示获取用户信息，需要用户手动确认。
            res.redirect(url);
        },
    }
    
从上面的栗子说明，要先通过授权，用户同意之后才能进入目标页面的，所以说在进入目标页面之前，先请求`/xxx/wechatAuth`进行授权认证，相当于`/xxx/wechatAuth`才是真正的入口。

想知道原本的授权过程是怎样的，戳这里[网页授权获取用户基本信息](http://mp.weixin.qq.com/wiki/17/c0f37d5704f0b64713d5d2c37b468d75.html)。

## 调用JS-SDK

有了前面的工作，现在来调用JS-SDK就顺利咯。

细看微信官方Demo : [http://demo.open.weixin.qq.com/jssdk](http://demo.open.weixin.qq.com/jssdk)，只需要五步就可以调用JS-SDK了。

> 1. 进入“公众号设置”的“功能设置”里填写“JS接口安全域名”
> 2. 在需要调用JS接口的页面引入如下JS文件(http://res.wx.qq.com/open/js/jweixin-1.0.0.js)
> 3. 通过config接口注入权限验证配置
> 4. 通过ready接口处理成功验证
> 5. 通过error接口处理失败验证

这里主要讲 **#3**和 **#4**

### 3.通过config接口注入权限验证配置

摘抄微信文档：
> 所有需要使用JS-SDK的页面必须先注入配置信息，否则将无法调用......

    wx.config({
        debug: true, // 开启调试模式,调用的所有api的返回值会在客户端alert出来，若要查看传入的参数，可以在pc端打开，参数信息会通过log打出，仅在pc端时才会打印。
        appId: '', // 必填，公众号的唯一标识
        timestamp: , // 必填，生成签名的时间戳
        nonceStr: '', // 必填，生成签名的随机串
        signature: '',// 必填，签名，见附录1
        jsApiList: [] // 必填，需要使用的JS接口列表，所有JS接口列表见附录2
    });
    
来个例子：

    module.exports = {
        weixin : function (req, res) {
            ...
        },
        
        wechatAuth : function (req, res) {    // 主要做授权的处理
            var redirectUrl = 'this is a redirectUrl';  // 这个就是通过认证的回调地址（可以理解为目标地址）
            var url = client.getAuthorizeURL(redirectUrl, 'ok', 'snsapi_userinfo');  // 还记得网页授权的两种方式吗，参数“snsapi_userinfo”表示获取用户信息，需要用户手动确认。
            res.redirect(url);
        },
        
        getAuthData : function (req, res) {
            var code = req.param('code');        //获取授权处理后的code值
            var url = req.param('url');          //获取动态加密的URL
            client.getAccessToken(code, function (err, result) {    // 这里或以下的接口，详情请看上面的两个模块（链接）
              var openid = result.data.openid;
              //根据openid 获取用户授权信息
              client.getUser(openid, function (err, result) {
                var userInfo = result;
                var param = {
                  debug: false,
                  jsApiList: ['onMenuShareTimeline', 'onMenuShareAppMessage'],
                  url: url
                };
                //设置js api 参数
                api.getJsConfig(param,function(err,result){
                  userInfo['jsconfig'] = result;   // 这里的jsconfig，其实就是一个"权限验证配置"对象
                  res.json(200,{code : 0, data:userInfo});
                })
            });
        });
    }
    
前端代码只要做：

    wx.config(data.jsconfig); 即可注入权限验证配置
    
### 4.通过ready接口处理成功验证

    wx.ready(function(){

        // config信息验证后会执行ready方法，所有接口调用都必须在config接口获得结果之后，config是一个客户端的异步操作，所以如果需要在页面加载时就调用相关接口，则须把相关接口放在ready函数中调用来确保正确执行。对于用户触发时才调用的接口，则可以直接调用，不需要放在ready函数中。
    });
    
上一步注入权限验证配置成功之后，就可以自定义接口了，例子如下：

    wx.ready(function () {
          var shareData = {
            title: "title", // 分享标题
            link: "the share link", // 分享链接
            desc: "desciption", // 分享描述
            imgUrl: location.protocol +'//' + location.host+ '/images/invite/packet10yuan.jpg' // 分享图标
          };
          wx.checkJsApi({
            jsApiList:['onMenuShareTimeline','onMenuShareAppMessage'],
            success:function(res){
                // alert(JSON.stringify(res));
            }
          });
          //分享到朋友圈
          wx.onMenuShareTimeline({
            title: shareData.title, // 分享标题
            link: shareData.link, // 分享链接
            imgUrl: shareData.imgUrl, // 分享图标
            success: function () {
                //  alert('ok');
                 // 用户确认分享后执行的回调函数
            },
            cancel: function () {
                //   alert('cancel');
                 // 用户取消分享后执行的回调函数
            }
          });
          //分享给朋友
          wx.onMenuShareAppMessage({
            title: shareData.title, // 分享标题
            desc: shareData.desc, // 分享描述
            link: shareData.link, // 分享链接
            imgUrl: shareData.imgUrl, // 分享图标
            type: 'link', // 分享类型,music、video或link，不填默认为link
            dataUrl: '', // 如果type是music或video，则要提供数据链接，默认为空
            success: function () {
                // alert('ok');
                // 用户确认分享后执行的回调函数
            },
            cancel: function () {
                //  alert('cancel');
                // 用户取消分享后执行的回调函数
            }
          });
        });

这里当使用分享功能时，分享的内容就是自定义的内容了。

## 总结几个注意(坑)点

 1. 公众帐号必须升级为认证的 **服务号**才能使用全部的微信js api（详细看“ **开发者中心**”——>“ **接口权限表**”）

 2. 需要在微信公众帐号后台进行: **服务器配置**

 3. 需要在微信公众帐号后台——>**公众号设置**——>**公众号设置(tab)** : **设置JS接口安全域名**

 4. 需要 **网页授权获取用户基本信息**时，在“ **开发者中心**”——>“ **接口权限表**” ——> “ **网页授权获取用户基本信息**”设置 **回调域名**，例如：www.kaolalicai.cn。

## 关于测试方面的建议

> 微信api本地调试比较麻烦，需要本地调试时，要做一些准备工作。
> 1.设置一个本地测试域名，例如： http://test_domain/。
> 2.该域名指向内网测试IP
> 3.还可以配置内网路由器，设置虚拟服务器，指向到个人电脑IP