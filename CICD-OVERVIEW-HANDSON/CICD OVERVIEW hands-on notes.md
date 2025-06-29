# Github Action CI/CD

### Goal: Build, Test and Deploy a web-app to AWS Elastic Beanstalk from scratch

## What is CI/ CD?
![Alt text](../images/CI_CD.png?raw=true)

### Continuous Integration
Successful **Continuous Integration** means code changes to a web-app are regularly built, tested, and merged to a shared
repository automatically via integration tools like Jenkins, Github Actions (GHA), Bitbucket pipelines.
It’s a solution to the problem of
1. having too many developers writing code changes to a web-app that might conflict with each other.
2. code testing and validation before merging.

### Continuous delivery
Continuous delivery usually means a developer’s changes to an application are automatically bug tested and
uploaded to a repository (like GitHub or a container registry), where they can then be deployed to a live
production environment by the operations team.
It’s an answer to the problem of poor visibility and communication between dev and business teams.
To that end, the purpose of continuous delivery is to ensure that it takes minimal effort to deploy new code.

### Continuous deployment
Continuous deployment (the other possible “CD”) can refer to automatically releasing a developer’s changes from
the repository to production, where it is usable by customers. It addresses the problem of overloading operations
teams with manual processes that slow down app delivery. It builds on the benefits of continuous delivery by
automating the next stage in the pipeline.


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
Create a sample React web-app skeleton (nextjs.js) with npx: https://react.dev/learn/start-a-new-react-project 

Set `my-app` as your project name and proceed with default options for other questions.
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

You can run locally the docker containers for testing your Dockerfile (`-f Dockerfile` is optional)
```
docker build -t my-app -f Dockerfile .
docker run --rm -d -p 3000:3000 --name nextjs-react-app my-app
```


## 3. Add the repo to Github
Create a repo in your github
![Alt text](../images/Create_new_repo.png?raw=true)

Follow the steps in the second section (``...or push an existing repository from the command line``) to push existing repo from the command line
![Alt text](../images/Steps.png?raw=true)


## 4. Let us setup Github Action CI
Note: 
+ The uses field in the Setup Node.js step specifies the actions/setup-node action to set up the Node.js environment. 
This action automatically installs the specified version of Node.js and sets up the environment variables.
+ Your dockerhub username and password were saved in the sample-docker-react repo's `Secrets and variables` (https://github.com/<your_org>/sample-docker-react/settings/secrets/actions)
<p align="center">
  <img src="../images/gha_secrets.png">
</p>

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

Now push the changes to your git repo
```
git checkout -b "feature/introduce_ci"
git add .
git commit -m "Introduce github action CI pipeline"
git push --set-upstream origin "feature/introduce_ci"
git checkout main
```
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
