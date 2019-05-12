---
title: "通过容器化Python web应用了解Docker容器核心功能"
catalog: true
toc_nav_num: true
date: 2019-05-12 20:51:24
subtitle: "通过实战了解Docker核心功能"
header-img: "/img/article_header/tizi.jpg"
tags:
- Docker
catagories:
- Docker
---
> 本篇文章通过一个典型的Python web应用，带你了解Docker的核心功能。需要依赖一台已经安装了Docker的Linux虚拟机。

## 构建一个镜像
首先我们准备一个应用。新建一个本文文件，起名叫 app.py，里面写入下面的内容，实现一个简单的web应用：
```pythonstub
from flask import Flask
import socket
import os

app = Flask(__name__)

@app.route('/')
def hello():
    html = "<h3>Hello {name}!</h3>" \
           "<b>Hostname:</b> {hostname}<br/>"           
    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname())
    
if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
```
在这段代码中，使用 Flask 框架启动了一个 Web 服务器，而它唯一的功能是：如果当前环境中有“NAME”这个环境变量，就把它打印在“Hello”后，否则就打印“Hello world”，最后再打印出当前环境的 hostname。
这个应用的依赖文件requirements.txt存在于与其同级的目录中，内容是：
```shell
cat requirements.txt
Flask
```
将这样一个应用在容器中跑起来，需要制作一个容器镜像。Docker提供了一种Dockerfile文件，来描述镜像的构建过程。下面的代码存在于与上面的应用（app.py）同级目录下的Dockerfile中。
```text
# 使用官方提供的python:3.6-alpine镜像，作为我们这个镜像的基础镜像，这样我们的镜像就有了python3.6环境
FROM python:3.6-alpine

# 将工作目录切换为 /app
WORKDIR /app

# 将当前目录下的所有内容复制到镜像的 /app 下
ADD . /app

# 使用 pip 命令安装这个应用所需要的依赖，RUN 指令就是在容器里执行 shell 命令的意思。
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# 允许外界访问容器的 80 端口
EXPOSE 80

# 设置环境变量
ENV NAME World

# 设置容器进程为：python app.py，即：这个 Python 应用的启动命令
CMD ["python", "app.py"]
```
这个Dockerfile文件的内容，描述了我们所要构建的 Docker 镜像。Dockerfile中每一行都是按顺序处理的。这个Dockerfile中用到的指令的具体含义已经在以注释的方式写在了Dockerfile中。

需要再详细介绍一下CMD指令。CMD指定了`python app.py`为这个容器启动后执行的进程。因为Dockerfile中使用WORKDIR切换了容器的工作路径是/app，所以 app.py 的实际路径是 /app/app.py。CMD ["python", "app.py"] 等价于 "docker run <imgname> python app.py"。

另外，在使用 Dockerfile 时，还有一种 ENTRYPOINT 指令。它和 CMD 都是 Docker 容器进程启动所必需的参数，完整执行格式是："ENTRYPOINT CMD"。

默认情况下，Docker 会为你提供一个隐含的 ENTRYPOINT，即：`/bin/sh -c`。所以，在不指定 ENTRYPOINT 时，比如在我们这个例子里，实际上运行在容器里的完整进程是：/bin/sh -c "python app.py"，即 CMD 的内容就是 ENTRYPOINT 的参数。正是基于这样的原理，Docker 容器的启动进程为实际为 ENTRYPOINT，而不是 CMD。

需要注意的是，Dockerfile 里的指令并不都是只在容器内部的操作。就比如 ADD，它指的是把当前目录（即 Dockerfile 所在的目录）里的文件，复制到指定容器内的目录当中。

根据前面的描述，现在我们的整个应用的目录结构应该如下这样：
```shell
ls
Dockerfile  app.py   requirements.txt
```
现在我们执行下面的指令构建镜像：
```shell
docker build -t helloworld .
```
其中，-t 的作用是给这个镜像加一个 Tag，即：起一个好听的名字。docker build 会自动加载当前目录下的 Dockerfile 文件，然后按照顺序执行Dockerfile文件中的指令。

Dockerfile 中的每个指令执行后，都会生成一个对应的镜像层。

