---
{"dg-publish":true,"permalink":"/Program/Container/Dockfile详解/","noteIcon":"","created":"2024-05-22T16:17:54.137+08:00"}
---

       

## 什么是Dockerfile

dockerfile其实就是记录'火箭发射'步骤的文件，它记载了从一个镜像创建另一个新镜像的步骤。撰写好Dockerfile文件之后，我们就可以轻而易举的使用`docker build`命令来创建镜像了。

Dockerfile 是一个文本文件，其内包含了一条条的指令(Instruction)，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建。有了 Dockerfile，当我们需要定制自己额外的需求时，只需在 Dockerfile 上添加或者修改指令，重新生成 image 即可，省去了敲命令的麻烦。

下面是一个dockerfile文件的例子：

```dockerfile
FROM openjdk:8-jdk-alpine
MAINTAINER Kurisu "makise_kurisuu@outlook.jp"
LABEL maintainer="makise_kurisuu@outlook.jp"
VOLUME /tmp
ARG JAR_FILE=target/alice-server.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

## Dockerfile 的组成部分

| 部分        | 命令                                               |
| --------- | ------------------------------------------------ |
| 基础镜像信息    | FROM                                             |
| 维护者信息     | MAINTAINER                                       |
| 镜像操作指令    | RUN、COPY、ADD、EXPOSE、WORKDIR、ONBUILD、USER、VOLUME等 |
| 容器启动时执行指令 | CMD、ENTRYPOINT                                   |

## DockerFile命令详解

###   FROM  
  >  构建的镜像设置基础镜像
    
```bash
FROM <image> [AS <name>]
FROM <image>[:<tag>] [AS <name>]
FROM <image>[@<digest>] [AS <name>]
```
    
   `FROM`指令初始化新的构建阶段，并为后续指令设置基础镜像，e.g: `FROM openjdk:8-jdk-alpine`。因此，Dockerfile文件必须以`FROM`指令开头。 而指定的镜像可以使任何有效的镜像（推荐使用公共仓库的镜像，因为拉取更为容易）。
    
   * 在Dockerfile文件中`ARG`是唯一一个可以用于`FROM`之前的指令。
    
```dockerfile
ARG  CODE_VERSION=latest
FROM base:${CODE_VERSION}
CMD  /code/run-app
FROM extras:${CODE_VERSION}
CMD  /code/run-extras
```
    
       一个Dockerfile文件可以有多个FROM指令，来创建多个镜像或者使用其中一个阶段作为另一个阶段的依赖项。以最后一个镜像的ID为输出值。e.g:
    
```dockerfile
FROM golang:1.10.3 
COPY server.go /build/
WORKDIR /build
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 GOARM=6 go build -ldflags '-w -s' -o server
ENTRYPOINT ["/build/server"]
```
    
*   我们可以通过在`FROM`指令中添加`AS 名称` 来指定一个名称给后续阶段的`FROM`和`COPY --from=<name|index>`使用。
*   tag和digest是可选的，如果不提供则使用latest。
*   RUN   在镜像的构建过程中执行特定的命令，并生成一个中间镜像。比如安装一些软件、配置一些基础环境，可使用\\来换行。
    
```bash
RUN <command> (shell格式)
RUN ["executable", "param1", "param2"] (exec格式)
```
    
    > 要注意的是，executable是命令，后面的param是参数  
    > 采用exec格式指令将会被解析成json格式所以不能使用单引号，并且使用反斜杠也是必须要转移的，这在windows上尤为重要。
    
###    CMD  
    指定容器运行时的默认参数。
    
```bash
CMD ["executable","param1","param2"]（exec格式，首选）
CMD ["param1","param2"]（给ENTRYPOINT提供默认参数）
CMD command param1 param2（shell格式）
```
    
   > 注意：`RUN`是在构建的时候执行，并生成一个新的镜像，`CMD`在构建时不进行任何操作，在容器运行的时候执行。  
    如果`CMD`用于为`ENTRYPOINT`指令提供缺省参数，那么`CMD`和`ENTRYPOINT`指令都应该使用JSON数组格式。
    
*   LABEL  
    给构建的镜像打标签。
    
```bash
LABEL <key>=<value> <key>=<value> <key>=<value> ...
```
    
  标签是键值对格式,要在标签中包含空格则需转义或用引号`"`括起来。e.g:
    
