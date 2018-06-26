---
title: 腾讯Tars初探
date: 2018-06-26 20:25:07
tags:
 - tars
categories:
 - 源码学习
---

官网

1. <https://github.com/Tencent/Tars>
2. <https://github.com/tars-node>

本文重点参考

1. <https://github.com/Tencent/Tars/blob/master/Install.md>
2. <https://github.com/Tencent/Tars/blob/master/docs/tars_cpp_quickstart.md>

# 环境准备
下载代码、apt-get安装依赖

``` bash
git clone git@github.com:Tencent/Tars.git
apt-get install flex bison libmysqlclient-dev
```

安装第三方库
``` bash
${TARS_ROOT}/Tars/cpp/thirdparty/thirdparty.sh
```

实际就下载一个`rapidjson`
``` bash
king@king:~/Tars/cpp/thirdparty$ tree -L 2
.
├── rapidjson
│   ├── appveyor.yml
│   ├── bin
│   ├── CHANGELOG.md
│   ├── CMakeLists.txt
│   ├── CMakeModules
│   ├── contrib
│   ├── doc
│   ├── docker
│   ├── example
│   ├── include
│   ├── include_dirs.js
│   ├── library.json
│   ├── license.txt
│   ├── package.json
│   ├── rapidjson.autopkg
│   ├── RapidJSONConfig.cmake.in
│   ├── RapidJSONConfigVersion.cmake.in
│   ├── RapidJSON.pc.in
│   ├── readme.md
│   ├── readme.zh-cn.md
│   ├── test
│   ├── thirdparty
│   └── travis-doxygen.sh
└── thirdparty.sh
```

#  编译核心组件/开发库
``` bash
cd ${TARS_ROOT}/Tars/cpp/build
chmod u+x build.sh
./build.sh all
```
需要重来，则
``` bash
./build.sh cleanall
```

由于ubuntu的mysql库路径和tars默认配置不一致，所以需要在编译前修改CMAKE配置,`${TARS_ROOT}/Tars/cpp/build/CmakeLists.txt`
``` bash
set(MYSQL_DIR_INC "/usr/include/mysql")
set(MYSQL_DIR_LIB "/usr/lib/x86_64-linux-gnu/")
```

## 安装开发库
编译成功之后
``` bash
cd /usr/local
sudo mkdir tars
sudo chown king:king ./tars/  # king酌情修改
```
``` bash
cd ${TARS_ROOT}/Tars/cpp/build
./build.sh install
```

文件内容如下，主要用于Cpp开发（头文件/库）

``` bash
king@king:/usr/local/tars$ tree -L 3
.
└── cpp
    ├── include
    │   ├── jmem
    │   ├── promise
    │   ├── servant
    │   ├── tup
    │   └── util
    ├── lib
    │   ├── libtarsparse.a
    │   ├── libtarsservant.a
    │   └── libtarsutil.a
    ├── makefile
    │   └── makefile.tars
    ├── script
    │   ├── create_http_server.sh
    │   ├── create_tars_server.sh
    │   ├── demo
    │   └── http_demo
    └── tools
        ├── tars2android
        ├── tars2c
        ├── tars2cpp
        ├── tars2cs
        ├── tars2node
        ├── tars2oc
        ├── tars2php
        └── tars2python
```

## 安装基础服务
``` bash
cd ${TARS_ROOT}/Tars/cpp/build
make framework-tar
```

会生成一个`framework.tgz`

``` bash
cd /usr/local/app
sudo mkdir tars
sudo chown king:king ./tars/  # king酌情修改
```
``` bash
cd ${TARS_ROOT}/Tars/cpp/build
cp ./framework.tgz /usr/local/app/tars/
cd /usr/local/app/tars
tar xzfv framework.tgz
```

内容如下

``` bash
king@king:/usr/local/app/tars$ tree -L 2
.
├── framework.tgz
├── tarsAdminRegistry
│   ├── bin
│   ├── conf
│   └── util
├── tarsconfig
│   ├── bin
│   ├── conf
│   ├── data
│   └── util
├── tars_install.sh
├── tarsnode
│   ├── bin
│   ├── conf
│   ├── data
│   ├── tmp
│   └── util
├── tarsnode_install.sh
├── tarspatch
│   ├── bin
│   ├── conf
│   ├── data
│   └── util
└── tarsregistry
    ├── bin
    ├── conf
    ├── data
    └── util
```

# 运行核心基础服务
## 准备Mysql
安装之后
``` sql
CREATE DATABASE IF NOT EXISTS db_tars default charset utf8 COLLATE utf8_general_ci;
CREATE DATABASE IF NOT EXISTS tars_stat default charset utf8 COLLATE utf8_general_ci;
CREATE DATABASE IF NOT EXISTS tars_property default charset utf8 COLLATE utf8_general_ci;
```

修改IP。`这里的IP可用域名`

