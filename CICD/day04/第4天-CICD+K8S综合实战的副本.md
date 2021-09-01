# 第4天 CI/CD+K8S综合实战

## 一、部署流程

1、研发push到github代码库

2、Jenkins 构建，pull git代码 使用maven进行编译打包

3、打包生成的代码，生成一个新版本的镜像，push到本地docker仓库harbor

4、发布，测试机器 pull 新版本的镜像，并删除原来的容器，重新运行新版本镜像。

## 二、环境说明

**服务及服务器说明-Aliyun环境**

> **1、代码仓库**			
>
> ​	github 或者 git-server 或者 gitlab
>
> ​	本次实验使用github仓库 https://github.com/
>
> ![image-20200512165810293](assets/image-20200512165810293.png)
>
> **2、容器镜像仓库**	
>
> ​	ip:
>
> ​	主机名：harbor
>
> **3、CI/CD服务器**
>
> ​	ip：
>
> ​	主机名：jenkins
>
> ​	软件：
>
> ​		jdk
>
> ​		jenkins
>
> ​		git
>
> ​		maven
>
> ​		docker
>
> **4、应用服务器**
>
> ​	ip：
>
> ​	主机名：docker
>
> ​	软件： 
>
> ​		jq
>
> ​		docker
>
> ​	或者 k8s集群

## 三、部署Harbor镜像仓库

### 1、下载安装

```bash
官方地址：
	https://github.com/goharbor/harbor/releases

下载离线安装包：需要翻墙
# wget https://storage.googleapis.com/harbor-releases/release-1.8.0/harbor-offline-installer-v1.8.0.tgz
# yum -y install  lrzsz

安装compose
# curl -L https://github.com/docker/compose/releases/download/1.22.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

# chmod +x /usr/local/bin/docker-compose

# tar xf harbor-offline-installer-v1.8.0.tgz

配置harbor
# cd harbor
# vim harbor.yml // 主机名要可以解析(需要部署dns服务器，用/etc/hosts文件没有用)，如果不可以解析，可以使用IP地址,需要修改的内容如下
hostname = 192.168.1.200
ui_url_protocol = https（如果要用https这里就需要改，现在我们先不用https，这里不需要改）
# ./install.sh
```

浏览器访问测试：http://192.168.1.200

![image-20200509173109582](assets/image-20200509173109582.png)

![image-20200509172959892](assets/image-20200509172959892.png)

创建仓库

![image-20200509173218474](assets/image-20200509173218474.png)

![image-20200509173227261](assets/image-20200509173227261.png)

创建账户

![image-20200509173242826](assets/image-20200509173242826.png)

![image-20200509173250977](assets/image-20200509173250977.png)

项目授权

![image-20200509173311987](assets/image-20200509173311987.png)

![image-20200509173319302](assets/image-20200509173319302.png)

### 2、测试Harbor

**上传测试**

```bash
 [root@docker ~]# docker login harbor.io
    Username: wing
    Password:
    WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
    Configure a credential helper to remove this warning. See
    https://docs.docker.com/engine/reference/commandline/login/#credentials-store
    
    Login Succeeded
    # docker images
    REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
    nginx               latest              be1f31be9a87        13 days ago         109MB
    
    # docker image tag daocloud.io/library/nginx:latest 172.22.211.175/jenkins/nginx

    # docker push 172.22.211.175/jenkins/nginx
    The push refers to repository [harbor.io/library/nginx]
    92b86b4e7957: Pushed
    94ad191a291b: Pushed
    8b15606a9e3e: Pushed
    latest: digest: sha256:204a9a8e65061b10b92ad361dd6f406248404fe60efd5d6a8f2595f18bb37aad size: 948

在web界面中查看镜像是否被上传到仓库中
```

### 【扩展】重置Harbor登陆密码

```bash
harbor现使用postgresql 数据库。不再支持mysql

注：
    卸载重新重新安装也不可以，原因是没有删除harbor的数据，harbor数据在/data/目录下边，如果真要重新安装需要将这个也删除，备份或者迁移，请使用这个目录的数据。

harbor版本为：1.8.0
官方的安装包为： harbor-offline-installer-v1.8.0.tgz

具体步骤：
1、进入[harbor-db]容器内部
     # docker exec -it harbor-db /bin/bash

2、进入postgresql命令行，
     psql -h postgresql -d postgres -U postgres  #这要输入默认密码：root123 。
     psql -U postgres -d postgres -h 127.0.0.1 -p 5432  #或者用这个可以不输入密码。

3、切换到harbor所在的数据库
     # \c registry

4、查看harbor_user表
     # select * from harbor_user;

5、例如修改admin的密码，修改为初始化密码Harbor12345 ，修改好了之后再可以从web ui上再改一次。
     # update harbor_user set password='a71a7d0df981a61cbb53a97ed8d78f3e', salt='ah3fdh5b7yxepalg9z45bu8zb36sszmr'  where username='admin';

6、退出 \q 退出postgresql，exit退出容器。
     # \q 
     # exit 

完成后通过WEB UI，就可以使用admin 、Harbor12345 这个密码登录了，记得修改这个默认密码哦，避免安全问题。

有更加狠点的招数，将admin账户改成别的名字，减少被攻击面：
     # update harbor_user set username='wing' where user_id=1;              #更改admin用户名为wing

```



