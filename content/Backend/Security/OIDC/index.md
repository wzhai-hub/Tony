---
# Documentation: https://hugoblox.com/docs/managing-content/

title: "OpenID Connect"
subtitle: ""
summary: "基于OAuth 2.0的身份认证授权协议.它结合了OAuth 2.0的授权功能和OpenID的认证功能，为客户端应用提供了一种安全、标准的方式来验证用户的身份并获取用户的基本信息。"
authors: []
# tags:
#   - Istio
categories: []
date: 2024-03-08T11:27:22+08:00
lastmod: 2024-03-15T11:27:22+08:00
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

OIDC（OpenID Connect）是一个基于OAuth 2.0的身份认证授权协议。它结合了OAuth 2.0的授权功能和OpenID的认证功能，为客户端应用提供了一种安全、标准的方式来验证用户的身份并获取用户的基本信息。

OIDC规定了一套将用户身份信息从授权服务器（身份提供方）传递给客户机应用（身份使用方）的标准流程、格式。这使得客户端应用可以基于用户在授权服务器上进行的身份验证来验证其身份，并获取用户的基本资料，如姓名、邮箱等。

在实际使用中，OAuth 2.0主要解决了授权问题，但并未实现认证部分，往往需要添加额外的API来实现认证。而OpenID是一个单独的认证协议。综合两者，OIDC既拥有OAuth 2.0的授权功能，又实现了OpenID的认证功能，从而成为了一种更加全面和高效的身份认证授权协议。

此外，OIDC还在OAuth 2.0的access_token基础上增加了身份认证信息，通过公钥私钥配合校验获取身份等其他信息，即idToken，使得身份认证更加安全、可靠。

总的来说，OIDC是一种安全、灵活且高效的身份认证授权协议，广泛应用于各种场景，为客户端应用提供了便捷的身份验证和用户信息管理功能。
