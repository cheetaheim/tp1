# TP 1 - Docker

## Database - Basics

### Démarrage du conteneur

```bash
docker run -d -p 5432:5432 --name postgre postgre
```

### Vérification du conteneur en cours d'exécution

```bash
docker ps
```

### Arrêt et suppression du conteneur PostgreSQL existant

```bash
docker stop postgre
postgre
docker rm postgre
postgre
```

### Variable d'environement

```bash
docker run -d -p 5432:5432 --name postgre --network app-network -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd postgre
```

## Init Database

### Création d'un network

```bash
docker network create app-network
```

### Lancement de adminer

```bash
docker run -p "8090:8080" --net=app-network --name=adminer -d adminer
```

### Conservation des données

```bash
-v /my/own/datadir:/var/lib/postgresql/data
```

## Backend API

### API simple back-end

```dockerfile
# Build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

# Run
FROM amazoncorretto:17
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/your-application-name.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar
```

#### Question

1-2 Pourquoi avons-nous besoin d'une construction en plusieurs étapes ? Expliquez chaque étape de ce fichier docker.

La construction en plusieurs étapes dans un Dockerfile permet d'optimiser la taille de l'image finale en séparant les étapes de construction et d'exécution. D'abord, on utilise une image Maven pour compiler et packager l'application Java sans exécuter les tests, réduisant ainsi le temps de construction. Ensuite, une image Amazon Corretto plus légère sert à exécuter l'application, où seulement le fichier JAR nécessaire est copié. Cette méthode assure une image finale propre et minimale, contenant uniquement les éléments essentiels pour l'exécution.

## API back-end

### Configuration

```yml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          lob:
            non_contextual_creation: true
    generate-ddl: false
    open-in-view: true
  datasource:
    url: jdbc:postgresql://postgre-db:5432/db
    username: usr
    password: pwd
    driver-class-name: org.postgresql.Driver
management:
  server:
    add-application-context-header: false
  endpoints:
    web:
      exposure:
        include: health,info,env,metrics,beans,configprops
```

## Docker-compose TO DO

### Config

```yml
version: "3.7"

services:
  httpd:
    build:
      context: ./server
    ports:
      - "8080:80"
    networks:
      - app-network
    depends_on:
      - backend
    container_name: httpd-container

  backend:
    build:
      context: ./simple-api-student-main
      dockerfile: Dockerfile
    environment:
      POSTGRES_DB: "db"
      POSTGRES_USER: "usr"
      POSTGRES_PASSWORD: "pwd"
    networks:
      - app-network
    depends_on:
      - database
    container_name: simple-api-student

  database:
    image: cheetaheim/postgres-db
    environment:
      POSTGRES_DB: "db"
      POSTGRES_USER: "usr"
      POSTGRES_PASSWORD: "pwd"
    networks:
      - app-network
    container_name: postgres-db

networks:
  app-network:
```

1-3 Commandes Docker-Compose Importantes:

docker-compose up: Démarre les services définis dans le fichier docker-compose.yml.
docker-compose down: Arrête et supprime les ressources créées par up.
docker-compose build: Construit ou reconstruit les services ayant une option de construction.
docker-compose logs: Affiche les logs de tous les services.

1-4 Explication du Fichier Docker-Compose:

Ce fichier définit trois services (httpd, backend, database) dans un réseau commun app-network. Le service httpd sert de reverse proxy, dépendant du backend qui, à son tour, dépend de la base de données database. Chaque service est configuré avec des ports spécifiques, des variables d'environnement pour la connexion à la base de données, et des noms de conteneurs personnalisés. Cette organisation assure que la base de données est lancée avant le backend, et le backend est lancé avant le httpd, respectant les dépendances entre services.

1-5 Documentez vos commandes de publication et vos images publiées dans Dockerhub

Construction de l'image : docker build -t username/imagename:tag.
Connexion Docker Hub : docker login.
Publication : docker push username/imagename:tag.

# TP 2 - Github Action

### Question

#### C'est quoi les testcontainers?

Testcontainers est une bibliothèque Java utilisée pour tester des applications en utilisant des conteneurs Docker de manière automatisée. Elle permet de créer des environnements de test temporaires et isolés pour des bases de données, des serveurs web, ou toute autre dépendance externe requise par les tests d'intégration. Cela simplifie le processus de mise en place et de nettoyage après les tests, rendant les tests plus reproductibles et moins dépendants de l'environnement de développement ou de production.

