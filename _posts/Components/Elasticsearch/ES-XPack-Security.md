---
title: ES_XPack_Security
date: 2019-03-05 22:56:20
tags: [Elasticsearch]
---

> 🚧

<!--more-->

## ElasticSearch && X_Pack_Security

#### X_Pack_Security基础配置

```yml
# 启动security配置 所有Master/Data节点都需要配置
xpack.security.enabled: true
# 设置是否禁用初始密码 「changeme」, 设置为false，禁用初始密码
xpack.security.authc.accept_default_password: false
# 配置relam
xpack.security.authc.realms:
	# 自定义领域名称
    realm1:
    	# 可选值有native, ldap, active_directory, pki,  file
        type: native
        # 优先级顺序 越低越高
        order: 0
        # 是否开启，默认开启
        enabled: true
    relam2:
    	type: ldap
    	order: 1
# 配置匿名用户，用户名，所属 
xpack.security.authc:
  anonymous:
    username: anonymous_user 
    # 指定匿名用户所具有的角色权限
    roles: role1, role2 
    # 当匿名用户执行相关请求时，若缺少权限，则返回http403 response，且不会提示(默认为true)
    # false时，无权限访问则返回401，且提示相关信息。若需要与HTTP结合使用，则选择false
    authz_exception: true 
```

#### 生成Security密码

```shell
cd bin
#自动生成
./elasticsearch-setup-passwords auto
#手动生成 注意：该命令仅仅第一次执行有效，后续若希望修改密码，可以通过KibanaUI/ES X_PACK_Security API进行修改
./elasticsearch-setup-passwords interactive
```

- elastic　　超级用户
- Kibana　　用于连接并且和Elasticsearch通信的
- logstash_system     用于在Elasticsearch中存储监控信息
- beats_system    用于在Elasticsearch中存储监控信息

> Tips : 千万要注意自己的配置，笔者在配置集群YML文件时，设置最小Cluster Master节点需要两个。因此执行密码手动重置时，抛出Master节点不足错误。

#### 

------



#### X_Pack_Security API

##### 用户CRUD API

```shell
# 查看所有用户
curl -X GET -u elastic "localhost:9200/_xpack/security/user"

# 查看指定用户
curl -X GET -u elastic "localhost:9200/_xpack/security/user/${userName}"

# 创建用户
curl -X POST -u elastic "localhost:9200/_xpack/security/user/${userName}" -H 'Content-Type: application/json' -d'
{	
   #声明密码 分配权限 
  "password" : "j@rV1s",
  "roles" : [ "admin", "other_role1" ],
  "full_name" : "Jack Nicholson",
  "email" : "jacknich@example.com",
  "metadata" : {
    "intelligence" : 7
  }
}

# 修改密码
curl -X POST "localhost:9200/_xpack/security/user/${userName}/_password" -H 'Content-Type: application/json' -d'
{
  "password" : "s3cr3t"
}

# 禁用、启用、删除用户
curl -X PUT "localhost:9200/_xpack/security/user/${userName}/_disable"
curl -X PUT "localhost:9200/_xpack/security/user/${userName}/_enable"
curl -X DELETE "localhost:9200/_xpack/security/user/${userName}"
```

##### 角色CRUD API

```shell
# 查询所有角色
curl -X GET "localhost:9200/_xpack/security/role"

# 查询具体role
curl -X GET "localhost:9200/_xpack/security/role/${roleName}"

# 删除角色
curl -X DELETE "localhost:9200/_xpack/security/role/${roleName}"

# 创建角色
curl -X POST "localhost:9200/_xpack/security/role/${roleName}" -H 'Content-Type: application/json' -d'
{
   # 角色 所能操作的cluster
  "cluster": ["all"],
  "indices": [
    {
      # role所能操作的cluster 符合条件的index(必选)
      "names": [ "index1", "index2" ], 
      # 具体权限(必选)
      "privileges": ["all"],
      "field_security" : { // 可选
        "grant" : [ "title", "body" ]
      },
      "query": "{"match": {"title": "foo"}}" // 可选
    }
  ],
  "run_as": [ "other_user" ], // 可选
  "metadata" : { // 可选
    "version" : 1
  }
}
'
```



------



## Kibana && X_Pack_Security



#### X_Pack_Security基础配置

当es集群启动后，Kibana想要连接到集群，则需要配置相关权限账户信息。

```yml
# 直接于yml文件标注帐密
elasticsearch.username: "kibana"
elasticsearch.password: "%{password}"
# 使用key_Store 安全标注帐密
//todo

# 标注启用xpack
xpack.security.enabled: true

# 可选，默认情况下kibana将自动生成该key,存放内存中。修改后，原有session失效
xpack.security.encryptionKey: "32位或更长自定义加密KEY"

# 可选，更改默认session过期时间
xpack.security.sessionTimeout: 600000
```

------



#### Kibana的用户认证方式

* Basic Authentication：**默认选项**，登录kibana时，需要填写帐密。基于ES的native  relam
* SAML Single Sign-On：允许用户使用外部身份提供者（如Okta或Auth0）登录Kibana。在Kibana中设置SAML之前，请确保在Elasticsearch中启用和配置SAML。



------



#### User authentication

为了访问受保护的资源，一个用户必须通过密码、凭证、或者其它方式（通常是token）来证明他们的身份标识。

认证过程由一个或多个被称为“realms”的认证服务来处理。

你可以用本机支持管理和认证用户，或者集成外部的用户管理系统（比如：LDAP 和 Active Directory）。

X-Pack安全特性提供了内置的realms，比如：native，ldap，active_directory，pki，file 和 saml。如果没有一个内置realms满足你的需求，你还可以构建自己的realm。

当启用X-Pack安全特性时，根据你配置的realms，你必须将用户凭证附加到发送到Elasticsearch的请求中。例如，当使用支持用户名和密码的realms时，你可以简单的将basic auth头信息添加到请求中。