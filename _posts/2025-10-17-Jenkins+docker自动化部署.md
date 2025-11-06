---
layout: post
title:  "Jenkins+docker自动化部署"
date:   2025-10-17 10:29:20 +0800
categories:
      - docker
      - jenkins
tags:
      - 自动化部署
      - jenkinsfile
---

测试环境部署项目，使用jenkins+docker确实方便。

这里记录下搭建流程，以备后用。

### 1. 搞基建

首先是安装Jenkins，这里直接使用docker安装。原有的Jenkins官方镜像里没有docker

需要根据官方镜像打包一个自定义jenkins镜像，这个镜像里面整合了docker

dockerfile内容

```
FROM jenkins/jenkins:2.528.1-jdk21
USER root
RUN apt-get update && apt-get install -y lsb-release ca-certificates curl && \
    install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc && \
    chmod a+r /etc/apt/keyrings/docker.asc && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
    https://download.docker.com/linux/debian $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" \
    | tee /etc/apt/sources.list.d/docker.list > /dev/null && \
    apt-get update && apt-get install -y docker-ce-cli && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean docker-workflow json-path-api"
```

构建镜像命令

```shell
docker build -t myjenkins-blueocean:2.528.1-1 .
```

启动镜像,这里记得修改真实的docker_host地址以及暴露的端口

```bash
docker run \
  --name jenkins-blueocean \
  --restart=on-failure \
  --detach \
  --network jenkins \
  --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client \
  --env DOCKER_TLS_VERIFY=1 \
  --publish 8080:8080 \
  --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  myjenkins-blueocean:2.528.1-1
```


启动后访问Jenkins，并完成安装及初始化。


### 2. 项目配置


接着java项目也需要修改，需要新增两个文件:

- dockerfile, 放置路径为 project/src/main/docker/dockerfile
  主要是讲项目jar包打包成镜像，dockerfile文件内容如下
  
  ```bash
    FROM ubantu_jdk_ssh
    
    VOLUME /tmp
    
    ADD datasync-wn-0.0.1-SNAPSHOT.jar /app.jar
    
    ENTRYPOINT ["nohup","java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
  ```
- jenkinsfile, 放置路径为项目跟路径, project/jenkinsfile
  这里主要是Jenkins自动化脚本，自动构建以及启停容器。Jenkinsfile文件内容如下
  
  ```
        pipeline {
          agent none
          environment {
              // 定义容器名
              containername='myproject1'
              // 定义镜像名
              imagename='myproject/datasync:v1'
              // jar包名, 这里根据实际项目更改包名
              jarfilename='datasync-wn-0.0.1-SNAPSHOT.jar'
          }
          stages {
              stage('Build') {
                 agent {
                        docker {
                                image 'maven:3.9.11-sapmachine-25'
                                args '-v /root/.m2:/root/.m2'
                            }
                        }
                  steps {
                      sh 'mvn -B -DskipTests clean package'
                  }
              }
      
             stage('编译镜像') {
                 agent any
                  steps {
                      echo pwd()
                      echo '复制包'
                      sh 'cp ./target/$jarfilename ./src/main/docker/$jarfilename'
                      echo '停止旧服务,忽略错误'
                      sh 'docker stop $containername  || true'
                      echo '删除旧容器,忽略错误'
                      sh 'docker rm $containername  || true'
                      echo '删除旧镜像,忽略错误'
                      sh 'docker rmi $imagename || true'
                      echo '构建新镜像,忽略错误'
                      sh 'docker build -t $imagename src/main/docker'
                      echo '启动新容器,忽略错误'
                      // 这里启动时，根据实际情况暴露包名
                      sh 'docker run -d --name=$containername -p 8080:8080 $imagename'
                  }
              }
          }
      }
  ```

两个文件放置路径如图：

<img width="410" height="431" alt="image" src="https://github.com/user-attachments/assets/8f6a8888-3791-4865-b4e5-20e0854f4578" />

上述内容记得修改一些文件名及项目名部分。


### 3. Jenkins配置

打开Jenkins：
- 新建一个项目，选择流水线 pipline 。
- 配置页面中选择 pipline script from scm， 源选择git， 填入地址
- 点击完成

经过上诉步骤后，现在就可以点击构建部署了， Jenkins拉取项目后，自动执行项目根目录下的jenkinsfile脚本， 脚本将会编译并创建镜像启动。

基建是比较麻烦，不过搞好后就很好弄了。 所有项目只需要增加这两个配置文件即可完成自动化部署了。


参考文档：
- [Jenkins官方文档,Jenkins整合docker](https://www.jenkins.io/doc/book/installing/docker/)
- [Jenkins官方文档,Jenkins部署java项目](https://www.jenkins.io/zh/doc/tutorials/build-a-java-app-with-maven/)
