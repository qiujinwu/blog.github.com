---
title: 证书与Nginx部署
date: 2017-07-14 21:04:43
tags:
 - 证书
 - 签名
 - nginx
categories:
 - 后端
---

# 证书类别
按照审核方式，一般的CA机构提供三种类型的SSL证书：域名型SSL(DV SSL)，企业型SSL(OV SSL)以及增强型SSL(EV SSL)证书。

三种证书底层的技术实现是一致的，区别在于证书发布方对申请方的审核程度，在证书的**颁发对象**中有很多字段，比如
1. CN Common Name
2. OU Organization Unit
3. O Organization Name
4. L Locality
5. S State/Provice
6. C Country

一个典型的EV证书颁发对象（工商银行）,可以看到各种字段非常详尽
``` bash
CN = corporbank-simp.icbc.com.cn
OU = Software Development Center
O = Industrial and Commercial Bank of China Limited
STREET = NO.55 Fuxingmen Nei Street Xicheng District, Beijing
L = Beijing
S = Beijing
PostalCode = 100140
C = CN
SERIALNUMBER = 100000000003965
2.5.4.15 = Private Organization
1.3.6.1.4.1.311.60.2.1.2 = BEIJING
1.3.6.1.4.1.311.60.2.1.3 = CN
```

百度的OV证书
``` bash
CN = baidu.com
OU = service operation department.
O = BeiJing Baidu Netcom Science Technology Co., Ltd
L = beijing
S = beijing
C = CN
```

在腾讯云上申请的DV证书，就剩下可怜的一行，这种免费证书不支持多域名和泛域名（比如*.qiujinwu.com），只能写死某个域名。
``` bash
CN = ycy.qiujinwu.com
```


