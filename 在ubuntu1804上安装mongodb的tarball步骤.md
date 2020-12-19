## 在ubuntu18.04上安装mongodb的tarball步骤
### 1 下载tarball
  到https://www.mongodb.com/try/download/community 选择
  version:4.4.2(current) 
  platform:Ubuntu 18.04
  Package:tgz
  或者直接wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu1804-4.4.2.tgz
### 2 解压文件
  tar -zxvf mongodb-linux-*-4.4.2.tgz
  mv mongodb-linux-x86_64-ubuntu1804-4.4.2 mongodb442
### 3 拷贝到常用工作路径
  sudo cp mongodb442/bin/* /usr/local/bin/
### 4 运行mongo
  (1)建立配置文件
    配置mongodb有两种方式，一种是通过mongod和mongos两个命令；另外一种方式就是配置文件的方式。因为更容易去管理，所以后者更受大家的青睐。
    linux下配置文件缺省为/etc/mongod.conf，当然我们可以在启动时候显式指定配置文件，例如
    ./mongod --config /usr/local/mongoDB/mongodbserver/etc/mongodb.conf
    我们此处采用缺省设置，并且设置好当前用户的读权限
    配置文件采用YAML格式(YAML是json的超集)，注意YAML不支持tab，必须使用空格缩进。
    
    重要参数说明:
    日志参数:
    systemLog.path
         类型:string
         作用：指定日志文件的目录
    syetemLog.logAppend
          类型：boolean
          默认值：False
          作用：当mongod或mongos重启时，如果为true，将日志追加到原来日志文件内容末尾；如果为false，将创建一个新的日志文件
    systemLog.destination:可以写厕file或syslog，如果选文件写入文件，否则日志输出到控制台


    processManagement参数:
    processManagement:
	   fork: <boolean>
	   pidFilePath: <string>
	   timeZoneInfo: <string>

    processManagement.fork:启用后台运行mongos或mongod进程的守护程序模式,缺省false
    processManagement.pidFilePath:指定用于存储mongos或mongod进程的进程ID（PID）的文件位置。 运行mongod或mongos进程的用户必须能够写入此路径


    net 参数
   net.port MongoDB实例在其上侦听客户端连接的TCP端口
   缺省值：
      27017 for mongod (if not a shard member or a config server member) or mongos instance
      27018 if mongod is a shard member
      27019 if mongod is a config server member

   net.bindIp:mongos或mongod应该在其上侦听客户端连接的主机名和/或IP地址和/或完整的Unix域套接字路径。 可以将mongos或mongod附加到任何接口。 要绑定到多个地址，请输入一个用逗号分隔的值的列表。如果想绑定所有 IPv4地址,输入 0.0.0.0.
   net.bindIpAll:如果为true，则mongos或mongod实例绑定到所有IPv4地址（即0.0.0.0）。 如果mongos或mongod以net.ipv6开头：true，net.bindIpAll也将绑定到所有IPv6地址（即::)。如果以net.ipv6开头，则mongos或mongod仅支持IPv6：true。 仅指定net.bindIpAll不能启用IPv6支持。

   security参数
   security.authorization:启用或禁用基于角色的访问控制（RBAC），以控制每个用户对数据库资源和操作的访问。缺省是disabled,应该设置成enabled

   storage参数
    storage.dbPath:mongod实例存储数据的目录，缺省是data/db
        RHEL / CentOS and Amazon /var/lib/mongo
        SUSE	 /var/lib/mongo
         Ubuntu and Debian /var/lib/mongodb

我的简化版配置文件(参数太多没有一一研究):
```$ cat /etc/mongod.conf
systemLog:
   destination: file
   path: "/var/log/mongodb/mongod.log"
   logAppend: true
storage:
   journal:
      enabled: true
processManagement:
   fork: true
net:
   bindIpAll: true
   port: 27017
setParameter:
   enableLocalhostAuthBypass: false
security:
   authorization: enabled
storage:
  dbPath: "/var/lib/mongo"
```
需要注意的是enableLocalhostAuthBypass这个参数，第一次启动时需要注释掉，否则没有用户就要求认证，创建用户之后开启可以提高linux操作系统下的安全性
还有TLS选项也很重要，生产环境下应该使用

   (2)创建data和log目录
    sudo mkdir -p /var/lib/mongo
    sudo mkdir -p /var/log/mongodb
    sudo chown `whoami` /var/lib/mongo 
    sudo chown `whoami` /var/log/mongodb
  (3)运行服务端
 mongod -f /etc/mongod.conf
  (4)创建用户
  $mongo
  >use admin
  >db.createUser({ user: 'admin', pwd: 'zhubajie', roles: [ { role: "userAdminAnyDatabase", db: "admin" } ] });
  >db.auth("admin","zhubajie");

### 5 在windows客户端安装MongoDB Compass
 下载，解压运行
  建立连接串
   mongodb://admin:zhubajie@192.168.100.108:27017/admin
  因为没有用域名，所以不能用 mongodb+srv://admin:zhubajie@192.168.100.108:27017/admin
"mongodb+srv://"
这种形式的URL只支持一个hostname，对应DNS server，进行SRV record查询，它也支持replSet和authSource（TXT record），需要注意的是它默认使用TLS，即ssl=True。

### 6 参考资料
https://docs.mongodb.com/manual/tutorial/install-mongodb-on-ubuntu-tarball/
