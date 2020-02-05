---
title: OpenLdap学习
date: 2020-02-03 00:00:00
tags:
 - openldap
categories:
 - Openldap
---

# 运行
``` bash
$ docker run -d --privileged -p 10004:80 --name php \
  --env PHPLDAPADMIN_HTTPS=false --env PHPLDAPADMIN_LDAP_HOSTS=172.17.0.1  \
  --detach osixia/phpldapadmin
```

> `172.17.0.1` 注意留意下本地docker网卡的ip地址

``` bash
$ docker run -p 389:389 --name ldap \
  --network bridge --hostname openldap-host \
  --env LDAP_ORGANISATION="example" --env LDAP_DOMAIN="example.org" \
  --env LDAP_ADMIN_PASSWORD="pwd" --detach osixia/openldap
```

## SSL
如果要使用SSL访问, 将地址从`ldap://localhost`换成`ldaps://localhost`

若使用自己的证书

``` bash
$ docker run --hostname ldap.example.org \
  --volume /path/to/certificates:/container/service/slapd/assets/certs \
  --env LDAP_TLS_CRT_FILENAME=my-ldap.crt \
  --env LDAP_TLS_KEY_FILENAME=my-ldap.key \
  --env LDAP_TLS_CA_CRT_FILENAME=the-ca.crt \
  --detach osixia/openldap
```

## [原生客户端](http://www.ldapadmin.org/)

