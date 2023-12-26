


### 使用docker maven插件制作springboot镜像


***因为使用M1电脑中间遇到了很多问题,下面会记录遇到的问题***

- maven 构建镜像配置

pom.xml 文件添加docker插件<br>
***一开始使用com.spotify插件,会出现下面现象***
-----
  错误：

引起原因：com.spotify.docker.client.shaded.javax.ws.rs.ProcessingException：java.lang.UnsatisfiedLinkError：无法加载 FFI 提供程序 jnr.ffi.provider.jffi.Provider

引起：java.lang.UnsatisfiedLinkError：java.lang.UnsatisfiedLinkError：/private/var/folders/hz/rgppp8250rsdp86kf_tfjvqw0000gp/T/jffi8502916075702391528.dylib：dlopen(/private/var/folders/hz/rgppp8250rs dp86kf_tfjvqw0000gp/T/jffi8502916075702391528.dylib ，0x0001）：尝试：'/private/var/folders/hz/hz/rgppp8250rsdp86kf_tf_tfjvqw0000gp/t/jffi850291602916075702391528.dylib' /lib/jffi8502916075702391528.dylib'（没有这样的文件）

---
参考:https://stackoverflow.com/questions/71300031/docker-image-build-failed-on-mac-m1-chip  <br>
另外com.spotify插件已经不再维护了
后来使用io.fabric8 maven插件
但是低版本已经会出现这个问题
使用最新的0.43.4解决了这个问题
```
<build>
        <plugins>
            <plugin>
                <groupId>io.fabric8</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>0.43.4</version>
                <configuration>
                    <dockerHost>unix:///Users/mac_1/.docker/run/docker.sock</dockerHost>
                    <registry>${registry.host}</registry>
                    <authConfig>
                        <username>${registry.username}</username>
                        <password>${registry.password}</password>
                    </authConfig>
                    <images>
                        <image>
                            <name>${registry.host.name}/${docker.project.build.finalName}:${buildVersion}</name>
                            <registry>${registry.host}</registry>
                            <build>
                                <dockerFile>${project.basedir}/Dockerfile</dockerFile>
                            </build>
                        </image>
                    </images>
                </configuration>
            </plugin>
        </plugins>
    </build>
```

- 添加dockerfile文件

再pom同级目录添加dockerfile文件

```
# 基础镜像
FROM java:8
# 作者信息
MAINTAINER "DS"
# 添加一个存储空间
VOLUME /tmp
# 暴露8080端口
EXPOSE 8080
ARG JAR_FILE=target/spring-boot-docker.jar
# 往容器中添加jar包
ADD ${JAR_FILE} app.jar

# 启动镜像自动运行程序
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/urandom","-jar","/app.jar"]
```
之后执行
mvn package (打jar包)<br>
docker build (制作镜像)<br>
docker pull (推镜像到私有仓库)<br>

***其中<registry>${registry.host}</registry>中的registry.host
是使用docker-registry ,官方提供的工具，可以用于构建私有的镜像仓库。
```
docker run -d  -p 5000:5000 -v /Users/mac_1/data/docker/registry:/var/lib/registry  registry
```

###  Jenkins + Docker + Github 实现自动化部署 Maven 项目

![img_1.png](https://tc.bian666.cf/file/3953fefe7aa9b98348ce1.png)
#### 安装Jenkins

##### 1、docker安装jenkins

```
docker run --hostname=b50b06a2b5c3 --user=root --mac-address=02:42:ac:11:00:02 --env=LANG=C.UTF-8 --env=JENKINS_HOME=/var/jenkins_home --env=JENKINS_SLAVE_AGENT_PORT=50000 --env=PATH=/opt/java/openjdk/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin --env=JENKINS_VERSION=2.435 --env=JENKINS_UC=https://updates.jenkins.io --env=JENKINS_INCREMENTALS_REPO_MIRROR=https://repo.jenkins-ci.org/incrementals --env=REF=/usr/share/jenkins/ref --env=COPY_REFERENCE_FILE_LOG=/var/jenkins_home/copy_reference_file.log --env=JENKINS_UC_EXPERIMENTAL=https://updates.jenkins.io/experimental --env=JAVA_HOME=/opt/java/openjdk --volume=/Users/mac_1/data/docker/jenkins/home:/var/jenkins_home --volume=/run/host-services/docker.proxy.sock:/var/run/docker.sock --volume=/Users/mac_1/data/docker/jenkins/software:/var/software --volume=/var/jenkins_home --privileged -p 9091:50000 -p 9090:8080 --restart=always --label='org.opencontainers.image.description=The Jenkins Continuous Integration and Delivery server' --label='org.opencontainers.image.licenses=MIT' --label='org.opencontainers.image.revision=5ef64f1eb4b16ba4640d8a4bedca7ac660aec41a' --label='org.opencontainers.image.source=https://github.com/jenkinsci/docker' --label='org.opencontainers.image.title=Official Jenkins Docker image' --label='org.opencontainers.image.url=https://www.jenkins.io/' --label='org.opencontainers.image.vendor=Jenkins project' --label='org.opencontainers.image.version=2.435' --runtime=runc -t -d jenkins/jenkins:jdk11
```
***上面需要注意的点*** <br>
1、挂载jenkins目录 --volume=/Users/mac_1/data/docker/jenkins/home:/var/jenkins_home  --volume=/Users/mac_1/data/docker/jenkins/software:/var/software <br>
2、挂载docker目录(容器内连接宿主机docker server)   --volume=/run/host-services/docker.proxy.sock:/var/run/docker.sock <br>
##### 2、jenkins容器安装docker客户端
```
apt-get update
apt-get install -y docker.io
```

##### 3、配置jenkins和 gitHub
参考文档: https://juejin.cn/post/7127302949797101604#heading-8 <br>

##### 4、视频文档

```
jenkins
现在很多公司,都不需要专业的运维来进行项目部署,
都使用了自动化部署流程,
开发完代码,直接就可以低成本部署
今天来给大家介绍并操作一下常规的自动化部署流程
比如这里是一个springboot项目
我随便修改一下代码
提交推送到github
进入jenkins就看到一个构建任务正在执行
完成后我们就看到项目的容器已经运行起来了
先用一张图简单说一下自动化部署的流程
我们一般将项目代码推送托管平台
在gitlab或者github平台设置钩子
通知另一个重要组件jenkins
jenkins可以自动化完成软件的持续构建和测试
通知jenkins后,jenkins拉去项目代码
安装构建项目所需要的环境
构建成功后,把项目包通过ssh远程连接工具推送到各个服务器,运行测试
整个过程疏通明白后
我们来实践一下
首先来安装一下jenkins
整个演示流程,选择在docker中进行
为了节省时间,就不再编写启动命令了
直接复制之前的启动命令
我这里之前已经部署过jenkins容器,这里我先把它给删除了
这是一个比较简单docker容器启动命令
映射的端口是9090
因为我本地已经有镜像了所以这里启动比较快
使用docker ps发现容器已经启动起来了
然后在浏览器输入服务地址
刚启动第一次反应会比较慢
这里需要密码才能进入
这需要进入容器才能查看这个密码
进入容器后在这个地址下找到密码
在这里需要安装插件
可以先按照建议的插件安装
创建自己的管理员账号密码
从流程图上知道我们需要jenkins去托管平台Github上拉取代码
为让两个机器之间使用ssh不需要用户名和密码。
采用了数字签名RSA来完成这个操作
这里使用ssh-keygen工具来生成密钥和公钥
复制公钥在github上添加认证key
完成后,我们可以测试一下
发现可以成功拉去代码
然后就是配置项目的构建环境
一个springboot项目构建需要maven和jdk
在jenkins管理这里进入插件管理
下载需要的插件maven
我下载的jenkins镜像有自带的jdk
就不需要额外下载了
然后设置tools
使用默认的maven settings文件
jdk就不需要配置了
选择自己去安装maven
版本是默认的3.9.6

下面就是配置GitHub
提交代码时触发Jenkins自动构建
这里需要在GitHub上添加Jenkins的webhook
因为我们的jenkins是在本机上部署
github想要访问本地服务
就需要做一个内网穿透
就可以在外部网络环境中访问内网服务
这里使用我们之前讲过的内网穿透工具natapp
复制穿透隧道的authtoken
启动客户端
然后就映射到这个网址上面
访问测试一下
继续配置GitHub webhook
填写刚才的映射地址加上后缀github-webhook
这里的触发机制
就选择默认的推送事件

然后在GitHub上创建一个access token
Jenkins做一些需要权限的操作
的时候就用这个access token去鉴权
权限这里主要设置hook就行
复制token,马上在jenkins使用

接着进入jenkins设置系统配置Configure System
添加github server
添加验证凭据
把刚才的token复制进去
然后测试一下

下面开始正式的在jenkins上创建任务
就是推送代码后触发的任务
创建任务
类型选择maven
在源码管理这里添加项目的地址
触发的分支
在触发器这里选择GitHub hook
然后在后置步骤这里
设置操作
先使用maven打包
clean package
按照正常的流程,
我们这里可以设置ssh,
把包传到服务器上运行
这里只做演示
就在本地运行了
使用shell命令
这是一个比较简单的shell
就是到jenkins的项目工作目录下
启动docker镜像
因为我项目下有已经写好的dockerfile文件
直接使用dockerfile启动
下面来测试一下
随便修改一下代码
提交推送到github
进入jenkins就看到一个构建任务正在执行
构建成功后
查看容器已经成功运行起来了
```

docker-registry参考文档:https://yeasy.gitbook.io/docker_practice/repository/registry <br>
docker fabric8 maven插件参考文档: https://blog.lonelyman.site/archives/35 <br>
fabric8官方文档:https://dmp.fabric8.io/ <br>
docker官方文档:https://docs.docker.com/guides/get-started/ <br>
dockerFile :https://docs.docker.com/engine/reference/builder/