``` bash
cd ${TARS_ROOT}/Tars/cpp/framework/sql
sed -i "s/192.168.2.131/192.168.10.6/g" `grep 192.168.2.131 -rl ./*`
sed -i "s/db.tars.com/server/g" `grep db.tars.com -rl ./*`
sed -i "s/tars2015/password/g" `grep tars2015 -rl ./*`
sed -i "s/dbuser=tars/dbuser=user/g" `grep dbuser=tars -rl ./*`
```

创建表
``` bash
king@king:/usr/local/app/tars$ mysql -uu -pp -hserver db_tars # user/password酌情修改
mysql> source ./cpp/framework/sql/db_tars.sql
```

若想使用默认的帐号密码
``` bash
CREATE USER 'tars'@'%' IDENTIFIED BY 'tars2015';
GRANT ALL ON db_tars.* TO 'tars'@'%';
GRANT ALL ON tars_stat.* TO 'tars'@'%';
GRANT ALL ON tars_property.* TO 'tars'@'%';
```

## 修改服务配置
### 修改IP地址

因为个人的Mysql没有在本机，所以单独处理

> `注意原来为IP的地方，不能用域名，否则会bind失败`（其实可以改进一下）


``` bash
export your_machine_ip=192.168.10.6
sed -i "s/192.168.2.131/${your_machine_ip}/g" `grep 192.168.2.131 -rl ./*`
sed -i "s/registry.tars.com/${your_machine_ip}/g" `grep registry.tars.com -rl ./*`
sed -i "s/web.tars.com/${your_machine_ip}/g" `grep web.tars.com -rl ./*`
```
``` bash
export your_machine_ip=server
sed -i "s/db.tars.com/${your_machine_ip}/g" `grep db.tars.com -rl ./*`
```

### 修改Mysql帐号密码
``` bash
king@king:/usr/local/app/tars$ grep dbuser -rl ./* | grep ".conf"
./tarsAdminRegistry/conf/adminregistry.conf
./tarsconfig/conf/tarsconfig.conf
./tarsconfig/bin/tarsconfig
./tarsregistry/conf/tarsregistry.conf
```

Mysql帐号密码通过下列的配置指定

``` xml
<db>
    charset=utf8
    dbhost=server
    dbname=db_tars
    dbpass=p
    dbport=3306
    dbuser=u
</db>
```

## 运行
``` bash
Cd /usr/local/app/tars
chmod u+x tars_install.sh
# 运行基础组件（共五个进程）
./tars_install.sh
# 运行rsync
sudo ./tarspatch/util/init.sh
```

进程如下

``` bash
/usr/local/app/tars/tarsregistry/bin/tarsregistry --config=/usr/local/app/tars/tarsregistry/conf/tarsregistry.conf
/usr/local/app/tars/tarsAdminRegistry/bin/tarsAdminRegistry --config=/usr/local/app/tars/tarsAdminRegistry/conf/adminregistry.conf
/usr/local/app/tars/tarsnode/bin/tarsnode --locator=tars.tarsregistry.QueryObj@tcp -h 192.168.10.6 -p 17890 --config=/usr/local/app/tars/tarsnode/conf/tarsnode.conf
/usr/local/app/tars/tarsconfig/bin/tarsconfig --config=/usr/local/app/tars/tarsconfig/conf/tarsconfig.conf
/usr/local/app/tars/tarspatch/bin/tarspatch --config=/usr/local/app/tars/tarspatch/conf/tarspatch.conf
```

# 运行前端
## 准备Java client

``` bash
cd ${TARS_ROOT}/Tars/java
mvn clean install 
mvn clean install -f core/client.pom.xml

# 这一步不需要，应该是用于Java开发的，类似于C++的include/lib
mvn clean install -f core/server.pom.xml
```

编译之后

``` bash
king@king:~/Tars/java$ ls ~/.m2/repository/com/tencent/tars/tars-client/
1.4.0  maven-metadata-local.xml
```

## 编译前端

切换到目录`${TARS_ROOT}/Tars/web/src/main/resources`

修改 `app.config.properties`
``` bash
# server为mysql地址
tarsweb.datasource.tars.addr=server:3306
tarsweb.datasource.tars.user=u
tarsweb.datasource.tars.pswd=p
```

maven打包
``` bash
mvn clean package
```

增加`/etc/hosts` （注意不是127.0.0.1）

``` bash
192.168.10.6 registry1.tars.com                                                    
192.168.10.6 registry2.tars.com
```

若找不到依赖 qq-cloud-central

``` bash
king@king:~/Tars/web$ diff pom.xml pom.xml.bak 
diff pom.xml.bak pom.xml
82c82
< 			<groupId>qq-cloud-central</groupId>
---
> 			<groupId>com.tencent.tars</groupId>
84c84
< 			<version>1.0.3</version>
---
> 			<version>1.4.0</version>
```

## resin运行
下载<http://caucho.com/download/resin-4.0.56.tar.gz> 解压到`/usr/local/app/resin`

``` bash
cd ${TARS_ROOT}/Tars/web/
cp ./target/tars.war /usr/local/app/resin/webapps/
```

修改conf/resin.xml

``` xml
<host id="" root-directory=".">
    <--  <web-app id="/" root-directory="webapps/ROOT"/> -->
    <web-app id="/" document-directory="webapps/tars"/>
</host>
```

``` bash
mkdir -p /data/log/tars
/usr/local/app/resin/bin/resin.sh start
```

进程如下

``` bash
/usr/lib/jvm/jdk1.8.0_111/bin/java -Dresin.watchdog=app-0 -Djava.util.logging.manager=com.caucho.log.LogManagerIm
     usr/lib/jvm/jdk1.8.0_111/bin/java -Dresin.server=app-0 -Djava.util.logging.manager=com.caucho.log.LogManager
```

## 发布可选组件
``` bash
# 其他服务
make tarsstat-tar
make tarsnotify-tar
make tarsproperty-tar
make tarslog-tar
make tarsquerystat-tar
make tarsqueryproperty-tar
```

安装framework之后，tarsnotify会跑步起来，需要将上面步骤发布的tarsnotify-tar发布上去
	
接下来可以将剩余的5个也发布上去


#  Cpp Hello world
生成server

```
/usr/local/tars/cpp/script/create_tars_server.sh TestApp HelloServer Hello
```

在Ubuntu有报错，rename不了，参见文末[问题]集锦

生成接口头文件

``` bash
/usr/local/tars/cpp/tools/tars2cpp Hello.tars
make
make tar
```

make tar 会生成.tgz包，用于上传发布到tars平台

可以按照[tars_cpp_quickstart](https://github.com/Tencent/Tars/blob/master/docs/tars_cpp_quickstart.md) 增加一个带参数的接口

## 客户端

创建目录[/home/tarsproto/],*必须这个，有点恶心*，在后回到刚刚服务器源码的目录，例如[/usr/local/tars/TestApp/HelloServer]
``` bash
make release
```
``` bash
king@king:/home/tarsproto/TestApp/HelloServer$ tree -L 2
.
├── Hello.h
├── HelloServer.mk
├── Hello.tars
└── makefile

0 directories, 4 files
```

这些内容用于客户端引用

按照步骤继续添加可执行程序源码

``` bash
king@king:/home/tarsproto/TestApp/TestHelloClient$ tree
.
├── main.cpp
├── makefile

0 directories, 2 files
```

注意修改IP地址

``` cpp
int main(int argc,char ** argv)
{
    Communicator comm;
    try
    {
        HelloPrx prx;
        comm.stringToProxy("TestApp.HelloServer.HelloObj@tcp -h 192.168.10.6 -p 20001" , prx);
```
``` bash
make
```
运行
``` bash
king@king:/home/tarsproto/TestApp/TestHelloClient$ ./TestHelloClient 
2018-06-26 15:22:52|CommunicatorEpoll::run id:12838
iRet:0 sReq:hello world sRsp:hello world
```


# 问题
## 访问http://127.0.0.1:8080报错

NoClassDefFoundError: org/apache/log4j/spi/ThrowableInformation

修改${TARS_ROOT}/Tars/java/core/client.pom.xml，增加下列依赖
``` xml
<!-- https://mvnrepository.com/artifact/log4j/log4j -->
<dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j</artifactId>
    <version>1.2.16</version>
</dependency>
```

## mysql时间搓错误

Error updating database.  Cause: com.mysql.jdbc.MysqlDataTruncation: Data truncation: Incorrect datetime value: '0000:00:00 00:00:00' for column 'patch_time' at row 1↵### The error may involve defaultParameterMap

`/etc/mysql/mysql.conf.d/mysqld.cnf`  去掉`NO_ZERO_DATE`选项
``` bash
sql_mode = "ONLY_FULL_GROUP_BY"
```

## 文件发布上传文件失败
检查文件/目录权限

## tarsnotify inactive

需要手动发布 `tarsnotify-tar`（另外还有五个可选组件）


## 生成hello world失败
``` bash
Bareword "DemoServer" not allowed while "strict subs" in use at 
```
参考 <http://haoningabc.iteye.com/blog/1688936>

``` bash
#rename "DemoServer" "$SERVER" $SRC_FILE  
rename "s/DemoServer/$SERVER/" $SRC_FILE  
#rename "DemoServant" "$SERVANT" $SRC_FILE  
rename "s/DemoServant/$SERVANT/" $SRC_FILE  
```

## 编译Hello world报错
``` bash
/usr/include/c++/5/bits/c++0x_warning.h:32:2: error: #error This file requires compiler
```

修改当前目录makeifle，`TARS2CPP_FLAG`增加`c11`支持
``` bash
TARS2CPP_FLAG:= -std=c++11
```





