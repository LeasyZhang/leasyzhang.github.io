---
layout:     post
title:      "IdP Flow和 SP Flow的SSO集成"
subtitle:   "Okta， Aws Cognito和PingFederate集成"
date:       2019-08-30 16:40:07 +0800
author:     "leasy"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - SSO
    - AWS Cognito
    - Okta
    - PingFederate
---


In this article I will introduce the IdP and SP initiated SSO, it contains three main parts:
- The terms of IdP and SP initiated SSO,
- IdP initiated SSO flow and SP initiated SSO flow, 
- Implement two SSO systems with different flow
    - IdP initiated SSO using Okta and AWS Cognito
    - SP initiated SSO using Okta and PingFederate.

### The terms

The Federation Authentication Request varies depending on the protocol used:
SAML 2.0: AuthnRequest
SAML 1.1: a URL with a parameter representing the SP
WS-Fed: a URL with a wtrealm parameter representing the SP and other optional parameters
OpenID 2.0: OpenID 2.0 Request

#### IdP
#### SP
#### SSO
#### Authentication Protocol

### Difference between IdP initiated SSO and SP initiated SSO

#### IdP initiated SSO
#### SP initiated SSO

### Implementation
#### IdP initiated SSO(AWS Cognito and Okta)
#### SP initiated SSO(PingFederate and Okta)

### Conclusion

### **References**
- [AWS Cognito Configuration](https://aws.amazon.com/cn/premiumsupport/knowledge-center/cognito-okta-saml-identity-provider/)
- [AWS Cognito user pool verify process](https://docs.aws.amazon.com/zh_cn/cognito/latest/developerguide/authentication-flow.html)
- [AWS Cognito IdP initiated Authentication Flow](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-saml-idp-authentication.html)