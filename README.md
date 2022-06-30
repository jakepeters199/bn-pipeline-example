# Jenkins pipelines

## Goal

We are going to create a Jenkins pipeline that compiles, tests and deploys our application.

## Setup

We are going to create two containers. One that runs our jenkins instance and the other will be used as a runner to
execute Docker commands inside the Jenkins pipeline.

1. Ensure you have **Docker desktop** installed
2. Open up a terminal window
3. Create a bridge network in Docker using the following docker network create command:
   `docker network create jenkins`
4. In order to execute Docker commands inside Jenkins nodes, download and run the docker:dind Docker image using the
   following docker run command:
    ```bash
    docker run \
      --name jenkins-docker \
      --rm \
      --detach \
      --privileged \
      --network jenkins \
      --network-alias docker \
      --env DOCKER_TLS_CERTDIR=/certs \
      --volume jenkins-docker-certs:/certs/client \
      --volume jenkins-data:/var/jenkins_home \
      --publish 2376:2376 \
      --publish 3000:3000 \
      docker:dind \
      --storage-driver overlay2 
    ```
5. Now you should see the container running like so in Docker Desktop.
   ![img.png](resources/dockerdesktop.png)
6. Next we need to build to Docker image from the Dockerfile: `docker build -t myjenkins-blueocean:2.332.3-1 .`
   Keep in mind that the process described above will automatically download the official Jenkins Docker image if this
   hasn’t been done before.
7. Now you can create a container from the image you have just created:
    ```
   docker run \
    --name jenkins-blueocean \
    --detach \
    --network jenkins \
    --env DOCKER_HOST=tcp://docker:2376 \
    --env DOCKER_CERT_PATH=/certs/client \
    --env DOCKER_TLS_VERIFY=1 \
    --publish 8080:8080 \
    --publish 50000:50000 \
    --volume jenkins-data:/var/jenkins_home \
    --volume jenkins-docker-certs:/certs/client:ro \
    --volume "$HOME":/home \
    --restart=on-failure \
    --env JAVA_OPTS="-Dhudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true" \
    myjenkins-blueocean:2.332.3-1
    ```
8. We should now see both conatiners running in Docker Desktop
   ![img.png](resources/dockerDesktop2.png)

## Unlocking Jenkins

We now need to perform a series of one time steps to unlock our Jenkins instance.

1. Go to http://localhost:8080 and you should see the Jenkins unlock page
   ![img.png](resources/unlockJenkins.png)
   Unlock Jenkins page
2. Display the Jenkins console log with the command: `docker logs jenkins-blueocean`
3. From your terminal/command prompt window again, copy the automatically-generated alphanumeric password (between the 2
   sets of asterisks).
   ![img.png](resources/jenkinsPassword.png)
4. On the Unlock Jenkins page, paste this password into the Administrator password field and click Continue.

### Customizing Jenkins with plugins

1. After unlocking Jenkins, the Customize Jenkins page appears.

2. On this page, click Install suggested plugins.

3. The setup wizard shows the progression of Jenkins being configured and the suggested plugins being installed. This process may take a few minutes.

### Creating the first administrator user

1. Finally, Jenkins asks you to create your first administrator user.

2. When the Create First Admin User page appears, specify your details in the respective fields and click Save and
   Finish.
3. When the Jenkins is ready page appears, click Start using Jenkins.

**Notes:**

- This page may indicate Jenkins is almost ready! instead and if so, click Restart.
- If the page doesn’t automatically refresh after a minute, use your web browser to refresh the page manually.
- If required, log in to Jenkins with the credentials of the user you just created and you’re ready to start using
  Jenkins!

### Create your own git project
You will need to create your own repository that will then link with Jenkins.

1. Copy the contents of the CICD folder into a new directory.
2. Delete the exisitng `.git` folder
3. Run `git init`

### Create your Pipeline project in Jenkins
Go back to Jenkins, log in again if necessary and click create new jobs under Welcome to Jenkins!  
Note: If you don’t see this, click New Item at the top left.

In the Enter an item name field, specify the name for your new Pipeline project (e.g. example-pipeline).

Scroll down and click Pipeline, then click OK at the end of the page.

( Optional ) On the next page, specify a brief description for your Pipeline in the Description field (e.g. An entry-level Pipeline demonstrating how to use Jenkins to build a simple Node.js and React application with npm.)

Click the Pipeline tab at the top of the page to scroll down to the Pipeline section.

From the Definition field, choose the Pipeline script from SCM option. This option instructs Jenkins to obtain your Pipeline from Source Control Management (SCM), which will be your locally cloned Git repository.

From the SCM field, choose Git.

In the Repository URL field, specify the directory path of your locally cloned repository above, which is from your user account/home directory on your host machine, mapped to the /home directory of the Jenkins container - i.e.

