---
title: "Docker Registry V2 的授权验证"
date: 2018-01-15T22:04:35+08:00
---

**Docker Registry**是一个无状态，高可扩展的Docker镜像发布与存储的服务器端应用。 通常我们可以将镜像提交到官方或者其他服务，但有时候也需要自己部署个人或公司的Registry。

本文不讲解Docker Registry的部署，只讲解如何做授权。

Docker Registry自身不提供授权服务器，而是把授权委托给第三方。一次完整的请求过程如下所示：

![v2-registry-auth](/images/post/docker-registry-auth/v2-registry-auth.png)

1. Attempt to begin a push/pull operation with the registry.
2. If the registry requires authorization it will return a 401 Unauthorized HTTP response with information on how to authenticate.
3. The registry client makes a request to the authorization service for a Bearer token.
4. The authorization service returns an opaque Bearer token representing the client’s authorized access.
5. The client retries the original request with the Bearer token embedded in the request’s Authorization header.
6. The Registry authorizes the client by validating the Bearer token and the claim set embedded within it and begins the push/pull session as usual.

## 发起请求

当你调用`docker login`, `docker pull`或者`docker push`的时候， docker命令都是向默认的registry服务(registry-1.docker.io)发起HTTP请求，如果请求携带的授权信息不够，Registry服务会拒绝服务，返回一个401状态码的响应, 例如：

如果用户执行命令：

`docker push repository:samalba/my-app:latest`

在未登录的情况下，Registry服务会返回如下响应：

```
HTTP/1.1 401 Unauthorized
Content-Type: application/json; charset=utf-8
Docker-Distribution-Api-Version: registry/2.0
Www-Authenticate: Bearer realm="https://auth.docker.io/token",service="registry.docker.io",scope="repository:samalba/my-app:pull,push"
Date: Thu, 10 Sep 2015 19:32:31 GMT
Content-Length: 235
Strict-Transport-Security: max-age=31536000

{"errors":[{"code":"UNAUTHORIZED","message":"access to the requested resource is not authorized","detail":[{"Type":"repository","Name":"samalba/my-app","Action":"pull"},{"Type":"repository","Name":"samalba/my-app","Action":"push"}]}]}
```
这个响应告诉客户端(docker client及其他客户端）如何及去哪儿里获取授权(Docker Registry v2 Bearer token)。

头信息里有三个信息：

* service. 
* scope
* realm
    
其中service是registry自己，realm授权服务器(authorization server)的地址，scope是要处理的资源及动作。

scope的格式有3个部分：

* 资源类型。目前而言有两个:registry, repository
* 资源名称。samalba/my-app
* 请求的动作。 pull，push，*。

## 获取授权

用401 回应里信息向授权服务器发起请求(GET):

```
https://auth.docker.io/token?service=registry.docker.io&scope=repository:samalba/my-app:pull,push

```
除了必须的service跟scope参数外，还可以附加其他信息，例如：offline_token， client_id。 scope一次可以传送多个。

授权服务器可以自己实现，也可以利用现成的。不管是那种方式，授权服务器必须支持Basic Auth，因为：Docker client目前只支持Basic Auth。

> From Docker 1.11 the Docker engine supports both Basic Authentication and OAuth2 for getting tokens. Docker 1.10 and before, the registry client in the Docker Engine only supports Basic Authentication. 

如果你提交的身份信息无误，授权服务器应该根据自己的策略检查你的申请。比如资源不是公开的，那么只有提交者自己能够下载，或者资源是公开的，但不是你的，那么push操作会被禁止。

> After authenticating the client (which may simply be an anonymous client if no attempt was made to authenticate), the token server must next query its access control list to determine whether the client has the requested scope. In this example request, if I have authenticated as user jlhawn, the token server will determine what access I have to the repository samalba/my-app hosted by the entity registry.docker.io.

> Once the token server has determined what access the client has to the resources requested in the scope parameter, it will take the intersection of the set of requested actions on each resource and the set of actions that the client has in fact been granted. If the client only has a subset of the requested access it must not be considered an error as it is not the responsibility of the token server to indicate authorization errors as part of this workflow.

> Continuing with the example request, the token server will find that the client’s set of granted access to the repository is [pull, push] which when intersected with the requested access [pull, push] yields an equal set. If the granted access set was found only to be [pull] then the intersected set would only be [pull]. If the client has no access to the repository then the intersected set would be empty, [].

> It is this intersected set of access which is placed in the returned token.


那么授权服务器应该返回一个json格式的token响应，里面会包含如下内容：

* token
* access_token
* expires_in
* issued_at
* refresh_token
    
token或者access_token的作用是一样的，只是名字不同，但是响应里必须有其中一个；其他的三个字段则是可选的。

```
HTTP/1.1 200 OK
Content-Type: application/json

{"token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJFUzI1NiIsImtpZCI6IlBZWU86VEVXVTpWN0pIOjI2SlY6QVFUWjpMSkMzOlNYVko6WEdIQTozNEYyOjJMQVE6WlJNSzpaN1E2In0.eyJpc3MiOiJhdXRoLmRvY2tlci5jb20iLCJzdWIiOiJqbGhhd24iLCJhdWQiOiJyZWdpc3RyeS5kb2NrZXIuY29tIiwiZXhwIjoxNDE1Mzg3MzE1LCJuYmYiOjE0MTUzODcwMTUsImlhdCI6MTQxNTM4NzAxNSwianRpIjoidFlKQ08xYzZjbnl5N2tBbjBjN3JLUGdiVjFIMWJGd3MiLCJhY2Nlc3MiOlt7InR5cGUiOiJyZXBvc2l0b3J5IiwibmFtZSI6InNhbWFsYmEvbXktYXBwIiwiYWN0aW9ucyI6WyJwdXNoIl19XX0.QhflHPfbd6eVF4lM9bwYpFZIV0PfikbyXuLx959ykRTBpe3CYnzs6YBK8FToVb5R47920PVLrh8zuLzdCr9t3w", "expires_in": 3600,"issued_at": "2009-11-10T23:00:00Z"}

```


## 使用token

当获取token后，在随后的请求携带Authorization header：

```
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJFUzI1NiIsImtpZCI6IkJWM0Q6MkFWWjpVQjVaOktJQVA6SU5QTDo1RU42Ok40SjQ6Nk1XTzpEUktFOkJWUUs6M0ZKTDpQT1RMIn0.eyJpc3MiOiJhdXRoLmRvY2tlci5jb20iLCJzdWIiOiJCQ0NZOk9VNlo6UUVKNTpXTjJDOjJBVkM6WTdZRDpBM0xZOjQ1VVc6NE9HRDpLQUxMOkNOSjU6NUlVTCIsImF1ZCI6InJlZ2lzdHJ5LmRvY2tlci5jb20iLCJleHAiOjE0MTUzODczMTUsIm5iZiI6MTQxNTM4NzAxNSwiaWF0IjoxNDE1Mzg3MDE1LCJqdGkiOiJ0WUpDTzFjNmNueXk3a0FuMGM3cktQZ2JWMUgxYkZ3cyIsInNjb3BlIjoiamxoYXduOnJlcG9zaXRvcnk6c2FtYWxiYS9teS1hcHA6cHVzaCxwdWxsIGpsaGF3bjpuYW1lc3BhY2U6c2FtYWxiYTpwdWxsIn0.Y3zZSwaZPqy4y9oRBVRImZyv3m_S9XDHF1tWwN7mL52C_IiA73SJkWVNsvNqpJIn5h7A2F8biv_S2ppQ1lgkbw
```

*官方文档：https://docs.docker.com/registry/spec/auth/token/*
