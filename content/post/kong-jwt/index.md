---
title: Kong + JWT
date: 2020-03-23 11:02:02
tags:
    - gateway
    - kong
    - nginx
    - lua
categories:
  - gateway
---
<!--more-->
## 配置服务

添加 Service

```shell
curl -i -X POST \
  --url http://localhost:8001/services/ \
  --data 'name=api-gateway-sv' \
  --data 'url=https://blog.bitqiu.cc'
```

 为服务添加路由

```shell
curl -i -X POST \
  --url http://localhost:8001/services/api-gateway-sv/routes \
  --data 'hosts[]=api.bitqiu.cc'
```

通过Kong转发您的请求

```shell
curl -i -X GET \
  --url http://localhost:8000/ \
  --header 'Host: api.bitqiu.cc'
```

## 配置 JWT

service 开启 jwt 插件

```shell
# 查看插件列表
curl -i -X GET \
	--url http://localhost:8000/
# 开启 jwt 插件
curl -i -X POST \
	--url http://localhost:8001/services/api-gateway-sv/plugins \
	--data 'name=jwt'
```

创建 consumer

```shell
curl -i -X POST \
	--url http://localhost:8000/consumers \
	--data 'username=bitqiu'
```

创建 jwt 凭证

> 可以指定算法`algorithm`，`iss`签发者`key`，密钥`secret`，也可以省略，会自动生成。

```shell
curl -i -X POST \
	--url http://localhost:8000/consumers/bitqiu/jwt \
	--data 'algorithm=HS256' \
	--data 'key=bitqiu_key' \
	--data 'secret=uFLMFeKPPL525ppKrqmUiT2rlvkpLc9u'

# 查看 jwt 凭证
curl -i -X GET \
	--url http://localhost:8000/consumers/bitqiu/jwt
# response
{
    "rsa_public_key": null,
    "created_at": 1553681782,
    "consumer": {
        "id": "2e34d380-ec48-4a0d-926f-6dd8696a7eca"
    },
    "id": "61ee520c-3387-42f0-8e5f-02e0dc34d3d4",
    "algorithm": "HS256",
    "secret": "uFLMFeKPPL525ppKrqmUiT2rlvkpLc9u",
    "key": "7Xc3L8TdFpU6kgPEeR4iqMAstqLewJSS"
}
```

### jwt 下发

业务服务器根据 `kong` 生成的 `jwt` 凭证中的 `algorithm、key（iss）、secret`进行 `token` 的演算和下发。请求`鉴权接口`需携带
` Authorization: Bearer jwt` 进行请求。测试的话可以用 [https://jwt.io](https://jwt.io/) 生成：


```shell
curl -i -X GET \
  --url http://localhost:8000/ \
  --header 'Host: api.bitqiu.cc'
  --header 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJkb3dudG93bl9rZXkiLCJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.nDvZa6Nu9cLxO36jnXsHwYIxrNLrDomJgvKJ5gihn4k'
```

![](https://i.loli.net/2021/03/11/COTYlxoSHP6m8yM.png)