For macOS - /home/Documents/CICD

For Linux - /home/CICD

For Windows - /home/Documents/CICD

Click Save to save your new Pipeline project. You’re now ready to begin creating your Jenkinsfile, which you’ll be checking into your locally cloned Git repository.

### Create your initial Pipeline as a Jenkinsfile
You’re now ready to create your Pipeline that will automate building your Node.js and React application in Jenkins. Your Pipeline will be created as a `Jenkinsfile`, which will be committed to your repository.

This is the foundation of "Pipeline-as-Code", which treats the continuous delivery pipeline as a part of the application to be versioned and reviewed like any other code.

We will start by creating a pipeline that has a "Build" stage.

Using your favorite text editor or IDE, create and save new text file with the name `Jenkinsfile` at the root of your local `CICD` Git repository.

Copy the following Declarative Pipeline code and paste it into your empty Jenkinsfile:
```
pipeline {
    agent {
        docker {
            image 'node:lts-bullseye-slim' 
            args '-p 3000:3000' 
        }
    }
    stages {
        stage('Build') { 
            steps {
                sh 'npm install' 
            }
        }
    }
}
```

### What does this all mean?  

1. The `image` parameter (of the agent section’s docker parameter) downloads the node:lts-bullseye-slim Docker image (if it’s not already available on your machine) and runs this image as a separate container.  

This means that:
- You’ll have separate Jenkins and Node containers running locally in Docker.

- The Node container becomes the agent that Jenkins uses to run your Pipeline project. However, this container is short-lived - its lifespan is only that of the duration of your Pipeline’s execution.

2. The args parameter makes the Node container (temporarily) accessible through port 3000. This will be used later when we 'deploy' our application
 
3. The `stage` setion of the code, defines a stage called Build that appears on the Jenkins UI. 
4. The `sh` step (of the steps section) executes the npm command to ensure that all dependencies required to run your application have been downloaded to the node_modules workspace directory.  
   Note: **If you want to run a shell command in the pipeline you need to put `sh` before it.**
   
Save your edited Jenkinsfile and commit it to your local `CICD` Git repository. E.g. Within the `CICD` directory, run the commands:  
`git add .`  
`git commit -m "Add initial Jenkinsfile"`

Go back to Jenkins again, log in again if necessary and click Open Blue Ocean on the left to access Jenkins’s Blue Ocean interface.

In the This job has not been run message box, click Run, then quickly click the OPEN link which appears briefly at the lower-right to see Jenkins building your Pipeline project. If you weren’t able to click the OPEN link, click the row on the main Blue Ocean interface to access this feature.  

Note: You may need to wait several minutes for this first run to complete. After making a clone of your local `CICD` Git repository itself, Jenkins:

- Initially queues the project to be run on the agent.

- Downloads the Node Docker image and runs it in a container on Docker.

- Downloading Node Docker image

- Runs the Build stage (defined in the Jenkinsfile) on the Node container. During this time, npm downloads many dependencies necessary to run your Node.js and React application, which will ultimately be stored in the node_modules workspace directory (within the Jenkins home directory).

- Downloading 'npm' dependencies

The Blue Ocean interface turns green if Jenkins built your Node.js and React application successfully.

Initial Pipeline runs successfully

Click the X at the top-right to return to the main Blue Ocean interface.

Main Blue Ocean interface

Add a test stage to your Pipeline
Go back to your text editor/IDE and ensure your Jenkinsfile is open.

Copy and paste the following Declarative Pipeline syntax immediately under the Build stage:

        stage('Test') {
            steps {
                sh './jenkins/scripts/test.sh'
            }
        }
so that you end up with:

pipeline {
    agent {
        docker {
            image 'node:lts-bullseye-slim'
            args '-p 3000:3000'
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'npm install'
            }
        }
        stage('Test') { 
            steps {
                sh './jenkins/scripts/test.sh' 
            }
        }
    }
}
Defines a stage (directive) called Test that appears on the Jenkins UI.
This sh step (of the steps section) runs the shell script test.sh located in the jenkins/scripts directory from the root of the `CICD` repository. Explanations about what this script does are covered in the test.sh file itself. As a general principle, it’s a good idea to keep your Pipeline code (i.e. the Jenkinsfile) as tidy as possible and place more complex build scripting steps into separate shell script files like the test.sh file. This ultimately facilitates the maintenance of your Pipeline, especially if it gains more complexity.
Save your edited Jenkinsfile and commit it to your local `CICD` Git repository. E.g. Within the `CICD` directory, run the commands:
git stage .
then
git commit -m "Add 'Test' stage"

Go back to Jenkins again, log in again if necessary and ensure you’ve accessed Jenkins’s Blue Ocean interface.

