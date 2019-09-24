## Setup VM JC

### Prerequisites

- Memory : 6GB
- Disk : 10 GB

### Outils utilisés dans ce lab
+ [Virtualbox](https://www.virtualbox.org/wiki/Downloads)
+ [Vagrant](https://www.vagrantup.com/downloads.html)
+ [Docker](https://docs.docker.com/install/)
+ [Jenkins](https://jenkins.io/download/)
+ [Sonarqube](https://www.sonarqube.org/downloads/)
+ [Gogs](https://gogs.io/docs/installation)

### Préparer le dossier de travail
```
mkdir vm
```

### initialiser une vm basé sur ubuntu/xenial64
```
vagrant box add --name ubuntu/xenial64 local_box_file.box
vagrant init -m ubuntu/xenial64
```

Un fichier Vagrantfile est crée, contenant la description de la VM.

### Démarrer la vm
```
vagrant up
```

### Appliquer 6Go de mémoire à la vm
Dans le fichier Vagrantfile rajouter
```
config.vm.provider "virtualbox" do |vb|
  vb.memory = "6144"
end
```

### Recharger la configuration de la vm et vérifier la taille memoire
```
vagrant reload
```

### Approvisionnement de la vm

Dans le fichier Vagrantfile rajouter

#### en root
```
config.vm.provision "shell", inline: <<-SHELL
  # appliquer : update, upgrade
  # installer les packages  wget zip unzip vim git openjdk-8-jdk
  # installer docker
  # installer docker-compose
  # patch de l'os avec l'instruction suivante
  # update max_map_count used by sonarqube avec cette instruction
  sysctl -w vm.max_map_count=262144  
SHELL
```

#### avec vagrant user
```
config.vm.provision "shell", privileged: false, inline: <<-SHELL
  # Maven installation
  curl -s "https://get.sdkman.io" | bash
  source $HOME/.sdkman/bin/sdkman-init.sh
  sdk version
  sdk install maven 3.6.0
  source .bashrc
  java -version
  mvn --version
SHELL
```  

### Relancer l'étape d'approvisionnement
```  
vagrant provision
```  

### Accéder à la vm
```  
vagrant ssh
```  

### Vérifier les outils installés
```  
java -version
mvn --version
docker -v
docker-compose -v
```  

### Préparation des conteneurs

#### Créer le dossier compose à la racine du dossier work
```  
mkdir compose
```  

#### Créer un fichier docker-compose.yml dans le dossier compose

Avec le contenu suivant :
https://github.com/kamel2k/setup-vm-jc/blob/master/compose/docker-compose.yml

#### Lancer le compose lors de l'approvisionnement de la vm
Dans Vagrantfile rajouter
```
config.vm.provision "shell", inline: <<-SHELL
   cd /vagrant/compose && docker-compose up
SHELL
```

#### Redirection de ports de la VM
Dans Vagrantfile rajouter
```
config.vm.network "forwarded_port", guest: 8080, host: 8080 # jenkins
config.vm.network "forwarded_port", guest: 3000, host: 3000 # git
config.vm.network "forwarded_port", guest: 8081, host: 8081 # application
config.vm.network "forwarded_port", guest: 9000, host: 9000 # sonarqube
```

### Tester les accès aux applications
+ [Jenkins](http://localhost:8080)
+ [Gogs](http://localhost:8080)
+ [Sonarqube](http://localhost:9000)
