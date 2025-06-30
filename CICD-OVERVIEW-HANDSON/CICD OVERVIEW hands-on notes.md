# Github Action CI/CD

### Goal: Build, Test and Deploy a web-app to AWS Elastic Beanstalk from scratch

# Hands-on

## At the end of this hands-on
* You will experience a step-by-step guidance for CD and will be able to set up a simple CD via Github Actions pipeline
  and deploy a react app on AWS (Note this tutorial is simply for illustration purpose; do not use in production)


## Pre-requisites
* Installed Docker
  
  P.S In Linux you can add your username to group docker so that you can run docker command without sudo (relogin is needed)
```
sudo usermod -aG docker $USER
```
* Installed Npm: https://nodejs.org/en/download
```
# Download and install nvm:
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
```
Note: Don't forget to setup the nvm environment 
```
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion

# Download and install Node.js:
nvm install 22

# Verify the Node.js version:
node -v # Should print "v22.13.1".
nvm current # Should print "v22.13.1".

# Verify npm version:
npm -v # Should print "10.9.2".

```

# i) Hands-on CI

## 1.Create and Run a web-app
首先，通过npx创建一个web-app框架，命名为my-app。Create a sample React web-app skeleton (nextjs.js) with npx. Set `my-app` as your project name and proceed with default options for other questions.

关于Node.js和next.js的关系：
简单来说：
✅ Node.js
	•	是一个运行 JavaScript 的 后端运行时环境（基于 Chrome V8 引擎）
	•	让你可以在服务器上用 JavaScript 编写后端逻辑
✅ Next.js
	•	是一个 基于 Node.js 的 React 框架
	•	用来开发服务端渲染（SSR）、静态生成（SSG）和客户端渲染（CSR）结合的现代 Web 应用
	•	它在 Node.js 环境下运行，用 Node.js 提供服务器端渲染能力

所以：
👉 Node.js 是基础运行环境
👉 Next.js 是在 Node.js 上构建的、专注前端渲染和全栈开发的框架

而npm（Node Package Manager）是 Node.js 的包管理器，用来：

✅ 安装、更新、卸载 JavaScript 库或依赖（package）
✅ 管理依赖版本，记录在 package.json
✅ 运行脚本，比如 npm run build、npm test

简单说：
	•	如果你在用 Node.js 做项目开发，npm 是管理依赖和脚本运行的核心工具。
	•	它也能用来发布、分享自己的 JS 库到 npm 官方仓库。