支持`Windows` 和`Linux` , 个人感觉体验不如**[phpldapadmin](http://phpldapadmin.sourceforge.net/wiki/index.php/Main_Page)**

+ <http://www.ldapadmin.org/download/ldapadmin.html>
+ <http://ivb.sweb.cz/en/ldap.html>
+ <https://github.com/ibv/LDAP-Admin/releases>

## 验证
打开本地浏览器, 访问<http://localhost:10004> 即可打开`PHPLdapAdmin`

> 端口10004和docker保持一致

需要登录, 登录帐号密码`cn=admin,dc=example,dc=org` / `pwd`

> 注意参数和docker中的保持一致, `缺省用户是固定的admin`

### 命令行
也可以登录docker shell用命令行测试

``` bash
$ docker ps
CONTAINER ID  IMAGE               COMMAND               CREATED        STATUS        PORTS                          NAMES
bfa797f2e0c7  osixia/phpldapadmin "/container/tool/run" 15 minutes ago Up 15 minutes 443/tcp, 0.0.0.0:10004->80/tcp php
e0c9ca20a8f1  osixia/openldap     "/container/tool/run" 15 minutes ago Up 15 minutes 0.0.0.0:389->389/tcp, 636/tcp  ldap
$ docker exec -it --privileged ldap /bin/bash
$ ldapsearch -x -H ldap://localhost -D "cn=admin,dc=example,dc=org" -w pwd -b "dc=example,dc=org"
# extended LDIF
#
# LDAPv3
# base <dc=example,dc=org> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# example.org
dn: dc=example,dc=org
objectClass: top
objectClass: dcObject
objectClass: organization
o: example
dc: example

# admin, example.org
dn: cn=admin,dc=example,dc=org
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: admin
description: LDAP administrator
userPassword:: e1NTSEF9cVNPeHRiYTFoQ0RtMlJPcWRwOW0vaTVULzRidzE2NHI=

# search result
search: 2
result: 0 Success

# numResponses: 3
# numEntries: 2
```


# 概念

ldap的数据是分层的数组结构, 并且可能有多个DB,每个DB一棵树

```
dc:         com
             |
dc:        genfic         ## (公司)
          /      \
ou:   People   servers    ## (公司部门)
      /    \     ..
uid: ..   John            ## (部门里的数据)
```

一些常用的概念

+ Distinguished Name(`DN`) 惟一辨别名，类似于Linux文件系统中的绝对路径，每个对象都有一个惟一的名称，如"uid=king,ou=qiu,dc=example,dc=org"，在一个目录树中DN总是惟一的
+ RDN, DN最左边的一节, 一直凭借父节点的RDN就构成了DN
+ OID, 可以理解为内部ID, 但是仍然需要申请并公示, 避免重复
+ Domain Component(`DC`) 域名的部分，其格式是将完整的域名分成几部分，如域名为example.org变成dc=example,dc=org
+ User Id(`uid`) 用户ID，如"king"
+ Organization Unit(`OU`) 组织单位，类似于Linux文件系统中的子目录，它是一个容器对象，组织单位可以包含其他各种对象（包括其他组织单元），如"lixin"
+ Common Name(`CN`) 公共名称，如"king qiu"
+ Surname(`SN`) 姓，如"king"
+ Relative dn(`RDN`) 相对辨别名，类似于文件系统中的相对路径，它是与目录树结构无关的部分，如"uid=qiu"或"cn=king"
+ Country(`C`) 国家，如"CN"或"US"等。
+ Organization(`O`) 组织名，如"Example, Inc."

## 通信协议/渠道

> 以下内容以具体实现[**`slapd`**](https://linux.die.net/man/8/slapd) 为准

slapd提供了几种方式客户端接入[连接渠道](https://www.openldap.org/doc/admin24/runningslapd.html#Command-Line%20Options)

| URL |	Protocol | Transport |
| - |  - |  - |  
| ldap:/// | LDAP	TCP | port 389 |
| ldaps:/// | LDAP over SSL | TCP port 636 |
| ldapi:/// | LDAP | IPC (Unix-domain socket) |

当使用官方的客户端工具时, 使用参数`-H`

## [后端(Backends)](https://www.openldap.org/doc/admin24/backends.html)

slapd 支持多种数据的后端存储方式

+ Berkeley DB
+ LDAP, not an actual database; instead it acts as a proxy to forward incoming requests to another LDAP server
+ LDIF, a basic storage backend that stores entries in text files in LDIF format
+ LMDB, the recommended primary backend for a normal slapd database. It uses OpenLDAP's own Lightning Memory-Mapped Database (LMDB) library to store data and is intended to replace the Berkeley DB backends
+ etc...

``` bash
# 从cn=config下过滤特定objectClass所有条目的dn
$ ldapsearch -LLL -Y EXTERNAL -H ldapi:/// -b cn=config \
    -s sub "(objectClass=olcDatabaseConfig)" dn
dn: olcDatabase={-1}frontend,cn=config
dn: olcDatabase={0}config,cn=config
dn: olcDatabase={1}mdb,cn=config
```

可以看到, 上述实例使用的`mdb`

## [数据库(Database)](https://www.openldap.org/doc/admin24/slapdconf2.html)

> 早期的slapd使用配置文件`slapd.conf`, 目前已不建议, 考虑`dynamic runtime configuration engine` , 后者有以下优势

+ is fully LDAP-enabled
+ is managed using the standard LDAP operations
+ stores its configuration data in an LDIF database, generally in the /usr/local/etc/openldap/slapd.d directory.
+ allows all of slapd's configuration options to be changed on the fly, generally without requiring a server restart for the changes to take effect.

从上面的环境可以看到, 缺省就存在三个`database`

+ frontend, 对所有DB有效的配置, 优先级较低(special database that is used to hold database-level options that should be applied to all the other databases)
+ config, 配置数据库
+ mdb(视bachend而不同), 实际的数据存储

> 实际上slapd可以有多个db, slapd会根据dn匹配各个库的[`olcSuffix`](https://www.openldap.org/doc/admin24/slapdconf2.html#olcSuffix:%20%3Cdn%20suffix%3E)来做路由

``` bash
$ ldapsearch -LLL -Y EXTERNAL -H ldapi:/// -b "olcDatabase={1}mdb,cn=config" \
   -s sub "(objectClass=olcDatabaseConfig)"  olcSuffix
dn: olcDatabase={1}mdb,cn=config
olcSuffix: dc=example,dc=org
```

官方提供了

+ [**`ldapsearch`**](https://www.zytrax.com/books/ldap/ch14/#ldapsearch) 查询(search LDAP entries)
+ [**`ldapadd`**](https://www.zytrax.com/books/ldap/ch14/#ldapadd) 新增(add LDIF entries to an LDAP directory)
+ [**`ldapdelete`**](https://www.zytrax.com/books/ldap/ch14/#ldapdelete) 删除(delete LDAP entries)
+ [**`ldapmodify`**](https://www.zytrax.com/books/ldap/ch14/#ladpadd) 修改(modify existing LDAP entries)


# config库
config库的根DN是`cn=config`, 这里包含了所有的配置

具体的每个字段的意义, 参考<https://www.openldap.org/doc/admin24/slapdconf2.html>

``` bash
$ ldapsearch -LLL -Y EXTERNAL -H ldapi:/// -b "cn=config" dn 2>/dev/null | head -n 5
dn: cn=config
dn: cn=module{0},cn=config
dn: cn=module{1},cn=config
```

## [数据库配置](https://www.openldap.org/doc/admin24/slapdconf2.html#Database-specific%20Directives)

几个重要的配置

+ [olcAccess](https://www.openldap.org/doc/admin24/slapdconf2.html#olcAccess:%20to%20%3Cwhat%3E%20[%20by%20%3Cwho%3E%20[%3Caccesslevel%3E]%20[%3Ccontrol%3E]%20]+) 访问控制记录, 一般会有多条
+ [olcRootDN](https://www.openldap.org/doc/admin24/slapdconf2.html#olcRootDN:%20%3CDN%3E)  超级管理员DN
+ [olcRootPW](https://www.openldap.org/doc/admin24/slapdconf2.html#olcRootPW:%20%3Cpassword%3E)  超级管理员密码
+ [olcSuffix](https://www.openldap.org/doc/admin24/slapdconf2.html#olcSuffix:%20%3Cdn%20suffix%3E) 路由的DN后缀

> 对于`olcSuffix`, 更具体的路由应该在前, 否则后面的db永远无法路由

# 添加/删除/配置
``` bash
$ cat > base.ldif  << EOF
> dn: ou=lixin,dc=example,dc=org
objectClass: organizationalUnit
ou: lixin
# 空行
dn: ou=tech,ou=lixin,dc=example,dc=org
objectClass: organizationalUnit
ou: tech
description: 产品部
# 空行
dn: cn=king1,ou=tech,ou=lixin,dc=example,dc=org
objectClass: top
objectClass: organizationalRole
objectClass: simpleSecurityObject
cn: king1
userPassword: 1
# 空行
dn: cn=king2,ou=tech,ou=lixin,dc=example,dc=org
objectClass: top
objectClass: organizationalRole
objectClass: simpleSecurityObject
cn: king2
userPassword: 1
> EOF
#
$ ldapadd -x -H ldap://localhost  -w pwd -D cn=admin,dc=example,dc=org -f base.ldif 
adding new entry "ou=lixin,dc=example,dc=org"
adding new entry "ou=tech,ou=lixin,dc=example,dc=org"
adding new entry "cn=king1,ou=tech,ou=lixin,dc=example,dc=org"
adding new entry "cn=king2,ou=tech,ou=lixin,dc=example,dc=org"
```
查询
``` bash
$ ldapsearch -x -H ldap://localhost  -w pwd -D cn=admin,dc=example,dc=org \
    -b "ou=lixin,dc=example,dc=org" dn
# lixin, example.org
dn: ou=lixin,dc=example,dc=org

# tech, lixin, example.org
dn: ou=tech,ou=lixin,dc=example,dc=org

# king1, tech, lixin, example.org
dn: cn=king1,ou=tech,ou=lixin,dc=example,dc=org

# king2, tech, lixin, example.org
dn: cn=king2,ou=tech,ou=lixin,dc=example,dc=org

# search result
search: 2
result: 0 Success

# numResponses: 5
# numEntries: 4
```
修改
``` bash
$ cat > c << EOF
> dn: cn=king2,ou=tech,ou=lixin,dc=example,dc=org
changetype: modify
add: description
description: 你懂的
> EOF
#
$ ldapmodify -x -H ldap://localhost  -w pwd -D cn=admin,dc=example,dc=org -f c
modifying entry "cn=king2,ou=tech,ou=lixin,dc=example,dc=org"
#
$ cat > c << EOF
> dn: cn=king2,ou=tech,ou=lixin,dc=example,dc=org
changetype: modify
delete: description
description: 你懂的
-
add: street
street: iii
> EOF""
$ ldapmodify -x -H ldap://localhost  -w pwd -D cn=admin,dc=example,dc=org -f c
modifying entry "cn=king2,ou=tech,ou=lixin,dc=example,dc=org"
```
多字段属性增删改
``` bash
$ ldapsearch -x -H ldap://localhost  -w pwd -D cn=admin,dc=example,dc=org \
    -b "cn=king2,ou=tech,ou=lixin,dc=example,dc=org" street | \
    sed -e "/^$/d" -e "/^#/d"
dn: cn=king2,ou=tech,ou=lixin,dc=example,dc=org
street: iii
street: jjj
street: kkk
# 删除stree=jjj的条目, 修改street=kkk的条目修改为lll
$ cat > c << EOF
> dn: cn=king2,ou=tech,ou=lixin,dc=example,dc=org
changetype: modify
delete: street
street: jjj
-
delete: street
street: kkk
-
add: street
street: lll
> EOF
#
$ ldapsearch -x -H ldap://localhost  -w pwd -D cn=admin,dc=example,dc=org \
    -b "cn=king2,ou=tech,ou=lixin,dc=example,dc=org" street | \
    sed -e "/^$/d" -e "/^#/d"
dn: cn=king2,ou=tech,ou=lixin,dc=example,dc=org
street: iii
street: lll
```

删除
``` bash
$ cat > base.ldif << EOF
> dn: cn=admin,ou=tech,ou=lixin,dc=example,dc=org
objectClass: top
objectClass: organizationalRole
objectClass: simpleSecurityObject
cn: admin
userPassword: 1
> EOF
#
$ ldapadd -x -H ldap://localhost  -w pwd -D cn=admin,dc=example,dc=org -f base.ldif
adding new entry "cn=admin,ou=tech,ou=lixin,dc=example,dc=org"
#
$ ldapdelete -H ldap://localhost -D cn=admin,dc=example,dc=org -w 1 \
    cn=king1,ou=tech,ou=lixin,dc=example,dc=org
#
$ ldapsearch -x -H ldap://localhost  -w pwd -D cn=admin,dc=example,dc=org \
    -b "ou=lixin,dc=example,dc=org" dn
# lixin, example.org
dn: ou=lixin,dc=example,dc=org

# tech, lixin, example.org
dn: ou=tech,ou=lixin,dc=example,dc=org

# admin, tech, lixin, example.org
dn: cn=admin,ou=tech,ou=lixin,dc=example,dc=org

# king2, tech, lixin, example.org
dn: cn=king2,ou=tech,ou=lixin,dc=example,dc=org

# search result
search: 2
result: 0 Success
```

## 查询/filter
ldap的优势是`快速`/`灵活`的查询, ldapsearch有一个filter参数, 用于定制查询条件

几个概念
+ basedn 从树的那个子(根)节点开始查找(`-b`)
+ filter 查询条件
+ attrs 查询结果返回哪些属性

> filter手册,参考<https://www.zytrax.com/books/ldap/apa/search.html>

``` bash
# 从`ou=lixin,dc=example,dc=org`开始查找
# 匹配属性cn等于`king2`的条目
$ ldapsearch -x -H ldap://localhost -w pwd -D cn=admin,dc=example,dc=org \
    -b "ou=lixin,dc=example,dc=org" \
    cn=king2 | sed -e "/^$/d" -e "/^#/d"
dn: cn=king2,ou=tech,ou=lixin,dc=example,dc=org
objectClass: top
objectClass: organizationalRole
objectClass: simpleSecurityObject
cn: king2
userPassword:: e1NTSEF9RzFqOGszdG82aHRWamRvQmYvRUlqbmo1dHNpb0NQcTE=
street: iii
description:: 5L2g5oeC55qE
# 接上, 只显示dn属性
$ ldapsearch -x -H ldap://localhost -w pwd -D cn=admin,dc=example,dc=org \
    -b "ou=lixin,dc=example,dc=org" \
    cn=king2 dn | sed -e "/^$/d" -e "/^#/d"
dn: cn=king2,ou=tech,ou=lixin,dc=example,dc=org
```

### 判断
+ = 等于
+ ~= 不等于
+ \>= 大于等于
+ <= 小于等于

> 比较逻辑支持通配符`*`

### 与或非

+ & (AND)
+ ! (NOT)
+ | (OR)

> 优先级用括号`(`界定

```
(&(exp1)(exp2)(exp3)) # exp1 AND exp2 AND exp3
(|(exp1)(exp2)(exp3)) # exp1 OR exp2 OR exp3
(!(exp1)) # NOT exp1
(&(!(exp1))(!(exp2))) # NOT exp1 AND NOT exp2
```

``` bash
# cn等于admin 或者 street等于iii
$ ldapsearch -x -H ldap://localhost  -w pwd -D cn=admin,dc=example,dc=org \
    -b "ou=lixin,dc=example,dc=org" \
    "(|(cn=admin)(street=iii))" dn | sed -e "/^$/d" -e "/^#/d"
dn: cn=admin,ou=tech,ou=lixin,dc=example,dc=org
dn: cn=king2,ou=tech,ou=lixin,dc=example,dc=org
```

### DN查找
用于查找DN中的每一段是否包含某个属性, `不支持通配符`
``` bash
# dn中包含ou等于tech的条目
$ ldapsearch -x -H ldap://localhost  -w pwd -D cn=admin,dc=example,dc=org \
    -b "ou=lixin,dc=example,dc=org" "ou:dn:=tech" dn | sed -e "/^$/d" -e "/^#/d"
dn: ou=tech,ou=lixin,dc=example,dc=org
dn: cn=admin,ou=tech,ou=lixin,dc=example,dc=org
dn: cn=king2,ou=tech,ou=lixin,dc=example,dc=org
# dn中包含cn等于admin的条目
$ ldapsearch -x -H ldap://localhost  -w pwd -D cn=admin,dc=example,dc=org \
    -b "ou=lixin,dc=example,dc=org" "cn:dn:=admin" dn | sed -e "/^$/d" -e "/^#/d"
dn: cn=admin,ou=tech,ou=lixin,dc=example,dc=org
```

## [匹配规则(matchingrules)](https://www.zytrax.com/books/ldap/ch3/#matchingrules)
每个属性的业务场景各有不同, 所以ldap引入了匹配规则, 用于约束属性的匹配规则, 例如姓名大小写不一致关系并不大

```
matchingRule ( 2.5.13.2 NAME 'caseIgnoreMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 )
```

> ldap内建了一大波规则

```
# Subschema
dn: cn=Subschema
matchingRules: ( 2.5.13.0 NAME 'objectIdentifierMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.38 )
matchingRules: ( 2.5.13.1 NAME 'distinguishedNameMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.12 )
matchingRules: ( 2.5.13.2 NAME 'caseIgnoreMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 )
matchingRules: ( 2.5.13.3 NAME 'caseIgnoreOrderingMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 )
matchingRules: ( 2.5.13.4 NAME 'caseIgnoreSubstringsMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.58 )
matchingRules: ( 2.5.13.5 NAME 'caseExactMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 )
matchingRules: ( 2.5.13.6 NAME 'caseExactOrderingMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 )
matchingRules: ( 2.5.13.7 NAME 'caseExactSubstringsMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.58 )
matchingRules: ( 2.5.13.8 NAME 'numericStringMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.36 )
matchingRules: ( 2.5.13.10 NAME 'numericStringSubstringsMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.58 )
matchingRules: ( 2.5.13.13 NAME 'booleanMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.7 )
matchingRules: ( 2.5.13.14 NAME 'integerMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 )
matchingRules: ( 2.5.13.15 NAME 'integerOrderingMatch' 
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 )
matchingRules: ( 2.5.13.16 NAME 'bitStringMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.6 )
matchingRules: ( 2.5.13.17 NAME 'octetStringMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.40 )
matchingRules: ( 2.5.13.18 NAME 'octetStringOrderingMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.40 )
matchingRules: ( 2.5.13.20 NAME 'telephoneNumberMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.50 )
matchingRules: ( 2.5.13.21 NAME 'telephoneNumberSubstringsMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.58 )
matchingRules: ( 2.5.13.23 NAME 'uniqueMemberMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.34 )
matchingRules: ( 2.5.13.27 NAME 'generalizedTimeMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.24 )
matchingRules: ( 2.5.13.28 NAME 'generalizedTimeOrderingMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.24 )
matchingRules: ( 2.5.13.29 NAME 'integerFirstComponentMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 )
matchingRules: ( 2.5.13.30 NAME 'objectIdentifierFirstComponentMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.38 )
matchingRules: ( 2.5.13.34 NAME 'certificateExactMatch'
  SYNTAX 1.2.826.0.1.3344810.7.1 )
matchingRules: ( 1.3.6.1.4.1.1466.109.114.1 NAME 'caseExactIA5Match'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
matchingRules: ( 1.3.6.1.4.1.1466.109.114.2 NAME 'caseIgnoreIA5Match'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
matchingRules: ( 1.3.6.1.4.1.1466.109.114.3 NAME 'caseIgnoreIA5SubstringsMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
matchingRules: ( 1.3.6.1.4.1.4203.1.2.1 NAME 'caseExactIA5SubstringsMatch' 
 SYNTAX 1.3.6.1.4.1.1466.115.121.1.26 )
matchingRules: ( 1.2.840.113556.1.4.803 NAME 'integerBitAndMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 )
matchingRules: ( 1.2.840.113556.1.4.804 NAME 'integerBitOrMatch'
  SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 )
```

在比较时, 可以后面加一个`oid` 修改缺省的匹配规则
``` bash
# default sn EQUALITY comparison behaviour
# is caseIgnoreMatch (2.5.13.2)
sn=smith

# override EQUALITY match with case sensitive match
sn:caseExactMatch:=Smith
# functionally same as above using OID
sn:2.5.13.5:=Smith

# if a wildcard appears in seach the SUBSTR 
# MatchingRule applies

# default sn SUBSTR comparison behavior
# is caseIgnoreSubstringMatch
sn=*s* # finds Smith or smith

# override SUBSTR match with case sensitive match
sn:caseExactSubstringMatch:=*S* # only finds Smith
# functionally same as above using OID
sn:2.5.13.7:=*S*
```

# [`权限`](https://www.openldap.org/doc/admin24/access-control.html)
到目前位置, 所有的操作,要不使用unix socket通道(`ldapi:///`), 不受权限控制, 要不就使用超级管理员`admin,dc=example,dc=org`

slapd提供了完善的权限控制, 让其他用户也能做全部/部分操作

缺省情况下, 其他用户只有对自己的权限
``` bash
$ ldapsearch -LLL -Y EXTERNAL -H ldapi:/// -b "olcDatabase={1}mdb,cn=config" \
    -s sub "(objectClass=olcDatabaseConfig)"  olcAccess
dn: olcDatabase={1}mdb,cn=config
olcAccess: {0}to attrs=userPassword,shadowLastChange by self write by dn="cn=a
 dmin,dc=example,dc=org" write by anonymous auth by * none
olcAccess: {1}to * by self read by dn="cn=admin,dc=example,dc=org" write by * 
 none
# 查询lixin部门
$ ldapsearch -x -H ldap://localhost -D "cn=admin,ou=tech,ou=lixin,dc=example,dc=org" \
    -w 1 -b "ou=lixin,dc=example,dc=org" dn
search: 2
result: 32 No such object
# 查询自己
$ ldapsearch -x -H ldap://localhost -D "cn=admin,ou=tech,ou=lixin,dc=example,dc=org" \
    -w 1 -b "cn=admin,ou=tech,ou=lixin,dc=example,dc=org" dn
# admin, tech, lixin, example.org
dn: cn=admin,ou=tech,ou=lixin,dc=example,dc=org

search: 2
result: 0 Success
```

现在给`cn=admin,ou=tech,ou=lixin,dc=example,dc=org` 加一个`read`权限
``` bash
$ cat > perm << EOF
> dn: olcDatabase={1}mdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to dn.subtree="ou=lixin,dc=example,dc=org"
    by dn.exact="cn=admin,ou=tech,ou=lixin,dc=example,dc=org" read
    by anonymous auth
    by * none
> EOF
$ ldapmodify -H ldapi:// -Y EXTERNAL -f perm
#
$ ldapsearch -LLL -Y EXTERNAL -H ldapi:/// -b "olcDatabase={1}mdb,cn=config" \
   -s sub "(objectClass=olcDatabaseConfig)"  olcAccess
dn: olcDatabase={1}mdb,cn=config
olcAccess: {0}to dn.subtree="ou=lixin,dc=example,dc=org"   by dn.exact="cn=adm
 in,ou=tech,ou=lixin,dc=example,dc=org" read   by anonymous auth   by * none
olcAccess: {1}to attrs=userPassword,shadowLastChange by self write by dn="cn=a
 dmin,dc=example,dc=org" write by anonymous auth by * none
olcAccess: {2}to * by self read by dn="cn=admin,dc=example,dc=org" write by * 
 none
```
``` bash
$ ldapsearch -x -H ldap://localhost -D "cn=admin,ou=tech,ou=lixin,dc=example,dc=org" \
    -w 1 -b "ou=lixin,dc=example,dc=org" dn
# lixin, example.org
dn: ou=lixin,dc=example,dc=org

# tech, lixin, example.org
dn: ou=tech,ou=lixin,dc=example,dc=org

# hr, tech, lixin, example.org
dn: cn=hr,ou=tech,ou=lixin,dc=example,dc=org

# admin, tech, lixin, example.org
dn: cn=admin,ou=tech,ou=lixin,dc=example,dc=org

# king1, tech, lixin, example.org
dn: cn=king1,ou=tech,ou=lixin,dc=example,dc=org

# king2, tech, lixin, example.org
dn: cn=king2,ou=tech,ou=lixin,dc=example,dc=org

search: 2
result: 0 Success
# 查分配的权限之外的dn就找不到了
$ ldapsearch -x -H ldap://localhost -D "cn=admin,ou=tech,ou=lixin,dc=example,dc=org" \
    -w 1 -b "dc=example,dc=org" dn
search: 2
result: 32 No such object
```

## 权限Index
由于权限规则定义的顺序非常关键, 所以slapd使用一个`{X}`来锁定顺序, 在上面的例子中,可以看到`{0}, {1}, {2}` 

> 我们的配置中手动写死{0}, 是为了确保新插入的规则优先级最高, 因为此调规则的DN范围是小于后两条

{X}的另外一个作用是在增删改查时用作定位, 例如删除第一条

```
dn: olcDatabase={1}mdb,cn=config
changetype: modify
delete: olcAccess
olcAccess: {0}
```

## 权限集合

| Level |  Privileges | Description |
|-|-|-|
| none  | 0  |  no access |
| disclose |  d  |  needed for information disclosure on error |
| auth  | dx |  needed to authenticate (bind) |
| compare |   cdx | needed to compare |
| search  |   scdx |    needed to apply search filters |
| read  | rscdx  |  needed to read search results |
| write | wrscdx |  needed to modify/rename |
| manage |    mwrscdx | needed to manage |

+ 高级别权限自动包含低级别权限
+ search和read的区别在于, search可以返回`查到了`, 但是没有`查到的具体结果`
+ 认证权限, 表示允许登录, 至于能不能成功, 则使用帐号密码认证

## 权限规则
规则语法
```
access to what： 
by who access control
```
+ control 就是之前的权限等级, 读/写/授权etc
+ 指令中包含1个to语句，可多个by语句
+ `what` 限定规则应用条目(entries)的集合, 全部用`*`
+ `who` 授权给哪些实体(entities)

### what

> 匹配所有使用`*`
> 多种匹配规则可同时使用

[scope](https://www.openldap.org/doc/admin24/access-control.html#What%20to%20control%20access%20to)(基于某个根DN匹配)

+ base matches only the entry with provided DN
+ one matches the entries whose parent is the provided DN
+ subtree matches all entries in the subtree whose root is the provided DN
+ children matches all entries under the DN (but not the entry named by the DN)

```
    0: o=suffix
    1: cn=Manager,o=suffix
    2: ou=people,o=suffix
    3: uid=kdz,ou=people,o=suffix
    4: cn=addresses,uid=kdz,ou=people,o=suffix
    5: uid=hyc,ou=people,o=suffix
```
+ dn.base="ou=people,o=suffix" match 2; 
+ dn.one="ou=people,o=suffix" match 3, and 5; 
+ dn.subtree="ou=people,o=suffix" match 2, 3, 4, and 5; and 
+ dn.children="ou=people,o=suffix" match 3, 4, and 5.

regex(基于DN正则表达式匹配)

```
access to dn.regex="(.+,)?ou=People,(dc=[^,]+,dc=[^,]+)$"
```

filter(基于过滤器)
``` bash
filter=(objectClass=person)
# 配合使用
dn.one="ou=people,o=suffix" filter=(objectClass=person)
```

attrs(只授权部分属性)

缺省授权所有的attr, 这里可以过滤限定属性的范围

```
to attrs=member,entry
```

> 有两个`伪属性`(pseudo), `entry` `children` , 需要特别注意

+ To read (and hence return) a target entry, the subject must have read access to the target's `entry` attribute
+ To perform a search, the subject must have search access to the search base's `entry` attribute.
+ To add or delete an entry, the subject must have write access to the entry's `entry` attribute AND must have write access to the entry's parent's `children` attribute
+ To rename an entry, the subject must have write access to entry's `entry` attribute AND have write access to both the old parent's and new parent's `children` attributes.


### who

用于限定权限赋予给哪些实体, 一些特殊的实体如下

| Specifier | Entities | 
|-|-|
| *	 | All, including anonymous and authenticated users | 
| anonymous	| Anonymous (non-authenticated) users | 
| users	| Authenticated users | 
| self|	User associated with target entry | 


剩下两种通用的匹配方式 `scope` `regex` 和what用于保持一致

一个特殊的`dnattr`用于限定DN在某种条目的attr列表中

> 这个attr只能存放其他条目的DN

### 权限评估(Evaluation)

当判断who是否具有what的access权限时
+ 根据权限结合的优先级, 从先到后
+ 找到第一条匹配what的记录(`dn/attr`)
+ 根据这条记录的by, 判断当前用户是否符合who
  - 如果没有who, 那么会append一个缺省的`access to * by * none` (**而不是继续评估下个access to**)
  - 如果有, 则检查by的access 是否满足客户端申请的权限
+ 如果没有what, 那么`then a default read is used`

> 所以, 一个`access to`必须把所有`by`的考虑清楚, 否则未考虑的`who`将无权限

## 密码

新建用户时, 可以用明文写到`userPassword` 一次性创建, 通过ldapsearch可以看到密码被加密了, 不过只是简单的`base64编码`, 安全性较低

``` bash
# admin, tech, lixin, example.org
dn: cn=admin,ou=tech,ou=lixin,dc=example,dc=org
userPassword:: MQ==
```

我们可以用`slappasswd` 生成带盐的强加密密码

再测试之前, 先调整权限, 让所有用户可以自己修改密码
``` bash
# 内建规则, 所有用户可以自己修改密码
olcAccess: {0}to attrs=userPassword,shadowLastChange 
  by self write 
  by dn="cn=admin,dc=example,dc=org" write 
  by anonymous auth 
  by * none
# 本地新增,移动到第二位
olcAccess: {1}to dn.subtree="ou=lixin,dc=example,dc=org"   by dn.exact="cn=adm
 in,ou=tech,ou=lixin,dc=example,dc=org" read   by anonymous auth   by * none
# 超级管理员
olcAccess: {2}to * by self read by dn="cn=admin,dc=example,dc=org" write by * 
 none
```

``` bash
$ slappasswd -h {SSHA}
New password: 
Re-enter new password: 
{SSHA}zDDMqF0N2uSGQfsFlYIYJVehKpVlCqhi
#
$ cat > pwd << EOF
> dn: cn=admin,ou=tech,ou=lixin,dc=example,dc=org
changetype: modify
replace: userPassword
userPassword: {SSHA}zDDMqF0N2uSGQfsFlYIYJVehKpVlCqhi
> EOF
# 
$ ldapmodify -x -H ldap://localhost  -w pwd \
   -D "cn=admin,ou=tech,ou=lixin,dc=example,dc=org" -f pwd
modifying entry "cn=admin,ou=tech,ou=lixin,dc=example,dc=org"
#
$ ldapsearch -x -H ldap://localhost -D "cn=admin,ou=tech,ou=lixin,dc=example,dc=org" \
  -w pwd -b "ou=lixin,dc=example,dc=org" userPassword
ldap_bind: Invalid credentials (49)
#
$ ldapsearch -x -H ldap://localhost -D "cn=admin,ou=tech,ou=lixin,dc=example,dc=org" \
   -w 123456 -b "ou=lixin,dc=example,dc=org" userPassword
# admin, tech, lixin, example.org
dn: cn=admin,ou=tech,ou=lixin,dc=example,dc=org
userPassword:: e1NTSEF9ekRETXFGME4ydVNHUWZzRmxZSVlKVmVoS3BWbENxaGk=
#
$ echo e1NTSEF9ekRETXFGME4ydVNHUWZzRmxZSVlKVmVoS3BWbENxaGk= | base64 -d
{SSHA}zDDMqF0N2uSGQfsFlYIYJVehKpVlCqhi
```

# 第三方授权
ldap一个很重要的用途就是做第三方的认证授权

通常的做法
+ 客户输入一个`cn`或者`uid`, 按照限定的规则拼装成DN
+ 根据限定的搜索条件, 查询DN是否存在
+ 使用DN和客户输入的密码做bind

## Group
slapd内建了一个`groupOfNames`的objectClass, 其包含一个member的属性, 该属性只能包含其他DN
```
# ou, lixin, example.org
dn: cn=ou,ou=lixin,dc=example,dc=org
member: cn=admin,ou=tech,ou=lixin,dc=example,dc=org
member: cn=king2,ou=tech,ou=lixin,dc=example,dc=org
objectClass: groupOfNames
objectClass: top
cn: ou
```

那么可以考虑将角色和groupOfNames关联起来, 如果某个用户在这个groupOfNames的member中,那么就具有这个角色

这里有个问题, 在user端, 没有渠道查到它属于哪些groups, 为此需要安装额外的插件

## memberOf

具体安装参考<https://mayanbin.com/post/enable-memberof-in-openldap.html>

``` bash
$ cat > a << EOF
> dn: cn=module,cn=config
cn: module
objectClass: olcModuleList
olcModulePath: /usr/lib64/openldap
# 空行
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: memberof.la
> EOF
#
$ ldapadd -Q -Y EXTERNAL -H ldapi:/// -f a
adding new entry "cn=module,cn=config"

modifying entry "cn=module{0},cn=config"
#
$ cat > a << EOF
> dn: olcOverlay=memberof,olcDatabase={1}mdb,cn=config
objectClass: olcConfig
objectClass: olcMemberOf
objectClass: olcOverlayConfig
objectClass: top
olcOverlay: memberof
olcMemberOfDangling: ignore
olcMemberOfRefInt: TRUE
olcMemberOfGroupOC: groupOfNames
olcMemberOfMemberAD: member     
olcMemberOfMemberOfAD: memberOf
> EOF
# 
$ ldapadd -Q -Y EXTERNAL -H ldapi:/// -f a
adding new entry "olcOverlay=memberof,olcDatabase={1}mdb,cn=config"
```

之后就有了`memberOf` 字段

``` bash
$ ldapsearch -x -H ldap://localhost -D "cn=admin,ou=tech,ou=lixin,dc=example,dc=org" \
    -w 123456 -b "cn=admin,ou=tech,ou=lixin,dc=example,dc=org" memberof
# admin, tech, lixin, example.org
dn: cn=admin,ou=tech,ou=lixin,dc=example,dc=org
memberOf: cn=ou,ou=lixin,dc=example,dc=org
memberOf: cn=ou2,ou=lixin,dc=example,dc=or
```

注意:
+ memberof不属于强制，ldapsearch查询需要单独指定
+ 老数据不会修复

当我们做组判断时, 
+ 根据三方系统配置的组名, 按照规则拼接完整DN, 
+ 根据搜索规则, 判断DN是否存在
+ 检查用户的memberOf里面是否存在这个组DN


# Python

``` bash
$ pip install flask_simpleldap
```
> 如果不使用flask, 也可以直接安装**[python-ldap](https://www.python-ldap.org/en/python-ldap-3.2.0/)**

Debian/Ubuntu:

``` bash
$ sudo apt-get install libsasl2-dev python-dev libldap2-dev libssl-dev
```

RedHat/CentOS:

``` bash
$ sudo yum install python-devel openldap-devel
```

``` python
import ldap
ldapconn = ldap.initialize('ldap://localhost:389')

def p(con, u, p):
    res = con.simple_bind_s(u, p)
    print(res)

    res = con.whoami_s()
    print(res)

    res = con.search_s(
        "cn=admin,dc=example,dc=org",
        ldap.SCOPE_SUBTREE,
        # ldap.SCOPE_BASE,
        # ldap.SCOPE_ONELEVEL,
        '(&(cn=admin))',
    )
    print(res)

p(ldapconn, 'cn=admin,dc=example,dc=org', "pwd")
```

新建用户, 注意`uid`, `objectClass`
``` ldif
# king, example.org
dn: uid=king,dc=example,dc=org
uid: king
userPassword:: e01ENX14TXBDT0tDNUk0SU56RkNhYjNXRW13PT0=
objectClass: account
objectClass: simpleSecurityObject
objectClass: top
```
``` python
from flask import Flask, g
from flask_simpleldap import LDAP

app = Flask(__name__)
app.secret_key = 'dev key'
app.debug = True

app.config['LDAP_OPENLDAP'] = True
app.config['LDAP_BASE_DN'] = 'dc=example,dc=org'
app.config['LDAP_USERNAME'] = 'cn=admin,dc=example,dc=org'
app.config['LDAP_PASSWORD'] = 'pwd'
app.config['LDAP_USER_OBJECT_FILTER'] = '(&(objectclass=simpleSecurityObject)(userid=%s))'

ldap = LDAP(app)

@app.route('/')
@ldap.basic_auth_required
def index():
    return 'Welcome, {0}!'.format(g.ldap_username)

if __name__ == '__main__':
    app.run(port=5010)
```

浏览器访问<http://127.0.0.1:5010/>, 即弹出对话框,输入`king`, `1` 登录成功

+ <https://github.com/admiralobvious/flask-simpleldap/tree/master/examples> 有若干sample可供参考
+ <https://medium.com/@chris.pruitt15/ldap-user-authentication-in-flask-api-e662b0aeb7de>

具体原理比较简单

+ 先使用环境变量配置的帐号密码(管理员)登录
  - `LDAP_USERNAME`/`LDAP_PASSWORD`
  - 登录见`simple_bind_s`
+ 根据输入的uid以及`LDAP_USER_OBJECT_FILTER`来搜索用户
+ 找到用户, 获取完整的`DN`
+ 使用DN和输入的密码再登录, 确认密码有效性


# Golang

<https://github.com/go-ldap/ldap/blob/master/examples_test.go> 有较详细的sample

> 如果`gopkg.in/asn1-ber.v1` 安装不上, 修改`go.mod`, 添加以下内容, 重新`tidy`

```
replace (
	gopkg.in/asn1-ber.v1 => github.com/go-asn1-ber/asn1-ber v0.0.0-20181015200546-f715ec2f112d
)
```

``` go
package main

import (
	"fmt"

	"github.com/go-ldap/ldap"
)

func main() {
	conn, err := ldap.Dial("tcp", "127.0.0.1:389")
	if err != nil {
		panic(err)
	}
	defer conn.Close()

	if err := conn.Bind("cn=admin,dc=example,dc=org", "pwd"); err != nil {
		panic(err)
	}

	searchRequest := ldap.NewSearchRequest(
		"dc=example,dc=org", // The base dn to search
		ldap.ScopeWholeSubtree, ldap.NeverDerefAliases, 0, 0, false,
		"(&(uid=king))", // The filter to apply
		[]string{},      // A list attributes to retrieve
		nil,
	)

	sr, err := conn.Search(searchRequest)
	if err != nil {
		panic(err)
	}

	for _, entry := range sr.Entries {
		fmt.Printf("%s: %v\n", entry.DN, entry.GetAttributeValue("cn"))
	}
	for _, entry := range sr.Referrals {
		fmt.Printf("%s: %v\n", entry)
	}
}

```

# 数据安全
缺省的[mdb数据存储](https://lmdb.readthedocs.io/en/release/)在`/var/lib/ldap/`, 一个数据文件,一个锁文件, 我们可以外挂docker数据目录(`/var/lib/ldap`)和配置目录(`/etc/ldap/slapd.d`)实现数据的持久化保存

``` python
import lmdb

# 注意传入的路径是一个目录, 下面包含两文件
env_db = lmdb.Environment('/home/user/mdb')

txn = env_db.begin()
for key, value in txn.cursor():  # 遍历
    print(key)

print("-")
data = txn.get("cn".encode("utf8"))
env_db.close()
```


## 安装问题
+ [I can't install python-ldap](https://stackoverflow.com/questions/4768446/i-cant-install-python-ldap)
+ [How to meet depends requirements for libldap-2.4-2](https://askubuntu.com/questions/815897/how-to-meet-depends-requirements-for-libldap-2-4-2)

``` bash
$ sudo aptitude install libsasl2-dev
The following NEW packages will be installed:
  libsasl2-dev{b} 
0 packages upgraded, 1 newly installed, 0 to remove and 5 not upgraded.
Need to get 254 kB of archives. After unpacking 831 kB will be used.
The following packages have unmet dependencies:
 libsasl2-dev : Depends: libsasl2-2 (= 2.1.26.dfsg1-14build1) but 2.1.26.dfsg1-14ubuntu0.1 is installed.
The following actions will resolve these dependencies:

     Keep the following packages at their current version:
1)     libsasl2-dev [Not Installed]                       



Accept this solution? [Y/n/q/?] n
The following actions will resolve these dependencies:

     Downgrade the following packages:                                                   
1)     libsasl2-2 [2.1.26.dfsg1-14ubuntu0.1 (now) -> 2.1.26.dfsg1-14build1 (xenial)]     
2)     libsasl2-2:i386 [2.1.26.dfsg1-14ubuntu0.1 (now) -> 2.1.26.dfsg1-14build1 (xenial)]



Accept this solution? [Y/n/q/?] y
The following packages will be DOWNGRADED:
  libsasl2-2 libsasl2-2:i386 
The following NEW packages will be installed:
  libsasl2-dev 
0 packages upgraded, 1 newly installed, 2 downgraded, 0 to remove and 5 not upgraded.
Need to get 354 kB of archives. After unpacking 831 kB will be used.
Do you want to continue? [Y/n/?] y
Get: 1 http://mirrors.aliyun.com/ubuntu xenial/main i386 libsasl2-2 i386 2.1.26.dfsg1-14build1 [51.8 kB]
Get: 2 http://mirrors.aliyun.com/ubuntu xenial/main amd64 libsasl2-2 amd64 2.1.26.dfsg1-14build1 [48.7 kB]
Get: 3 http://mirrors.aliyun.com/ubuntu xenial/main amd64 libsasl2-dev amd64 2.1.26.dfsg1-14build1 [254 kB]
Fetched 354 kB in 0s (384 kB/s)       
dpkg: warning: downgrading libsasl2-2:i386 from 2.1.26.dfsg1-14ubuntu0.1 to 2.1.26.dfsg1-14build1
(Reading database ... 530388 files and directories currently installed.)
Preparing to unpack .../libsasl2-2_2.1.26.dfsg1-14build1_i386.deb ...
De-configuring libsasl2-2:amd64 (2.1.26.dfsg1-14ubuntu0.1) ...
Unpacking libsasl2-2:i386 (2.1.26.dfsg1-14build1) over (2.1.26.dfsg1-14ubuntu0.1) ...
dpkg: warning: downgrading libsasl2-2:amd64 from 2.1.26.dfsg1-14ubuntu0.1 to 2.1.26.dfsg1-14build1
Preparing to unpack .../libsasl2-2_2.1.26.dfsg1-14build1_amd64.deb ...
Unpacking libsasl2-2:amd64 (2.1.26.dfsg1-14build1) over (2.1.26.dfsg1-14ubuntu0.1) ...
Selecting previously unselected package libsasl2-dev.
Preparing to unpack .../libsasl2-dev_2.1.26.dfsg1-14build1_amd64.deb ...
Unpacking libsasl2-dev (2.1.26.dfsg1-14build1) ...
Processing triggers for libc-bin (2.23-0ubuntu11) ...
Processing triggers for man-db (2.7.5-1) ...
Setting up libsasl2-2:amd64 (2.1.26.dfsg1-14build1) ...
Setting up libsasl2-2:i386 (2.1.26.dfsg1-14build1) ...
Setting up libsasl2-dev (2.1.26.dfsg1-14build1) ...
Processing triggers for libc-bin (2.23-0ubuntu11) ...
```



# 参考
1. [Docker安装OpenLDAP及PHPLdapAdmin客户端](https://blog.mylitboy.com/article/operation/docker-install-openldap/)
2. [使用OpenLDAP实现集中式认证](https://wiki.gentoo.org/wiki/Centralized_authentication_using_OpenLDAP/zh)
3. [openldap介绍和使用](https://www.cnblogs.com/woshimrf/p/ldap.html)

## 应用支持
+ <https://grafana.com/docs/grafana/v6.5/auth/ldap/>
+ <https://github.com/nginxinc/nginx-ldap-auth>
+ <https://github.com/Banno/getsentry-ldap-auth>
+ <https://docs.gitlab.com/ee/administration/auth/ldap.html>
+ <https://confluence.atlassian.com/adminjiraserver/connecting-to-an-ldap-directory-938847052.html>
+ <https://wiki.jenkins.io/display/JENKINS/LDAP+Plugin>
+ <https://resources.collab.net/blogs/subversion-ldap-authentication-with-apache>
+ <https://github.com/flowable/flowable-engine/tree/master/modules/flowable-ldap>
+ <https://www.zentao.net/extension-viewExt-76.html>
