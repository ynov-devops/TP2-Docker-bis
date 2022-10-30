# Conteneurisation

## Prérequis

Ce TP nécessite d'avoir Docker d'installé dans un environnement Linux/Unix. Si vous utilisez Windows, assurez-vous d'avoir WSL2 d'installé et configuré pour fonctionner avec Docker Desktop.

Pour la suite de ce TP, vous utiliserez exclusivement votre environnement Linux/Unix préalablement configuré. **Veillez à ne pas utiliser votre environnement Windows ainsi que Powershell**

## Présentation de l'API

L'API permet de créer et de lister des utilisateurs. L'API nécessite une base de données MongoDB dans laquelle seront stockées les données des utilisateurs.

## Démarrage de la base données.

Docker (et les conteneurs de manière générale) permet de mettre en place un environnement de développement très simplement en quelques lignes de commande.

La commande suivante permet de démarrer un conteneur à partir de l'image `mongodb:latest` cette image est stockée sur le DockerHub, un registry public sur lequel on peut retrouver une grande quantité d'images maintenues par la communauté.

Voici la [documentation](https://hub.docker.com/_/mongo) associée à l'image `mongodb:latest`.

```bash
docker run -e MONGO_INITDB_ROOT_USERNAME="root" -e MONGO_INITDB_ROOT_PASSWORD="toto" --hostname mongo --name my-mongodb-container -p 27018:27017 -d mongo:latest
```

Décortiquons la commande précédente :
- `docker run` est une commande Docker qui permet de démarrer un conteneur à partir d'une image.
- Le flag `-e <VARIABLE_NAME>:<VALUE>` permet d'ajouter une variable d'environnement au moment où le conteneur démarre. En Linux, une variable d'environnement est une variable accessible par n'importe quel processus du système. Dans le cadre de l'image `mongo` les variables d'environnement `MONGO_INITDB_ROOT_USERNAME` et `MONGO_INITDB_ROOT_PASSWORD` permettent de définir l’utilisateur `root` (celui qui a tous les droits) ainsi que son mot de passe comme définit dans la [documentation](https://hub.docker.com/_/mongo) de l'image.
- Le flag `--hostname mongo` permet de définir un alias avec comme valeur `mongo` permettant de résoudre l'adresse IP du conteneur dans le réseau auquel il a été associé.
- Le flag `--name mongo` permet de donner un nom au conteneur qui pourra être réutiliser lorsque vous ferez référence à ce conteneur.
- Le flag `-p <HOST_PORT>:<CONTAINER_PORT>` permet de forward le traffic du port de l'hôte vers le port du conteneur. Cela permet d'accéder au port `27017` du conteneur à partir du port `27018` de l'hôte. Par défaut mongodb démarre son service sur le port `27017`.
- `-d` permet de faire tourner le conteneur en tâche de fond.
- Le dernier paramètre `mongo:latest` correspond au nom de l'image depuis laquelle sera créé le conteneur.

Une fois exécuté, la commande va réaliser 2 actions :
- `pull` l'image `mongo:latest` si celle-ci n'est pas présent en local.
- Démarrer le conteneur à partir de l'image précédemment télécharger.

Si tout s'est bien passé, un conteneur avec le nom `my-mongodb-container` s'est créé avec l'ensemble des paramètres qui ont été spécifiés.

Vous pouvez le vérifier avec la commande `docker ps` qui liste l'ensemble des conteneurs actifs sur l'hôte.

Si le conteneur n'est pas présent, il a du y avoir une erreur au démarrage et le conteneur s'est stoppé. La commande `docker ps -a` permet de lister l'ensemble des conteneurs, même ceux qui sont stoppé.

Afin d'avoir plus d'information sur un conteneur pour connaître la raison pour laquelle il ne démarre pas, il peut être intéressant d'inspecter les logs du conteneur avec la commande `docker logs <CONTAINER_NAME>`.

Vérifier que votre nouvelle base de données est accessible à partir du partir du port `27018`de l'hôte. Vous pouvez pour cela utiliser un client MongoDB comme [MongoDB Compass](https://www.mongodb.com/try/download/shell) (il faut le télécharger).

Une fois MongoDB Compass démarré, cliquez sur "New connection" et utilisez `mongodb://root:toto@localhost:27018` comme "Connection string". `localhost` est un alias qui fait référence à `127.0.0.1` l'adresse IP de votre hôte. On utilise le port `27018` car il s'agit du port de l'hôte qui permet d'accéder au port `27017` du conteneur. 

### Démarrage de l'API

Maintenant que la base de données est fonctionnelle et accessible, l'API peut être démarrée.

Avant de démarrer l'API, assurez d'avoir bien installé `node` et `yarn`: 

```bash
# Installe NVM
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash

# Installe la dernière version de node
nvm use node
Ò
# Installe Yarn
npm install --global yarn
```

Pour démarrer l'API, il faut au préalable installer et ses dépendances et la builder:

```bash
# Se rendre le dossier "api" contenant le code source de l'application.
cd api

# Installe les dépendances de l'API dans le dossier "node_modules"
yarn install

# Génère le build de l'application dans le dossier "dist"
yarn build

# Définir la variable d'environnement permettant de connecter l'API à la base de données MongoDB

export MONGODB_URI="mongodb://root:toto@localhost:27018"

# Démarre l'application
node dist/main.js
```

L'API doit démarrée sans erreurs.

Vous pouvez le vérifier en essayant d'ajouter des utilisateurs avec la commande suivante :
```bash
curl --location --request POST 'http://localhost:3000/users' \
--header 'Content-Type: application/json' \
--data-raw '{"email":"alexis.bel@ynov.com", "firstName": "Alexis", "lastName": "Bel"}'
```

Vous pouvez lister les utilisateurs avec la commande suivante : 
```bash
curl 'http://localhost:3000/users'
```

Une fois les utilisateurs créé, assurez vous qu'ils ont bien été insérés dans MongoDB en utilisant MongoDB Compass.

## Conteneurisation de l'API

Dans cette partie, vous allez conteneuriser l'API, ce qui permettra de la déployer dans n'importe quel environnement.

### Création de l'image

#### Création du Dockerfile

Afin de conteneurisé l'API, il est dans un premier temps nécessaire de créer son image. On utilise pour cela un `Dockerfile`. Un `Dockerfile` est un fichier de configuration qui va décrire les différentes étapes (appelées `layers`) permettant de créer une image. Vous trouverez plus d'information sur le Dockerfile et sa configuration sur la [documentation de Docker](https://docs.docker.com/engine/reference/builder/). 

Voici une liste des directives les plus utilisés dans un Dockerfile:

- La première étape d'un Dockerfile consiste à définir son image de base avec la directive `FROM <IMAGE>`. Par exemple la directive `FROM ubuntu:22.04` permet de partir de l'image `ubuntu:22.04` du [DockerHub](https://hub.docker.com/_/ubuntu). Cela permet d'avoir les outils déjà préinstaller pour les futurs étapes du Dockerfile.

- `WORKDIR <DIRECTORY>` permet de définir le répertoire courant à utiliser dans l'image. Cela signifie que toutes les futures layers seront exécutées à partir du répertoire spécifié.

- `COPY <SRC> <DESTINATION>` permet de copier un fichier ou un répertoire de l'hôte directement dans le conteneur. Le chemin du fichier/répertoire source est relatif à l'emplacement du `Dockerfile`.

- `RUN <COMMAND>` permet d'exécuter une commande et créera une nouvelle couche à la fin de son exécution. Cela est utile afin d'installer des composants nécessaires au démarrage du conteneur.

- `CMD <COMMAND>` permet de définir la commande qui sera exécutée au démarrage du conteneur. Il s'agit généralement de la dernière instruction dans un `Dockerfile`. Cette commande doit lancer le processus permettant de démarrer le service de l'application conteneurisée, par exemple `node dist/main.js` dans le cadre de notre API.

Il est important à ne pas confondre son utilisation avec `RUN` qui sert uniquement à exécuter des étapes nécessaires au processus lancé par `CMD`.

> Créer une fichier `Dockerfile` dans le dossier `api` et ajoutez y le contenu suivant:

```Dockerfile
# Image de build
FROM node:alpine 

# Défine le répertoire par défault
WORKDIR /app

# Copie le contenu du répertoire courant (celui où se situe le Dockerfile) dans l'image. En Linux/Unix le symbole "." fait référence au répertoire courant.
COPY . .

# On build l'app --> cela créé le dossier dist
RUN yarn build

CMD node /app/dist/main.js
```

> Il manque une étape dans le Dockerfile afin de le rendre fonctionnel. **Ajoutez y cette étape**.


#### Contexte de build

Il est possible d'exclure certains fichiers/répertoire en créant un fichier `.dockerignore` dans le même répertoire que le Dockerfile, contenant la liste des fichiers/répertoires à ignorer lors de la construction de l'image. Créer le fichier `.dockerignore` et ajoutez-y le contenu suivant :

```
dist
node_modules
Dockerfile
```

Cela va permettre d'exclure ces fichiers/répertoires du contexte de build de l'image.
Si on n'ajoute pas ce fichier et que l'on prend le `Dockerfile` précédent, il y a un risque que les dossier `dist` et `node_modules` soient copiés dans l'image avec la directive `COPY . .`

#### Build de l'image

L'image peut ensuite être construite à partir du `Dockerfile` avec la commande :

```
docker build -t <TAG> . # Le "." est important et correspond au contexte de build.
```

Avant de construire l'image à partir du `Dockerfile`, Docker va charger l'ensemble des fichiers et répertoires se trouvant dans le dossier spécifié dans la commande `docker build`, le répertoire courant dans la commande suivante (`.` correspond au répertoire courant dans les systèmes Linux/Unix), il s'agit du "contexte de build".

Le flag `-t` permet de donner un nom à l'image (un tag) en suivant la forme `<REGISTRY>/<IMAGE>:<TAG>`:
- Le `REGISTRY` est optionnel et permet de pouvoir partager l'image sur un autre registry que celui par défaut (le DockerHub).
- L'`IMAGE` est le nom principale de l'image.
- Le `TAG` (ou version) est utilisé pour donner une version spécifique à l'image. Si le tag n'est pas spécifié, alors il a une valeur par défaut à `latest`.

Voici quelques exemples de tag d'image :

- `my-api`
- `my-api:1.0.0`
- `my-api:dev`
- `another-registry/my-api:1.1.0`

> Construire l'image de l'api à partir du `Dockerfile` précédent en utilisant la commande `docker build` en spécifiant un tag à l'image.

Vous allez voir les différentes étapes du Dockerfile s'exécuter avant que l'image soit créée.

Une fois terminé, lancer la commande `docker image ls` qui permet de lister la liste des images. Vous devriez voir votre image

### Création d'un conteneur à partir de l'image.

Un conteneur est lancé à partir d'une image. Il s'agit d'une instance de l'image qui va exécuter le processus spécifié dans la directive `CMD` du `Dockerfile` ayant permis la construction de l'image. **Le conteneur s'arrête de fonctionner dès lors que le processus se termine**. 

Il est donc important de spécifier une instruction `bloquante` afin de s'assurer que le conteneur ne s'arrête pas de fonctionner. C'est d'autant plus nécessaire dans le cadre d'un service web telle qu'une API qui attend la réception de requêtes. 

La commande `docker run` permet de créer un conteneur à partir d'une image. 

Lancez la commande `docker run <VOTRE_IMAGE>`. Cela va lancer un conteneur à partir de votre image.

Cependant l'API ne va pas démarrer (et c'est normal) à cause de l'erreur suivante :

```bash
ERROR [ExceptionHandler] The `uri` parameter to `openUri()` must be a string, got "undefined". Make sure the first parameter to `mongoose.connect()` or `mongoose.createConnection()` is a string.
```

Rappelez vous qu'il est nécessaire de configurer la variable d'environnement `MONGODB_URI` afin que l'API puisse se connecter à la base de données.

Une variable d'environnement définit au niveau de l'hôte n'est pas accessible par le conteneur. Il faut donc définir cette variable **avant** que le conteneur ne démarre en utilisant le flag `-e <VARIABLE>=<VALUE>`.

Essayez de relancer la `docker run` en spécifiant la variable d'environnement : 

```bash
docker run -e MONGODB_URI="mongodb://root:toto@localhost:27018" <VOTRE_IMAGE> # Il est important de spécifier tous les flags avant le nom de l'image, sinon la commande ne fonctionnera pas.
```

L'API ne démarre toujours pas, et c'est normal. Cette fois ci, le problème vient de la valeur de la variable d'environnement. La valeur spécifiée ne permet pas à l'API de se connecter à la base de données MongoDB.

## Faire communiquer les conteneurs entre eux

Pour en comprendre la raison de l'erreur précédente, il faut comprendre la manière dont fonctionne le réseau.

Nous allons décortiquer la valeur de la connection string spécifiée `mongodb://root:toto@localhost:27018` :

- `mongodb` : il s'agit du protocole de communication entre un client et un serveur MongoDB. Cette valeur ne change pas dans notre cas.
- `root:password` : correspond au `<USER>:<PASSWORD>` permettant de se connecter à la base de données. Cette valeur ne change que lorsque que l'on souhaite se connecter avec un autre utilisateur, ou lorsque que le mot de passe a changé.
- `@` indique que tout ce qui précède ce caractère correspond à l'adresse du serveur.
- `localhost` : Cette partie est importante car elle permet de donner l'adresse du serveur auquel se connecter. Cela peut être une **adresse IP** ou un **nom de domaine**. `localhost` est un alias correspondant à l'adresse IP `127.0.0.1` qui est une adresse `loopback`. En réseau une adresse `loopback` est l'adresse locale du système. Dans le contexte de l'hôte, `localhost` correspond à l'adresse locale de l'hôte. Cependant dans le cadre du conteneur, il s'agit de l'adresse locale du conteneur.
- `27018` : Le numéro de port sur lequel le serveur MongoDB doit recevoir les requêtes. `27018` correspond à la valeur du port de l'hôte depuis lequel il renvoie les requêtes vers le port du conteneur `27017` (Voir la première partie du TP quand le conteneur MongoDB est démarré).

Si l'on devait comparer un serveur avec un immeuble, l'adresse postale de l'immeuble serait l'adresse IP du serveur, les appartements seraient les processus accessibles du serveur, et les boîtes aux lettres seraient les ports (ouverts) du serveur. 

Pour rester dans la métaphore, un conteneur serait une petite maison avec une seule boîte aux lettres, car de manière générale, un conteneur n'expose qu'un seul processus à travers un seul port. Il y a bien entendu des cas d'utilisation où un conteneur n'expose aucun processus en dehors de celui-ci et d'autres cas (assez spécifiques) ou plusieurs processus peuvent être exposés.

Pour revenir à l'erreur de connection à MongoDB, elle s'explique par le fait que le `localhost:27018` correspond à l'adresse locale du conteneur (de l'API) sur le port 27018. Hors, il n'y a aucun processus `mongodb` qui fonctionne sur le conteneur de l'API.

Il faut trouver un moyen de se connecter au conteneur MongoDB précédemment déployé, depuis le conteneur (et non depuis l'hôte comme c'était le cas au début).

La manière la plus simple d'y parvenir est de créer un réseau privé dans lequel on démarrera le conteneur MongoDB ainsi que le conteneur de l'API.

#### Création d'un réseau privé avec Docker.

Docker permet de créer des réseau afin de connecter les conteneurs entre eux. Dans ce TP nous allons nous contenter du driver `bridge`, il s'agit du driver par défaut.

Créer un réseau `my-private-network`:

```bash
docker network create my-private-network
```

#### Connecter les conteneurs au réseau.

Maintenant que le réseau `my-private-network` est créé, nous allons recréer les conteneurs dans ce réseau.

Le plus simple est de supprimer les conteneurs existants. Lister les conteneurs existants avec la commande `docker ps -a` et supprimer les conteneurs MonogDB et de l'API (s'ils existent) en utilisant la commande `docker rm -f <CONTAINER_ID>`.

Créer le conteneur MongoDB en spécifiant le nouveau réseau :

```
docker run -e MONGO_INITDB_ROOT_USERNAME="root" -e MONGO_INITDB_ROOT_PASSWORD="toto" --hostname mongo --network my-private-network  --name my-mongodb-container -p 27018:27017 -d mongo:latest
```
Avec Docker, quand plusieurs conteneurs sont connectés sur le même réseau, il est possible d'accéder à l'adresse IP du conteneur à partir de son `hostname`, comme spécifié dans la commande.

> Créer le conteneur de l'API dans ce nouveau réseau en vous inspirant de la commande précédente. Pensez à utiliser le hostname `mongo` (spécifié dans la commande précédente) afin d'identifier l'adresse IP du container MongoDB. Pensez également à utiliser le bon port. Cette fois-ci vous accéder au conteneur MongoDB sans passer par l'hôte (comme c'était le cas au début du TP).

Cette fois-ci, le conteneur devrait démarrer sans erreur si toutes les étapes ont été respectées.

#### Exposer l'API depuis l'hôte

L'API n'est toujours pas accessible depuis l'hôte. Supprimez le conteneur de l'API et le recréer avec la bonne configuration afin de l'exposer sur le port `8080` de l'hôte. Inspirez vous pour cela de la manière dont le conteneur mongo a été configuré.

Assurez que vous pouvez communiquer avec l'API conteneurisée depuis l'hôte en utilisant les commandes suivantes : 

Vous pouvez le vérifier en essayant d'ajouter des utilisateurs avec la commande suivante :

```bash
# Ajouter un utilisateur
curl --location --request POST 'http://localhost:8080/users' \
--header 'Content-Type: application/json' \
--data-raw '{"email":"alexis.bel@ynov.com", "firstName": "Alexis", "lastName": "Bel"}'

# Lister les utilisateurs 
curl 'http://localhost:8080/users'
```

## Persister les données de la base de données

Supprimez les 2 conteneurs que vous avez créé précédemment et recréez (avec la même configuration qu'avant).

Lancez la commande requêtant l'API afin d'afficher la liste des utilisateurs : 

```bash
# Lister les utilisateurs 
curl 'http://localhost:8080/users'
```

La liste est vide. Vous pouvez également le constater en vous connectant sur la base de données MongoDB en utilisant le client Compass.

Un conteneur est éphémère, ce qui signifie qu'une fois supprimé, toutes les fichiers qui ont été créé dans le conteneur sont également perdus.

Lorsque l'on insert des données dans MongoDB, celui-ci créé et met à jour des fichiers dans lesquels sont stockées ces données.

Afin de conserver les données écrites dans le conteneur, il est nécessaire de les stocker dans un autre endroit.


Docker permet de créer des volumes afin de persister des données. Il existe plusieurs type de driver permettant de stocker les données à différent endroits. Par défaut un volume utilise le driver `local` qui stocke les données l'hôte du conteneur.

Créer un volume `mongodb-data` afin de stocker les données :

```bash
docker volume create mongodb-data
```

Recréez ensuite le conteneur MongoDB en ajoutant le flag `-v <VOLUME_NAME>:<CONTAINER_FOLDER_NAME>`. 

Par défaut, MongoDB stocke ces données dans le dossier `/data/db`, c'est donc ce dossier là qu'il faut associer au volume.

La commande finale permettant de lancer le conteneur MongoDB est celle là :

```bash
docker run -e MONGO_INITDB_ROOT_USERNAME="root" -e MONGO_INITDB_ROOT_PASSWORD="toto" --hostname mongo --network my-private-network  --name my-mongodb-container -p 27018:27017 -v mongodb-data:/data/db -d --rm mongo:latest
```

Maintenant le dossier `/data/db` est persisté dans le volume `mongodb-data`. Vous pouvez le vérifier avec la commande `docker inspect my-mongodb-container`. Vous pouvez tester de supprimer le conteneur et de le recréer, les données seront toujours présentes.

## Infrastructure as Code avec Docker Compose

Depuis le début du TP, la configuration des conteneurs se fait en ligne de commande, et nous répétons les mêmes actions à quelques paramètres près. 

Il serait intéressant de pouvoir définir toutes notre configuration sous forme de code et de pouvoir démarrer et stopper notre environnement en une seule commande. C'est ce qu'on appelle plus généralement **l'Infrastructure as Code (IaC)**.

Cela apporte plusieurs avantages :

- Efficacité : L'environnement peut être démarré et stoppé en une seule commande.
- Partage : Il est beaucoup plus facile de partager la configuration de l'environnement. Par exemple la configuration peut être partagé entre plusieurs développeur afin de faciliter la création d'un environnement de développement.
- Auditabilité : Comme l'environnement est décrit dans un ou plusieurs fichiers, il est facilement lisible et auditable.

Il est possible d'utiliser Docker Compose afin de définir la configuration Docker dans un fichier. Docker Compose utilise un fichier de configuration au format `YAML` dans lequel on peut définir toute notre configuration Docker.

Créer un fichier `docker-compose.yaml` dans le dossier racine de ce TP (en dehors du dossier `api`) et ajoutez-y le contenu suivant :

```yaml
version: '3.1'

services:

  # Conteneur MongoDB
  mongo:
    image: mongo # Correspond à l'image mongo du DockerHub : https://hub.docker.com/_/mongo
    volumes:
      - mongodb-data:/data/db # Les données des bases de données de MongoDB sont stockées dans le répertoire /data/db. Il faut donc monter un volume sur cet emplacement. 
    ports:
      - 27018:27017 # On forward le traffic du port 27018 de l'hôte vers le port 27017 du conteneur qui correspond au port de MongoDB. Cela permettra de s'y connecter avec un client lourd comme Compass.
    environment: # Variables d'environnement permettant l'initialisation du user de base. Les valeurs sont récupérées dans le fichier .env
      MONGO_INITDB_ROOT_USERNAME: ${MONGODB_USERNAME} 
      MONGO_INITDB_ROOT_PASSWORD: ${MONGODB_PASSWORD}
  
  api: # API NodeJS
    image: myapi:dev # Image qui a été précédemment construite
    ports:
      - 8080:3000 # On forward le traffic du port 8080 de l'hôte vers le port 3000 du conteneur qui correspond au port de l'API afin de la rendre accessible depuis l'host.
    environment:
      MONGODB_URI: "mongodb://${MONGODB_USERNAME}:${MONGODB_PASSWORD}@mongo:27017"  # On définit la variable d'environnement permettant de se connecter à MongoDB 

volumes:
  mongodb-data:
    external: true # Permet de réutiliser le volume précédemment créé, sinon un nouveau volume sera créé et les données seront perdues.
```

Créez également un fichier `.env` au même niveau que le fichier `docker-compose.yaml` avec le contenu suivant :

```bash
MONGODB_USERNAME="root"
MONGODB_PASSWORD="toto"
```

Docker compose va ensuite injecter la valeurs des variables définies dans ce fichier dans la configuration Docker compose, si ces variables sont utilisées.

L’intérêt de cette pratique est double :
- Elle permet de dissocier la configuration fixe de celle qui varie. Cela facilite la réutilisation de la configuration fixe dans des environnement différents.
- Le secrets (comme le mot de passe de MongoDB) peuvent être géré indépendamment. On peut par exemple envoyer sur le Git le fichier `docker-compose.yaml` mais garder en local le fichier `.env`

L'environnement est prêt à être déployé. Supprimer les anciens conteneurs en utilisant la commande `docker rm`.

Lancez la commande `docker-compose up` afin de déployer l'environnement Docker (la commande doit être exécutée dans le répertoire où se situe le fichier `docker-compose.yaml`).

Si tout se passe bien, les 2 conteneurs devraient se lancer. Vous pouvez le vérifier en listant les conteneurs. Assurez vous que l'API fonctionne toujours.

Essayez de comprendre la manière dont fonctionne Docker Compose :

- Lister et inspectez les conteneurs avec la commande `docker inspect` 
- Lister et inspectez les réseaux avec la commande `docker network ls` et `docker network inspect`
- Lister et inspectez les volumes avec la commande `docker volume ls` et `docker network volume`

Docker compose créé automatiquement un `network` en y ajoutant l'ensemble des services définis dans le `docker-compose.yaml` avec un alias équivalent au nom du service (`mongo` et `api`) pour chacun d'eux. Cela permet aux conteneur de communiquer entre eux.


## Loadbalancer le trafique de l'api vers plusieurs conteneurs (Avancé)

Dans l'environnement actuel, il n'y a qu'un seul réplica de l'API. C'est suffisant dans un environnement de développement. Cependant, dans un environnement de production il serait nécessaire d'avoir plusieurs réplicas de l'API afin :

- D'avoir de la Haute Disponibilité (HA) : dans le cas où un des conteneurs n'est plus disponible, une autre peut prendre le relais.
- D'adapter la charge : Ajouter ou supprimer le nombre de réplicas afin de s'adapter au trafique.

Pour se faire, la technique consiste à ajouter un autre service en frontend, c'est à dire celui qui est visible par l'utilisateur et qui s'occupe de répartir les requêtes qu'il reçoit entres les différents réplicas.

`NGINX` est un outil opensource qui permet de faire de load balancing.

Dans cette partie, vous allez modifier l'architecture de l'application pour quelle ressemble à celle-ci : 

![LB Architecture](./img/lb-archi.svg "LB Architecture")

### Ajouter des réplicas de l'API.

Modifier le `docker-compose.yaml` pour ajouter 2 autres réplicas de l'API. Il y aura en tout 3 réplicas : `api-1`, `api-2` et `api-3`. 

Pensez à changer les numéro de port de l'hôte qui redirige vers les ports des conteneurs de l'API. **Un même port ne pas être utilisé pour renvoyé vers des conteneurs différents !**. Utilisez par exemple les ports `8081`, `8082` et `8083` afin d'éviter tout conflit.

S'assurer que les 3 réplicas sont fonctionnels et essayant de les requêter un par un. 

### Ajouter un service NGINX.

`NGINX` va service de load balancer entre le client (c'est à dire nous) et les 3 réplicas de l'API.

Modifier le `docker-compose.yaml` pour y ajouter un service `nginx`. Utilisez pour cela l'image du DockerHub `nginx`. Par défaut NGINX fonctionne sur le port 80 (dans le conteneur). Pensez à bien forward un des ports disponible de l'hôte (le `8080` devrait être disponible si vous avez bien suivi l'étape précédente) vers le port `80` du conteneur NGINX.

Assurez vous d'accéder à NGINX avec la commande `curl localhost:8080` ou en ouvrant un navigateur à l'adresse `localhost:8080`.

NGINX est normalement fonctionnelle.

### Configurer NGINX pour effectuer le load balancing entre les réplicas de l'API.

Créer un fichier `nginx.conf` avec la contenu suivant : 
```bash
events {}
http {
    log_format apm '"$time_local" client=$remote_addr '
               'method=$request_method request="$request" '
               'request_length=$request_length '
               'status=$status bytes_sent=$bytes_sent '
               'body_bytes_sent=$body_bytes_sent '
               'referer=$http_referer '
               'user_agent="$http_user_agent" '
               'upstream_addr=$upstream_addr '
               'upstream_status=$upstream_status '
               'request_time=$request_time '
               'upstream_response_time=$upstream_response_time '
               'upstream_connect_time=$upstream_connect_time '
               'upstream_header_time=$upstream_header_time';
    upstream api {
        server api-1:3000 weight=1;
        server api-2:3000 weight=1;
        server api-3:3000 weight=1;
    }

    server {
        listen 80;
        access_log /dev/stdout apm;

        location / {
            proxy_pass http://api;
            proxy_next_upstream     error timeout invalid_header http_500;
            proxy_connect_timeout   2;
        }
    }
}
```

Ce fichier modifie la configuration de base d'NGINX en ajoutant notamment la capacité de load balancer. Assurez vous que le nom des réplicas de l'API dans le `docker-compose.yaml` correspondent bien au nom des `server` dans la partie `upstream api` de la nouvelle configuration NGINX.

> Trouver une manière de remplacer la configuration de base d'NGINX par cette nouvelle configuration. Il y a plusieurs méthode possible, mais la plus simple consiste à utiliser un volume.

Une fois la configuration mise en place. Requêtez l'API non plus en passant par les ports `8081`,`8082` et `8083` mais directement par NGINX avec le port `8080` comme expliqué sur le schéma d'architecture.

Vous pouvez directement requêter la route `/` avec `curl localhost:8080`. Celle-ci affiche un `Hello World` avec l'id du conteneur. Cela vous permet de valider que le traffique est bien load balancer entre les différentes API. Vous pouvez également le constater en allant vérifier les logs des conteneurs de l'API directement depuis Docker Compose ou avec la commande `docker logs <CONTAINER_NAME>`.

Essayez de supprimer un des conteneurs de l'API et de continuer d'envoyer des requêtes. Que remarquez vous ?














