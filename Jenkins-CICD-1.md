## Install Jenkins on Docker

This is to guide how to install Jenkins and run it in a Docker container.


正确理解层级结构：
[ GCP VM ]
   └── Ubuntu（操作系统）
         └── Docker（容器引擎）
               └── Jenkins（容器中的 CI/CD 服务）

	•	GCP VM：提供计算资源（相当于远程机器）
	•	Ubuntu：运行 Docker 的基础系统
	•	Docker：用来跑 Jenkins 的容器平台
	•	Jenkins：CI/CD 工具，运行在 Docker 容器中

### Pre-requisites

* A VM (GCP VM is recommended, new users can get US$300 credits for 90 days) or a laptop

* 创建完VM后，要把这个VM链接到本地终端的步骤是：
* 先在终端输入指令：ssh-keygen 去创建密钥，会显示：                                      Generating public/private ed25519 key pair.

* 然后按回车把私钥公钥自动保存到：(/Users/eleanorcollins/.ssh/id_ed25519)

* 接着cd ~/.ssh 通过cat id_ed25519.pub 可以提取公钥内容：
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIC/k87ckXCbbHng5bnTGALgIpVe0bEvsa9M46a8/exE4 eleanorcollins@Mac
* 复制粘贴到刚刚创建的vm ssh里。以上是准备工作。

* 如果之前已经创建过密钥并添加到vm里，但是忘了密钥id，就先cd ~/.ssh 然后ls 罗列出包含的文件夹，找到钥匙文件夹，分别有id_ed25519      id_ed25519.pub

* 接着ssh -i ~/.ssh/id_ed25519 eleanorcollins@34.125.253.245
其中id_ed25519是我的私钥文件名，eleanorcollins是我的终端用户名，同时和公钥结尾注释匹配。@34.125.253.245这串数字是vm external ip

* Software: Install docker on Ubuntu: https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository
* 可以在终端输入sudo snap install docker去快速下载docker到ubuntu系统里

P.S You can add your username to group `docker` so that you can run docker command without `sudo`
```
sudo usermod -aG docker <yourusername>
```
当目标是在GCP 上运行 Jenkins，并使用 Docker，操作流程：

* 1.GCP VM 是你租的远程服务器（像一台云端电脑）
→ 因为 Jenkins 要“跑”在某个机器上。

* 2.在这台 GCP VM 云端电脑上选择装 Ubuntu 这个操作系统是因为它是流行的 Linux 系统
→ 稳定、免费、适合跑 Jenkins、Docker 等服务。

* 3.在这台 GCP VM 云端电脑的 Ubuntu 操作系统里装 Docker 是为了用Docker container运行 Jenkins
→ 用容器跑 Jenkins 有以下好处：
	•	简单：不用手动配置很多系统依赖；
	•	可移植：跑在任何装了 Docker 的地方；
	•	可升级/销毁/重建更方便。

* 4.最终，只需要在 Docker 容器里运行 Jenkins 就行
 
* 5.另外，容器里的 Jenkins 要自己构建 Docker 镜像，并运行容器。所以在成功验证可以进入jenkins操作页面后，还要去返回jenkins container里面安装Docker CLI
→ 部署 Jenkins 只是为了运行 Jenkins 服务，而Jenkins 如果需要运行 Docker 命令来构建、运行或推送镜像等，那这些命令就需要用 
  Docker CLI（Command-Line Interface）来运行。 这里有一个“容器里再跑容器”的感觉！
  这里有一个概念Docker CLI不是Docker 。Docker CLI（Command-Line Interface） 是你在终端中输入的 docker 命令；它不是 Docker 引擎（daemon），而是 客户端程序；它向 Docker 后台服务（dockerd）发送命令，比如创建容器、构建镜像、推送仓库等。



在做 Jenkins + Docker 自动化部署时经常会碰到Docker in/outside of Docker 情况

🎯 什么是 “Docker in Docker”（简称：DinD）

Docker in Docker 就是指：

在一个 Docker 容器内部，再运行 Docker 守护进程（Docker Daemon），并让这个容器可以运行其它容器。

也就是说：你在容器里又跑了一个 Docker 引擎，相当于“容器里再跑容器”。

宿主机 Docker
└── Jenkins 容器（运行 Docker 守护进程）
     └── 构建时运行的容器（比如运行 docker build 的时候）


❓为什么有人要用 Docker in Docker？
主要是为了解决这种需求：

