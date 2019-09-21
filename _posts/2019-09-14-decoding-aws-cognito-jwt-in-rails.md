---
layout:     post
title:      "用Rails解析AWS Cognito JWT数据"
subtitle:   "Go Web Application With Heroku"
date:       2019-09-16
author:     "leasy"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - AWS Cognito
    - Rails
    - Ruby
---

> "Decoding and verifing AWS Cognito in Rails"

这篇文章介绍一下我用Rails去解析AWS Cognito JWT数据的实践经验。

### JWT
JWT是JSON Web Token的缩写，是一个开放的互联网标准，在[RFC 7519](https://tools.ietf.org/html/rfc7519)中有详细的定义。JWT可以用来做身份认证，比如服务端生成一个包含用户属性的token，这些属性表示用户在系统中的角色，这个token返回给客户端，客户端在后续的请求中带上这个token(通常是在request header里面带上)，服务端拿到token之后可以做解析和校验，来验证用户是否合法。

### AWS Cognito
AWS Cognito是AWS(Amazon Web Service)提供的用户身份认证服务，它包含两个功能：用户池和身份池，用户池提供注册和登录功能，身份池可以向用户授予其他AWS服务的访问权限。我使用的是用户池服务。

### AWS Cognito JWT
Cognito授权成功之后会返回三个token([参考文档](https://docs.aws.amazon.com/cognito/latest/developerguide/token-endpoint.html))
- Access Token: JWT数据,包含用户基本信息,数据格式如下
```json
header:
{
  "kid": "xxxx",
  "alg": "RS256"
}

payload:
{
  "sub": "xxx",
  "cognito:groups": [
    ""
  ],
  "token_use": "access",
  "scope": "aws.cognito.signin.user.admin phone openid profile email",
  "auth_time": 1231,
  "iss": "https://cognito-idp.region.amazonaws.com/user-pool-id",
  "exp": 1231231,
  "iat": 112412412,
  "version": 2,
  "jti": "xxxxxx",
  "client_id": "xxxx",
  "username": "xxxx"
}
```
- OIDC Token: JWT,包含更丰富的信息，比如第三方IdP的信息，格式如下
```json
header:
{
  "kid": "xxxx",
  "alg": "RS256"
}

payload:
{
  "at_hash": "xxxxxx",
  "sub": "xxxx",
  "cognito:groups": [
    "xxxxx"
  ],
  "email_verified": false,
  "iss": "https://cognito-idp.region.amazonaws.com/user-pool-id",
  "cognito:username": "xxxx",
  "aud": "xxxx",
  "identities": [
    {
      "userId": "email address",
      "providerName": "provider name",
      "providerType": "SAML",
      "issuer": "dadqweoif",
      "primary": "true",
      "dateCreated": "1566895337105"
    }
  ],
  "token_use": "id",
  "auth_time": 1566957171,
  "exp": 1566960771,
  "iat": 1566957171,
  "email": "email address"
}
```
相比于access token， OIDC token还包含了identities字段，这个字段包含了第三方IdP的信息。可以用来验证第三方IdP的信息。
- Refresh Token: 用来换取新的access token.
OIDC token包含的字段在Cognito配置界面(常规设置-->属性菜单)
![OIDC token fields](https://leasyzhang.github.io/img/in-post/cognito-integration/oidc-token-fields.jpg)
这里面所有字段都可以包含在OIDC token里面。

### 解码JWT
JWT的数据包含三部分，header.payload.signature.所有的数据都经过Base64编码处理，但是注意这不是加密处理，可以直接解码。所以不要在JWT中保存密码等敏感信息。
关于三个部分的含义，参考[JWT组成](https://jwt.io/introduction/#what-is-the-json-web-token-structure-).
如果要获取JWT的数据，只需要使用Base64解码就可以
```ruby
# oidc_token is Base64 encoded JWT data
oidc_token = params["id_token"]
header = oidc_token.split(".")[0]
payload = oidc_token.split(".")[1]
decoed_header = Base64.decode64(header)
decoded_payload = Base64.decode64(payload)
```

### 校验&解码 Cognito JWT
- Cognito 使用了[JWK](https://tools.ietf.org/html/rfc7517)来做JWT数据的校验，我们用[json-jwt](https://github.com/nov/json-jwt)这个库来做JWT的校验和解析.
- Cognito [JWKS](https://auth0.com/docs/jwks)存储的位置如下
```html
https://cognito-idp.region.amazonaws.com/userPoolId/.well-known/jwks.json
```
把region和userPoolId替换成自己创建的用户池的region和poolId，从上述链接获取JWKS数据,
- 构造JWKS对象
```ruby
# keys_url refers to https://cognito-idp.region.amazonaws.com/userPoolId/.well-known/jwks.json
jwk_set = JSON::JWK::Set.new(
            JSON.parse(
                HTTP.get(keys_url).body
            )
        )
```
用JWKS和JWT校验数据
```ruby
JSON::JWT.decode jwt_string, jwk_set
```
如果顺利返回数据，说明校验正确。如果不正确，这个方法会抛出JSON::JWS::VerificationFailed异常，需要手动处理异常信息。而且校验通过的话返回值是JWT数据本身，需要自己用Base64解码。

### 结论
JWT的校验不管用什么库来处理，流程都是相同的
- 从给定的url获取JWKS数据
- 构建JWK对象
- 用JWT和JWKS来验证数据
完整的代码在[ruby tools repo](https://github.com/LeasyZhang/ruby-tools/tree/master/decode-aws-jwt)
AWS官方的仓库有一个python版本的实现，在 [aws-support-tools](https://github.com/awslabs/aws-support-tools/tree/master/Cognito/decode-verify-jwt)。