### 3、Dockerfile文件

```yaml
# cd /root/jenkins/docker-file/maven-docker-test_war
# vim Dockerfile
# Version 1.0
# Base images. 
FROM tomcat:8.0.36-alpine

# Author.
MAINTAINER wing <276267003@qq.com>

# Add war.
ADD maven-docker-test.war /usr/local/tomcat/webapps/

# Define working directory.
WORKDIR /usr/local/tomcat/bin/

# Define environment variables.
ENV PATH /usr/local/tomcat/bin:$PATH

# Define default command. 
CMD ["catalina.sh", "run"]

# Expose ports.
EXPOSE 8080
```

### 4、Harbor权限相关

harbor仓库的权限得配置一下，不然curl命令访问不到

![image-20200509174138114](assets/image-20200509174138114.png)

## 四、业务服务器

### 1、安装软件

```bash
# yum install docker jq -y   //后面的脚本会用到,jq类似于sed/awk专门处理json格式的文件
# systemctl start docker
```



### 2、预先配置

```bash
在业务机器上配置：
# visudo
#
#Defaults    requiretty
Defaults:root !requiretty

否则在机器业务机器上执行脚本时会报错：
[SSH] executing...
sudo: sorry, you must have a tty to run sudo
docker: invalid reference format.
```



## 五、Jenkins服务部署配置

### 1、软件安装

```bash
# yum install java git maven -y
安装插件：Maven Integration
```



### 2、预先配置

```bash
由于在Jenkins机器上docker是使用root用户运行的，而Jenkins是使用普通用户jenkins运行的，所以要先配置下jenkins用户可以使用docker命令。
[root@jenkins ~]# visudo
jenkins ALL=(root)      NOPASSWD: /usr/bin/docker

另外在Jenkins机器上配置：

# Disable "ssh hostname sudo <cmd>", because it will show the password in clear.
#         You have to run "ssh -t hostname sudo <cmd>".
#
#Defaults    requiretty
Defaults:jenkins !requiretty

如果不配置这个，在执行下面脚本时，会报错误：
+ cp -f /home/jenkins/.jenkins/workspace/godseyeBranchForNov/godseye-container/target/godseye-container-wisedu.war /home/jenkins/docker-file/godseye_war/godseye.war
+ sudo docker login -u jkzhao -p Wisedu123 -e 01115004@wisedu.com 172.16.206.32
sudo: sorry, you must have a tty to run sudo
```

### 3、安装插件

登录Jenkins，点击“系统管理”，点击“管理插件”，搜索插件“SSH plugin”，进行安装。

### 4、配置远程机器

登录Jenkins，点击“Credentials”，点击“Add domain”。

![image-20200512101026784](assets/image-20200512101026784.png)

![image-20200512101109339](assets/image-20200512101109339.png)

![image-20200512101120858](assets/image-20200512101120858.png)

![image-20200512101145738](assets/image-20200512101145738.png)

点击“系统管理”，“系统配置”，找到“SSH remote hosts”。

![image-20200512101210294](assets/image-20200512101210294.png)



## 六、Jenkins构建Job

###  1、构建Maven风格的Job

代码地址： https://github.com/yanqiang20172017/easy-springmvc-maven.git

![image-20200512101348331](assets/image-20200512101348331.png)

Goals and options填写：clean package -Dmaven.test.skip=true

![image-20200512101411044](assets/image-20200512101411044.png)

### 2、配置Post Steps

注：脚本中用到的仓库和认证的账号需要先在harbor新建好。

![image-20200512101638389](assets/image-20200512101638389.png)



