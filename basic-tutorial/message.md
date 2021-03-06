# 模板消息和统一服务消息

## 功能概述

本文对应实现例子为[tcb-demo-basic](https://github.com/TencentCloudBase/tcb-demo-basic)中的 `模板消息` 功能，类似一个访客预约系统，当用户预约成功后，下发一条消息。这个消息可以是单纯的小程序模板消息，如果小程序与公众号进行了关联，那么也可以使用统一服务消息，按需发送小程序的模板消息或是公众号的模板消息。

## 体验功能

<p align="center">
    <img src="https://main.qcloudimg.com/raw/f36ab01f3fd9e0f899c879f71d11fdff.png" width="500px">
    <p align="center">扫码体验</p>
</p>

## 体验 DEMO

本章的案例代码，是在 [tcb-demo-basic](https://github.com/TencentCloudBase/tcb-demo-basic)。

1.用小程序登录微信公众平台，选择 `访客登记通知` 消息模板。添加到个人模板库，复制保存后的模板ID，供后面使用。

<p align="center">
    <img style="background:white;padding:20px;" src="../assets/message1.png" width="800x">
</p>

2.在小程序公众号平台， `设置` -〉 `开发设置` 里，找到 `AppSecret` ，复制保存供后面使用。

<p align="center">
    <img style="background:white;padding:20px;" src="../assets/appsecret.png" width="800px">
</p>

3.在公众号平台登录小程序绑定的公众号，在 `首页` -〉`功能` -〉 `添加功能` 插件里里选择模板消息，先添加，需要公众平台审核通过后才能开始使用。审核通过后，选择`来访申请提醒`模板，搜索结果列表中编号为OPENTM410586240的那一个，添加到个人模板库后，复制保存其ID，供后面使用。

<p align="center">
    <img style="background:white;padding:20px;" src="../assets/message2.png" width="800px">
</p>

4.在公众号管理平台，`开发` -〉`基本配置`里找到开发者 `appId` ，复制并保存，供后面使用。

<p align="center">
    <img style="background:white;padding:20px;" src="../assets/appid.png" width="800px">
</p>

5.在 `cloud/functions` 目录下，找到 `send-message` 云函数。在其根目录下新建 `config` 目录，复制 `example.js` 到 `config` 下，并改名为 `index.js` 。其内容如下所示，将上面收集保存的信息分别填入对应字段

```js
module.exports = {
  weappSecret: '', // 小程序 secret id
  weappTemplateId: '', // 小程序模板消息 id
  mpAppId: '', // 公众号 appid
  mpTemplateId: '' // 公众号模板消息 id
}

```
然后上传并安装云函数依赖。

6. 打开云控制台，选择 `数据库` -〉 `添加集合`，新建一个名为 `reserves` 的集合。

做完以上步骤，编译预览就可以体验本demo了。


## 源码介绍

### 自定义消息模板
发送模板消息，首先需要在模板库里选择对应的模板，并按需选择所需字段。如果模板库中已有模板不能满足业务需求，可以自己定义后提交审核，审核通过后即可选择使用。
微信公众平台模板消息选择界面

#### 使用wx-js-utils发送模板消息

官方的发送模板消息API如下
```js
POST https://api.weixin.qq.com/cgi-bin/message/wxopen/template/send?access_token=ACCESS_TOKEN
```
具体的参数可以移步[官网](https://developers.weixin.qq.com/miniprogram/dev/api/open-api/template-message/sendTemplateMessage.html)查看

这里我们介绍下wx-js-utils封装方法的使用，非常简单
```js
  const wxMiniUser = new WXMINIUser({ appId, secret });
  const access_token = await wxMiniUser.getAccessToken();

  const wxMiniMessage = new WXMINIMessage({ openId, formId, templateId });

  return wxMiniMessage.sendMessage({
    access_token,
    data,
    page
  });
```
我们只需要关注我们发布的消息内容即可。

### 统一服务消息

统一服务消息这个就比较厉害了！

统一服务消息既可以发送小程序的模板消息，如有公众号与之绑定，则它也可以发送公众号模板消息，此时只需要提供公众号的一些配置即可，详细的参数可以查看demo代码。通过些消息发送的公众号推送消息，点击可以跳转到小程序页面，为小程序的访问提供了便利的入口。

用统一服务消息API发送小程序模板消息，具体的参数就是上面模板消息例子中的那些，只不过格式有点变化，这里就不多说，主要说下公众号的。

#### 选择模板

同样的，发送公众号模板消息也需要在公众号管理平台先选择。具体如何选择前文已有介绍，这里就不再说。


#### 官网接口
官方的发送统一服务消息API如下
```js
POST https://api.weixin.qq.com/cgi-bin/message/wxopen/template/uniform_send?access_token=ACCESS_TOKEN
```
具体的参数可以移步[官网](https://developers.weixin.qq.com/miniprogram/dev/api/open-api/uniform-message/sendUniformMessage.html)查看

**特别需要注意的**是，利用小程序发送绑定公众号的消息模板，需要当前小程序已上线，不然接口始终会报如下错误
```js
{"errcode":40165,"errmsg":"invalid weapp pagepath hint: [uQN5tA0640shc2]"}
```
网上有博客说把pagepath改成path可以发送成功，但这样一来，收到消息后没有地址回到小程序了，也是没有意义的。

#### 使用 wx-js-utils 接口发送统一服务消息
主要代码如下：
```js
  const {
    WXMINIUser,
    WXUniformMessage,
  } = require('wx-js-utils');
  const {
    touser,
    appId,
    secret,
    weapp_template_msg,
    mp_template_msg
  } = event

  const wxMiniUser = new WXMINIUser({ appId, secret });
  const access_token = await wxMiniUser.getAccessToken();

  const wxUniformMessage = new WXUniformMessage();

  return wxUniformMessage.sendMessage({
    access_token,
    touser,
    weapp_template_msg,
    mp_template_msg
  });

```

这里需要注意 weapp_template_msg 和 mp_template_msg 的关系，两者都有，发送 weapp_template_msg 所代表的小程序模板消息，weapp_template_msg 不存在而 mp_template_msg 存在时，则发送公众号模板消息。
