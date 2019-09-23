
1. créer un dossier vm

2. initialiser une vm basé sur ubuntu/xenial64

vagrant init ubuntu/xenial64

3. démarrer la vm
vagrant up

4. appliquer 6Go de mémoire à la vm

config.vm.provider "virtualbox" do |vb|
  vb.memory = "8192"
end

5. recharger la configuration de la vm et vérifier la taille memoire
vagrant reload


6. approvisionnement de la vm

* en root
  - appliquer : update, upgrade
  - installer les packages  wget zip unzip vim git openjdk-8-jdk
  - installer docker :

  apt-get install apt-transport-https ca-certificates curl software-properties-common
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
  add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
  apt-get update -y
  apt-get install docker-ce -y

  - installer docker-compose
  curl -L "https://github.com/docker/compose/releases/download/1.22.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
  chmod +x /usr/local/bin/docker-compose
  docker-compose --version

* avec l'utilisateur vagrant

  - installer maven

  curl -s "https://get.sdkman.io" | bash
  source $HOME/.sdkman/bin/sdkman-init.sh
  sdk version
  sdk install maven 3.6.0
  source .bashrc
  java -version
  mvn --version

* syntaxe pour executer des commandes root :

config.vm.provision :shell, :inline  => '
    '

* syntaxe pour executer des commandes avec vagrant

config.vm.provision :shell,:inline  => '
    ', :privileged => false

7. relancer l'etape de provisionning
vagrant provision

8. accéder à la vm
vagrant ssh

9. vérifier les outils installés
java -version
mvn --version
docker -v
docker-compose -v

10. preparation des conteneurs (jenkins, gogs, sonarqube)

10.1 créer un dossier compose

10.2. copier tous les fichiers necessaires pour les conteneurs

11. ouverture des ports applicatifs (a la fin)

    config.vm.network "forwarded_port", guest: 8080, host: 8080 # jenkins
    config.vm.network "forwarded_port", guest: 3000, host: 3000 # git
    config.vm.network "forwarded_port", guest: 8081, host: 8081 # application
    config.vm.network "forwarded_port", guest: 9000, host: 9000 # sonarqube

12. Tester les accès aux applications

13. ceux qui ont des difficultées pour la mise en place de la vm

sudo sysctl -w vm.max_map_count=262144


## Hand-on Devops with Jenkins and Docker Tooling for Mule project

### Event information

### Place