```bash
# Jenkins机器：编译完成后，build生成一个新版本的镜像，push到远程docker仓库
 
# Variables
JENKINS_WAR_HOME='/root/.jenkins/workspace/maven-docker-test/target'
DOCKERFILE_HOME='/root/jenkins/docker-file/maven-docker-test_war'
HARBOR_IP='172.22.211.175'
REPOSITORIES='jenkins/maven-docker-test'
HARBOR_USER='wing'
HARBOR_USER_PASSWD='Harbor12345'
HARBOR_USER_EMAIL='276267003@qq.com'
 
# Copy the newest war to docker-file directory.
\cp -f ${JENKINS_WAR_HOME}/easy-springmvc-maven.war ${DOCKERFILE_HOME}/maven-docker-test.war
 
# Delete image early version.
sudo docker login -u ${HARBOR_USER} -p ${HARBOR_USER_PASSWD} ${HARBOR_IP} 
IMAGE_ID=`sudo docker images | grep ${REPOSITORIES} | awk '{print $3}'`
if [ -n "${IMAGE_ID}" ];then
    sudo docker rmi ${IMAGE_ID}
fi
 
# Build image.
cd ${DOCKERFILE_HOME}
TAG=`date +%Y%m%d-%H%M%S`
sudo docker build -t ${HARBOR_IP}/${REPOSITORIES}:${TAG} . &>/dev/null
 
# Push to the harbor registry.
sudo docker push ${HARBOR_IP}/${REPOSITORIES}:${TAG} &>/dev/null
```

注：war包的名字为git项目的名字
/root/.jenkins/workspace/maven-docker-test/target/easy-springmvc-maven.war

**拉取镜像、发布**

![image-20200512101901486](assets/image-20200512101901486.png)

![image-20200512101918476](assets/image-20200512101918476.png)



```bash
# 拉取镜像，发布
HARBOR_IP='172.22.211.175'
REPOSITORIES='jenkins/maven-docker-test'
HARBOR_USER='wing'
HARBOR_USER_PASSWD='Harbor12345'
 
# 登录harbor
docker login -u ${HARBOR_USER} -p ${HARBOR_USER_PASSWD} ${HARBOR_IP}
 
# Stop container, and delete the container.
CONTAINER_ID=`docker ps | grep "maven-docker-test" | awk '{print $1}'`
if [ -n "$CONTAINER_ID" ]; then
    docker stop $CONTAINER_ID
    docker rm $CONTAINER_ID
else #如果容器启动时失败了，就需要docker ps -a才能找到那个容器
    CONTAINER_ID=`docker ps -a | grep "maven-docker-test" | awk '{print $1}'`
    if [ -n "$CONTAINER_ID" ]; then  # 如果是第一次在这台机器上拉取运行容器，那么docker ps -a也是找不到这个容器的
        docker rm $CONTAINER_ID
    fi
fi
 
# Deleteeasy-springmvc-maven image early version.
IMAGE_ID=`sudo docker images | grep ${REPOSITORIES} | awk '{print $3}'`
if [ -n "${IMAGE_ID}" ];then
    docker rmi ${IMAGE_ID}
fi
 
# Pull image.
# TAG=`curl -s http://${HARBOR_IP}/api/repositories/${REPOSITORIES}/tags | jq '.[-1]' | sed 's/\"//g'` 
TAG=`curl -s http://172.22.211.175/api/repositories/jenkins/maven-docker-test/tags | jq '.[-1]| {name:.name}' | awk -F '"' '/name/{print $4}'`
docker pull ${HARBOR_IP}/${REPOSITORIES}:${TAG} &>/dev/null
 
# Run.
docker run -d --name maven-docker-test -p 8080:8080 ${HARBOR_IP}/${REPOSITORIES}:${TAG}
```

### 3、构建

![image-20200512102013778](assets/image-20200512102013778.png)



### 4、控制台输出过程

```bash
执行中控制台输出
Started by user admin
Building on master in workspace /root/.jenkins/workspace/maven-docker-test
No credentials specified
 > git rev-parse --is-inside-work-tree # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://github.com/yanqiang20172017/easy-springmvc-maven.git # timeout=10
Fetching upstream changes from https://github.com/yanqiang20172017/easy-springmvc-maven.git
 > git --version # timeout=10
 > git fetch --tags --progress https://github.com/yanqiang20172017/easy-springmvc-maven.git +refs/heads/*:refs/remotes/origin/*
 > git rev-parse refs/remotes/origin/master^{commit} # timeout=10
 > git rev-parse refs/remotes/origin/origin/master^{commit} # timeout=10
Checking out Revision 67604f7f9f30505e3bb3e8935c745154f04aa372 (refs/remotes/origin/master)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f 67604f7f9f30505e3bb3e8935c745154f04aa372
Commit message: "修改standard/1.1.2的依赖"
 > git rev-list --no-walk 67604f7f9f30505e3bb3e8935c745154f04aa372 # timeout=10