## Workflow

```yml
name: CI devops 2023
on:
  # To begin, you want to launch this job in main and develop
  push:
    branches:
      - main
      - develop
  pull_request:

jobs:
  test-backend:
    runs-on: ubuntu-22.04
    steps:
      # Checkout your GitHub code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

      # Set up JDK 17
      - name: Set up JDK 17
        uses: actions/setup-java@v2
        with:
          distribution: "adopt"
          java-version: "17"

      # build of the app with the latest command
      - name: Build and test with Maven
        #run: mvn clean verify --file "simple-api-student-main/pom.xml"
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=cheetaheim_tp1 -Dsonar.organization=cheetaheim -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }} --file "simple-api-student-main/pom.xml"
```

### Question

2-2 Configurations d'actions Github (1ère partie):
Cette partie récupère le code avec actions/checkout@v2.5.0, configure l'environnement Java 17 via actions/setup-java@v2 en utilisant AdoptOpenJDK, et lance ensuite les commandes Maven pour construire l'application et exécuter les tests, y compris l'analyse avec SonarCloud, en utilisant les secrets pour l'authentification.

## Workflow Build and publish

```yml
build-and-push-docker-image:
  needs: test-backend
  runs-on: ubuntu-22.04

  steps:
    - name: Checkout code
      uses: actions/checkout@v2.5.0

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1

    - name: Login to DockerHub
      uses: docker/login-action@v1
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}

    - name: Build and push backend image
      uses: docker/build-push-action@v3
      with:
        context: ./simple-api-student-main # Chemin vers le dossier contenant le Dockerfile du backend
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-simple-api:latest
        push: ${{ github.ref == 'refs/heads/main' }} # Pousse l'image seulement si le commit est sur la branche main

    - name: Build and push frontend image
      uses: docker/build-push-action@v3
      with:
        context: ./devops-front-main
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-frontend:latest
        push: ${{ github.ref == 'refs/heads/main' }}

    - name: Build and push postgres image
      uses: docker/build-push-action@v3
      with:
        context: ./db
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-postgres:latest
        push: ${{ github.ref == 'refs/heads/main' }}

    - name: Build and push httpd image
      uses: docker/build-push-action@v3
      with:
        context: ./server
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/tp-devops-httpd:latest
        push: ${{ github.ref == 'refs/heads/main' }}
```

### Question

2-2 Configurations d'actions Github (2ème partie):
Cette partie s'occupe de la récupération du code, configuration de Docker Buildx, connexion à Docker Hub avec des identifiants sécurisés, construction et publication des images Docker pour le backend, le frontend, la base de données PostgreSQL, et le serveur HTTPD. Les images sont taguées avec le nom d'utilisateur Docker Hub et le tag latest, mais ne sont poussées sur Docker Hub que si les modifications sont faites sur la branche main.

Documentez la configuration de votre portail qualité :

Reliability : Détecte les bugs pouvant causer des crashs ou des comportements inattendus.
Maintainability : Évalue la facilité avec laquelle le code peut être compris, modifié ou étendu.
Security : Identifie les vulnérabilités potentielles qui pourraient être exploitées.
Security Review : Analyse manuelle et automatique pour les problèmes de sécurité.
Coverage : Mesure le pourcentage de code testé.
Duplications : Identifie les blocs de code similaires ou dupliqués.

# TP 3 - Ansible

## Installation et connexion

Installation ansible :

```bash
sudo apt install ansible
```

Lecture clef ssh :

```bash
chmod 400 /home/ahmet/Downloads/id_rsa
```

Connexion SSH :

```bash
ssh-keygen -f "/root/.ssh/known_hosts" -R "ahmet.turk.takima.cloud"
```

## Ping

Changement de droit :

```bash
sudo chmod 600 /home/ahmet/Downloads/id_rsa
```

Ping success :

```bash
ansible all -m ping
ahmet.turk.takima.cloud | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```

## Inventories

Inventory :

```bash
all:
 vars:
   ansible_user: centos
   ansible_ssh_private_key_file: /home/ahmet/Downloads/id_rsa
 children:
   prod:
     hosts: ahmet.turk.takima.cloud
```

