# Docker Ctrip Apollo.
使用 Docker 部署携程 Apollo，为提高扩展性拆为 3 个 Docker Image。

#### 部署说明：

1. 修改官方项目 apollo-configservice 、apollo-adminservice 项目下 application.yml 配置文件，此处可参考[官网文档](https://github.com/ctripcorp/apollo/wiki/分布式部署指南#14网络策略)：

   ```yaml
   spring:
       application:
           name: apollo-configservice
       profiles:
       	active: ${apollo_profile}
       cloud:
           inetutils:
               ignoredInterfaces:
               - docker0
               - veth.*
   eureka:
       instance:
           preferIpAddress: true
           ip-address: ${ip}
   ```


2. 按照官方文档配置完成后，由于官方版本迭代，需自行修改 Dockerfile 中 VERSION，然后执行 ./build.sh 编译打包。

3. 下载并解压本项目到官方项目根目录，分别编译 apollo-configservice 、apollo-adminservice 、apollo-portal，编译及运行说明可参考下方或目录中 Dockerfile 文件。

   ```shell
   # Build with:
   docker build -t apollo-configservice .
   # Run with:
   docker run -p 8080:8080 -d --name apollo-configservice apollo-configservice
   ```

4. 访问 http://IP:PORT 查看部署是否成功。

==================================================

# Docker-compose Ctrip Apollo.
使用Docker-compose方式将一些配置信息提到外面，这样就不用我们每次都修改源码打包了， 方便了部署。

1、启动时修改compose文件,以adminservice为例
```
version: "3"
services:
  apollo-adminservice:
    container_name: apollo-adminservice
    build: apollo-adminservice/
    image: apollo-adminservice
    ports:
      - 8090:8090
    volumes:
      - "/docker/apollo/logs/100003172:/opt/logs/100003172"
    environment:
      - spring_datasource_url=jdbc:mysql://xxx:8306/ApolloConfigDB_TEST?characterEncoding=utf8
      - spring_datasource_username=xxx
      - spring_datasource_password=xxxxxx
```
- 1、build 参数为dockerfile路径
- 2、volumes : 为日志文件挂载到本地目录， 格式 本地目录：容器内目录
- 3、environment 配置数据库的连接信息

configservice的compose文件修改和adminservice基本相同。下面我们单独说下portal的

2、portal 的compose 文件修改注意项
```
version: "3"
services:
  apollo-portal:
    container_name: apollo-portal
    build: apollo-portal/
    image: apollo-portal
    ports:
      - 8070:8070
    volumes:
      - "/docker/apollo/logs/100003173:/opt/logs/100003173"
      - "Apollo/docker-image/apollo-portal/config/apollo-env.properties:/apollo-portal/config/apollo-env.properties"
    environment:
      - spring_datasource_url=jdbc:mysql://xx.xx.xx.xx:8306/ApolloPortalDB?characterEncoding=utf8
      - spring_datasource_username=root
      - spring_datasource_password=xxxxxx
```
其它和adminservice的相同，特别注意以下事项
- 1、volumes 将apollo-env.properties文件挂载到容器外部，特别注意先创建好物理机的目录、将修改好的apollo-env.properties文件放到对应的目录中，在启动执行启动。（不要出现挂在一个不存在文件地址）

如果不想分开启动，也可以执行apollo-compose.yml，不过里面改修改的还是和单个修改的地方相同。

3、启动命令
1、首次启动需要build镜像
```
docker-compose -f apollo-compose.yml up --build -d
```

在启动只需要
```
docker-compose -f apollo-compose.yml up -d
```

20190911
-------------------------------------------
在使用docker搭建apollo集群时，发现adminservice、configservice、的注册地址为docker容器内部
地址，导致使用时网络不通的问题。

故，将添加了 eureka.instance.ip-address 环境变量的配置，
我们在打包时（adminservice,configservice）时需要在bootstrap.yml中添加
```
eureka:
  instance:
    ip-address: ${eureka.instance.ip-address}
```

在apollo-compose.yml（adminservice,configservice）的环境变量中添加
```
eureka.instance.ip-address=内网地址
```
