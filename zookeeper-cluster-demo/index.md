# 使用Docker快速搭建ZooKeeper集群


<!--more-->
# 使用Docker快速搭建ZooKeeper集群

## 背景 
* 使用Docker 安装zookeeper只是安装单机的。现在，我们要用 docker-compose 来编排集群容器。因为一个一个地启动 ZK 太麻烦了.

## 1. docker-compose.yml  文件

* 首先创建一个名为 docker-compose.yml 的文件, 其内容如下:

```yaml
version: '3'
services:
    zoo1:
        image: zookeeper
        restart: always
        container_name: zoo1
        ports:
            - "2181:2181"
        environment:
            ZOO_MY_ID: 1
            ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2182 server.3=zoo3:2888:3888;2183 server.4=zoo4:2888:3888;2184
 
    zoo2:
        image: zookeeper
        restart: always
        container_name: zoo2
        ports:
            - "2182:2181"
        environment:
            ZOO_MY_ID: 2
            ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2182 server.3=zoo3:2888:3888;2183 server.4=zoo4:2888:3888;2184
 
    zoo3:
        image: zookeeper
        restart: always
        container_name: zoo3
        ports:
            - "2183:2181"
        environment:
            ZOO_MY_ID: 3
            ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2182 server.3=zoo3:2888:3888;2183 server.4=zoo4:2888:3888;2184

    zoo4:
        image: zookeeper
        restart: always
        container_name: zoo4
        ports:
            - "2184:2181"
        environment:
            ZOO_MY_ID: 4
            ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2182 server.3=zoo3:2888:3888;2183 server.4=zoo4:2888:3888;2184
```            

### 编排配置说明           
*  1、version: 版本号，一般来说，和你的 Docker Engine 的版本相匹配；这里选择 3；
*  2、image: 镜像版本：默认使用的是latest版本，可缺省；
*  3、ZOO_MY_ID（环境变量），该id在集群中必须是唯一的，并且其值应介于1和255之间；
 请注意，如果使用已包含 myid 文件的 /data 目录启动容器，则此变量将不会产生任何影响。相当于你在zoo.cfg中指定的dataDir目录中创建文件myid，并且文件的内容就是范围为1到255的整数；
* 4、ZOO_SERVERS（环境变量）:
 > * 此变量允许您指定Zookeeper集群的计算机列表；
 > * 每个条目都应该这样指定：server.id=<address1>:<port1>:<port2>[:role];[<client port address>:]<client port> 条目间空格分隔
 > * id 是一个数字，表示集群中的服务器ID；
 > * address1表示这个服务器的ip地址；
 > * 集群通信端⼝: port1 表示这个服务器与集群中的 Leader 服务器交换信息的端口；
 > * 集群选举端⼝: port2 表示万一集群中的 Leader 服务器挂了，需要一个端口来重新进行选举，选出一个新的 Leader，而这个端口就是用来执行选举时服务器相互通信的端口；
 > * role: 默认是 participant,即参与过半机制的⻆⾊，选举，事务请求过半提交，还有⼀个是observer, 观察者，不参与选举以及过半机制。
 > * client port address 是可选的，如果未指定，则默认为 “0.0.0.0” ；
 > * client port 位于分号的右侧。从3.5.0开始，zoo.cfg中不再使用clientPort和clientPortAddress配置参数。作为替代，client port 用来表示客户端连接 Zookeeper 服务器的端口，Zookeeper 会监听这个端口，接受客户端的访问请求。

* 请注意，如果使用已包含zoo.cfg文件的/conf目录启动容器，则此变量不会产生任何影响。换句话说，zoo.cfg文件中包含集群的计算机列表，此时该环境变量不生效。

## 2. 启动集群
* 打开命令行提示符，进入 docker-compose.yml 所在的目录，接着执行
  ``` shell 
  docker-compose -f docker-compose.yml up
  ```
* 最后检查一下运行情况：容器 zoo1,zoo2,zoo3,zoo4 都在运行中。