```dockerfile
    LABEL "com.example.vendor"="ACME Incorporated"
    LABEL com.example.label-with-value="foo"
    LABEL version="1.0"
    LABEL description="This text illustrates \
    that label-values can span multiple lines."
    
```
    
   一个镜像可以有多个标签，如果基础镜像也有标签则继承，名字相同的话则会覆盖。如果使用多个标签，建议合并成一个标签指令，如果使用多个标签指令， 则每个标签指令都会生成一个图层，这会导致镜像生成效率低下。举个栗子：
    
```dockerfile
LABEL multi.label1="value1" multi.label2="value2" other="value3"

```
    
   也可以写成：
    
```dockerfile
LABEL multi.label1="value1" \
multi.label2="value2" \
other="value3"

```
    
###   MAINTAINER（已弃用）  
  设置作者信息。
    
```bash
MAINTAINER <name>
```
    
    `LABEL`比`MAINTAINER`更灵活，推荐使用`LABEL`，弃用`MAINTAINER`。
    
```bash
LABEL maintainer="makise_kurisuu@outlook.jp"
```
    
### *   EXPOSE  
    为构建的镜像设置监听端口。
    
```bash
EXPOSE <port> [<port>/<protocol>...]
```
    
 `EXPOSE`指令让docker容器在运行的时候监听指定的端口，可以指定端口是upd还是tcp协议，如果没有指定则默认tcp协议。  
 
 `EXPOSE`指令并不会发布端口，如果发布端口，则需要在docker run时使用-p来发布和映射一个或多个容器端口、或者使用-P来发布所有公开的端口 并将它们映射到高阶端口。
    
### *   ENV  
   在构建的镜像中设置环境变量，在后续的Dockerfile指令中可以直接使用，也可以固化在镜像里，在容器运行时仍然有效。 `ENV`指令有两种格式：
    
    1.  ENV ：把第一个空格之后的所有值都当做的值，无法在一行内设定多个环境变量。
    2.  ENV = ...：可以设置多个环境变量，如果中存在空格，需要转义或用引号`"`括起来。
    
```bash
ENV myName John Doe
ENV myDog Rex The Dog
ENV myCat fluffy
```
或者
    
```bash
ENV myName="John Doe" myDog=Rex\ The\ Dog \
    myCat=fluffy
```
    
 推荐使用第二种，同样的这样能减少图层提高效率。
    
 > 注意：可以在容器运行时指定环境变量，替换镜像中的已有变量，`docker run --env <key>=<value>`。  
  使用ENV可能会对后续的Dockerfile指令造成影响，如果只需要对一条指令设置环境变量，可以使用这种方式：`RUN <key>=<value> <command>`
    
###    ADD  
构建镜像时，复制上下文中的文件到镜像内。
    
```bash
ADD <src>... <dest>
ADD ["<src>",... "<dest>"](路径包含空格的必须使用这种格式)

```
    
`<src>`可以是文件、目录，也可以是文件URL。可以使用模糊匹配(wildcards，类似shell的匹配)，可以指定多个`<src>`，必须是在上下文目录和子目录中， 无法添加`../a.txt`这样的文件。如果`<src>`是个目录，则复制的是目录下的所有内容，但不包括该目录。如果`<src>`是个可被docker识别的压缩包， docker会以tar -x的方式解压后将内容复制到`<desct>`。`<dest>`可以是绝对路径，也可以是相对WORKDIR目录的相对路径。 所有文件的UID和GID都是0。
    
   > 如果docker发现文件内容被改变，则接下来的指令都不会再使用缓存。  
    关于复制文件时需要处理的/，基本跟正常的copy一致。
    
### *   COPY  
  将主机的文件复制到镜像内，如果目的位置不存在，Docker会自动创建所有需要的目录结构，但是它只是单纯的复制，并不会去做文件提取和解压工作。
    
```bash
COPY <src>... <dest>
COPY ["<src>",... "<dest>"](路径包含空格的必须使用这种格式)

```
    
> 注意：需要复制的目录一定要放在Dockerfile文件的同级目录下。  

> 因为构建环境将会上传到Docker守护进程，而复制是在Docker守护进程中进行的。任何位于构建环境之外的东西都是不可用的。 COPY指令的目的的位置则必须是容器内部的一个绝对路径。  

