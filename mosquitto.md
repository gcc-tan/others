mosquitto是一款mqtt协议的broker（server），mqtt的broker有很多，大家从[这里](https://github.com/mqtt/mqtt.github.io/wiki/software?id=software)找到一款合适的broker。

###安装
在这个[下载页面](http://mosquitto.org/download/)，可以下载到对应的版本。在ubuntu系统下可以使用`sudo apt-get install mosquitto`进行安装，如果是比较老版本的ubuntu可以先执行`sudo apt-add-repository ppa:mosquitto-dev/mosquitto-ppa`添加源，然后`sudo apt-get update`更新源。


上面安装的是mosquitto的broker，如果需要测试mqtt的发布和订阅功能，还需要使用命令`sudo apt-get install mosquitto-clients`安装mosquitto_pub和mosquitto_sub两个客户端。从客户端的名字上看可以清楚地了解到这两个客户端的作用。

>如果安装过后mosquitto作为daemon已经在后台运行，那么可以使用`sudo service mosquitto stop`终止该守护进程。在调试的过程中一般需要启用日志，需要修改默认配置文件（/etc/mosquitto/mosquitto.conf），启用日志（log_dest的none修改为stdout），选择日志输出类型（所有log_type全部选上）


测试安装是否成功可以首先开启一个终端运行`mosquitto`作为broker，然后一个终端执行`mosquitto_sub -h localhost -t test -v`作为subscriber，另外一个终端执行`mosquitto_pub -h localhost -t test -m "Hello world"`作为publisher。最后可以在subscriber的终端看到Hello world这条消息


###开启SSL/TLS传输
为了保证数据传输的安全，需要开启SSL/TLS模式。

首先使用openssl创建需要的证书文件。

a.产生CA的证书和私钥，用于签发其他证书。
下面命令的解释：openssl req生成证书请求，让第三方权威CA来签发。没有-key参数时会生成一个私钥文件（该私钥默认是有密码保护的，-nodes（no des），可以明确指定不需要密码保护），再生成证书请求。-keyout 指定产生的私钥文件名字是ca.key。-out指定输出的证书文件名为ca.crt。-new表示生成一个新的证书请求文件。365表示证书的有效期
```
openssl req -new -x509 -days 365 -extensions v3_ca -keyout ca.key -out ca.crt -nodes
```
b.对于服务端：
执行下面命令产生产生server的私钥server.key，私钥的长度是2048
```
openssl genrsa -out server.key 2048
```
执行下面命令发送证书签名请求到CA。其实下面命令的作用就是产生一个server.csr文件，包含申请者国家，域名，公钥。
```
openssl req -out server.csr -key server.key -new
```
执行下面命令，将server.crs发送到CA，使用CA的私钥签名。生成的server.crt是CA签名后的证书文件
```
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365
```

c.对于客户端执行同样的命令：
```
openssl genrsa -out client.key 2048
openssl req -out client.csr -key client.key -new
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 365
```

>在生成server的crs需要将common name设置成服务器的ip地址或者是url地址，并且使用mosquitto_sub时-h参数需要指定连接对应的ip地址或者url，否则会遇到tls error。而使用paho客户端连接时则不会出现这个问题，很奇怪


>这里的`openssl genrsa -out server.key 2048`这条命令的输出server.key其实包括的不仅仅是私钥。可以使用`openssl rsa -in server.key -text`，这个key中包含的内容。我们可以看到包含的一些rsa算法需要的或者中间过程的变量：p（prime1)，q（prime2），n（modulus），e（在openssl中固定65537）....。公钥是{e, n}，私钥是{d，n}。因此，使用`openssl rsa -in server.key -pubout`，可以查看对应的公钥。同样可以使用`openssl req -in server.csr -text`查看server.csr的内容


然后，在/etc/mosquitto/conf.d/目录下新增一个.conf文件，我新增的是tls.conf文件，文件的内容如下：
```
#监听端口
listener 8883
#CA的证书文件
cafile /home/tan/Documents/code/java/platform/secure/ca.crt
#服务器的证书文件
certfile /home/tan/Documents/code/java/platform/secure/server.crt
#服务器的私钥
keyfile /home/tan/Documents/code/java/platform/secure/server.key
#服务器对客户端进行认证
require_certificate true
#使用客户端证书作的common name为username
use_identity_as_username true
```

最后在（/home/tan/Documents/code/java/platform/secure/）目录执行下面的脚本进行tls安全传输的测试：
```
#!/bin/bash
if [ "$1" = "" ]; then
	printf "usage:$0 [sub|pub messages] {topic}\n"
	exit 1
fi

if [ "$1" = "sub" ]; then
	mosquitto_sub -h localhost -p 8883 -t "$2" --cafile ca.crt -i "client-sub" --cert client.crt --key client.key
fi

if [ "$1" = "pub" ]; then
	mosquitto_pub -h localhost -p 8883 -m "$2" -t "$3" --cafile ca.crt -i "cilent-pub" --cert client.crt --key client.key
	if [ $? -eq 0 ]; then
		printf "send message ok\n"
	else
		printf "send failed\n"
	fi
fi

```


###认证和授权
首先先理解认证和授权这两个概念。

**认证**：

认证是确认单个数据或实体的属性的真实性的行为。这意味着认证用于验证个人、设备或应用程序是否具有他们声称拥有的身份。呈现一个旅行中的简单例子给你。在你能够乘坐飞机之前，机场安检要求你的护照或身份证件。显示所请求的ID以认证身份。在这种情况下，您的护照证实您的身份和您的姓名。每个人都可以说出你的名字，但是只有你能够出示护照作为你身份的证明。

**授权**：

授权是指定特定资源的访问权限的功能。这包括策略的定义和实施，它们指定谁可以访问某个资源。因此，以下术语至关重要：

主体或用户：想访问资源
资源，对象或服务：应防止未经授权的访问
政策：指定主体是否可以访问资源

让我们来看一下身份验证博客文章的旅行示例，并继续执行。我们已经看到，登机时可以使用护照来认证一个人的身份。所以在身份确认后，预订确认书或登机证用于授权/授予访问权限以获得特定的飞机。所以在预订航班后，有关您的人的信息和确切的航班日期，时间和目的地作为飞行的授权。

正如我们已经在现实世界中看到的那样，认证和授权是两个非常重要的安全概念。用户或主体的授权在事先没有认证的情况下没有太多价值，从而确认其身份。授权对于限制访问非常重要，只允许符合条件的人员，客户或主体访问某些资源，数据或事物。

授权的两种实现方案：

简写 | 名称 | 说明 | 示例
----|------|-----|----
ACL	| 访问控制列表 | ACL将资源与权限列表相关联。权限包括谁可以访问资源（例如文件）以及哪些操作（例如，读取，写入，执行）被允许。| Unix文件权限
RBAC | 基于角色的访问控制 | RBAC始终将某个资源的权限与角色相关联。角色是用户和资源之间的额外抽象。这使得更容易将用户与角色相关联，以便维护所有用户的权限 | Active Directory, SELinux, PostgreSQL


除了上面两个概念。还有就是如果服务器遇到没有授权的请求，broker该怎么处理客户端的请求。（认证就不用说了，认证不通过过是拒绝连接）。如果在发布（Publish）过程中有两种解决方案：第一种是断开客户端的连接，第二种是客户端正常向broker推送消息，broker正常响应（响应PUBACK或PUBREL），但是broker不向订阅者转发消息。由于mqtt协议规范没有进行定义，所以导致不同的broker会采取不同的策略，mosquitto采取的就是第二种策略。如果在订阅（Subscribe）过程中，遇到了非授权请求。那么mqtt的处理会和正常订阅请求一样返回一个SUBACK的确认消息，这个SUBACK消息中包含Return Code字段，通过返回128表示订阅失败。下面是Return Code和对应含义表：

Return Code | Return Code Response
------------|--------------------
0 | Success – Maximum QoS 0
1 | Success – Maximum QoS 1
2 | Success – Maximum QoS 2
128 | Failure


对上面的基础知识有了了解之后看看mosquitto怎样做授权和认证：

mqtt仅支持最基本的帐号和密码认证。要添加授权功能则需要自己利用mqtt broker提供的接口进行实现。好在github上已经有开源的项目实现了这个插件。下面就介绍插件的配置过程。

[mosquitto-auth-plug](https://github.com/jpmens/mosquitto-auth-plug)从这个地方可以找到插件对应的源代码和配置文档。

首先编译源代码，编译源代码需要将config.mk.in重命名成config.mk，然后在config.mk里面填写编译插件需要的信息

```
# Select your backends from this list
BACKEND_CDB ?= no
BACKEND_MYSQL ?= yes
BACKEND_SQLITE ?= no
BACKEND_REDIS ?= no
BACKEND_POSTGRES ?= no
BACKEND_LDAP ?= no
BACKEND_HTTP ?= no
BACKEND_JWT ?= no
BACKEND_MONGO ?= no
BACKEND_FILES ?= no
BACKEND_MEMCACHED ?= no

# Specify the path to the Mosquitto sources here
# MOSQUITTO_SRC = /usr/local/Cellar/mosquitto/1.4.12
MOSQUITTO_SRC = /home/tan/Documents/code/java/platform/secure/auth/mosquitto-1.5.3

# Specify the path the OpenSSL here
OPENSSLDIR = /usr/include/openssl

# Add support for django hashers algorithm name
SUPPORT_DJANGO_HASHERS ?= no

# Specify optional/additional linker/compiler flags here
# On macOS, add
#	CFG_LDFLAGS = -undefined dynamic_lookup
# as described in https://github.com/eclipse/mosquitto/issues/244
#
# CFG_LDFLAGS = -undefined dynamic_lookup  -L/usr/local/Cellar/openssl/1.0.2l/lib
# CFG_CFLAGS = -I/usr/local/Cellar/openssl/1.0.2l/include -I/usr/local/Cellar/mosquitto/1.4.12/include
CFG_LDFLAGS =
CFG_CFLAGS =
```
然后运行make命令进行编译

>需要编译成功还得安装一些必要的开发库，以backend为mysql为例，需要执行`sudo apt-get install libmysqld-dev`，安装开发mysql插件的库，否则执行make时会产生找不到`<mysql.h>`这个头文件错误。还需要安装libmosquitto-dev这个动态库，否则会产生ld error找不到mosquitto动态库。

接着在/etc/mosquitto/conf.d中添加一个mysql_auth.conf的配置文件，文件内容如下：

```
auth_plugin /home/tan/Documents/code/java/platform/secure/auth/auth-plug.so
#配置的认证后端，可以有多个
auth_opt_backends mysql

#表示关闭acl cache，防止在测试时更新db的acl权限表后mqtt连接没有响应
acl_cacheseconds 0

#mysql服务器的地址
auth_opt_host localhost

#mysql服务器运行端口
auth_opt_port 3306

#使用mysql的数据库名字
auth_opt_dbname platform

#登录mysql的帐号
auth_opt_user root

#登录mysql的密码
auth_opt_pass tan5

#指定用户认证的查询结果，SQL查询的结果最多只能返回一列，表示这个用户对应的PBKDF2密码。这个加密算法在插件中已经给出，叫做np，可直接使用np进行加密，得出密码。注意数据库中存储的是密码加密后的密文。%s会被替换成mqtt客户端传递的username
auth_opt_userquery SELECT pw FROM users WHERE username = '%s'

#超级用户的认证。SQL必须返回一行一列。0表示不是超级用户，1表示是超级用户。超级用户不受acl控制
auth_opt_superquery SELECT COUNT(*) FROM users WHERE username = '%s' AND super = 1

#用户授权。%d会被替换成客户端的访问动作对应的值。访问动作有read topic，write topic（publish），subscribe topic，对应的值分别是1, 2, 4。这里使用的SQL和官方文档不一样，就是因为新版的Mosquitto broker将subscribe和read区分开来，如果使用老版的SQL会出现订阅topic错误，从而导致重复订阅topic的情况
auth_opt_aclquery SELECT topic FROM acls WHERE (username = '%s') AND (rw & %d)

#匿名mqtt连接使用的用户名
auth_opt_anonusername AnonymouS
```



最后在mysql新建数据库platform，然后使用下面的建表语句建立相关的数据表

```
DROP TABLE IF EXISTS users;

CREATE TABLE users (
	id INTEGER AUTO_INCREMENT,
	username VARCHAR(25) NOT NULL,
	pw VARCHAR(128) NOT NULL,
	super INT(1) NOT NULL DEFAULT 0,
	PRIMARY KEY (id)
  );

CREATE UNIQUE INDEX users_username ON users (username);

INSERT INTO users (username, pw) VALUES ('jjolie', 'PBKDF2$sha256$901$pN94c3+KCcNvIV1v$LWEyzG6v/gtvTrjx551sNcWWfwIZKAg0');
INSERT INTO users (username, pw) VALUES ('a', 'PBKDF2$sha256$901$XPkOwNbd05p5XsUn$1uPtR6hMKBedWE44nqdVg+2NPKvyGst8');
INSERT INTO users (username, pw, super)
	VALUES ('su1',
	'PBKDF2$sha256$901$chEZ4HcSmKtlV0kf$yRh2N62uq6cHoAB6FIrxIN2iihYqNIJp',
	1);
INSERT INTO users (username, pw, super)
	VALUES ('S1',
	'PBKDF2$sha256$901$sdMgoJD3GaRlTF7y$D7Krjx14Wk745bH36KBzVwHwRQg0a+z6',
	1);
INSERT INTO users (username, pw, super)
	VALUES ('m1',
	'PBKDF2$sha256$2$NLu+mJ3GwOpS7JLk$eITPuWG/+WMf6F3bhsT5YlYPY6MmJHvM',
	0);
-- PSK
INSERT INTO users (username, pw, super)
	VALUES ('ps1',
	'deaddead',
	0);

DROP TABLE IF EXISTS acls;

CREATE TABLE acls (
	id INTEGER AUTO_INCREMENT,
	username VARCHAR(25) NOT NULL,
	topic VARCHAR(256) NOT NULL,
	rw INTEGER(1) NOT NULL DEFAULT 1,	-- 1: read-only, 2: read-write
	PRIMARY KEY (id)
	);
CREATE UNIQUE INDEX acls_user_topic ON acls (username, topic(228));

INSERT INTO acls (username, topic, rw) VALUES ('jjolie', 'loc/jjolie', 1);
INSERT INTO acls (username, topic, rw) VALUES ('jjolie', 'loc/ro', 1);
INSERT INTO acls (username, topic, rw) VALUES ('jjolie', 'loc/rw', 2);
INSERT INTO acls (username, topic, rw) VALUES ('jjolie', '$SYS/something', 1);
INSERT INTO acls (username, topic, rw) VALUES ('a', 'loc/test/#', 1);
INSERT INTO acls (username, topic, rw) VALUES ('a', '$SYS/broker/log/+', 1);
INSERT INTO acls (username, topic, rw) VALUES ('su1', 'mega/secret', 1);
INSERT INTO acls (username, topic, rw) VALUES ('nop', 'mega/secret', 1);

INSERT INTO acls (username, topic, rw) VALUES ('jog', 'loc/#', 1);
INSERT INTO acls (username, topic, rw) VALUES ('m1', 'loc/#', 1);

INSERT INTO acls (username, topic, rw) VALUES ('ps1', 'x', 1);
INSERT INTO acls (username, topic, rw) VALUES ('ps1', 'blabla/%c/priv/#', 1);
```



内容来自：

[[译]MQTT安全基础：授权](http://fanzhenyu.me/2017/09/28/译-MQTT安全基础：授权/)