上面的命令执行完成后，就形成了一个镜像。可以通过下面的指令查看：
```text
docker image ls

REPOSITORY    TAG        IMAGE ID      CREATED             SIZE
helloworld    latest     5bacb9617bcf  7 minutes ago       89.8MB
```
还可以通过 `docker inspect helloworld:latest` 查看镜像的元信息。

## 运行镜像
有了镜像，就可以通过下面的指令来运行容器了。
```text
docker run -p 5000:80 helloworld
```
在这一句命令中，镜像名 helloworld 后面，什么都不用写，因为在 Dockerfile 中已经指定了 CMD。否则，我就得把进程的启动命令加在后面：
```text
docker run -p 5000:80 helloworld python app.py
```
如果上面的指令执行后，能够输出下面的内容，则表示容器启动成功了：
```text
 * Serving Flask app "app" (lazy loading)
 * Environment: production
   WARNING: Do not use the development server in a production environment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://0.0.0.0:80/ (Press CTRL+C to quit)
```
可以通过运行`docker ps`指令，查看运行中的容器。
```text
docker ps

CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                  NAMES
a242ecaf6cf6        helloworld          "python app.py"     3 minutes ago       Up 3 minutes        0.0.0.0:5000->80/tcp   dazzling_khayyam
```
从输出中可以看到，容器的ID，容器是基于哪个镜像的启动的，容器中的进程，容器的启动时间及端口映射情况。

从现在看，容器已经正确启动，我们使用curl命令通过宿主机的IP和端口号，来访问容器中的web应用：
```text
curl localhost:5000
<h3>Hello World!</h3><b>Hostname:</b> a242ecaf6cf6<br/>
```
还有一种访问容器中应用的方法是使用容器的IP和应用端口号，应用的端口号我们已经在Dockerfile中指定为80，那么容器的IP如何获取呢？

依然是使用`docker inspect CONTAINER ID`命令查看容器的元数据。容器的IP地址就在其中。

