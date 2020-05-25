MySQL 8.0的安装过程和MySQL5.7，大致安装过程可以参考5.7的安装，除此之外，还需要下面的准备步骤。

#### 1. Windows下OpenSSL的安装

可以通过下载自动安装包：http://slproweb.com/products/Win32OpenSSL.html ，否则手动安装OpenSSL库很麻烦

#### 2. 下列文件需要另存为UTF-8格式

否则编译通不过。

`sql_locale.cc`,`sql_commands_help_data.h`,`sql_initialize.cc`,`test_string_service_charset.cc`
