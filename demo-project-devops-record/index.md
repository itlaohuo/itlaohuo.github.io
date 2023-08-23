# 演示项目-运维部署记录（demo project devops record）

<!--more-->

## demo-common

### Dockerfile

```shell
FROM it-registry.itlaohuo.top/libary/centos7-jdk1.8:1.0.22-arms-agent-0510

WORKDIR /home

COPY ./democommon-service/target/democommon-service-*.jar /home

COPY ./docker-file/demo-ccommon/tools /home/tools/

COPY ./docker-file/demo-ccommon/conf /home/conf/

COPY ./docker-file/demo-ccommon/bin/entrypoint /

RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]

```

### tools 目录文件

* arthas_conn.sh

```shell
#!/bin/bash
JAVA_TOOL_OPTIONS=
APP_NAME="vowrk-common"
ARTHAS_PATH = "/home/webapps/ArmsAgent2/arthas/arthas-boot.jar"
JAVA_PID=$(ps -ef |grep ${APP_NAME} |grep -v grep |awk '{print $2}')
TELNET_PORT=$(netstat -lnp |grep 127.0.0.1 |awk '{print $4}' |awk -F':' '{print $2}')

if [[ -n ${TELNET_PORT}]]; then
    java - jar ${ARTHAS_PATH} ${JAVA_PID} --telnet-port ${TELNET_PORT}
else 
    java -jar ${ARTHAS_PATH} ${JAVA_PID}  
fi
```

* check_health.sh

```shell
#!/bin/bash
http_code=("200",
"503")
if [[ -n "${http_code[@]}" = ~ `curl --connect-timeout 8 -m 8 -sL -w '%{http_code}' http://127.0.0.1:8080/actuator/health -o /dev/null` ]]; then
    echo "server health"
    exit 0
else 
    echo "check health result" : `curl --connect-timeout 8 -m 8 -sL  http://127.0.0.1:8080/actuator/health `
    echo "server not health"
    exit 1
fi
```

* check_ready.sh

```shell
#!/bin/bash
http_code=("200",
"503")
if [[ -n "${http_code[@]}" = ~ `curl --connect-timeout 8 -m 8 -sL -w '%{http_code}' http://127.0.0.1:8080/actuator/health -o /dev/null` ]]; then
    echo "server health"
    exit 0
else  
    echo "check health result" : `curl --connect-timeout 8 -m 8 -sL  http://127.0.0.1:8080/actuator/health `
    echo "server not health"
    exit 1
fi
```

* exit_pod.sh

```shell
#!/bin/bash
curl --connect-timeout 8 -m 8 -sL  http://127.0.0.1:22222/offline
sleep 20
ps -ef | grep java | awk '{print $2}' | awk 'NR==1' | xargs kill -15
http_code=("200",
"503")
for i in {1..60}
do
    if [[ -n "${http_code[@]}" = ~ `curl --connect-timeout 8 -m 8 -sL -w '%{http_code}' http://127.0.0.1:8080/actuator/health -o /dev/null` ]]; then
        echo "server running"
        exit 0
    else 
        echo "server stop success"
        exit 0
    fi
    sleep 3
done  
```

* fullgc_dump.sh

```shell
#!/bin/bash
export JAVA_TOOL_OPTIONS=
FGC_NUMD=0
FGC_NUM=0
tmp_file="/tmp/fullgc"
health_file="/tmp/tools/check_health.sh"
ready_file="/tmp/tools/ready_health.sh"

while true;do
    sleep 10
    [[ -f ${tmp_file} ]] && FGC_NUMD=$(cat ${tmp_file} |tail -1)
    FGC_NUM=$(jstat -gc 1 |awk '{print $15}' |tail -1)
    if [ ${FGC_NUM} -gt ${FGC_NUMD} ]; then
        mv ${health_file}  ${health_file}.bak
        echo  1 >  ${health_file}
        mv ${ready_file}  ${ready_file}.bak
        echo  0 >  ${ready_file}
        jmap -F -dump:live,format=b,file=./dump-${date +%F-%M-%S}.hprof 1
        echo ${FGC_NUM} >> ${tmp_file}
        rm -f ${health_file} ${ready_file}
        mv ${health_file}.bak  ${health_file}
        mv ${ready_file}.bak  ${ready_file}
    fi
done
  
```

### conf 目录

* cn-dev.sh

```shell
JAVA_OPTION="-DsetaEnv=dev -Dserver.port=8080 \
-Dapp.id=demo-cn-common -Dapollo.meta=http://it-apollo-dev.itlaohuo.top/ \
-Denv=dev -DserverAddr=it-seata-nacos.itlaohuo.top \
-Xmx8000M -Xms8000M -Xss512K -XX:MaxMetaspaceSize=1024M -XX:MetaspaceSize=1024M -XX:+UseG1GC \
-XX:MaxGCPauseMillis=200 -XX:G1HeapRegionSize=16M \
-Dit.log.switch=true \
-Dit.app.name=demo-common \
-Dit.app.env=dev \
-D.server.tomcat.max-thread=500 \
-Dnamespace=2506f038-7c4b-4af3-8fa1-c4915ac6694f -Dencrptionkey=23asdwee2343afwe \
-Duser.home=/data01/demo-common/logs"