## 3. 检测集群状态
* 检测集群状态的命令是 zkServer.sh status /conf/zoo.cfg。使用该命令的前提是要先登入容器中，可以使用 ``docker exec -it zoo1 bash`` 登入 zoo1 中。

  ``` Shell
  he@LAPTOP-0PME2UF0:~$ docker exec -it zoo1 bash
  root@aaa4ead811f0:/apache-zookeeper-3.8.2-bin# zkServer.sh status /conf/zoo.cfg
  ZooKeeper JMX enabled by default
  Using config: /conf/zoo.cfg
  Client port found: 2181. Client address: localhost. Client SSL: false.
  Mode: follower

  he@LAPTOP-0PME2UF0:~$ docker exec -it zoo2 bash
  root@14fc28ab859e:/apache-zookeeper-3.8.2-bin# zkServer.sh status /conf/zoo.cfg
  ZooKeeper JMX enabled by default
  Using config: /conf/zoo.cfg
  Client port found: 2182. Client address: localhost. Client SSL: false.
  Mode: follower

  he@LAPTOP-0PME2UF0:~$ docker exec -it zoo3 bash
  root@7d5c0792e1d5:/apache-zookeeper-3.8.2-bin# zkServer.sh status /conf/zoo.cfg
  ZooKeeper JMX enabled by default
  Using config: /conf/zoo.cfg
  Client port found: 2183. Client address: localhost. Client SSL: false.
  Mode: follower

  he@LAPTOP-0PME2UF0:~$ docker exec -it zoo4 bash
  root@30e423d9997b:/apache-zookeeper-3.8.2-bin# zkServer.sh status /conf/zoo.cfg
  ZooKeeper JMX enabled by default
  Using config: /conf/zoo.cfg
  Client port found: 2184. Client address: localhost. Client SSL: false.
  Mode: leader
  ```


## 4. ZooKeeper 集群角色介绍
* Zookeeper 集群模式⼀共有三种类型的⻆⾊：
* Leader	:  处理所有的事务请求（写请求），可以处理读请求，集群中只能有⼀个Leader
* Follower : 只能处理读请求，同时作为 Leader的候选节点，即如果Leader宕机，Follower节点要参与到新的Leader选举中，有可能成为新的Leader节点。|
* Observer : 只能处理读请求。不能参与选举|

## 5. 不宕机动态扩容和缩容
* ZooKeeper 支持通过修改 zoo.cfg 并重启来改变ZooKeeper集群中的服务器，但是这种方式，不是很方便，上线一台服务器或者下线一台服务器需要对所有的服务器进行重启。

* ZooKeeper 3.5.0 提供了⽀持动态扩容/缩容的新特性。但是通过客户端 API可以变更服务端集群状态是件很危险的事情，所以在ZooKeeper 3.5.3 版本要⽤动态配置，需要开启超级管理员身份验证模式 ACLs。

* 关于ZooKeeper的ACL权限控制，

* Step1: 我为 super:qwer1234 生成了 digest 为 super:YjJhp1/a1jnzeGTDN7nAUxFcep8=；

* Step2: ``` docker-compose -f docker-compose.yml down ``` 停掉并删除原来的容器；

* Step3: 修改 docker-compose.yml 的内容：

```yaml
version: '3'
services:
    zoo1:
        image: zookeeper
        restart: always
        container_name: zoo1
        ports:
            - "2181:2181"
        environment:
            ZOO_MY_ID: 1
            ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181 server.4=zoo4:2888:3888;2181
            ZOO_CFG_EXTRA: "reconfigEnabled=true"
            JVMFLAGS: "-Dzookeeper.DigestAuthenticationProvider.superDigest=super:YjJhp1/a1jnzeGTDN7nAUxFcep8="

    zoo2:
        image: zookeeper
        restart: always
        container_name: zoo2
        ports:
            - "2182:2181"
        environment:
            ZOO_MY_ID: 2
            ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181 server.4=zoo4:2888:3888;2181
            ZOO_CFG_EXTRA: "reconfigEnabled=true"
            JVMFLAGS: "-Dzookeeper.DigestAuthenticationProvider.superDigest=super:YjJhp1/a1jnzeGTDN7nAUxFcep8="

    zoo3:
        image: zookeeper
        restart: always
        container_name: zoo3
        ports:
            - "2183:2181"
        environment:
            ZOO_MY_ID: 3
            ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181 server.4=zoo4:2888:3888;2181
            ZOO_CFG_EXTRA: "reconfigEnabled=true"
            JVMFLAGS: "-Dzookeeper.DigestAuthenticationProvider.superDigest=super:YjJhp1/a1jnzeGTDN7nAUxFcep8="

    zoo4:
        image: zookeeper
        restart: always
        container_name: zoo4
        ports:
            - "2184:2181"
        environment:
            ZOO_MY_ID: 4
            ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181 server.4=zoo4:2888:3888;2181
            ZOO_CFG_EXTRA: "reconfigEnabled=true"
            JVMFLAGS: "-Dzookeeper.DigestAuthenticationProvider.superDigest=super:YjJhp1/a1jnzeGTDN7nAUxFcep8="
```
> * ZOO_CFG_EXTRA: "reconfigEnabled=true" 开启了默认关闭的客户端 reconfig API
> * JVMFLAGS: "-Dzookeeper.DigestAuthenticationProvider.superDigest=super:YjJhp1/a1jnzeGTDN7nAUxFcep8=" 设置了超级管理员账号为 super，明文密码为 qwer1234

