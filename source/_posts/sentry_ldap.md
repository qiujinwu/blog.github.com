---
title: sentry配置ldap登录授权
date: 2020-02-04 00:00:00
tags:
 - openldap
categories:
 - Openldap
---

快速安装sentry见<http://blog.kimq.cn/2017/03/10/sentry/>

了解ldap见<http://blog.kimq.cn/2020/02/02/openldap/>

# 安装插件
进入docker
``` bash
$ docker exec -it my-sentry /bin/bash
# 最好先修改源
$ apt update
$ apt-get install libsasl2-dev python-dev libldap2-dev libssl-dev 
# for debug
$ apt-get vim procps -y
# 安装sentry插件
pip install https://github.com/Banno/getsentry-ldap-auth/archive/2.8.1.zip
```

> `getsentry-ldap-auth`的版本自行评估

## Debug
``` bash
/usr/local/lib/python2.7/site-packages/sentry_ldap_auth$ find . -type f
./backend.py
./__init__.py
```

若有问题, 可自行修改代码, 然后重启docker镜像, 日志查看
``` bash
$ docker logs -f my-sentry
```

# 配置ldap群组和帐号

两个群组 
+ `owner_group` 对应sentry的owner角色
+ `admin_group` 对应sentry的admin角色

---

`admin`属于`owner_group`, `king1`属于`admin_group`


``` bash
$ ldapsearch -x -H ldap://localhost  -w 123456 -D cn=admin,ou=tech,ou=lixin,dc=example,dc=org \
   -b "ou=lixin,dc=example,dc=org"
# admin_group, lixin, example.org
dn: cn=admin_group,ou=lixin,dc=example,dc=org
objectClass: groupOfNames
objectClass: top
cn: admin_group
member: cn=king1,ou=tech,ou=lixin,dc=example,dc=org

# owner_group, lixin, example.org
dn: cn=owner_group,ou=lixin,dc=example,dc=org
objectClass: groupOfNames
objectClass: top
cn: owner_group
member: cn=admin,ou=tech,ou=lixin,dc=example,dc=org

# admin, tech, lixin, example.org
dn: cn=admin,ou=tech,ou=lixin,dc=example,dc=org
objectClass: top
objectClass: organizationalRole
objectClass: simpleSecurityObject
cn: admin
userPassword:: e1NTSEF9ekRETXFGME4ydVNHUWZzRmxZSVlKVmVoS3BWbENxaGk=

# king1, tech, lixin, example.org
dn: cn=king1,ou=tech,ou=lixin,dc=example,dc=org
objectClass: top
objectClass: organizationalRole
objectClass: simpleSecurityObject
cn: king1

# king2, tech, lixin, example.org
dn: cn=king2,ou=tech,ou=lixin,dc=example,dc=org
objectClass: top
objectClass: organizationalRole
objectClass: simpleSecurityObject
cn: king2
```

# 配置Sentry服务器

将下列配置附到`/etc/sentry/sentry.conf.py` 后面

``` python
#################################################################
# ldap auth
import ldap
from django_auth_ldap.config import LDAPSearch, GroupOfNamesType

# ldap服务器地址
AUTH_LDAP_SERVER_URI = 'ldap://172.17.0.1'
# 具有查询权限的内部帐号
AUTH_LDAP_BIND_DN = 'cn=admin,ou=tech,ou=lixin,dc=example,dc=org'
# 密码
AUTH_LDAP_BIND_PASSWORD = '123456'

# 用户登录的过滤条件, 我们输入的帐号是ldap的`cn`
# @attention user占位符别修改
AUTH_LDAP_USER_SEARCH = LDAPSearch(
            'ou=tech,ou=lixin,dc=example,dc=org',
            ldap.SCOPE_SUBTREE,
            '(cn=%(user)s)',
            )

# 查找组的过滤条件
AUTH_LDAP_GROUP_SEARCH = LDAPSearch(
            'ou=lixin,dc=example,dc=org',
            ldap.SCOPE_SUBTREE,
            '(objectClass=groupOfNames)'
            )

# GroupOfNamesType
# NestedGroupOfNamesType
# GroupOfUniqueNamesType
# NestedGroupOfUniqueNamesType
# ActiveDirectoryGroupType
# NestedActiveDirectoryGroupType
# OrganizationalRoleGroupType
# NestedOrganizationalRoleGroupType
AUTH_LDAP_GROUP_TYPE = GroupOfNamesType()

# 如果设置了AUTH_LDAP_REQUIRE_GROUP那么只有这个组的成员才会登录成功
AUTH_LDAP_REQUIRE_GROUP = None
# AUTH_LDAP_DENY_GROUP正好相反，这个组的用户登录都会被拒绝。
AUTH_LDAP_DENY_GROUP = None

# 用于映射sentry用户字段, 因为ldap测试的对象比较简单, 这里全部都用`cn`
AUTH_LDAP_USER_ATTR_MAP = {
        'name': 'cn',
        'email': 'cn'
        }

AUTH_LDAP_FIND_GROUP_PERMS = True
AUTH_LDAP_CACHE_GROUPS = True
AUTH_LDAP_GROUP_CACHE_TIMEOUT = 3600

# Sentry缺省的Organization, 填写错误会导致登录后无权限, 一般不用修改
AUTH_LDAP_DEFAULT_SENTRY_ORGANIZATION = u'Sentry'
# 自动加入到这个缺省的群组
AUTH_LDAP_SENTRY_SUBSCRIBE_BY_DEFAULT = True
# 配置哪些群组里的用户,具有哪个角色
# 例如owner_group 这个群组里的用户具有owner角色
AUTH_LDAP_SENTRY_GROUP_ROLE_MAPPING = {
        'owner': ['owner_group'],
        'admin': ['admin_group'],
        'member': [], # 缺省, 这里就不用配置了
        }
# 缺省角色(如果不属于我们配置的群组)
AUTH_LDAP_SENTRY_ORGANIZATION_ROLE_TYPE = 'member'
#
AUTH_LDAP_SENTRY_ORGANIZATION_GLOBAL_ACCESS = True
#
AUTH_LDAP_SENTRY_USERNAME_FIELD = 'cn'

# 启用ldap登录
AUTHENTICATION_BACKENDS = AUTHENTICATION_BACKENDS + (
        'sentry_ldap_auth.backend.SentryLdapBackend',
        )

# 日志相关
import logging
logger = logging.getLogger('django_auth_ldap')
logger.addHandler(logging.StreamHandler())
logger.setLevel('DEBUG')
```

> `AUTH_LDAP_SENTRY_GROUP_ROLE_MAPPING` 字典的value值, 只需要写对应群组的`RDN`即可, 无需填写完整的`DN`

如果一个用户分别属于多个群组, 那么系统会取登记最高的角色, 角色优先级如下(由低到高)
``` python
def _get_effective_sentry_role(group_names):
    role_priority_order = [
        'member',
        'admin',
        'manager',
        'owner',
    ]
```


# 登录验证
打开<http://127.0.0.1:8080> 然后分别用`admin`, `king1`, `king2` 登录, 在member页面<http://127.0.0.1:8080/settings/sentry/members/> 可以看到三个用户属于不同的角色

# 参考
1. <https://github.com/Banno/getsentry-ldap-auth>
2. <https://django-auth-ldap.readthedocs.io/en/latest/reference.html>
3. <https://darkcooking.gitbooks.io/django-auth-ldap/content/chapter4.html>