Parsing POMs
Established TCP socket on 36798
[maven-docker-test] $ java -cp /root/.jenkins/plugins/maven-plugin/WEB-INF/lib/maven3-agent-1.12.jar:/usr/share/maven/boot/plexus-classworlds.jar org.jvnet.hudson.maven3.agent.Maven3Main /usr/share/maven /root/.jenkins/war/WEB-INF/lib/remoting-3.29.jar /root/.jenkins/plugins/maven-plugin/WEB-INF/lib/maven3-interceptor-1.12.jar /root/.jenkins/plugins/maven-plugin/WEB-INF/lib/maven3-interceptor-commons-1.12.jar 36798
<===[JENKINS REMOTING CAPACITY]===>channel started
Executing Maven:  -B -f /root/.jenkins/workspace/maven-docker-test/pom.xml clean package -Dmaven.test.skip=true
[INFO] Scanning for projects...
[WARNING] 
[WARNING] Some problems were encountered while building the effective model for springmvc-maven:easy-springmvc-maven:war:0.0.1-SNAPSHOT
[WARNING] 'build.plugins.plugin.version' for org.apache.maven.plugins:maven-war-plugin is missing. @ line 22, column 15
[WARNING] 
[WARNING] It is highly recommended to fix these problems because they threaten the stability of your build.
[WARNING] 
[WARNING] For this reason, future Maven versions might no longer support building such malformed projects.
[WARNING] 
[INFO]                                                                         
[INFO] --------------------------------------------------------------
[INFO] Building springmvc-maven 0.0.1-SNAPSHOT
[INFO] --------------------------------------------------------------
[INFO] 
[INFO] --- maven-clean-plugin:2.4.1:clean (default-clean) @ easy-springmvc-maven ---
[INFO] Deleting /root/.jenkins/workspace/maven-docker-test/target
[INFO] 
[INFO] --- maven-resources-plugin:2.5:resources (default-resources) @ easy-springmvc-maven ---
[debug] execute contextualize
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory /root/.jenkins/workspace/maven-docker-test/src/main/resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ easy-springmvc-maven ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 2 source files to /root/.jenkins/workspace/maven-docker-test/target/classes
[INFO] 
[INFO] --- maven-resources-plugin:2.5:testResources (default-testResources) @ easy-springmvc-maven ---
[debug] execute contextualize
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory /root/.jenkins/workspace/maven-docker-test/src/test/resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.1:testCompile (default-testCompile) @ easy-springmvc-maven ---
[INFO] Not compiling test sources
[INFO] 
[INFO] --- maven-surefire-plugin:2.10:test (default-test) @ easy-springmvc-maven ---
[INFO] Tests are skipped.
[WARNING] Attempt to (de-)serialize anonymous class hudson.maven.reporters.BuildInfoRecorder$1; see: https://jenkins.io/redirect/serialization-of-anonymous-classes/
[INFO] 
[INFO] --- maven-war-plugin:2.1.1:war (default-war) @ easy-springmvc-maven ---
[INFO] Packaging webapp
[INFO] Assembling webapp [easy-springmvc-maven] in [/root/.jenkins/workspace/maven-docker-test/target/easy-springmvc-maven]
[INFO] Processing war project
[INFO] Copying webapp resources [/root/.jenkins/workspace/maven-docker-test/src/main/webapp]
[INFO] Webapp assembled in [43 msecs]
[INFO] Building war: /root/.jenkins/workspace/maven-docker-test/target/easy-springmvc-maven.war
[INFO] WEB-INF/web.xml already added, skipping
[INFO] --------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] --------------------------------------------------------------
[INFO] Total time: 2.647s
[INFO] Finished at: Sun Jun 09 16:12:01 CST 2019
[INFO] Final Memory: 19M/189M
[INFO] --------------------------------------------------------------
Waiting for Jenkins to finish collecting data
[JENKINS] Archiving /root/.jenkins/workspace/maven-docker-test/pom.xml to springmvc-maven/easy-springmvc-maven/0.0.1-SNAPSHOT/easy-springmvc-maven-0.0.1-SNAPSHOT.pom
[JENKINS] Archiving /root/.jenkins/workspace/maven-docker-test/target/easy-springmvc-maven.war to springmvc-maven/easy-springmvc-maven/0.0.1-SNAPSHOT/easy-springmvc-maven-0.0.1-SNAPSHOT.war
[maven-docker-test] $ /bin/sh -xe /tmp/jenkins6873694180184993727.sh
channel stopped
+ JENKINS_WAR_HOME=/root/.jenkins/workspace/maven-docker-test/target
+ DOCKERFILE_HOME=/root/jenkins/docker-file/maven-docker-test_war
+ HARBOR_IP=172.22.211.175
+ REPOSITORIES=jenkins/maven-docker-test
+ HARBOR_USER=wing
+ HARBOR_USER_PASSWD=Harbor12345
+ HARBOR_USER_EMAIL=276267003@qq.com
+ cp -f /root/.jenkins/workspace/maven-docker-test/target/easy-springmvc-maven.war /root/jenkins/docker-file/maven-docker-test_war/maven-docker-test.war
+ sudo docker login -u wing -p Harbor12345 172.22.211.175
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
++ sudo docker images
++ grep jenkins/maven-docker-test
++ awk '{print $3}'
+ IMAGE_ID=a6676d1090aa
+ '[' -n a6676d1090aa ']'
+ sudo docker rmi a6676d1090aa
Untagged: 172.22.211.175/jenkins/maven-docker-test:20190609-153307
Untagged: 172.22.211.175/jenkins/maven-docker-test@sha256:a0dbff5acc2284554aef955e32cb3325b0716c13ec66ec70b0705fe17d425d8b
Deleted: sha256:a6676d1090aa51212861be156d81e2e96ab7ffb8ace2f8e4428debd1404ba8dd
Deleted: sha256:675a50c43c8b6b890dd86f9ef7958f849956d922735a3a721741af7bfcc0a50d
Deleted: sha256:fb5bd69c508668d13c4da89770d8c54fbc7e330c45543a80be52dc94a021730e
Deleted: sha256:7e47412dc00e4b0e753cf076ac4b97bbc972c524aeae4243ef4c441a7bf9d8e4
Deleted: sha256:2e6bb2e854e3a6d14b35797198758503dcd924a1c365e653daef5708659f3b6c
Deleted: sha256:e036e0839cbd1676552163ab0001e9e968ef924d2d17b0b7eee245ce29f35f6b
Deleted: sha256:1f35ccae7e2fec67145bd7b3f8b50d7f0a66f3386cc1d444136e24c8ef667763
+ cd /root/jenkins/docker-file/maven-docker-test_war
++ date +%Y%m%d-%H%M%S
+ TAG=20190609-161201
+ sudo docker build -t 172.22.211.175/jenkins/maven-docker-test:20190609-161201 .
+ sudo docker push 172.22.211.175/jenkins/maven-docker-test:20190609-161201
[SSH] script:
USER="root"