> [COPY指令](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.docker.com%2Fv17.09%2Fengine%2Freference%2Fbuilder%2F%23copy "https://docs.docker.com/v17.09/engine/reference/builder/#copy")
    
###    ENTRYPOINT  
指定镜像的执行程序。`ENTRYPOINT`指令有两种格式：
    
```bash
ENTRYPOINT ["executable", "param1", "param2"] （执行格式，首选）
ENTRYPOINT command param1 param2 （shell格式）
```
    
详细参考：[ENTRYPOINT指令](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.docker.com%2Fv17.09%2Fengine%2Freference%2Fbuilder%2F%23entrypoint "https://docs.docker.com/v17.09/engine/reference/builder/#entrypoint")
    
`CMD`和`ENTRYPOINT`至少得使用一个。`ENTRYPOINT`应该被当做docker的可执行程序，CMD应该被当做`ENTRYPOINT`的默认参数。 `docker run <image> <arg1> <arg2> ...`会把之后的参数传递给`ENTRYPOINT`，覆盖CMD指定的参数。可以用`docker run --entrypoint` 来重置默认的`ENTRYPOINT`。
    
| — | No ENTRYPOINT | ENTRYPOINT exec\_entry p1\_entry | ENTRYPOINT \["exec\_entry", "p1\_entry"\] |
| --- | --- | --- | --- |
| No CMD | error, not allowed | /bin/sh -c exec\_entry p1\_entry | exec\_entry p1\_entry |
| CMD \["exec\_cmd", "p1\_cmd"\] | exec\_cmd p1\_cmd | /bin/sh -c exec\_entry p1\_entry | exec\_entry p1\_entry exec\_cmd p1\_cmd |
| CMD \["p1\_cmd", "p2\_cmd"\] | p1\_cmd p2\_cmd | /bin/sh -c exec\_entry p1\_entry | exec\_entry p1\_entry p1\_cmd p2\_cmd |
| CMD exec\_cmd p1\_cmd | CMD exec\_cmd p1\_cmd | /bin/sh -c exec\_entry p1\_entry | exec\_entry p1\_entry /bin/sh -c exec\_cmd p1\_cmd |
###    VOLUME  
  > 指定镜像内的目录为数据卷。
    
```bash
VOLUME ["/data"]
```
    
   在容器运行的时候，docker会把镜像中的数据卷的内容复制到容器的数据卷中去。 如果在接下来的Dockerfile指令中，修改了数据卷中的内容，则修改无效。
    
###    USER  
    为接下来的Dockerfile指令指定用户。受影响的指令有：`RUN`、`CMD`、`ENTRYPOINT`。
    
```bash
USER <user>[:<group>] or
USER <UID>[:<GID>]
```
    
> 注意：当用户没有主要组时，镜像（或下一条指令）将与该root组一起运行。
    
###    WORKDIR  
    为接下来的Dockerfile指令指定当前工作目录，可多次使用，如果使用的是相对路径，则相对的是上一个工作目录，类似shell中的cd命令。
    
```bash
WORKDIR /path/to/workdir
```
    
受影响的指令有：`RUN`、`CMD`、`ENTRYPOINT`、`COPY`和`ADD`。 可以在Dockerfile中多次使用`WORKDIR`指令。如果提供了相对路径，则它将相对于上一个`WORKDIR`指令的路径。例如：
    
```dockerfile
WORKDIR /a
WORKDIR b
WORKDIR c
RUN pwd
```
    
此Dockerfile中`pwd`指令将输出 `/a/b/c.`。  

`WORKDIR`指令可以解析之前使用`ENV`设置的环境变量，只能使用Dockerfile中显式设置的环境变量。例如：

```dockerfile
ENV DIRPATH /path
WORKDIR $DIRPATH/$DIRNAME
RUN pwd


```

此Dockerfile中`pwd`指令将输出 `/path/$DIRNAME`。
    
###    ARG  
   指定用户在`docker build --build-arg <varname>=<value>`时可以使用的参数。
    
```bash
ARG <name>[=<default value>]

```

 如果用户指定了未在Dockerfile中定义的构建参数，则构建会输出警告。
    
```bash
[Warning] One or more build-args [foo] were not consumed.

```

 构建参数在定义的时候生效而不是在使用的时候。如下面第三行开始的user才是用户构建参数传递过来的user：
    