### Question

3-1 Documentation inventory :

all : Il définit des variables globales (ansible_user et ansible_ssh_private_key_file) qui s'appliquent à tous les hôtes répertoriés dans l'inventaire.
prod : Il regroupe les hôtes sous la catégorie prod, avec une liste d'hôtes (ahmet.turk.takima.cloud dans mon cas).

## Playbooks

playbook.yml :

```yml
---
- name: Déploiement de l'application
  hosts: all
  gather_facts: false
  become: yes

  vars:
    postgres_db: "db"
    postgres_usr: "usr"
    postgres_pwd: "pwd"
    postgres_container: "postgres-db"
    postgres_port: "5432"

  roles:
    - install-docker
    - create-network
    - launch-database
    - launch-app
    - launch-front
    - launch-proxy
```

### Question

3-2 Documentation playbook :

Ce playbook est utilisé pour automatiser le déploiement de l'application sur les hôtes cibles.

Facts: Désactivés pour ne pas collecter de détails sur les hôtes cibles.

Privilèges: Activés pour exécuter les tâches avec des privilèges élevés (sudo).

Variables: Définit des variables pour la configuration du serveur PostgreSQL.

Rôles: Installe Docker, crée un réseau Docker, lance les conteneurs PostgreSQL, déploie l'application backend et frontend, et configure un serveur proxy.

Documentation de la configuration des tâches docker_container :

```yml
---
- name: Create Docker network
  docker_network: # Utilise le module docker_network pour gérer les réseaux Docker
    name: app-network # Spécifie le nom du réseau Docker à créer
    state: present # Indique que le réseau doit être présent, s'il n'existe pas déjà
  vars: # Définit les variables pour cette tâche
    ansible_python_interpreter: /usr/bin/python3 # Spécifie l'interpréteur Python à utiliser pour l'exécution de cette tâche
```

```yml
---
- name: Install device-mapper-persistent-data
  yum: # Utilise le module yum pour gérer les packages via le gestionnaire de paquets yum
    name: device-mapper-persistent-data # Spécifie le nom du package à installer
    state: latest # Indique que la dernière version du package doit être installée

- name: Install lvm2
  yum:
    name: lvm2
    state: latest

- name: Add Docker repository
  command: # Utilise la commande pour exécuter une action dans le shell
    cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo # Ajoute le référentiel Docker au gestionnaire de paquets yum

- name: Install Docker
  yum:
    name: docker-ce
    state: present # Indique que Docker doit être installé et présent sur le système

- name: Install python3
  yum:
    name: python3
    state: present # Installe Python 3 sur le système

- name: Install Docker Python library
  pip: # Utilise le module pip pour installer des packages Python
    name: docker # Spécifie le nom du package Python à installer
    executable: pip3 # Spécifie l'exécutable pip à utiliser (pip3 pour Python 3)
  vars: # Définit les variables pour cette tâche
    ansible_python_interpreter: /usr/bin/python3 # Spécifie l'interpréteur Python à utiliser pour l'exécution de cette tâche

- name: Make sure Docker is running
  service: # Utilise le module service pour gérer les services système
    name: docker # Spécifie le nom du service à gérer (Docker)
    state: started # Indique que le service doit être démarré
  tags: docker # Ajoute un tag pour cette tâche, permettant de l'exécuter sélectivement via les balises
```

```yml
---
- name: Run App
  docker_container: # Utilise le module docker_container pour gérer les conteneurs Docker
    name: simple-api-student # Spécifie le nom du conteneur à gérer
    image: cheetaheim/tp-devops-simple-api:latest # Spécifie l'image Docker à utiliser pour le conteneur
    state: started # Indique que le conteneur doit être démarré
    env: # Définit les variables d'environnement à passer au conteneur
      POSTGRES_DB: "{{ postgres_db }}" # Variable d'environnement pour la base de données PostgreSQL
      POSTGRES_USER: "{{ postgres_usr }}" # Variable d'environnement pour l'utilisateur PostgreSQL
      POSTGRES_PASSWORD: "{{ postgres_pwd }}" # Variable d'environnement pour le mot de passe PostgreSQL
      POSTGRES_HOSTNAME: "{{ postgres_container }}" # Variable d'environnement pour l'hôte PostgreSQL
      POSTGRES_PORT: "{{ postgres_port }}" # Variable d'environnement pour le port PostgreSQL
    networks: # Spécifie le réseau auquel le conteneur doit être connecté
      - name: app-network # Nom du réseau auquel connecter le conteneur
    pull: true # Indique que l'image Docker doit être tirée depuis le registre Docker avant de démarrer le conteneur
  become: true # Indique que cette tâche doit être exécutée avec les privilèges de superutilisateur
  vars: # Définit les variables pour cette tâche
    ansible_python_interpreter: /usr/bin/python3 # Spécifie l'interpréteur Python à utiliser pour l'exécution de cette tâche
```

