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
* Software: Install docker on Ubuntu: https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository

P.S You can add your username to group `docker` so that you can run docker command without `sudo`
```
sudo usermod -aG docker <yourusername>
```
当目标是在GCP 上运行 Jenkins，并使用 Docker，操作流程：

  1.	GCP VM 是你租的远程服务器（像一台云端电脑）
→ 因为 Jenkins 要“跑”在某个机器上。
	2.	在这台 GCP VM 云端电脑上选择装 Ubuntu 这个操作系统是因为它是流行的 Linux 系统
→ 稳定、免费、适合跑 Jenkins、Docker 等服务。
	3.	在这台 GCP VM 云端电脑的 Ubuntu 操作系统里装 Docker 是为了用Docker container运行 Jenkins
→ 用容器跑 Jenkins 有以下好处：
	•	简单：不用手动配置很多系统依赖；
	•	可移植：跑在任何装了 Docker 的地方；
	•	可升级/销毁/重建更方便。
	4.	最终，只需要在 Docker 容器里运行 Jenkins 就行
  5.  在成功进入jenkins操作页面后，还要去jenkins container里面安装Docker CLI
  因为部署 Jenkins 本身只是运行 Jenkins 服务，而Jenkins 如果需要运行 Docker 命令来构建、运行或推送镜像等，那这些命令就需要在 
  Jenkins 所在的容器内运行。 这里有一个“容器里再跑容器”的感觉！
  
  如果你在 Jenkins、CI/CD 脚本中看到要“安装 Docker CLI”，它的意思就是要能运行 docker build、docker push 等命令，但通常不
  会再装一个后台服务。这里有一个概念Docker CLI不是Docker 。Docker CLI（Command-Line Interface） 是你在终端中输入的 docker 命令；它不是 Docker 引擎（daemon），而是 客户端程序；它向 Docker 后台服务（dockerd）发送命令，比如创建容器、构建镜像、推送仓库等。



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

```cmd
// Windows CMD
docker run --name jenkins ^
           -u root ^
           -d ^
           -v %cd%:/var/jenkins_home ^
           -v //var/run/docker.sock:/var/run/docker.sock ^
           -p 80:8080 ^
           -p 50000:50000 ^
           jenkins/jenkins:lts-jdk21
```

```PowerShell
// Windows PowerShell
$pwd = (Get-Location).Path
$pwd = $pwd -replace '\\', '/'
docker run --name jenkins `
           -u root `
           -d `
           -v "${pwd}:/var/jenkins_home" `
           -v //var/run/docker.sock:/var/run/docker.sock `
           -p 80:8080 `
           -p 50000:50000 `
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

You can rebuild the Jenkins images by following the steps in [Jenkins official documentation](https://www.jenkins.io/doc/book/installing/docker/ ) (recommended). 

Or install docker tool sets inside your Jenkins container. The release version of Jenkins image is based on Debian:

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
