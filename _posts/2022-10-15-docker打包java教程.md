---
layout: post
title:  "docker打包java教程"
date:   2022-10-15 10:29:20 +0800
categories:
      - java
      - docker
tags:
      - docker打包
---

公司项目有打包成docker的需求。这里记录下操作流程，记录下。

前提：

- 电脑或打包环境有安装docker
- 使用jdk17
- 使用maven依赖管理
- docker开启2375端口并允许访问

### 1. 修改项目

在java项目, main 同级目录中新增 `docker` 目录, 目录下新增`dockerfile`文件, 内容如下:

```docker
FROM wojiaojsjzs123/jdk17

VOLUME /tmp

ADD datasync-wn-0.0.1-SNAPSHOT.jar /app.jar

ENTRYPOINT ["nohup","java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

第一行是使用的基本镜像, 第3行是需要打包的目标包名, 这两个可以根据实际情况调整。

基础镜像可以使用`docker search ` 命令来查找，比如我要查找 jdk17的基础镜像 ，则使用`docker search jdk17`。


目录结构如下：

<img width="344" height="356" alt="image" src="https://github.com/user-attachments/assets/e6f3761f-53ed-4517-85ae-dcdf4d39e3f1" />


接着修改 mavem 配置，pom中需要做以下两个修改:

1. 新增一个配置`docker.image.prefix`,内容如下:

   ```xml
     <properties>
        <java.version>17</java.version>
        <docker.image.prefix>springio</docker.image.prefix>
      </properties>
   ```
   

2. 新增一个编译插件，也就是build下新增一个插件，内容如下:
    ```xml
            <plugin>
                    <groupId>com.spotify</groupId>
                    <artifactId>docker-maven-plugin</artifactId>
                    <version>0.4.14</version>
                    <configuration>
                        <imageName>${docker.image.prefix}/${project.artifactId}</imageName>
                        <!--指定docker镜像的版本号-->
                        <imageTags>
                            <!--使用maven项目的版本号-->
                            <imageTag>${project.version}</imageTag>
                            <imageTag>latest</imageTag>
                        </imageTags>
                        <dockerDirectory>src/main/docker</dockerDirectory>
                        <resources>
                            <resource>
                                <targetPath>/</targetPath>
                                <directory>${project.build.directory}</directory>
                                <include>${project.build.finalName}.jar</include>
                            </resource>
                        </resources>
                    </configuration>
                    <dependencies>
                      <!--这个依赖可要可不要, 如果打包报错可以打开-->
                      <dependency>
                          <groupId>javax.activation</groupId>
                          <artifactId>activation</artifactId>
                          <version>1.1.1</version>
                      </dependency>
                    </dependencies>
                </plugin>
    ```

### 2. 打包项目

经过以上修改, 项目已经改造完成。再经过以下流程即可启动服务和镜像。

- 切换到项目跟目录执行 `mvn package  docker:build` 打包镜像
- 执行`docker images ls` 可以看到打包的新增的镜像
- 执行`docker run -d --restart always -p 8080:8080 d38e49d7` 启动镜像
- 使用浏览器访问服务

上面第三步骤命令中, `-d` 为后台启动, `--restart always` 为自动启动，比如开机自动重启。 `-p 8080:8080` 为将端口进行映射，格式为`-p 本地端口:容器端口`


#### 3. 结论

这样打包其实还是挺简单的，主要是可以结合 jenkins 自动发布到测试环境之类的，很方便。早用早舒心