## 分享镜像 
大家一定用过代码分享平台GitHub，在Docker世界中分享镜像的平台是[Docker Hub](https://hub.docker.com/)，它"学名"叫镜像仓库（Repository）。

为了能够上传镜像，首先需要注册一个 Docker Hub 账号，然后使用 docker login 命令登录:
```text
$ docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: liuchunming
Password:
Login Succeeded
```
在push到Docker Hub之前，需要先给镜像指定一个版本号：
```text
docker tag helloworld liuchunming/helloworld:v1
```
liuchunming是我在Docker Hub 上的账户名。v1是我给这个镜像本次起的版本号。

接着执行下面的指令就可以镜像push到Docker Hub上了：
```text
docker push liuchunming/helloworld:v1
```
一旦提交到Docker Hub上，其他人就可以通过`docker pull helloworld:v1`指令将镜像下载下来了。

在企业内部，也可以搭建一个跟 Docker Hub 类似的镜像存储系统。感兴趣的话，可以查看VMware 的 Harbor 项目。

## 进入正在运行的容器中玩玩
运行web服务的容器，通常是以后台进程启动的。就是在`docker run`指令后面加上-d选项。比如以后台方式运行上面的web容器：
```text
docker run -d -p 5000:80 helloworld
```
如果你想进入到一个正在运行的容器做一些操作，可以通过`docker run`指令。我们需要先通过`docker ps`命令查看容器的ID,然后执行下面的命令。
```text
docker exec -it 1695ed10e2cb75e /bin/sh
```
-it选项指的是连接到容器后，启动一个terminal(终端)并开启input(输入)功能。`/bin/sh`表示进入到容器后执行的命令。现在我们就可以在终端上进行一些操作了，比如在容器中新建一个readme.md文件：
```text
/app # ps
PID   USER     TIME  COMMAND
    1 root      0:00 python app.py
   24 root      0:00 /bin/sh
   29 root      0:00 ps
/app# touch readme.md
/app# exit
```
我们还可以将正在运行的容器，commit成新的镜像。
```text
docker commit 1695ed10e2cb75e liuchunming033/helloworld:v2
```
docker exec 的实现原理，其实是利用了容器的三大核心技术之一的Namespace。一个进程可以选择加入到某个进程（运行中的容器）已有的 Namespace 当中，从而达到“进入”这个进程所在容器的目的。更细节的原理这里不在细究。
## 容器与宿主机之间如何共享文件
容器技术使用了 rootfs 机制和 Mount Namespace，构建出了一个同宿主机完全隔离开的文件系统环境。但是我们使用过程中经常会遇到这样两个问题：

- 容器里进程新建的文件，怎么才能让宿主机获取到？

- 宿主机上的文件和目录，怎么才能让容器里的进程访问到？

这正是 Docker Volume 要解决的问题：Volume 机制，允许你将宿主机上指定的目录或者文件，挂载到容器里面进行读取和修改。

在 Docker 项目里，它支持两种 Volume 声明方式，可以把宿主机目录挂载进容器的 /test 目录当中：
```text
docker run -v /test ...
docker run -v /home:/test ...
```
而这两种声明方式的本质，实际上是相同的：都是把一个宿主机的目录挂载进了容器的 /test 目录。

只不过，在第一种情况下，由于你并没有显示声明宿主机目录，那么 Docker 就会默认在宿主机上创建一个临时目录 `/var/lib/docker/volumes/[VOLUME_ID]/_data`，然后把它挂载到容器的 /test 目录上。而在第二种情况下，Docker 就直接把宿主机的 /home 目录挂载到容器的 /test 目录上。

启动容器时，给他声明一个volume
```text
docker run -d -v /test helloworld
```
容器启动之后，我们来查看一下这个容器 的Volume 在宿主机上的对应的目录:
```shell
docker volume ls
DRIVER              VOLUME NAME
local               dc195c8ad14ad505832461d9f37da889c54ef284ebaf777a100e10a932217ad3
```
或者执行`docker inspect CONTAINER_ID`命令查看，命令输出的Mounts字段中Source的值就是宿主机上的目录：
```text
"Mounts": [
            {
                "Type": "volume",
                "Name": "dc195c8ad14ad505832461d9f37da889c54ef284ebaf777a100e10a932217ad3",
                "Source": "/var/lib/docker/volumes/dc195c8ad14ad505832461d9f37da889c54ef284ebaf777a100e10a932217ad3/_data",
                "Destination": "/test",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
        ],
```
然后，查看宿主机上的路径：
```
ls /var/lib/docker/volumes/dc195c8ad14ad505832461d9f37da889c54ef284ebaf777a100e10a932217ad3/_data
```
这个 _data 文件夹，就是这个容器的 Volume 在宿主机上对应的临时目录了。接下来，我们在容器的 Volume 里，添加一个文件 text.txt：
```text
docker exec -it cf53b766fa6f /bin/sh
cd test/
touch text.txt
```
这时，我们再回到宿主机，就会发现 text.txt 已经出现在了宿主机上对应的临时目录里了：
```text
ls /var/lib/docker/volumes/dc195c8ad14ad505832461d9f37da889c54ef284ebaf777a100e10a932217ad3/_data
text.txt
```
因为容器运行时产生的文件，在容器停止后将会消失。因此，将容器的目录映射到宿主机的某个目录，一个重要使用场景是持久化容器中产生的文件，比如应用的日志。

## 给容器加上资源限制
其实容器是运行的宿主机上的特殊进程，多个容器之间是共享宿主机的操作系统内核的。默认情况下，容器并没有被设定使用操作系统资源的上限。

有些情况下，我们需要限制容器启动后占用的宿主机操作系统的资源。Docker可以利用Linux Cgroups机制可以给容器设置资源使用限制。

Linux Cgroups 的全称是 Linux Control Group。它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。Docker正是利用这个特性限制容器使用宿主上的CPU、内存。

下面启动容器的方式，给这个 Python 应用加上 CPU 和 Memory 限制：
```text
docker run -it --cpu-period=100000 --cpu-quota=20000 -m 300M helloworld
```
--cpu-period和--cpu-quota组合使用来限制容器使用的CPU时间。表示在--cpu-period的一段时间内，容器只能被分配到总量为 --cpu-quota 的 CPU 时间。-m选项则限制了容器使用宿主机内存的上限。

上面启动容器的命令，将容器使用的CPU限制设定在最高20%，内存使用最多是300MB。

## 总结
本篇文章通过非常经典的 Python web应用作为案例，讲解了 Docker 容器使用的主要场景。包括构建镜像、启动镜像、分享镜像、在镜像中操作、在镜像中挂在宿主机目录以及对容器使用的资源进行限制。熟悉了这些操作，你也就基本上摸清了 Docker 容器的核心功能。