# 拉取镜像，发布
HARBOR_IP='172.22.211.175'
REPOSITORIES='jenkins/maven-docker-test'
HARBOR_USER='wing'
HARBOR_USER_PASSWD='Harbor12345'
 
# 登录harbor
docker login -u ${HARBOR_USER} -p ${HARBOR_USER_PASSWD} ${HARBOR_IP}
 
# Stop container, and delete the container.
CONTAINER_ID=`docker ps | grep "maven-docker-test" | awk '{print $1}'`
if [ -n "$CONTAINER_ID" ]; then
    docker stop $CONTAINER_ID
    docker rm $CONTAINER_ID
else #如果容器启动时失败了，就需要docker ps -a才能找到那个容器
    CONTAINER_ID=`docker ps -a | grep "maven-docker-test" | awk '{print $1}'`
    if [ -n "$CONTAINER_ID" ]; then  # 如果是第一次在这台机器上拉取运行容器，那么docker ps -a也是找不到这个容器的
        docker rm $CONTAINER_ID
    fi
fi
 
# Deleteeasy-springmvc-maven image early version.
IMAGE_ID=`sudo docker images | grep ${REPOSITORIES} | awk '{print $3}'`
if [ -n "${IMAGE_ID}" ];then
    docker rmi ${IMAGE_ID}
fi
 
# Pull image.
# TAG=`curl -s http://${HARBOR_IP}/api/repositories/${REPOSITORIES}/tags | jq '.[-1]' | sed 's/\"//g'` 
TAG=`curl -s http://172.22.211.175/api/repositories/jenkins/maven-docker-test/tags | jq '.[-1]| {name:.name}' | awk -F '"' '/name/{print $4}'`
docker pull ${HARBOR_IP}/${REPOSITORIES}:${TAG} &>/dev/null
 
# Run.
docker run -d --name maven-docker-test -p 8080:8080 ${HARBOR_IP}/${REPOSITORIES}:${TAG}

[SSH] executing...
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
afa55c8e5fe64bc2f652fdc3813f077173baff4e8cfd4301c5c49c75bdb3d953
[SSH] completed
[SSH] exit-status: 0

Finished: SUCCESS
```