[关于证书的支持范围](http://www.barretlee.com/blog/2016/04/24/detail-about-ca-and-certs/)

|   | 单域名 | 多域名 | 泛域名 | 多泛域名 |
|--------|--------|--------|--------|--------|
|DV|    支持    | 支持 | 不支持 | 不支持|
|OV|    支持    | 支持 | 支持 | 支持|
|EV|    支持    | 支持 | 不支持 | 不支持|
|举例|www.barretlee.com | www.barretlee.com <br/>www.xiaohuzige.com <br/>www.barret.cc | *.barretlee.com | *.barretlee.com <br/> *.xiaohuzige.com <br/>*.barret.cc|

## 配置Nginx

配置服务器证书即可，实现Https的访问，对于上面的免费DV证书，就可以实现全网的可信任Https

``` nginx
server {
    listen       443 ssl;
    root /var/www/html;
    ssl on;
    ssl_certificate /tmp/a/server.crt;
    ssl_certificate_key /tmp/a/server.key;

    ssl_session_cache    shared:SSL:1m;
    ssl_session_timeout  5m;

    ssl_ciphers  HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers  on;
}
```

# 自建CA
证书支持一个树状的层级认证链关系，例如Google首页的证书
``` bash
Builtin Object Token:GeoTrust Global CA
  Google Internet Authority G2
    *.google.com.hk
```
按照通行的做法，会建多级ca，一个root ca和若干个和若干层intermediate ca，然后再一堆的服务器/客户端证书，此内容重点参考<http://www.barretlee.com/blog/2016/04/24/detail-about-ca-and-certs/>

## 工具shell
由于创建公私钥，证书、签名的过程需要输入密码等各种参数，为了方便测试，下面的脚本进行了封装，支持批量化运行
``` bash
#!/bin/bash
# 常用后缀
key=".key" # 私（公）钥
crt=".crt" # 证书
csr=".csr" # 证书签名请求
p12=".p12" # 打包了私钥的证书

# 创建私（公）钥
create_key(){
    # 加密方式，可以但不限这些，
    # aes256/des3/aes128

    # 2048 可以作为参数，亦可调整大小
    name="${1}${key}"
    if [ "${#}" -gt 1 ];then
        openssl genrsa -aes256 -passout pass:${2} -out "${name}" 2048
    else
    	# 不填写参数则无密码
        openssl genrsa -out "${name}" 2048
    fi
}

# 移除私（公）钥的密码
remove_key_pwd(){
    # notify passin & passout
    name="${1}${key}"
    if [ "${#}" -gt 1 ];then
        openssl rsa -in "${name}" -passin pass:"${2}" -out "${name}"
    else
        openssl rsa -in "${name}" -out "${name}"
    fi
}

# 输出证书详情
check_crt(){
    openssl x509 -noout -text -in "${1}${crt}"
}

# 基于测试，SUJECT作为一个全局的参数存在，除了Common Name
C="CN"
ST="GD"
L="SZ"
O="0MyCompany"
OU="Department"

# 自签名
self_sign(){
	# SUJECT 的Common Name
    common_name=$1
    output=$2
    # 是否输入了密码
    password_param=""
    if [ "${#}" -gt 2 ];then
        password_param="-passin pass:${3}"
    fi

    SUBJECT="/C=${C}/ST=${ST}/L=${L}/O=${O}/OU=${OU}/CN=${common_name}"
    # 留意-config -extensions
    openssl req -config `pwd`/ca_openssl.cnf \
        ${password_param} -subj ${SUBJECT}  \
        -extensions v3_ca \
        -key "${output}${key}" -new -x509 \
        -days 7200 -sha256 -out "${output}${crt}"
}

# 创建证书签名请求
create_csr(){
	# SUJECT 的Common Name
    common_name=$1
    output=$2
    # 和函数self_sign一样
    SUBJECT="/C=${C}/ST=${ST}/L=${L}/O=${O}/OU=${OU}/CN=${common_name}"
    # 是否输入了密码
    password_param=""
    if [ "${#}" -gt 2 ];then
        password_param="-passin pass:${3}"
    fi
    # 留意-config -extensions
    openssl req -config `pwd`/imm_openssl.cnf ${password_param} -new -sha256 -subj ${SUBJECT} \
        -key "${output}${key}" -out "${output}${csr}"
}

# ca签名，包括root ca给intermediate签名，以及intermediate ca和业务证书签名
ca_sign(){
    input=$1
    cafile=$2
    # 是否输入了密码
    password_param=""
    if [ "${#}" -gt 2 ];then
        password_param="-passin pass:${3}"
    fi

    extensionsparam=""
    cnfparam="-config `pwd`/imm_openssl.cnf"
    if [ "${cafile}" == "ca" ];then
    	# root ca签名的情形，使用ca_openssl.cnf，以及指定extensions
        cnfparam="-config `pwd`/ca_openssl.cnf"
        extensionsparam="-extensions v3_intermediate_ca"
    fi

    # -batch without （Y/N）prompt
    openssl ca ${cnfparam} ${password_param} ${extensionsparam} \
        -batch -days 3600 -notext -md sha256 \
        -in "${input}${csr}" -out "${input}${crt}" #\
        # 由于使用了conf指定ca，这些参数是多余的
        #-cert "${cafile}${crt}" -keyfile "${cafile}${key}"
}

# 验证证书
verify_crt(){
    input=${1}
    cafile=${2}
    openssl verify -CAfile "${cafile}${crt}" "${input}${crt}"
}

# 将证书和私钥打包成p12格式
export_p12() {
    input=${1}
    opassword=${2}
    # 是否输入了密码
    password_param=""
    if [ "${#}" -gt 2 ];then
        password_param="-passin pass:${3}"
    fi
    openssl pkcs12 ${password_param} -passout pass:${opassword} \
        -export -clcerts -in "${input}${crt}" \
        -inkey "${input}${key}" -out "${input}${p12}"
}
```

由于不同的ca有不同的会话环境，添加工具函数
``` bash
create_conf(){
    rdir="${1}"
    # 生成环境应该考虑相对路径，以及安全性的问题
    if [ "${rdir}" == "/" ];then
        return
    else
        rm -rf "${rdir}"
    fi

    mkdir -p "${rdir}"
    cd "${rdir}"
    mkdir -p certs crl csr newcerts private
    touch index.txt
    echo 1000 > serial
    cd -
}
```

### 创建证书
``` bash
# 全局的密码，方便测试
pass=123456
# ca chain，方便@1批量导入@2nginx需要
cachain="cachain"

# 创建证书（含intermediate证书和业务证书）并签名
create_and_sign(){
	# 输入文件，不含后缀
    output=${1}
    # common name
    cname=${2}
    # ca文件名称，不含后缀
    caname=${3}
    # 创建证书私钥
    create_key ${output} ${pass}
    # 创建证书签名请求
    create_csr ${cname} ${output} ${pass}
    # 用指定的ca进行签名
    ca_sign ${output} ${caname} ${pass}
    # 打印信息
    check_crt ${output}
    # 校验
    verify_crt ${output} ${cachain}
    # 导出p12证书
    export_p12 ${output} ${pass} ${pass}

    if [ ${caname} = "ca" ];then
    	# intermediate证书，需要更新cachain并且放置在最签名，rootca在最底部（注意顺序）
        tmpfile="/tmp/$$_tmp_$$"
        cat "${output}${crt}" | cat - "${cachain}${crt}" > "${tmpfile}" \
            && mv "${tmpfile}" "${cachain}${crt}"
    else
    	# 业务证书移除密码
        remove_key_pwd "${output}" ${pass}
    fi
}

# 在这里创建一个root ca和intermediate ca
# init两个ca的环境，置于如何关联到这两个目录，在conf文件中配置
create_conf ca
create_conf imm

# 创建root ca的私钥
create_key ca ${pass}
# 自签名
self_sign root_ca ca ${pass}
check_crt ca
# 更新到cachain
cat "ca${crt}" > "${cachain}${crt}"

# 创建intermediate ca 证书
create_and_sign intermediate intermediate_ca ca
# 创建服务器证书，common name必须和域名匹配
create_and_sign server "ycy.qiujinwu.com" intermediate
# 创建客户证书，用于作客户端认证（双向认证）
create_and_sign kingqiu "kingqiu" intermediate
```

### [cnf配置](http://www.barretlee.com/blog/2016/04/24/detail-about-ca-and-certs/)
上述的脚本有两个cnf文件，分别是ca_openssl.cnf和imm_openssl.conf,原始文件在<https://github.com/barretlee/autocreate-ca>
``` bash
wget -O ca_openssl.cnf \
	https://raw.githubusercontent.com/barretlee/autocreate-ca/master/cnf/root-ca
wget -O imm_openssl.cnf \
	https://raw.githubusercontent.com/barretlee/autocreate-ca/master/cnf/intermediate-ca
```

两者的区别
``` diff
king@king:/tmp/c$ diff *.cnf
1,2c1,2
< # OpenSSL root CA configuration file.
< # Copy to `/root/ca/openssl.cnf`.
---
> # OpenSSL intermediate CA configuration file.
> # Copy to `/root/ca/intermediate/openssl.cnf`.
10c10
< dir               = /root/ca
---
> dir               = /root/ca/intermediate
19,20c19,20
< private_key       = $dir/private/ca.key.pem
< certificate       = $dir/certs/ca.cert.pem
---
> private_key       = $dir/private/intermediate.key.pem
> certificate       = $dir/certs/intermediate.cert.pem
24c24
< crl               = $dir/crl/ca.crl.pem
---
> crl               = $dir/crl/intermediate.crl.pem
35c35
< policy            = policy_strict
---
> policy            = policy_loose
```

这些配置和运行上述脚本的路径，以及文件名相关，我做了如下修改
``` diff
king@king:/tmp/b$ diff ca_openssl.cnf imm_openssl.cnf 
1,2c1,2
< # OpenSSL root CA configuration file.
< # Copy to `/root/ca/openssl.cnf`.
---
> # OpenSSL intermediate CA configuration file.
> # Copy to `/root/ca/intermediate/openssl.cnf`.
10c10
< dir               = /tmp/b/ca
---
> dir               = /tmp/b/imm
19,20c19,20
< private_key       = $dir/../ca.key
< certificate       = $dir/../ca.crt
---
> private_key       = $dir/../intermediate.key
> certificate       = $dir/../intermediate.crt
24c24
< crl               = $dir/ca.crl.pem
---
> crl               = $dir/intermediate.crl.pem
35c35
< policy            = policy_strict
---
> policy            = policy_loose
```

注意：
1. CA_default/dir的路径和证书的路径匹配
2. private_key和certificate的路径正确

``` ini
[ CA_default ]
# Directory and file locations.
dir               = /tmp/b/ca
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/.rand

# The root key and root certificate.
private_key       = $dir/../ca.key
certificate       = $dir/../ca.crt

# For certificate revocation lists.
crlnumber         = $dir/crlnumber
crl               = $dir/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30
```

运行之后，生成如下文件

>.
├── ca
│   ├── certs
│   ├── crl
│   ├── csr
│   ├── index.txt
│   ├── index.txt.attr
│   ├── index.txt.old
│   ├── newcerts
│   ├── private
│   ├── serial
│   └── serial.old
├── cachain.crt
├── ca.crt
├── ca.key
├── ca_openssl.cnf
├── imm
│   ├── certs
│   ├── crl
│   ├── csr
│   ├── index.txt
│   ├── index.txt.attr
│   ├── index.txt.attr.old
│   ├── index.txt.old
│   ├── newcerts
│   ├── private
│   ├── serial
│   └── serial.old
├── imm_openssl.cnf
├── intermediate.crt
├── intermediate.csr
├── intermediate.key
├── intermediate.p12
├── kingqiu.crt
├── kingqiu.csr
├── kingqiu.key
├── kingqiu.p12
├── server.crt
├── server.csr
├── server.key
└── server.p12

## 配置Nginx服务器证书
``` nginx
server {
    listen       443 ssl;
    root /var/www/html;
    ssl on;
    ssl_certificate /tmp/b/server.crt;
    ssl_certificate_key /tmp/b/server.key;
}
```

添加Host之后即可访问<https://ycy.qiujinwu.com/index.nginx-debian.html>，由于是自建ca，浏览器是不信任的

### Linux下添加信任
chrome和opera打开的界面都是一样的，并且数据互通。打开【证书管理器】之后，打开tab【授权中心】，选择【cachain.crt】，就一次性将root ca和intermediate ca一并导入了（必须都导入，否则无法通过信任，业务证书也无法看到完整的证书链），

### Windows下添加信任
双击ca.crt和intermediate.ca依次安装到【受信任的根证书颁发机构】

再次刷新，即可看到锁变绿了，chrome有时需要关闭重新打开才能信任。

## 双向认证
注意nginx客户端的ca证书，**使用的是cachain.crt证书链**
``` nginx
server {
    listen       443 ssl;
    root /var/www/html;
    ssl on;
    ssl_certificate /tmp/b/server.crt;
    ssl_certificate_key /tmp/b/server.key;

    ssl_verify_client on;
    ssl_verify_depth 2;
    ssl_client_certificate /tmp/b/cachain.crt;
    proxy_set_header X-SSL-Client-Cert $ssl_client_cert;
}
```

Linux下，打开【证书管理器】之后，打开tab【您的证书】，选择【kingqiu.p12】，输入密码，导入证书

Windows下，直接双击kingqiu.p12导入到默认的位置即可。

接下来再次访问就会弹出对话框选择证书，若无匹配的证书，就报错【400 Bad Request。No required SSL certificate was sent】


## 典型的问题

提示O/S等不匹配

``` bash
Using configuration from /etc/pki/tls/openssl.cnf
Check that the request matches the signature
Signature ok
The countryName field needed to be the same in the
CA certificate (NL) and the request (RO)
```

king@king:/tmp/a$ sudo vi /etc/ssl/openssl.cnf
``` bash
# For the CA policy
[ policy_match ]
countryName     = optional
stateOrProvinceName = optional
organizationName    = optional
#countryName        = match
#stateOrProvinceName    = match
#organizationName   = match
organizationalUnitName  = optional
commonName      = supplied
emailAddress        = optional
```

无法作为ca加入信任的授权中心

这通常是v3 extension的**CA flag**未启用或者不存在
``` bash
king@king:/tmp/c$ openssl x509 -noout -text -in intermediate.crt 
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 4096 (0x1000)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=CN, ST=fasdf, OU=R&D, CN=kingqiu
        Validity
            Not Before: Jul 14 10:00:14 2017 GMT
            Not After : May 23 10:00:14 2027 GMT
        Subject: C=CN, ST=afdasf, OU=R&D, CN=intermediate_ca
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:c0:81:d1:fb:2b:b8:0f:b6:8b:30:04:46:d0:78:
                    28:8d
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier: 
                0D:6A:86:F1:62:AE:6A:74:05:12:31:09:BD:25:3B:B6:4E:98:BF:8E
            X509v3 Authority Key Identifier: 
                keyid:E1:79:75:26:9D:C9:B4:FF:03:77:F9:5D:2F:BD:CB:C7:C6:DE:2A:20

            X509v3 Basic Constraints: critical
            	# 注意这里
                CA:TRUE, pathlen:0
            X509v3 Key Usage: critical
                Digital Signature, Certificate Sign, CRL Sign
    Signature Algorithm: sha256WithRSAEncryption
         ce:79:c0:61:5c:39:66:f2:cf:30:34:ee:8b:7c:e1:f5:24:53:
         1e:d7:cf:ed

```

