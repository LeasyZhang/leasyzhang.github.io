---
layout:     post
title:      "AWS Cognito不支持Okta IDP 发起的flow的解决方法"
subtitle:   "Okta cognito integration"
date:       2019-12-28
author:     "leasy"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - AWS
    - Cognito
    - Okta
---

> "Okta创建Bookmark App"

我参考[AWS Cognito文档](https://aws.amazon.com/cn/premiumsupport/knowledge-center/cognito-okta-saml-identity-provider/) 将Okta设置为SAML提供商，但是当我从Okta中发起登陆流程的时候发现会报错，错误信息是

```bash
Invalid samlResponse or relayState from identity provider
```

搜索了一下发现AWS Cognito目前是不支持IDP主动发起的Flow,只支持SP发起的flow,关于这两种flow 的区别可以参考之前的文章[IDP and SP 发起的flow](https://leasyzhang.github.io/2019/08/30/IdP-and-SP-initiated-SSO/)。

配置好SAML提供商之后AWS Cognito会提供一个链接，通过链接来进行登陆，链接的格式是

```javascript
https://yourDomainPrefix.auth.region.amazoncognito.com/login?response_type=token&client_id=yourClientId&redirect_uri=redirectUrl
```

这种情况下可以通过Okta的Bookmark APP来解决，不过在新版的Okta里面Bookmarp APP已经被移除，用户只能手动创建书签来解决这个问题：

- 在Okta主界面选择"添加应用"
![添加应用程序](https://leasyzhang.github.io/img/in-post/okta-cognito-integration/okta-create-app.jpg)
- 这个时候搜索Bookmark APP会被告知不存在，但是下面有个添加书签的按钮，点击添加书签
- 在应用登陆URL中填写AWS提供的链接，应用名称填写自己要配置的名字然后选择添加，就可以看到一个书签的图标出现在应用程序主界面
![创建应用程序](https://leasyzhang.github.io/img/in-post/okta-cognito-integration/add-book-mark.jpg)
![应用程序图标](https://leasyzhang.github.io/img/in-post/okta-cognito-integration/the-app.jpg)

这样就可以在Okta中发起AWS Cognito的授权流程。