容器里的 Jenkins 要自己构建 Docker 镜像，并运行容器。

如果你不挂载宿主机的 Docker socket，而是让 Jenkins 容器自己运行一个独立的 Docker 引擎（daemon），那你就用到了 Docker-in-Docker。

❗ 但是 “Docker in Docker” 一般 不推荐生产使用！！！



✅ 替代方案：Docker Outside of Docker（DooD）

这是最常见、推荐的做法，不需要在容器里运行 Docker 守护进程，只有 CLI，使用宿主的 Daemon。

你只在 Jenkins 容器里安装 Docker CLI，然后把宿主机的 Docker socket 挂进去。

这样 Jenkins 容器内部调用 docker build，实际操作的还是 宿主机的 Docker 引擎，更安全高效。

配置方式：
docker run -d --name jenkins \
  -p 8080:8080 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts




### Steps

#### 1. Create a folder to hold Jenkins data

```bash
mkdir jenkins_home
cd jenkins_home
```

#### 2. Start a Jenkins container

```bash
// Linux, macOS, WSL
docker run --name jenkins \
           -u root \
           -d \
           -v $(pwd):/var/jenkins_home \
           -v /var/run/docker.sock:/var/run/docker.sock \
           -p 80:8080 \
           -p 50000:50000 \
           jenkins/jenkins:lts-jdk21
```
> Note:
>
> * `-u root` configures to run Jenkins by root user.
> * `-d` detaches the container
> * `-v $(pwd):/var/jenkins_home` mounts the current directory to Jenkins home inside the container
> * `-v /var/run/docker.sock:/var/run/docker.sock` mounts Docker socks
> * `-p 80:8080` maps port 80 in the host to port 8080 inside the container
> * `-p 50000:50000` maps ports 50000 which is the default port for agent registration
> * `jenkins/jenkins:lts-jdk21` is the image maintained by `jenkin`.
> * If you have error with port 80, please change it to 8080 or other port numbers.

#### 3. Open <http://your.ip.addr> and you should see screen below

![Alt text](images/docker-install-01.png?raw=true)

#### 4. Get your password by opening `secrets/initialAdminPassword` file in the directory you created in the first step

```bash
sudo cat secrets/initialAdminPassword
```

- You can also get the initial password from your Docker container log
```
docker logs <jenkins_container_id>
```

- or you can bash into your Docker container and get it from inside:

To see the docker container id

```bash
docker container ls 
```
Then enter in bash inside a container.

```bash
docker exec -it CONTAINER_ID bash
```

#### 5. Continue installation in step #3 with password from step #4

You should see screen below after full installation.
![Alt text](images/docker-install-02.png?raw=true)

#### 6. Install `docker` CLI in your Jenkins container

The official Jenkins image doesn’t contain Docker CLI. If you need to run docker related tasks, you need to install docker CLI in your Jenkins container.
当你在 Jenkins、CI/CD 脚本中看到要“安装 Docker CLI”，它的意思就是要能运行 docker build、docker push 等命令。

常见方案是：Docker Outside of Docker（DooD）
即不需要在容器里运行 Docker 守护进程。你只在 Jenkins 容器里安装 Docker CLI，然后把宿主机的 Docker socket 挂进去。这样 Jenkins 容器内部调用 docker build，实际操作的还是 宿主机的 Docker 引擎，更安全高效。

这里有一个概念Docker CLI不是Docker 。Docker CLI（Command-Line Interface） 是你在终端中输入的 docker 命令；它不是 Docker 引擎（daemon），而是 客户端程序；它向 Docker 后台服务（dockerd）发送命令，比如创建容器、构建镜像、推送仓库等。


 Install docker tool sets inside your Jenkins container. The release version of Jenkins image is based on Debian:

1. Go into your Jenkins docker container
   ```
   docker exec -it <Jenkins_container_id> bash        
   ```
1. Set up Docker's apt repository
   ```
   # Add Docker's official GPG key:
   apt-get update
   apt-get install ca-certificates curl
   install -m 0755 -d /etc/apt/keyrings
   curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
   chmod a+r /etc/apt/keyrings/docker.asc
           
   # Add the repository to Apt sources:
   echo \
     "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
     $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
     tee /etc/apt/sources.list.d/docker.list > /dev/null
   apt-get update
   ```
2. Install the Docker packages
   ```
   apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   ```
3. Verify that the installation is successful
   ```
   docker -v
   Docker version 27.5.1, build 9f9e405
   ```