```

* cn-pro.sh

```shell
JAVA_OPTION="-Dserver.port=8080 \
-Dapp.id=demo-cn-common  \ 
-Dapollo.meta=http://it-apollo-pro.itlaohuo.top/ \
-Dapollo.bootstarp.namespaces=application,v2 \
-Dapollo.clientAuthKeyName=d6e25b4a0cf4b60  -Denv=pro -Dnamespace=759623-xxxxxxx\
-DserverAddr=it-seata-nacos.itlaohuo.top -DsetaEnv=pro -Dencrptionkey=07cxxxxxxxx \
-Duser.home=/data01/democommon \
-Xmx21000M -Xms21000M -XX:MaxMetaspaceSize=1024M -XX:MetaspaceSize=1024M -XX:+UseG1GC \
-XX:MaxGCPauseMillis=200  -XX:+ParallelRefProcEnabled \
-XX:ErrorFile=/home/logs/gc/hs_err_pid%p.log -Xloggc:/home/gc/gc.log \
-XX:HeadDumpPath=/home/logs/gc/commondump \ 
-XX:+HeadDumpOnOutOfMemoryError \
-XX:+PrintGCDetails -XX:+PrintGCDateStamps \
-XX:+PrintGCApplicationConcurrentTime \
-XX:+PrintGCApplicationStoppedTime \
-XX:+PrintHeadAtGC \
-Dit.log.switch=true \
-Dit.app.name=demo-common \
-Dit.app.env=pro \
-D.rocket.mq.gray.switch=true \
-Ddubbo.labels=stag=v2 \
-Dxxl.job.executor.gray=false"
```

* cn-pro-grey.sh

```shell
JAVA_OPTION="-Dserver.port=8080 \
-Dapp.id=demo-cn-common  \ 
-Dapollo.meta=http://it-apollo-pro.itlaohuo.top/ \
-Dapollo.bootstarp.namespaces=application,v1 \
-Dapollo.clientAuthKeyName=d6e25b4a0cf4b60  -Denv=pro -Dnamespace=759623-xxxxxxx\
-DserverAddr=it-seata-nacos.itlaohuo.top -DsetaEnv=pro -Dencrptionkey=07cxxxxxxxx \
-Duser.home=/data01/democommon \
-Xmx22000M -Xms22000M -XX:MaxMetaspaceSize=1024M -XX:MetaspaceSize=1024M -XX:+UseG1GC \
-XX:MaxGCPauseMillis=200  -XX:+ParallelRefProcEnabled \
-XX:ErrorFile=/home/logs/gc/hs_err_pid%p.log -Xloggc:/home/gc/gc.log \
-XX:HeadDumpPath=/home/logs/gc/commondump \ 
-XX:+HeadDumpOnOutOfMemoryError \
-XX:+PrintGCDetails -XX:+PrintGCDateStamps \
-XX:+PrintGCApplicationConcurrentTime \
-XX:+PrintGCApplicationStoppedTime \
-XX:+PrintHeadAtGC \
-Dit.log.switch=true \
-Dit.app.name=demo-common-grey \
-Dit.app.env=pro \
-D.rocket.mq.gray.switch=true \
-Ddubbo.labels=stag=v1 \
-Dxxl.job.executor.gray=true"

```

### bin目录

* entrypoint.sh

```shell
#!/bin/bash
JAVA_DUBBO_GROUP=""
if [ $DUBBO_GROUP ];then
    JAVA_DUBBO_GROUP="-Ddubbo.provider.group=${DUBBO_GROUP} -Dbubbo.consumer.group=${DUBBO_GROUP}"
fi

fi [[$ENV ==in* ]] ;then
    ln -sf /usr/share/zoneinfo/Asia/Calcutta /etc/licaltime
elif [[$ENV ==eur-pro* ]] ;then
    ln -sf /usr/share/zoneinfo/Europe/Paris /etc/licaltime
fi 

mv /home/conf/${ENV}.sh /tmp/${ENV}.sh
rm -rf /home/conf/*.sh
mv /tmp/${ENV}.sh /home/conf/${ENV}.sh
source /home/conf/${ENV}.sh

JAVA_OPTION = "${JAVA_DUBBO_GROUP}" \
${JAVA_OPTION}

JAR_FILE= `find /home/*.jar | awk "{print $1}"`

exec java jar -server $JAVA_OPTION -jar $JAR_FILE

```

### devops 应用流水线

* 输入源

  > 代码源  git仓库地址 默认分支
  >
* 构建应用

  > 应用类型：JAVA
  > 构建类型：Maven
  > JAVA版本 工具版本
  > 构建命令：
  >

  ```shell
  docker run --rm -u 5001 -v `pwd`:/home/code it-registry.itlaohuo.top/libray/centos7-jdk1.8:1,0.16-arms-docker-build /bin/bash -c "export LC_ALL=en_US.UTF-8 && mvn -B clean deploy -U -Dmaven.test.skip=true -Dautoconfig.skip -Djob.skip -f ./pom.xml"
  ```
* 代码扫描

  > sonar
  >
* 构建镜像

  > Dockerfile 输入方式： 勾选 外部输入源（其他选项：本地输入源，自定义Dockerfile）
  > Dockerfile仓库：http://it-gitlab.itlaohuo.top/v-work/docker-file.git
  > 分支名称：master
  > Dockerfile路径：./demo-common/Dockerfile
  >
* 推送镜像
* 部署

  > 制品路径：使用上一步推送的镜像
  > 部署逻辑集群：
  > 命名空间：
  > 启动参数：
  > 环境变量：ENV=cn-dev
  > 停止前命令：sh /home/tools/exit_pod.sh
  > 规格：CPU-Request = 1.0 核 CPU-Limited = 1.0核心 内存—Request = 2.0GB 内存-Limited = 2.0GB ； 副本数= 2
  > 缩放/升级策略：
  > 健康检查：命令探针；sh /home/tools/check_health.sh ; 设定频率，超时时间，失败阈值
  >