```
$ npx create-next-app@latest

Need to install the following packages:
create-next-app@15.1.6
Ok to proceed? (y)

✔ What is your project named? … my-app
✔ Would you like to use TypeScript? … No / Yes
✔ Would you like to use ESLint? … No / Yes
✔ Would you like to use Tailwind CSS? … No / Yes
✔ Would you like your code inside a `src/` directory? … No / Yes
✔ Would you like to use App Router? (recommended) … No / Yes
✔ Would you like to use Turbopack for `next dev`? … No / Yes
✔ Would you like to customize the import alias (`@/*` by default)? … No / Yes
Creating a new Next.js app in /home/ubuntu/git/my-app.
.
.
.
```

Run the web-app
```
$ cd my-app
$ npm run build
$ npm start
```

You should see but on localhost:3000:
<p align="center">
  <img src="../images/Nextjs.png" width="50%">
</p>



## 2.Dockerise your web-app
Add a Dockerfile to my-app folder
第二步，创建完web并且成功访问localhost:3000后，下一步就是把这个web镜像化。这里要提到一点，my-app其实就是一个本地repo,因为当你处于my-app里，然后在终端输入ls -a，就可以看到隐藏文件夹.git，.gitignore 和.github。
综上，我们要像镜像化这个web就要先在my-app文件夹里创建一个dockerfile文档，就去terminal 里面输入commond: vi Dockerfile，然后复制粘贴下方的信息,保存后退出，就实现了在my-app里创建dockerfile的任务，dockerfile里的代码是用来把刚刚建的web镜像化。

```
FROM node:22 AS build

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22 as runtime

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=build /app/.next ./.next
COPY --from=build /app/public ./public

EXPOSE 3000
USER node
CMD ["npm", "start"]
```


Your folder structure should now looks like this:
<p align="center">
  <img src="../images/Structure-nextjs.png" width="50%" >
</p>

要测试dockerfile是否成功，可以运行以下指令，You can run locally the docker containers for testing your Dockerfile (`-f Dockerfile` is optional)
```
docker build -t my-app -f Dockerfile .
docker run --rm -d -p 3000:3000 --name nextjs-react-app my-app
```
以上两行指令的意义：
把dockerfile和当前目录的所有文件都作为构建的上下文去建一个名为my-app的镜像；
在后台运行一个基于my-app镜像的容器，这个容器命名为nextjs-react-app，将容器内的3000端口映射到宿主机的3000端口，容器停止后自动删除。

## 3. Add the repo to Github
在github创建一个远程repo，命名为sample-docker-react，并与本地分支建立首次追踪关系，Create a repo in your github
![Alt text](../images/Create_new_repo.png?raw=true)

并通过第2小节展示的指令可知，我们要做的是把这个远程仓库url添加到本地，并取名为origin；接着把my-app所在的本地分支命名为main，再把main首次推送到origin仓库，由此my-app repo和sample-docker-react repo建立追踪关系。
Follow the steps in the second section (``...or push an existing repository from the command line``) to push existing repo from the command line
![Alt text](../images/Steps.png?raw=true)


## 4. Let us setup Github Action CI
Note: 
+ The uses field in the Setup Node.js step specifies the actions/setup-node action to set up the Node.js environment. 
This action automatically installs the specified version of Node.js and sets up the environment variables.这一步就是自动化CI的关键，把本地repo my-app与GitHub远程repo sample-docker-react关联起来，并通过在远程repo的‘secrets and variables'里添加dockerhub的登入名和登入密码去创建与docker的关联，进而实现把这个装有my-app代码的repo镜像化。

+ 把你的dockerhub登入名和登入密码存在刚刚创建并与本地repo建立追踪的repo“机密与变量”里。Your dockerhub username and password were saved in the sample-docker-react repo's `Secrets and variables` (https://github.com/<your_org>/sample-docker-react/settings/secrets/actions)
<p align="center">
  <img src="../images/gha_secrets.png">
</p>

在终端输入：mkdir -p .github/workflows (注意是workflows,s不能少，不然系统不能辨别是pipeline的文件)去创建workflows文件夹在.github文件夹里面。
接着在终端输入：code . 打开vscode，在.github/workflows文件夹里创建一个yml文件：ci.yml,复制粘贴以下文档;在这个ci.yml文档里我们可以看到自动化ci的流程，包括docker用户名和密码被编辑。保存退出后，在终端里输入：git add. ,git commit -m "adding ci.yml" ,git push. 完成后去到github sample-docker-react repo，点击actions，可以看到这个ci成功build。

要注意ci.yml正确创建在workflows这个pipeline文件夹里而不是.github文件夹里，否则无法action.

Create a new YAML file named `ci.yml` or something similar in the `.github/workflows` directory (create the directory if it does not exist) in your repository. 
Copy and paste the example workflow code into the file and commit it to your repository. 
Whenever you submit a pull request, the workflow will run automatically and perform the specified tests.
```
name: React next.js CI

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v2

    - name: Setup Node.js
      uses: actions/setup-node@v2
      with:
        node-version: 22

    - name: Build Docker image
      run: docker build -t ${{ secrets.DOCKER_USERNAME }}/sample-app:gha .

    - name: Run tests
      run: echo "Put your testing command here if you have"

    - name: Log in to Docker Hub
      uses: docker/login-action@f4ef78c080cd8ba55a85445d5b36e214a81df20a
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Push Docker image
      run: docker image push ${{ secrets.DOCKER_USERNAME }}/sample-app:gha

```
This GitHub Actions workflow sets up a Node.js environment, builds a Docker image, and runs tests using the image.

The workflow uses the on field to specify when the workflow should run: whenever there is a push or pull request to the main branch.

The `jobs` section defines a single job named **build**, which runs on the **ubuntu-latest** environment. This job consists of the following steps:

* Checking out the code from the repository
* Setting up Node.js on the environment
* Building a Docker image using the file `Dockerfile`
* Running tests  
* Loging in to DockerHub and and pushing the image

👉 jobs 部分 定义了一个叫 build 的任务，这个任务在 ubuntu-latest 这个环境里跑。
👉 这个任务会做这些事：
	•	把你仓库里的代码拉下来
	•	在环境里装好 Node.js
	•	用 Dockerfile 来构建 Docker 镜像
	•	跑测试
	•	登录 DockerHub，把镜像推上去

Now push the changes to your git repo
```
git checkout -b "feature/introduce_ci"
git add .
git commit -m "Introduce github action CI pipeline"
git push --set-upstream origin "feature/introduce_ci"
git checkout main
```
	•	新建并切换到一个分支叫 feature/introduce_ci
	•	把改动都加入暂存区
	•	提交改动，写上说明 Introduce github action CI pipeline
	•	把这个分支推到远程仓库
	•	最后切回 main 分支
Click the link in the GitHub UI to create a pull request and see what happens.

Further reading for pushing images to dockerhub: https://docs.github.com/en/actions/use-cases-and-examples/publishing-packages/publishing-docker-images#publishing-images-to-docker-hub

# ii) Hands-on CD

## 0. Create IAM role and EC2 Key-Pairs for AWS Elastic Beanstalk
Before we creating the AWS Elastic Beanstalk, we need to create an EC2 Key Pairs and an IAM role with the required permissions.
+  ### Create the EC2 Key Pair
Create a new EC2 Key Pair with the name `tutorial` (you need to download and save the new key pair immediately) 
![Alt text](../images/create-ec2-key-pair.png?raw=true) 
+  ### Create the IAM role
In IAM create a new IAM role with the name `BTN_EC2_profile`
![Alt text](../images/iam_create_role_1.png?raw=true) 
![Alt text](../images/iam_create_role_2.png?raw=true) 
![Alt text](../images/iam_create_role_3.png?raw=true) 

## 1. Create AWS Elastic Beanstalk
AWS Elastic Beanstalk is an easy-to-use service for deploying and scaling web applications and services developed with
Java,. NET, PHP, Node. js, Python, Ruby, Go, and Docker on familiar servers such as Apache, Nginx, Passenger, and IIS.

Now, let’s configure AWS manually via the AWS UI to help you understand the bare minimum steps required.

We need to sign in to AWS, hover over Services, select Elastic Beanstalk, and then click `Create application`.

![Alt text](../images/Create_web_app_new.png?raw=true)

Let us name it `sample-docker-react` and fill in the platform info :
![Alt text](../images/aws_docker_env_new.png?raw=true)
![Alt text](../images/create_app_platform.png?raw=true)
In step2, create and use new service role (the default name will be `aws-elasticbeanstalk-service-role`), then select the EC2 key pair and EC2 instance profile created previously
![Alt text](../images/create_app_access.png?raw=true)

Now, Skip to review and create the app. You will see something like this:
![Alt text](../images/create_app_health_ok.png?raw=true)

## 2. Set up IAM user
We need an IAM user with proper permissions so that github action can deploy the app on your behalf:
![Alt text](../images/iam.png?raw=true)

add a new user 
![Alt text](../images/iam_add_new_user.png?raw=true)

on next page let us attach the following policy to this user
![Alt text](../images/iam_user_review.png)

Create the access key for the new user and download the csv immediately, otherwise, you won't be able to see the secrets anymore
![Alt text](../images/aws_create_access_key.png)

Then go to your Git repository, navigate to `Settings -> Secrets and variables -> Actions -> New Repository Secrets`, and add **AWS_ACCESS_KEY_ID** and **AWS_SECRET_ACCESS_KEY**, setting their corresponding values.


## 3. Add a S3 folder for ElasticBeanstalk app
Amazon S3 or Amazon Simple Storage Service is a service offered by Amazon Web Services (AWS) that provides object
storage through a web service interface.

Also, while creating an elastic beanstalk environment, AWS selects a server location closest to you for hosting it
(You can select a different location too).

Also, after the environment is created, an S3 bucket is also created bearing a name that’s contains an identifier of
this server location (e.g us-east-1, us-west-1) and it is used to store applications that you deployed in environments
existing in this same server location.

![Alt text](../images/s3.png)

Okay, you will probably see a s3 bucket called elasticbeanstalk-us-east-1-xxx,
let us create a folder called "EBApptest" as illustrated below.

![Alt text](../images/s3_folder.png)

## 4. Now, let us add a CD pipeline file

create a new YAML file named `cd.yml` or something similar in the `.github/workflows` directory in your repository.
Copy and paste the example workflow code into the file and commit it to your repository.
Whenever you close a pull request, the workflow will run automatically and deploy relevant code.

```
name: Deploy to Elastic Beanstalk

on:
  pull_request:
    types: [closed]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Create Dockerrun.aws.json
        run: |
          cat > Dockerrun.aws.json <<EOF
          {
            "AWSEBDockerrunVersion": "1",
            "Image": {
              "Name": "${{ secrets.DOCKER_USERNAME }}/sample-app:gha",
              "Update": "true"
            },
            "Ports": [
              {
                "ContainerPort": "3000"
              }
            ]
          }
          EOF

      - name: Create deployment package
        run: zip deploy_package.zip Dockerrun.aws.json

      - name: Install AWS CLI
        run: |
          sudo apt-get update
          sudo apt-get install -y python3-pip
          pip3 install --user awscli

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-2

      - name: Upload package to S3 bucket
        run: aws s3 cp deploy_package.zip s3://elasticbeanstalk-ap-southeast-2-482739392776/EBApptest/

      - name: Deploy to Elastic Beanstalk
        run: |
          aws elasticbeanstalk create-application-version \
            --application-name sample-docker-react \
            --version-label ${{ github.sha }} \
            --source-bundle S3Bucket="elasticbeanstalk-ap-southeast-2-482739392776",S3Key="EBApptest/deploy_package.zip"
          
          aws elasticbeanstalk update-environment \
            --environment-name Sample-docker-react-env \
            --version-label ${{ github.sha }}
```
Make sure you adjust the following according to your settings
```
  aws-region: ap-southeast-2
```
```
  run: aws s3 cp deploy_package.zip s3://elasticbeanstalk-ap-southeast-2-482739392776/EBApptest
```
```
  --source-bundle S3Bucket="elasticbeanstalk-ap-southeast-2-482739392776",S3Key="EBApptest/deploy_package.zip"
```
```
  --environment-name Sample-docker-react-env \
```

Checkout a new branch and Commit your changes. Then raise and merge a PR, see what will happen. 

Q&A


问题1，在创建一个新的 IAM role，取名为BTN_EC2_profile时，选择EC2作为use case并赋予这个role一些elastic beanstalk的权限，但我从未专门去ec2创建任何instance，那为什么在创建elastic beanstalk application时，在configure service access里会自动出现ec2 key pair的选项，而且ec2 instance profile可以选择我刚刚建的IAM role with the name BTN_EC2_profile？这里为什么需要用到ec2 key pair？

回答：

 IAM Role是「一套权限」，是AWS 给你一个可以被 EC2 实例使用的「身份凭证」，；它用来 临时授权，常被 EC2、Lambda 等 AWS 服务「扮演」来访问资源。你在 IAM 里创建的 role（比如你给它取名 BTN_EC2_profile），如果在创建时选了 EC2 use case，那相当于你告诉 AWS这个角色是给 EC2 实例用的。如此一来，AWS 就会自动为你生成一个 同名的 EC2 instance profile。这是AWS干的，显然你所做的只是创建 IAM role，并没有专门跑到EC2去建一个instance。而且新手在这里要特别注意： EC2 instance profile不是 EC2 instance！！！这是两回事：

✅ EC2 Instance Profile：
	•	本质是一个 IAM Role 的「包装器」，用来附加到 EC2 实例上。
	•	它 不是一台机器，只是一个「权限身份」容器，即装着这个IAM Role 所拥有的权限的容器。
那可以理解为，你每创建一个选择EC2 作为use case的IAM Role ，AWS就会自动帮你生成一个装着这个IAM Role 所拥有的权限的EC2 instance profile容器，之后可以用来附加到 EC2 实例上用。

✅ EC2 Instance：
	•	就是实际运行在 AWS 上的 虚拟机服务器。

小结：

所以可以明确一点，你创建 IAM role ≠ 创建 EC2 实例，只是做了准备，以后你在 启动 EC2 实例时，可以把这个AWS帮你自动生成的 Instance Profile容器 附加到这台机器，也就是把这个EC2 实例和这个IAM Role绑定了，即这个EC2 实例拥有了这个IAM Role带有的权限，这样 这个EC2 实例上的应用就能用这个 role 的权限访问 AWS 资源（比如 S3）。


Elastic Beanstalk 环境里一定会有 EC2 实例，你的 app 实际会部署并运行在这些 EC2 上。当你在 Elastic Beanstalk 上创建应用环境时，AWS 会 自动在 EC2 里帮你启动一个（或多个）虚拟机实例，来运行你的代码。这些 EC2 实例需要选择一个EC2 Instance Profile，也就是绑定一个 IAM Role，才能实现用于让这些实例拥有这个IAM Role权限去访问 AWS 资源的目的（比如拉取镜像、写日志到 CloudWatch 等）。

Elastic Beanstalk：AWS 提供的「全托管平台」，它自动帮你创建、配置、伸缩、更新 EC2 实例，并且能整合负载均衡、存储、日志等。而EC2只是AWS 提供的裸虚拟机，你自己搭建、配置、管理环境

关于 Key Pair：因为 Elastic Beanstalk 部署应用的底层，其实就是在 AWS EC2 上建机器，AWS 给你一个选项：

「如果你需要，将来手动 SSH 到 EC2 排查问题，请选择或创建一个 Key Pair」

如果你没选 key pair，将来你就不能用 SSH 登录进去看日志或调试机器。创建一个 Key Pair不是必须（你可以选择「不生成 key pair」）
	•	但 AWS 给你留了「后门」，如果你想 SSH 登录机器，就必须先生成并下载好 key pair 文件（.pem），因为 AWS 只在机器创建时设置公钥

🎯 总结

Beanstalk = 「帮你自动创建+管理 EC2 机器」的服务
	•	所以需要 EC2 key pair（可选） → 如果你想手动连进去看机器
	•	需要 EC2 instance profile（必须） → 让机器拥有 IAM 角色权限





问题2，结合上文，我想接着问为什么We need an IAM user with proper permissions so that github action can deploy the app on your behalf？是因为这个CD pipeline最终要在Github Actions里部署，前面的一系列操作是在AWS service进行的，所以需要创建一个 IAM USER并给这个USE创建access key，才能最终实现把这个IAM USER和github关联起来的目的吗？

回答：

是的！因为 GitHub Actions 运行在 GitHub 的服务器上（即「云端」的 CI/CD 服务），它 不是 AWS 自己的服务。为了让 GitHub Actions (AWS 外部) 能用 AWS API 部署应用（比如把包上传 S3、更新 Elastic Beanstalk 环境）时，需要 AWS 的身份凭证。，你要在 AWS 创建一个 IAM User，授予它权限，并把 Access Key 填到 GitHub Secrets 中，这样 GitHub Actions 就能代表你操作 AWS 了！

✅ 整个流程逻辑：
1️⃣ 你在 AWS 上配置 Elastic Beanstalk、EC2 Role、Key Pair 等，这是 AWS 内部的资源和角色
2️⃣ 但要让 GitHub Actions（外部）「远程控制 AWS」部署新版本，需要用 IAM User + Access Key 来让 GitHub 拥有权限去调用 AWS API




对于以上2问题及其答案，我们可以逆推：

这个大语境是，我们的最终任务是部署CD pipeline,它可以在 GitHub Actions、CodePipeline 等任何 CI/CD 工具里完成。本次我们选用GitHub Actions。而GitHub Actions 本质只是一个「自动化脚本运行环境」和「运行脚本」的平台。GitHub Actions 的工作环境 运行在 GitHub 的云端服务器上，它可以：
	•	获取你的代码
	•	编译、测试、打包
	•	执行各种命令行脚本

但它本身并不能创建脚本，更不能部署工作流水线。要明确CD的目的是把已经打包好的应用部署到 AWS 环境（比如 Elastic Beanstalk）。

由于 GitHub Actions 运行在 GitHub 的服务器上（即「云端」的 CI/CD 服务），它 不是 AWS 自己的服务。为了让 GitHub Actions (AWS 外部) 能用 AWS API 部署应用（比如把包上传 S3、更新 Elastic Beanstalk 环境）时，需要 AWS 的身份凭证。，你要在 AWS 创建一个 IAM User，授予它权限，并把 Access Key 填到 GitHub Secrets 中，这样 GitHub Actions 就能代表你操作 AWS 了！通俗总结，GitHub Actions 是一个外部工人，它不住在 AWS 家里，要想帮你部署 AWS 上的应用，就得 敲门（用 API）+ 出示凭证（Access Key）。

回顾CI的部署，并不需要用到AWS，那为什么CD需要呢？

这里的区别在于 CI 和 CD 阶段各自的职责：

✅ CI（Continuous Integration） —— 持续集成
	•	目的是在 GitHub Actions 等 CI 工具里 拉取代码、编译、运行测试、打包
	•	这些Build 的任务只在 GitHub 环境里就能完成，不需要接触 AWS

✅ CD（Continuous Deployment/Delivery） —— 持续部署
	•	目标是 把已经打包好的应用部署到 AWS 环境（比如 Elastic Beanstalk）
	•	GitHub 环境里只能build不能deploy，这些Deploy 的任务就需要 用到 AWS API、上传部署包、触发环境更新
	•	因为 GitHub Actions 本身不是 AWS 服务，所以必须先拿到 AWS 的权限（通过 IAM User + Access Key），才能在 CD 阶段控制 AWS 上的资源

⸻

🎯 总结
	•	CI 只是build and test→ 不需要 AWS，github环境里就能完成；
	•	CD 是把产物deploy到 AWS → 必须让 GitHub Actions 拥有访问 AWS 的权限




