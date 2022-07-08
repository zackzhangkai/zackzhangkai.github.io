---
layout: post
published: true
title:  openstack之keystone讲解
categories: [document]
tags: [openstack,私有云]
---
* content
{:toc}

## 前言
Keystone是openstack的服务，通过identify API提供API客户端认证，服务发现，和分布式多租户认证  
官网 https://docs.openstack.org/keystone/latest/getting-started/architecture.html
## Services
通过暴露服务的endpoints，让user/project认证，成功后通过Token service创建并返回token

### Idenify
通过LDAP来做CRUD操作

#### Users  
usrs并不是全局惟一的，它只是在在其domain下惟一

#### Groups  
它是多个user的集合，与users一样，只在其domain下惟一

### Resource
提供projects和domains的数据

#### projects
只在其domain下惟一，若不指定其doain，则默认为 **Default**

#### domains
它是Projects/users/groups的一个容器，通过namespace隔离，默认为Default，domain name全局惟一，通过assginment可以让User在不同的domain下访问数据

### Assignment
提供role和 role Assignments的数据

#### Roles
定义user的认证级别，roles可以授权到doamin级别，也可以授权到project级别；一个role要绑定一个独立的user或是一个组;
#### Role Assignments
它是个元组包含三个元素[Role,Resource,Identify]

### Token
通过Token service提供，访问令牌

### Catalog
通过Catalog sevice为endpint discovery 提供endpoint registry

## Application Construction
keystone对其它的服务而言一个HTTP的Frontend；跟其它应用一样，通过python的WSGI动态网关接口；应用的HTTP endpoint通过WSGI的pipeline组装而成，如：
```bash
[pipeline:api_v3]
pipeline = healthcheck cors sizelimit http_proxy_to_wsgi osprofiler url_normalize request_id build_auth_context json_body ec2_extension_v3 s3_extension service_v3
```

## Data Model
+ User  
+ Group  
+ Project  
+ Domain  
+ Role  
+ Token  
+ Extra  
+ Rule  

