The OAuth 2.0 Authorization Framework

# 1.介绍

## 1.1.角色

- resource owner

- resource server 

- client 

- authorization server 



## 1.3. 授权模式（四种）

### 1.3.1. Authorization Code
应用对象：机密client（client是与用户代理分开）

客户端与资源所有者存在严重的不信任，特别是用户代理（浏览器）
资源所有者的身份凭证和client的身份凭证先后通过之后，授权服务器才能返回token。

必须有重定向的url，是要指定在哪个client中使用

### 1.3.2. Implicit
应用对象：公共client（client是与用户代理一块）
所以没有client身份验证，只有对应的重定向url

client的实现是在用户代理上（浏览器）实现的情况下，直接向授权服务器发送资源所有者的身份凭证，
即可直接返回token


### 1.3.3. Resource Owner Password Credentials
应用对象：本机client

资源所有者与客户端存在高度的信任

直接向授权服务器发送资源所有者与客户端各自的身份凭证即可返回token


### 1.3.4. Client Credentials

client就是资源所有者的情况下，直接向授权客户端发送client的身份凭证即可，直接返回token


## 1.4. Access Token

访问令牌是通过资源所有同意，授权服务器发放的，client只能用来使用，不能更改token，里边的内容也是不透明的。

## 1.5. Refresh Token

刷新token只能向授权服务器发送请求。


# 2. Client Registration

## 2.1. client类型
- 机密client

- 公共client

- 本机client

## 2.2. Client 标识
由授权服务器发送给client

## 2.3. client 身份验证
client身份凭证是授权服务器颁发的

### 2.3.1. Client Password




# 3. 协议Endpoints

授权服务器Endpoints
- 授权Endpoints  从资源所有者那里获取授权

- token Endpoints 通过验证客户端身份，发放token


客户端Endpoints
- 重定向Endpoints 将授权服务器发放的token发给client

## 3.1. 授权Endpoints

与资源所有者进行交互，并授予权限。授权服务器必须首先验证资源所有者的身份。
授权端点由授权代码授权类型和隐式授权类型流使用。

### 3.1.1. Response Type
必输项

授权码类型：code
implicit ：token

### 3.1.2.重定向Endpoints

完成与资源所有者的交互后，授权服务器将资源所有者的用户代理重定向到client。

#### 3.1.2.1.Endpoint Request的机密性

保证http传输的安全性

#### 3.1.2.2.注册要求

授权服务器可允许客户端注册多个重定向端点。

- 公共client
- 机密client

#### 3.1.2.3.动态配置

可以动态配置url

## 3.2. token Endpoint

除了Implicit授权模式外，其他授权模式都使用token endpoint

### 3.2.1.client身份认证

保证发送的token是对应的client，而不是被授权码所取代.

## 3.3. Access Token Scope

scope由授权服务器进行定义



# 4. 获得授权

## 4.1.Authorization Code Grant

由于是基于重定向来实现的,所以客户端必须与用户代理(web浏览器)进行交互。接受授权服务器返回的code。

### 4.1.1.授权请求

```
    GET /authorize?response_type=code&client_id=s6BhdRkqt3&state=xyz
        &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb HTTP/1.1
    Host: server.example.com
```

state:主要是用来防止跨站点请求伪造。

redirect_uri:在访问授权服务器完成之后，需要指引到响应的client中。

发出授权请求之后，资源所有者在授权服务器的引导下，进行身份验证和授权准予。



### 4.1.2.授权响应

响应的url是请求时重定向的url

```
     HTTP/1.1 302 Found
     Location: https://client.example.com/cb?code=SplxlOBeZQQYbYS6WxSbIA
               &state=xyz
```

#### 4.1.2.1.错误响应

```
   HTTP/1.1 302 Found
   Location: https://client.example.com/cb?error=access_denied&state=xyz
```

### 4.1.3. Access Token 请求

```
     POST /token HTTP/1.1
     Host: server.example.com
     Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
     Content-Type: application/x-www-form-urlencoded
     
grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb
```

### 4.1.4. Access Token 响应

```
     HTTP/1.1 200 OK
     Content-Type: application/json;charset=UTF-8
     Cache-Control: no-store
     Pragma: no-cache
     
     {
       "access_token":"2YotnFZFEjr1zCsicMWpAA",
       "token_type":"example",
       "expires_in":3600,
       "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
       "example_parameter":"example_value"
     }
```

## 4.2. Implicit Grant

不支持刷新token

### 4.2.1. 授权请求

```
    GET /authorize?response_type=token&client_id=s6BhdRkqt3&state=xyz
        &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb HTTP/1.1
    Host: server.example.com
```

发出授权请求之后，资源所有者在授权服务器的引导下，进行身份验证和授权准予。

### 4.2.2. Access Token 响应

```
     HTTP/1.1 302 Found
     Location: http://example.com/cb#access_token=2YotnFZFEjr1zCsicMWpAA
               &state=xyz&token_type=example&expires_in=3600
```

#### 4.2.2.1. Error Response

```
   HTTP/1.1 302 Found
   Location: https://client.example.com/cb#error=access_denied&state=xyz
```

## 4.3. Resource Owner Password Credentials Grant

### 4.3.2.Access Token请求

```
     POST /token HTTP/1.1
     Host: server.example.com
     Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
     Content-Type: application/x-www-form-urlencoded
     
grant_type=password&username=johndoe&password=A3ddj3w
```

### 4.3.3.Access Token响应

```
     HTTP/1.1 200 OK
     Content-Type: application/json;charset=UTF-8
     Cache-Control: no-store
     Pragma: no-cache
     
    {
       "access_token":"2YotnFZFEjr1zCsicMWpAA",
       "token_type":"example",
       "expires_in":3600,
       "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
       "example_parameter":"example_value"
     }
```

## 4.4.Client Credentials Grant

只能适合机密客户端

### 4.4.2.Access Token请求

授权服务器必须对客户端进行身份验证。

```
     POST /token HTTP/1.1
     Host: server.example.com
     Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
     Content-Type: application/x-www-form-urlencoded
     
     grant_type=client_credentials
```

### 4.4.3.Access Token响应

不应包括刷新令牌

```
     HTTP/1.1 200 OK
     Content-Type: application/json;charset=UTF-8
     Cache-Control: no-store
     Pragma: no-cache
     
    {
       "access_token":"2YotnFZFEjr1zCsicMWpAA",
       "token_type":"example",
       "expires_in":3600,
       "example_parameter":"example_value"
     }
```

# 5. 发放 Access  token

## 5.1. 成功响应

```
     HTTP/1.1 200 OK
     Content-Type: application/json;charset=UTF-8
     Cache-Control: no-store
     Pragma: no-cache
     
    {
       "access_token":"2YotnFZFEjr1zCsicMWpAA",
       "token_type":"example",
       "expires_in":3600,
       "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
       "example_parameter":"example_value"
     }
```

## 5.2. 错误响应

```
     HTTP/1.1 400 Bad Request
     Content-Type: application/json;charset=UTF-8
     Cache-Control: no-store
     Pragma: no-cache
     
     {
       "error":"invalid_request"
     }
```



# 6. 刷新Access  token

```
     POST /token HTTP/1.1
     Host: server.example.com
     Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
     Content-Type: application/x-www-form-urlencoded
     
     grant_type=refresh_token&refresh_token=tGzv3JOkF0XG5Qx2TlKWIA
```



# 7.访问受保护的资源

由token的类型来进行对应的token验证操作。

## 7.1. Access Token 类型

最常用的token类型是bearer



# 11.IANA的思考