Click Run at the top left, then quickly click the OPEN link which appears briefly at the lower-right to see Jenkins running your amended Pipeline project. If you weren’t able to click the OPEN link, click the top row on the Blue Ocean interface to access this feature.
Note: You’ll notice from this run that Jenkins no longer needs to download the Node Docker image. Instead, Jenkins only needs to run a new container from the Node image downloaded previously. Also, from now on, no (new) npm dependencies should need to be downloaded during the "Build" stage. Therefore, running your Pipeline this subsequent time should be much faster.
If your amended Pipeline ran successfully, here’s what the Blue Ocean interface should look like. Notice the additional "Test" stage. You can click on the previous "Build" stage circle to access the output from that stage.

Test stage runs successfully (with output)

Click the X at the top-right to return to the main Blue Ocean interface.

Add a final deliver stage to your Pipeline
Go back to your text editor/IDE and ensure your Jenkinsfile is open.

Copy and paste the following Declarative Pipeline syntax immediately under the Test stage of your Jenkinsfile:

        stage('Deliver') {
            steps {
                sh './jenkins/scripts/deliver.sh'
                input message: 'Finished using the web site? (Click "Proceed" to continue)'
                sh './jenkins/scripts/kill.sh'
            }
        }
so that you end up with:

pipeline {
    agent {
        docker {
            image 'node:lts-buster-slim'
            args '-p 3000:3000'
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'npm install'
            }
        }
        stage('Test') {
            steps {
                sh './jenkins/scripts/test.sh'
            }
        }
        stage('Deliver') { 
            steps {
                sh './jenkins/scripts/deliver.sh' 
                input message: 'Finished using the web site? (Click "Proceed" to continue)' 
                sh './jenkins/scripts/kill.sh' 
            }
        }
    }
}
Defines a new stage called Deliver that appears on the Jenkins UI.
This sh step (of the steps section) runs the shell script deliver.sh located in the jenkins/scripts directory from the root of the `CICD` repository. Explanations about what this script does are covered in the deliver.sh file itself.
This input step (provided by the Pipeline: Input Step plugin) pauses the running build and prompts the user (with a custom message) to proceed or abort.
This sh step runs the shell script kill.sh, also located in the jenkins/scripts directory. Explanations about what this script does are covered in the kill.sh file itself.
Save your edited Jenkinsfile and commit it to your local `CICD` Git repository. E.g. Within the `CICD` directory, run the commands:
git stage .
then
git commit -m "Add 'Deliver' stage"

Go back to Jenkins again, log in again if necessary and ensure you’ve accessed Jenkins’s Blue Ocean interface.

Click Run at the top left, then quickly click the OPEN link which appears briefly at the lower-right to see Jenkins running your amended Pipeline project. If you weren’t able to click the OPEN link, click the top row on the Blue Ocean interface to access this feature.
If your amended Pipeline ran successfully, here’s what the Blue Ocean interface should look like. Notice the additional "Deliver" stage. Click on the previous "Test" and "Build" stage circles to access the outputs from those stages.

Deliver stage pauses for user input

Ensure you are viewing the "Deliver" stage (click it if necessary), then click the green ./jenkins/scripts/deliver.sh step to expand its content and scroll down until you see the http://localhost:3000 link.

Deliver stage output only

Click the http://localhost:3000 link to view your Node.js and React application running (in development mode) in a new web browser tab. You should see a page/site with the title Welcome to React on it.
Tip: If you’re feeling a little adventurous, you can try accessing the terminal/command prompt of your Jenkins Docker container, then using vi editor, tweak and save the App.js source file and see the results appear on the Welcome to React page. To do this, run the following commands:

docker exec -it <docker-container-name> bash 
cd /var/jenkins_home/workspace/`CICD example`/src 
vi App.js 
This command provides access to the terminal/command prompt of your Jenkins Docker container. The <docker-container-name> can be obtained using the command docker ps. Otherwise, it would be jenkins-tutorials (if you specified this in the command you used to run this container above - i.e. --name jenkins-tutorials).
Once in the container, change directory to the Node.js and React source directory (in the Jenkins workspace directory within Jenkins home).
Access, edit and save changes to your application’s App.js file using vi editor.
When you are finished viewing the page/site, click the Proceed button to complete the Pipeline’s execution.

Deliver stage runs successfully

Click the X at the top-right to return to the main Blue Ocean interface, which lists your previous Pipeline runs in reverse chronological order.

Main Blue Ocean interface with all previous runs displayed

Wrapping up
Well done! You’ve just used Jenkins to build a simple Node.js and React application with npm!

The "Build", "Test" and "Deliver" stages you created above are the basis for building more complex Node.js and React applications in Jenkins, as well as Node.js and React applications that integrate with other technology stacks.

Because Jenkins is extremely extensible, it can be modified and configured to handle practically any aspect of build orchestration and automation.