## Policy
在每个服务的对应的配置目录，如![/etc/keystone/policy.json](/styles/images/keystone1.jpg)  
具体对应的每个API的policy，参考[点我](https://docs.openstack.org/keystone/latest/getting-started/policy_mapping.html)；关于policy.json文件的语法，请参考[点我](https://docs.openstack.org/ocata/config-reference/policy-json-file.html);在此放上一篇Nova的policy.json的示例，但是请注意，该未例有个变量没有定义，需要自己手动加上，已经跟官网提了bug，后续如果加上了，那么这个可以忽略，为
```bash
  "admin_api": "is_admin:True or (role:admin and is_admin_project:True)"
```
[compute服务示例参考](https://docs.openstack.org/mitaka/config-reference/compute/policy.json.html)

## Plugins
+ keystone.auth.plugins.external.Base  
+ keystone.auth.plugins.mapped.Mapped  
+ keystone.auth.plugins.oauth1.OAuth  
+ keystone.auth.plugins.password.Password  
+ keystone.auth.plugins.token.Token  
+ keystone.auth.plugins.totp.TOTP  

## 通过curl调用API获取数据
1. 第一步首先要先获取token  
```bash
curl -i \
  -H "Content-Type: application/json" \
  -d '
{ "auth": {
    "identity": {
      "methods": ["password"],
      "password": {
        "user": {
          "name": "admin",
          "domain": { "id": "default" },
          "password": “xxxx"
        }
      }
    },
    "scope": {
      "project": {
        "name": “admin",
        "domain": { "id": "default" }
      }
    }
  }
}' \
  "https://x.x.x.x:5000/v3/auth/tokens” -k |python -m json.tool
```  

2. 拿到token后，就可以获取项目上的数据了，如获取虚机   
```bash
token=37a9799a28f94d5498737cf0018107f8
project_id=6f12225f5fc946c7bae62646fff5dfb2
url="https://x.x.x.x:8774/v2.1/$project_id/servers"
curl -s -H "X-Auth-Token:$token" $url -k |python -m json.tool
```
project_id：通过openstack project list获取  

3. 同理依次类推，如要获取什么数据，最重要的就是拿到对应的API：

  + /v3/domains:列出domains  
    ```bash
    curl -s \
     -H "X-Auth-Token: $OS_TOKEN" \
     "http://localhost:5000/v3/domains" | python -mjson.tool
   ```

  + /v3/domains: 创建一个domain  
    ```bash
    curl -s \
     -H "X-Auth-Token: $OS_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{ "domain": { "name": "newdomain"}}' \
     "http://localhost:5000/v3/domains" | python -mjson.tool
    ```  

  + /v3/projects:列出projects
    ```bash
    curl -s \
     -H "X-Auth-Token: $OS_TOKEN" \
     "http://localhost:5000/v3/projects" | python -mjson.tool
    ```

  + /v3/services:列出services
    ```bash
    curl -s \
     -H "X-Auth-Token: $OS_TOKEN" \
     "http://localhost:5000/v3/services" | python -mjson.tool
    ```

  + /v3/endpoints:获取endpoints
    ```bash
    curl -s \
     -H "X-Auth-Token: $OS_TOKEN" \
     "http://localhost:5000/v3/endpoints" | python -mjson.tool
    ```  
  + /v3/users 列出users  
     ```bash
     curl -s \
      -H "X-Auth-Token: $OS_TOKEN" \
      "http://localhost:5000/v3/users" | python -mjson.tool
     ```  
  + /v3/users 创建user
      ```bash
      curl -s \
       -H "X-Auth-Token: $OS_TOKEN" \
       -H "Content-Type: application/json" \
       -d '{"user": {"name": "newuser", "password": "changeme"}}' \
       "http://localhost:5000/v3/users" | python -mjson.tool
      ```

  + /v3/users/{user_id}：列出用户的详细信息：
      ```bash
       USER_ID=ec8fc20605354edd91873f2d66bf4fc4
       curl -s \
        -H "X-Auth-Token: $OS_TOKEN" \
        "http://localhost:5000/v3/users/$USER_ID" | python -mjson.tool
      ````

  + /v3/users/{user_id}/password：修改密码:普通用户
    ```bash
    USER_ID=b7793000f8d84c79af4e215e9da78654
    ORIG_PASS=userpwd
    NEW_PASS=newuserpwd

    curl \
     -H "X-Auth-Token: $OS_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{ "user": {"password": "'$NEW_PASS'", "original_password": "'$ORIG_PASS'"} }' \
     "http://localhost:5000/v3/users/$USER_ID/password"
    ```

  + PATCH  /v3/users/{user_id}:修改密码，用admin修改普通用户密码
    ```bash
    USER_ID=b7793000f8d84c79af4e215e9da78654
    NEW_PASS=newuserpwd

    curl -s -X PATCH \
     -H "X-Auth-Token: $OS_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{ "user": {"password": "'$NEW_PASS'"} }' \
     "http://localhost:5000/v3/users/$USER_ID" | python -mjson.tool
    ```
  + PUT /v3/projects/{project_id}/groups/{group_id}/roles/{role_id}:
  在一个项目里创建一个group role assignment
    ```bash
    curl -s -X PUT \
     -H "X-Auth-Token: $OS_TOKEN" \
     "http://localhost:5000/v3/projects/$PROJECT_ID/groups/$GROUP_ID/roles/$ROLE_ID" |
     python -mjson.tool
    ```
  + 其他还有很多，请参考[官方API](https://docs.openstack.org/keystone/pike/api_curl_examples.html)

## keystone package
Keystone的安装包对应的[代码模块](https://docs.openstack.org/keystone/latest/api/keystone.assignment.backends.base.html)

## 命令行
### keystone-manage
keystone-manage初始化或是更新Keystone内部数据用，只在在HTTP API不能完成时才使用这个命令，如数据库迁移、数据迁入迁出；
+ bootstrap: 产生新的bootstrap进程  
+ credential_migrate: 用新的私钥产生一个新的认证  
+ credential_setup
+ db_sync: 同步数据库  
+ db_version  
+ doctor: 常见问题诊断  
+ domain_config_upload: 上传新的domain配置文件  
+ ...  

### keystone-status
升级deployment

+ upgrade  
+ upgrade check  
