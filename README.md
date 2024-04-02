# NodeJS Application Setup Guide

## Overview

Your team members want to collaborate on your NodeJS application, where developers are listed with their associated projects. To facilitate collaboration and ensure code quality, you decide to set up a Git repository and implement a continuous integration (CI) pipeline. Additionally, you plan to Dockerize the application for easier deployment and access.


## Steps:

### STEP 1: Dockerize your NodeJS App
- Configure your application to be built as a Docker image.
- Dockerize your NodeJS app.

1. **Installing Docker**:

If you haven't already, you'll need to install Docker on your machine. You can download Docker from the [official website](https://www.docker.com/).

2. **Creating Dockerfile**:

In the root directory of your NodeJS project, create a file named `Dockerfile`. This file will contain instructions for Docker to build your application image.

3. **Writing Dockerfile**:

Open the `Dockerfile` in a text editor and add the following contents:

```Dockerfile

FROM node:20-alpine

WORKDIR /usr/app

COPY app/ .

EXPOSE 3000

RUN npm install

CMD ["node", "server.js"]
```

<img src="https://i.imgur.com/UN5u0F4.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


### STEP 2: Create a Full Pipeline for your NodeJS App

You want the following steps to be included in your pipeline:

1. **Increment Version**: 
   - Increment the application's version and Docker image version.

2. **Run Tests**: 
   - Test the code to ensure only working code is deployed. If tests fail, the pipeline should abort.

3. **Build Docker Image with Incremented Version**: 
   - Build the Docker image with the incremented version.

4. **Push to Docker Repository**: 
   - Push the built Docker image to the Docker repository.

5. **Commit to Git**: 
   - Commit the application version increment to a remote Git repository.

**Create Jenkins Credentials**
- Create usernamePassword credentials for docker registry called `docker-credentials`
- Create usernamePassword credentials for git repository called `gitlab-credentials`

<img src="https://i.imgur.com/Je7GTf1.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


**Configure Node Tool in Jenkins Configuration**
- Name should be `node`, because that's how it's referenced in the below Jenkinsfile in `tools` block

**Install plugin**
- Install `Pipeline Utility Steps` plugin. This contains readJSON function, that we will use to read the version from package.json 

<img src="https://i.imgur.com/Oqks1Dj.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


```Jenkinsfile

pipeline {
    agent any
    tools {
        nodejs "node"
    }
    stages {
        stage('increment version') {
            steps {
                script {
                    // enter app directory, because that is where package.json is located
                    dir("/jenkins-exercises/app") {
                        // update application version in the package.json file with one of these release types: patch, minor or major
                        // this will commit the version update
                        sh "npm version minor"

                        // read the updated version from the package.json file
                        def packageJson = readJSON file: 'package.json'
                        def version = packageJson.version

                        // set the new version as part of IMAGE_NAME
                        env.IMAGE_NAME = "$version-$BUILD_NUMBER"
                    }
                }
            }
        }
        stage('Run tests') {
            steps {
                script {
                    // enter app directory, because that is where package.json is located
                    dir("/jenkins-exercises/app") {
                        // install all dependencies needed for running tests
                        sh "npm install"
                        sh "npm run test"
                    }
                }
            }
        }
        stage('Build and Push docker image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        // git config here for the first time run
                        sh 'git config --global user.email "amina.iprahim@gmail.com"'
                        sh 'git config --global user.name "amina"'
                        sh 'git remote set-url origin https://$USER:$PASS@https://gitlab.com/aminaahmed-cloud/jenkins-exercises.git'
                        sh 'git add .'
                        sh 'git commit -m "ci: version bump"'
                        sh 'git push origin HEAD:jenkins-jobs'
                    }
                }
            }
        }
    }
}
```

<img src="https://i.imgur.com/HxZ1plm.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


<img src="https://i.imgur.com/nc5C8rs.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


<img src="https://i.imgur.com/wqvVPlM.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


<img src="https://i.imgur.com/TcTaico.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


<img src="https://i.imgur.com/ka2swSG.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>


<img src="https://i.imgur.com/6BykwDx.png" height="80%" width="80%" alt="Disk Sanitization Steps"/>

  
### STEP 3: Manually Deploy New Docker Image on Server
After the pipeline has run successfully, you:
- Manually deploy the new Docker image on the droplet server.

**steps:**

```sh
# ssh into your droplet server
ssh -i ~/id_rsa root@{server-ip-address}

# login to your docker hub registry
docker login

# pull and run the new docker image from registry
docker run -p 3000:3000 aminaaahmed323/myapp:{image-name}

```

### STEP 4: Extract into Jenkins Shared Library
A colleague from another project tells you that they are building a similar Jenkins pipeline and they could use some of your logic. So you suggest creating a Jenkins Shared Library to make your Jenkinsfile code reusable and shareable.
- Extract all logic into Jenkins-shared-library with parameters and reference it in Jenkinsfile.
