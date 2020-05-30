MySQL 8.0的安装过程和MySQL5.7，大致安装过程可以参考5.7的安装，除此之外，还需要下面的准备步骤。

Win10下5.7的安装步骤在这里：https://github.com/inCeit/Win10_MySQL_Installation

#### 1. Windows下OpenSSL的安装

可以通过下载自动安装包：http://slproweb.com/products/Win32OpenSSL.html ，否则手动安装OpenSSL库很麻烦

#### 2. 下列文件需要另存为UTF-8格式

否则编译通不过。

`sql_locale.cc`,`sql_commands_help_data.h`(cmake生成的文件，不在源码目录),`sql_initialize.cc`,`test_string_service_charset.cc`

#### 3. Windows下调试需要将VS“附加”到mysqld进程

在Windows下使用VS调试MySQL的时候，VS的版本要>= 1027。另外，还有一个问题，VS启动调试时默认是跟踪的`monitor mysqld`进程，而我们要跟踪的是mysqld进程。实现的方法是：选择“调试”->“附加到进程”，搜索mysqld进程名，并选中跟踪。

#### 4. 初始化data目录之后，`show databases`的时候可能会如下错误

``` mysql
ERROR 1449 (HY000): The user specified as a definer ('mysql.infoschema'@'localhost') does not exist
```
需要在mysqld启动时加入`--upgrade=FORCE`，加了之后启动mysqld它会自动升级系统表用户，另外，可能会显示升级失败，不用关心，去掉这个选项重启之后再登录就可以正常`show databases`了。