**[JCC](https://www.jasmineconseil.com)**   
September 25, 2019

### About me

Kamel NAJJAR, is an Architect and Devops/Cloud Expert for several years, and is involved in various Devops projects in major international companies.

### Summary

A practical workshop focuses on the implementation of DevOps tooling for :
- Understand the DevOps life cycle with associated tooling and specifically designed for this workshop (Jenkins, Sonarqube, Gogs, etc.)
- Define and implement a powerful pipeline of continuous deployment with associated tooling
- Using the docker container combined with the Vagrant solution to automate its software deployment chain

### Prerequisites

- Memory : 6GB
- Disk : 10 GB

### Tools used in the lab
+ [Virtualbox](https://www.virtualbox.org/wiki/Downloads)
+ [Vagrant](https://www.vagrantup.com/downloads.html)
+ [Docker](https://docs.docker.com/install/)
+ [Jenkins](https://jenkins.io/download/)
+ [Sonarqube](https://www.sonarqube.org/downloads/)
+ [Gogs](https://gogs.io/docs/installation)

### Application overview

This is a hello world application designed to work with mule runtime inside a docker container.

For more information :  
+ [github](https://github.com/kamel2k/hello-mule4)

### Setting up Vagrant environment

Start by cloning the following github repository, then run and access the vm with the following command :

```markdown
vagrant up && vagrant ssh
```

Clone the application inside /home/vagrant
```markdown
cd /home/vagrant
git clone https://github.com/kamel2k/hello-mule4.git
```

Compile the application
```markdown
cd /home/vagrant/hello-mule4
mvn clean install
```

Download Mule Server
[link](https://www.mulesoft.com/lp/dl/mule-esb-enterprise)

Deploy
```markdown
copy the jar file inside standalone apps directory
```

Launch the application
```markdown
inside /bin
./mule
```

The application is available on http://localhost:8081/test/hello

### Gogs integration

Perform the first steps of configuring Gogs at http://localhost:3000/ with the following informations :

> **Type de base de données** : PostgreSQL  
> **Hôte** : postgres:5432  
> **Utilisateur** : dbuser  
> **Mot de passe** : dbpass  
> **Nom de base de données** : gogs  
> **Domaine** : localhost  
> **Port SSH** : 10022  
> **Nom d'utilisateur** : administrateur  
> **Mot de passe** : administrateur


#### Push the application

Before pushing the application, configure git with user.email and user.name

```markdown
git config --global user.email "admin@admin.com"
git config --global user.name "administrateur"
```

Start by creating **hello-mule4** repository in Gogs and enable security. Then put the application sources in Gogs

```markdown
cd /home/vagrant/hello-mule4
rm -rf .git
git init
git add .
git commit -m "first commit"
git remote add origin http://localhost:3000/administrateur/hello-mule4.git
git push -u origin master
```

Check that the repository contains the sources of the application in Gogs.

### Creating a Jenkins Pipeline

Jenkins is available here:

#### Configure Jenkins
> http://localhost:8080/

Start by unlocking jenkins with the following command

```markdown
sudo docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Then, Install the suggested plugins, define an administrator account (admin/admin) and finally put http://localhost:8080/ in instance configuration.

Restart Jenkins
```markdown
sudo docker restart jenkins
```

Install the [Blue Ocean plugin](https://jenkins.io/doc/book/blueocean/getting-started/) for better pipeline visibility.  
Blue Ocean puts Continuous Delivery in reach of any team without sacrificing the power and sophistication of Jenkins.

Follow these steps :
```markdown
Jenkins > Administration > Plugin Management > Available > Blue Ocean
```

#### Create pipeline

```markdown
New item> pipeline> Name: hello-mule4-pipeline
```
In the
```markdown
Pipeline > Script section
```

Put
```markdown
pipeline {
  agent {
    docker {
      image 'maven:3-alpine'
      args '--network=compose_labnet -u root:root -v /home/vagrant/.m2:/root/.m2'
    }
  }
  stages {
    stage('Maven version') {
      steps {
        sh 'mvn --version'
      }
    }
  }
}
```
Launch the pipeline. This pipeline will show the maven version of the docker agent

***Explanation***
- **maven:3-alpine** is a lightweight image for maven 3  
- Argument **--network=compose_labnet** allows the container to be inside the labnet network which hosts all lab tools (Jenkins, Nexus, etc.)
- The shared folder **/home/vagrant/.m2:/root/.m2** perform a cache operation for maven repository folder between host and container


#### Using Snippet Generator

Clone the application using the snippet generator

> In hello-mule4-pipeline > Configure > Pipeline > Syntax Pipeline

**Sample Step** : git: Git  
**Repository URL** : http://gogs:3000/administrateur/hello-mule4.git
**Branch** : master  
**Credentials** : (Add) administrateur/******  

Hit the Generate Pipeline Script Button

```markdown
git credentialsId: 'gogs-credentials', url: 'http://gogs:3000/administrateur/hello-mule4.git'
```

Here is the result of the pipeline snippet generator

```markdown
pipeline {
  agent {
    docker {
      image 'maven:3-alpine'
      args '--network=compose_labnet -u root:root -v $HOME/.m2:/root/.m2'
    }
  }
  stages {
    stage('Checkout') {
      steps {
        git credentialsId: 'gogs-credentials', url: 'http://gogs:3000/administrateur/hello-mule4.git'
      }
    }
  }
}
```

Launch the hello-mule4-pipeline job and see the result

#### Pipeline as Code

Pipeline as Code describes a set of features that allow Jenkins users to define pipelined job processes with code, stored and versioned in a source repository. These features allow Jenkins to discover, manage, and run jobs for multiple source repositories and branches — eliminating the need for manual job creation and management.

Begin by creating a file named Jenkinsfile in the root of hello-mule4 project which contain :

```markdown
pipeline {
  agent {
    docker {
      image 'maven:3-alpine'
      args '--network=compose_labnet -u root:root -v /home/vagrant/.m2:/root/.m2'
    }
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }
  }
}
```

**Configure hello-mule4-pipeline to use the Jenkinsfile of our project**

>Jenkins > Configure Job "hello-mule4-pipeline" > Pipeline > Definition : Pipeline Script from SCM

Save job modification, push Jenkinsfile to Gogs and launch hello-mule4-pipeline

#### Webhook configuration      

**Jenkins configuration**

Start by installing [gogs plugin](https://github.com/jenkinsci/gogs-webhook-plugin)

**Gogs configuration**

>Hello-Mule4 project > Settings > Webgooks > Add webhook -> Gogs  
> **Data URL** : http://jenkins:8080/gogs-webhook/?job=hello-mule4-pipeline  
> **Content type** : application/json  

Hit the **Test version** button and check that the Jenkins build is triggered

#### Build hello-mule4

Integrate the build step of the application into Jenkinsfile

```markdown
stage('build') {
  steps {
    sh 'mvn clean install'
  }
}
```

After pushing to gogs check if the Jenkins Build is triggered automatically

### Code Quality

Sonarqube is available here:

#### Configure Sonarqube
> http://localhost:9000/

Login credentials are:
> admin/admin

Give a name for the token : **my-token** for example

Choose **Java** as project's main language and **Maven** as build technology

Then copy the generated maven command and enrich the Jenkinsfile with the "Quality" stage

```markdown
stage('Code Quality') {
  steps {
    sh 'mvn sonar:sonar \
    -Dsonar.host.url=http://sonarqube:9000 \
    -Dsonar.login=2a0af4e245f2866510b5980d3fd567397489c11b'
  }
}
```

Push the file to Gogs and check the result in Sonarqube


### Deploy application

Create a Dockerfile in the root application folder with this content

```markdown
FROM javastreets/mule:latest

COPY ./target/hello-mule4*.jar /opt/mule/apps/

CMD [ "/opt/mule/bin/mule"]
```

Add deploy stage inside the Jenkinsfile with the given snippet

```markdown
stage('deploy') {  	   
  agent any           
    steps {             
      sh 'docker ps -a | grep hello-mule4 && docker stop hello-mule4 && docker rm hello-mule4 || echo "Build and execute container" '               
      sh 'docker build -t hello-mule4 .'    	     
      sh 'docker run --name hello-mule4 -d -p 8081:8081 hello-mule4'  
    }        
}
```

Push modified files, wait for jenkins build to finish and browse http://localhost:8081/test/hello