* Step4: 重启集群
在 docker-compose.yml 所在目录执行以下命令：
```shell
docker-compose -f docker-compose.yml up
```
* Step5: 登录客户端 ,使用如下命令在容器内连接该节点的 ZooKeeper 服务
  ```
  zkCli.sh -server localhost:2181
  ```

* 接下来就是使用ZooKeeper客户端的reconfig命令了。

> 5.1 查看当前集群信息
> ``get /zookeeper/config``  执行结果如下：
```
[zk: localhost:2181(CONNECTED) 0] get /zookeeper/config
server.1=zoo1:2888:3888:participant;0.0.0.0:2181
server.2=zoo2:2888:3888:participant;0.0.0.0:2181
server.3=zoo3:2888:3888:participant;0.0.0.0:2181
server.4=zoo4:2888:3888:participant;0.0.0.0:2181
version=100000000
```
> 5.2 扩缩容前的认证超级管理员身份 ,客户端命令行执行``addauth digest super:qwer1234``
> 5.3 移除集群中的一台服务器 ``reconfig -remove <id>``比如说，从集群中移除ID为3的ZooKeeper服务：
```
[zk: localhost:2181(CONNECTED) 6] reconfig -remove 3
Committed new configuration:
server.1=zoo1:2888:3888:participant;0.0.0.0:2181
server.2=zoo2:2888:3888:participant;0.0.0.0:2181
server.4=zoo4:2888:3888:participant;0.0.0.0:2181
version=300000001
```
> 5.4 为集群新增一台服务器
```
reconfig -add server.id=<address1>:<port1>:<port2>[:role];[<client port address>:]<client port>
```
> 比如说，把刚才移除的服务器再添加回来：
```
[zk: localhost:2181(CONNECTED) 7] reconfig -add server.3=zoo3:2888:3888;2181
Committed new configuration:
server.1=zoo1:2888:3888:participant;0.0.0.0:2181
server.2=zoo2:2888:3888:participant;0.0.0.0:2181
server.3=zoo3:2888:3888:participant;0.0.0.0:2181
server.4=zoo4:2888:3888:participant;0.0.0.0:2181
version=400000001
```
## 6. 过半机制
participant ⻆⾊能够形成集群（过半机制），如果集群内的 participant 有一半及以上都宕机了，此时客户端将不再可用。

> 6.1 半数服务器宕机
> 比如说，停掉了 zoo3 和 zoo4：此时，没有停掉的服务，也启动不了客户端：

> 6.2 半数以上服务器可用,然后重启了 zoo3：
> 稍等一会之后，集群再次变得可用了。

## 问题记录
### 1 : Client port not found in server configs
 * 参考官网 ZOO_SERVERS 的配置，以zoo1为例: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888 server.4=zoo4:2888:3888 没有加上端口号，实践后遇上了问题。
 * 解决方案：立即停止和删除容器、网络、卷：
    ```
    docker-compose -f docker-compose.yml down
    ```
 * 然后，改成上面给出的 docker-compose.yml 文件内容，然后用 docker-compose -f docker-compose.yml up 重启集群服务器。

### 2 :KeeperErrorCode = Reconfig is disabled
* 从3.5.0开始到3.5.3之前，没有办法禁用动态重新配置功能。我们希望提供禁用重新配置功能的选项，因为启用重新配置后，我们存在一个安全问题，即恶意参与者可以对ZooKeeper集合的配置进行任意更改，包括向集合中添加受损服务器。我们倾向于让用户自行决定是否启用它，并确保适当的安全措施到位。因此，在3.5.3中引入了reconfigEnabled configuration选项，以便可以完全禁用重新配置功能，并且通过reconfig API重新配置集群的任何尝试（无论是否具有身份验证）在默认情况下都将失败，除非reconfigEnabled设置为true。要将选项设置为true，配置文件（zoo.cfg）应包含：
   
    ```
    reconfigEnabled=true
    ```
### 3: Insufficient permission
* 解决方案：addauth digest super:qwer1234 赋予超级管理员权限
    ```
    [zk: localhost:2181(CONNECTED) 4] reconfig -remove 3
    Insufficient permission :
    [zk: localhost:2181(CONNECTED) 5] addauth digest super:qwer1234
    [zk: localhost:2181(CONNECTED) 6] reconfig -remove 3
    Committed new configuration:
    server.1=zoo1:2888:3888:participant;0.0.0.0:2181
    server.2=zoo2:2888:3888:participant;0.0.0.0:2181
    server.4=zoo4:2888:3888:participant;0.0.0.0:2181
    version=300000001
    ```