```yml
---
- name: Launch Database
  docker_container: # Utilise le module docker_container pour gérer les conteneurs Docker
    name: "{{ postgres_container }}" # Spécifie le nom du conteneur à gérer, défini par la variable postgres_container
    image: cheetaheim/tp-devops-postgres:latest # Spécifie l'image Docker à utiliser pour le conteneur
    state: started # Indique que le conteneur doit être démarré
    env: # Définit les variables d'environnement à passer au conteneur
      POSTGRES_DB: "{{ postgres_db }}" # Variable d'environnement pour le nom de la base de données PostgreSQL
      POSTGRES_USER: "{{ postgres_usr }}" # Variable d'environnement pour le nom d'utilisateur PostgreSQL
      POSTGRES_PASSWORD: "{{ postgres_pwd }}" # Variable d'environnement pour le mot de passe PostgreSQL
    networks: # Spécifie le réseau auquel le conteneur doit être connecté
      - name: app-network # Nom du réseau auquel connecter le conteneur
    pull: true # Indique que l'image Docker doit être tirée depuis le registre Docker avant de démarrer le conteneur
  become: true # Indique que cette tâche doit être exécutée avec les privilèges de superutilisateur
  vars: # Définit les variables pour cette tâche
    ansible_python_interpreter: /usr/bin/python3 # Spécifie l'interpréteur Python à utiliser pour l'exécution de cette tâche
```

```yml
---
- name: Run Front-End
  docker_container: # Utilise le module docker_container pour gérer les conteneurs Docker
    name: devops-front # Spécifie le nom du conteneur à gérer
    image: cheetaheim/tp-devops-frontend:latest # Spécifie l'image Docker à utiliser pour le conteneur
    state: started # Indique que le conteneur doit être démarré
    ports: # Définit les ports à mapper pour le conteneur
      - "80:80" # Mappe le port 80 du conteneur au port 80 de l'hôte
    networks: # Spécifie le réseau auquel le conteneur doit être connecté
      - name: app-network # Nom du réseau auquel connecter le conteneur
    pull: true # Indique que l'image Docker doit être tirée depuis le registre Docker avant de démarrer le conteneur
  become: true # Indique que cette tâche doit être exécutée avec les privilèges de superutilisateur
  vars: # Définit les variables pour cette tâche
    ansible_python_interpreter: /usr/bin/python3 # Spécifie l'interpréteur Python à utiliser pour l'exécution de cette tâche
```

```yml
---
- name: Run Proxy
  docker_container: # Utilise le module docker_container pour gérer les conteneurs Docker
    name: httpd-container # Spécifie le nom du conteneur à gérer
    image: cheetaheim/tp-devops-httpd:latest # Spécifie l'image Docker à utiliser pour le conteneur
    ports: # Définit les ports à mapper pour le conteneur
      - "8080:80" # Mappe le port 80 du conteneur au port 8080 de l'hôte
    state: started # Indique que le conteneur doit être démarré
    networks: # Spécifie le réseau auquel le conteneur doit être connecté
      - name: app-network # Nom du réseau auquel connecter le conteneur
    pull: true # Indique que l'image Docker doit être tirée depuis le registre Docker avant de démarrer le conteneur
  become: true # Indique que cette tâche doit être exécutée avec les privilèges de superutilisateur
  vars: # Définit les variables pour cette tâche
    ansible_python_interpreter: /usr/bin/python3 # Spécifie l'interpréteur Python à utiliser pour l'exécution de cette tâche
```