```dockerfile
FROM busybox
USER ${user:-some_user}
ARG user
USER $user
```

您可以使用`ARG`或`ENV`指令指定`RUN`指令可用的变量。使用`ENV`指令定义的环境变量始终覆盖同名的`ARG`指令。
    
```dockerfile
FROM ubuntu
ARG CONT_IMG_VER
ENV CONT_IMG_VER v1.0.0
RUN echo $CONT_IMG_VER 
```
    
正确的用法：

```dockerfile
FROM ubuntu
ARG CONT_IMG_VER
ENV CONT_IMG_VER ${CONT_IMG_VER:-v1.0.0}
RUN echo $CONT_IMG_VER
```

要在多个阶段中使用arg，每个阶段都必须包含该ARG指令。
    
```dockerfile
FROM busybox
ARG SETTINGS
RUN ./run/setup $SETTINGS

FROM busybox
ARG SETTINGS
RUN ./run/other $SETTINGS

```

此外docker还内置了一批构建参数，可以不用在Dockerfile中声明：`HTTP_PROXY`、`http_proxy`、`HTTPS_PROXY`、`https_proxy`、`FTP_PROXY`、 `ftp_proxy`、`NO_PROXY`、`no_proxy`
    
> 注意：在使用构建参数(而不是在构建参数定义的时候)的指令中，如果构建参数的值发生了变化，会导致该指令发生变化，会重新寻找缓存。
    
###    ONBUILD  
    向镜像中添加一个触发器，当以该镜像为基础镜像再次构建新的镜像时，会触发执行其中的指令。
    
```dockerfile
ONBUILD [INSTRUCTION]
```
    
比如我们生成的镜像是用来部署Python代码的，但是因为有多个项目可能会复用该镜像。你可以这样使用：
    
```dockerfile
[...]

ONBUILD ADD . /app/src

ONBUILD RUN /usr/local/bin/python-build --dir /app/src
[...]

```

> 不允许`ONBUILD`使用链接指令`ONBUILD ONBUILD`。  
   `ONBUILD`只会继承给子节点的镜像，不会再继承给孙子节点。  
   `ONBUILD`不会触发`FROM`、`MAINTAINER`指令。
    
###    STOPSIGNAL  
    触发系统信号。
    
```bash
STOPSIGNAL signal

```
    
 `STOPSIGNAL`指令设置将发送到容器以退出的系统调用信号。此信号可以是与内核的SysCall表中的位置匹配的有效无符号数字(例如9)，也可以是格式为SIGNAME的信号名称(例如SIGKILL)。
    
*   HEALTHCHECK  
  增加自定义的心跳检测功能，多次使用只有最后一次有效。格式：
    
```bash
HEALTHCHECK [OPTIONS] CMD command （通过在容器内运行命令来检查容器运行状况）
HEALTHCHECK NONE （禁用从基础映像继承的任何运行状况检查）
```
    
    可选的OPTION：

1.  `--interval=DURATION`（检测间隔，默认值：30s）
2.  `--timeout=DURATION`（命令超时时间，默认值：30s）
3.  `--start-period=DURATION`（初始化后开始检查时间，默认值：0s）
4.  `--retries=N`（连续N次失败后标记为不健康，默认值：3次）

> Start Period为需要时间引导的容器提供初始化时间。在此期间的探测失败将不计入最大重试次数。但是，如果运行状况检查在开始期间成功，则容器将被视为已启动，并且所有连续失败将计入最大重试次数。

`<command>`可以是shell脚本，也可以是exec格式的json数组。 docker以`<command>`的退出状态码来区分容器是否健康，这一点同shell一致：

*   0：命令返回成功，容器健康
*   1：命令返回失败，容器不健康
*   2：保留状态码，不要使用 举个栗子：每5分钟检测本地网页是否可访问，超时设为3秒：

```dockerfile
HEALTHCHECK --interval=5m --timeout=3s \
  CMD curl -f http://localhost/ || exit 1
```

你可以使用`docker inspect`命令查看健康状态。  
当容器的运行状况发生更改时，将生成带有新状态的health_status事件。

###    SHELL  
更改后续的Dockerfile指令中所使用的shell。默认的shell是`["bin/sh", "-c"]`。可多次使用，每次都只改变后续指令。
    
```dockerfile
SHELL ["executable", "parameters"]
```