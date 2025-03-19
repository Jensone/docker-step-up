# Guide complet : Pipeline CI/CD Symfony 7 avec Docker Compose sur Ubuntu 24.04

Ce guide détaillé explique comment mettre en place un pipeline **CI/CD** complet pour déployer une application **Symfony 7** sur un **VPS Ubuntu 24.04 (Noble)** en utilisant **Git**, **Docker**, **Docker Compose**, **DockerHub** comme registre d’images, ainsi que **Portainer** et **Nginx Proxy Manager** pour la gestion des conteneurs et du proxy inverse. Nous couvrirons toutes les étapes, de la configuration initiale du serveur jusqu’aux bonnes pratiques de maintenance du pipeline, avec des explications claires, des commandes pas-à-pas et des illustrations lorsque possible. Ce guide s’adresse à un développeur souhaitant automatiser efficacement le déploiement de son application Symfony 7.

## 1. Configuration du VPS Ubuntu

Pour commencer, nous allons configurer le serveur Ubuntu en installant les dépendances nécessaires et en appliquant quelques mesures de sécurité de base.

### Installation des dépendances (Git, Docker, Docker Compose, Portainer, Nginx Proxy Manager)

**Mettre à jour le système et installer Git :**  
Connectez-vous en SSH à votre VPS (idéalement avec un compte utilisateur non-root disposant de `sudo`) et mettez à jour la liste des paquets, puis installez Git :  
```bash
sudo apt update && sudo apt upgrade -y  
sudo apt install git -y
``` 
Git servira à récupérer votre code depuis le dépôt.

**Installer Docker Engine :**  
Ubuntu 24.04 ne fournit pas Docker par défaut, il faut donc ajouter le dépôt officiel Docker. Exécutez les commandes suivantes pour installer les prérequis, ajouter la clé GPG Docker, le dépôt apt, puis installer Docker CE et ses composants ([CrownCloud Wiki - How To Install Docker Portainer On Ubuntu 24 04](https://wiki.crowncloud.net/?How_to_Install_Docker_Portainer_on_Ubuntu_24_04#:~:text=apt%20install%20apt,y)) ([CrownCloud Wiki - How To Install Docker Portainer On Ubuntu 24 04](https://wiki.crowncloud.net/?How_to_Install_Docker_Portainer_on_Ubuntu_24_04#:~:text=Install%20Docker%3A)) :  
```bash
# Installer les paquets requis pour utiliser un dépôt HTTPS
sudo apt install apt-transport-https ca-certificates curl software-properties-common gnupg -y

# Ajouter la clé GPG officielle de Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Ajouter le dépôt Docker stable à APT
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Mettre à jour les sources et installer Docker
sudo apt update 
sudo apt install docker-ce docker-ce-cli containerd.io -y
```  
Une fois l’installation terminée, démarrez le démon Docker et activez-le pour qu’il se lance au boot ([CrownCloud Wiki - How To Install Docker Portainer On Ubuntu 24 04](https://wiki.crowncloud.net/?How_to_Install_Docker_Portainer_on_Ubuntu_24_04#:~:text=Start%20and%20enable%20Docker%3A)) :  
```bash
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker   # pour vérifier que Docker tourne correctement
```  
Vérifiez que Docker fonctionne en lançant `sudo docker run hello-world` (ce qui télécharge et exécute un conteneur de test).  

**Installer Docker Compose :**  
Docker Compose est souvent inclus sous forme de plugin (compose v2) avec Docker CE. Vérifiez la version avec `docker compose version`. Si ce n’est pas le cas, installez-le :  
```bash
sudo apt install docker-compose-plugin -y   # Installe le plugin Docker Compose v2
```  
*Remarque:* Sous Docker v2, on utilise la commande `docker compose ...` (espace) au lieu de `docker-compose ...` (tiré). Les exemples de ce guide utiliseront la nouvelle syntaxe.

**Installer Portainer :**  
Portainer est une interface web de gestion de Docker. On le déploie lui-même en tant que conteneur Docker. Tout d’abord, créez un volume dédié pour Portainer, puis lancez le conteneur ([CrownCloud Wiki - How To Install Docker Portainer On Ubuntu 24 04](https://wiki.crowncloud.net/?How_to_Install_Docker_Portainer_on_Ubuntu_24_04#:~:text=Once%20Docker%20is%20installed%20and,a%20Docker%20volume%20for%20Portainer)) :  
```bash
sudo docker volume create portainer_data
sudo docker run -d -p 8000:8000 -p 9443:9443 \
  --name=portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```  
Cette commande télécharge l’image Portainer Community Edition et l’exécute en arrière-plan (`-d`). Les ports **8000** (Edge agent), **9443** (interface web HTTPS) et **8000** sont publiés. Une fois le conteneur démarré, accédez à l’interface Portainer en HTTPS sur le port 9443 de votre serveur (par exemple *https://votre-ip:9443*). Au premier accès, Portainer vous invite à définir un utilisateur administrateur et un mot de passe ([CrownCloud Wiki - How To Install Docker Portainer On Ubuntu 24 04](https://wiki.crowncloud.net/?How_to_Install_Docker_Portainer_on_Ubuntu_24_04#:~:text=Open%20a%20web%20browser%20and,navigate%20to%3A%20https%3A%2F%2F%3A9443)). Créez ces identifiants pour sécuriser l’accès à Portainer.

 ([CrownCloud Wiki - How To Install Docker Portainer On Ubuntu 24 04](https://wiki.crowncloud.net/?How_to_Install_Docker_Portainer_on_Ubuntu_24_04)) *Écran d’initialisation de Portainer demandant de créer le compte administrateur.* ([CrownCloud Wiki - How To Install Docker Portainer On Ubuntu 24 04](https://wiki.crowncloud.net/?How_to_Install_Docker_Portainer_on_Ubuntu_24_04#:~:text=Open%20a%20web%20browser%20and,navigate%20to%3A%20https%3A%2F%2F%3A9443))

Ensuite, Portainer vous demandera de choisir un environnement Docker à gérer. Sélectionnez l’environnement « local » (votre Docker sur le VPS) pour le contrôler via l’UI. Vous devriez alors voir le tableau de bord Portainer et pouvoir lister vos conteneurs Docker.  

**Installer Nginx Proxy Manager (NPM) :**  
Nginx Proxy Manager est un outil web simplifié pour gérer des hôtes Nginx (virtual hosts) avec une interface et la prise en charge de Let’s Encrypt. Lui aussi se déploie comme conteneur Docker, généralement via un fichier *docker-compose.yml* car il nécessite un volume de données et, par défaut, une base de données SQLite ou MariaDB. 

Sur votre VPS, assurez-vous que les ports 80 et 443 ne sont pas déjà occupés (par exemple par Apache ou un autre Nginx). Si vous n’avez pas de serveur web installé, NPM pourra utiliser ces ports. Créez un répertoire (par exemple `/opt/nginx-proxy-manager`) et à l’intérieur, créez un fichier `docker-compose.yml` minimal pour NPM : 

```yaml
version: "3"
services:
  npm:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - "80:80"    # HTTP
      - "443:443"  # HTTPS
      - "81:81"    # Admin UI
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```  

Ce fichier définit un service `npm` utilisant l’image officielle. Il publie les ports 80 et 443 (pour le proxy public) et le port 81 (pour l’interface d’admin) ([Full Setup Instructions | Nginx Proxy Manager](https://nginxproxymanager.com/setup/#:~:text=services%3A%20app%3A%20image%3A%20%27jc21%2Fnginx,Admin%20Web%20Port)) ([Full Setup Instructions | Nginx Proxy Manager](https://nginxproxymanager.com/setup/#:~:text=volumes%3A%20)). Les volumes mappent le dossier de données (configurations, certificats SQLite) et le dossier Let’s Encrypt pour persister les certificats. Enregistrez le fichier, puis lancez Nginx Proxy Manager :  
```bash
sudo docker compose up -d
```  
La première exécution peut prendre quelques instants le temps de télécharger l’image. Une fois démarré, accédez à l’UI web de NPM sur le port 81 (http://votre-ip:81). Utilisez les identifiants par défaut (**email:** *admin@example.com* / **mot de passe:** *changeme*) pour vous connecter, puis créez un utilisateur admin personnalisé et un mot de passe fort dans les paramètres.

### Sécurisation de base du serveur (Firewall UFW, SSH)

**Configurer le firewall UFW :**  
Ubuntu fournit **UFW (Uncomplicated Firewall)** pour gérer les règles réseau. On va l’utiliser pour n’autoriser que les ports nécessaires. Tout d’abord, assurez-vous d’autoriser **SSH (port 22)** avant d’activer le firewall, pour ne pas vous bloquer vous-même :  
```bash
sudo ufw allow OpenSSH    # autorise le profil SSH (port 22)
```  
Ensuite, autorisez les ports web dont on aura besoin :  
- HTTP (**port 80**) pour le trafic web non chiffré (utile pour redirections ou ACME challenge) :  
  ```bash
  sudo ufw allow 80/tcp
  ```  
- HTTPS (**port 443**) pour le trafic web chiffré :  
  ```bash
  sudo ufw allow 443/tcp
  ```  
- Portainer (**port 9443** en TCP) si vous comptez accéder à son interface admin à distance. Vous pouvez l’ouvrir, ou mieux, le restreindre à votre IP publique pour plus de sécurité (via `ufw allow from <votre-IP> to any port 9443`).  
- L’interface admin de Nginx Proxy Manager (**port 81**) peut également être restreinte ou laissée fermée si vous y accédez via un tunnel SSH. Pour un accès web, autorisez : `sudo ufw allow 81/tcp`. (Vous pourrez aussi décider de mettre l’interface NPM derrière un VPN ou la protéger avec un accès privé.)

Une fois les règles ajoutées, activez le firewall :  
```bash
sudo ufw enable
```  
Confirmez par “y” si demandé. UFW va activer les règles au prochain boot également. Vous pouvez lister les règles actives avec `sudo ufw status`. **Exemple :** les règles devraient au minimum montrer 22, 80, 443 ouverts ([How to Set Up a Firewall with UFW on Ubuntu | DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu#:~:text=SSH%20on%20port%20,can%20also%20do%20this%20for)).  

**Sécuriser SSH :**  
Assurez-vous d’utiliser une **authentification par clé SSH** plutôt que par mot de passe pour vous connecter au serveur. Éditez le fichier `/etc/ssh/sshd_config` pour vous assurer que les paramètres suivants sont définis :  
- `PubkeyAuthentication yes`  
- `PasswordAuthentication no` (désactive le login par mot de passe)  
- `PermitRootLogin no` (empêche la connexion directe en root)

Redémarrez le service SSH `sudo systemctl restart sshd` pour appliquer. **Attention :** ne faites cela qu’après avoir configuré vos clés publiques sur le serveur pour ne pas perdre l’accès.

**Mises à jour système régulières :**  
Enfin, gardez votre système à jour pour appliquer les derniers patchs de sécurité : `sudo apt update && sudo apt upgrade -y`. Vous pouvez envisager d’installer *unattended-upgrades* pour automatiser les MAJ de sécurité.

## 2. Mise en place de l’environnement de développement local

Avant de configurer le pipeline CI/CD, il est important d’organiser un environnement de développement Docker pour votre application Symfony 7. Cela permet de développer et tester en local dans des conditions proches de la production (mêmes conteneurs Docker), et d’éviter les “écarts” entre l’environnement de dev et celui de prod.

### Docker Compose pour l’application Symfony 7

**Structure du projet Symfony avec Docker :**  
Votre projet Symfony 7 devrait contenir à sa racine un fichier `Dockerfile` (pour construire l’image de l’application) ainsi qu’un fichier `docker-compose.yml` (pour définir les services nécessaires en dev). Si vous avez créé l’application via Symfony CLI ou Composer, ajoutez ces fichiers manuellement.  

**Exemple de Dockerfile pour Symfony 7 :**  
Vous pouvez utiliser une image PHP officielle comme base. Par exemple, un Dockerfile basique pourrait ressembler à : 

```Dockerfile
# Image de base : PHP 8.2 avec Apache (mod_php)
FROM php:8.2-apache

# Installation des extensions PHP nécessaires (ex: pdo_mysql, intl, etc.)
RUN apt-get update && apt-get install -y \ 
    libicu-dev libpq-dev git unzip libzip-dev \
    && docker-php-ext-install intl opcache pdo_mysql zip

# Activer mod_rewrite pour Apache (nécessaire pour Symfony)
RUN a2enmod rewrite

# Installer Composer (si besoin de composer install pendant l'image)
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# Définir le répertoire de travail
WORKDIR /var/www/html

# Copier le code source Symfony (en excluant les fichiers ignorés grâce à .dockerignore)
COPY . .

# Installer les dépendances via Composer (en production, on pourrait utiliser --no-dev)
RUN composer install --no-dev --optimize-autoloader

# Ajuster les permissions si nécessaire, par exemple pour le cache/ et le log/
RUN chown -R www-data:www-data var

# L'image expose par défaut le port 80 via Apache
EXPOSE 80

# Pas de CMD nécessaire, l'image php:apache lance Apache par défaut
```

Cet exemple d’image utilise **Apache** comme serveur web intégré à PHP, ce qui simplifie le déploiement (Apache servira les fichiers statiques et exécutera PHP). Vous pouvez alternativement utiliser PHP-FPM avec un serveur Nginx séparé, mais cela complexifie la configuration. L’image ci-dessus installe quelques extensions courantes et copie le code Symfony. **Adaptez-la** en fonction des besoins de votre application (par exemple installer `pdo_pgsql` si PostgreSQL est utilisé, etc.). Vérifiez aussi que **Composer** est disponible (ici on l’importe depuis l’image officielle composer).  

**Fichier docker-compose.yml en développement :**  
Créez un `docker-compose.yml` pour orchestrer les services en local. En développement, on inclut généralement : 
- Un service pour l’app Symfony (qui peut être construit à partir du Dockerfile). 
- Une base de données (MySQL ou PostgreSQL selon votre projet).
- D’autres services comme **Redis** (pour le cache, les sessions ou Messenger) si votre app en a besoin.
- Eventuellement un service pour un serveur SMTP de test (ex: Mailhog), etc., en fonction des besoins.

Par exemple, un `docker-compose.yml` de développement minimal avec MySQL : 

```yaml
version: "3.9"
services:
  symfony:
    build: .                        # construit l'image depuis le Dockerfile du répertoire courant
    container_name: symfony_app_dev
    ports:
      - "8080:80"                   # expose le port 80 du conteneur sur le port 8080 local
    volumes:
      - .:/var/www/html:rw          # monte le code local dans le conteneur pour du hot-reload
    environment:
      # Variables d'environnement Symfony (base de données, environnement)
      - APP_ENV=dev
      - DATABASE_URL=mysql://symfony:password@db:3306/app_db?serverVersion=8.0
    depends_on:
      - db

  db:
    image: mysql:8.0
    container_name: symfony_db_dev
    environment:
      - MYSQL_DATABASE=app_db
      - MYSQL_USER=symfony
      - MYSQL_PASSWORD=password
      - MYSQL_ROOT_PASSWORD=rootpass
    volumes:
      - db_data:/var/lib/mysql

  # Exemple de service Redis si nécessaire
  redis:
    image: redis:7-alpine
    container_name: symfony_redis_dev

volumes:
  db_data:
```

Quelques explications : 
- Le service **symfony** est construit à partir du Dockerfile (`build: .`). On mappe le dossier courant dans le conteneur (`.:/var/www/html`) pour que toute modification de code sur votre machine soit reflétée immédiatement dans le conteneur (pratique pour développer sans reconstruire l’image à chaque fois). On expose le port 80 du conteneur sur le port **8080** de votre machine pour accéder à l’application via http://localhost:8080.  
- Le service **db** utilise l’image MySQL officielle. On fixe des variables d’env pour créer la base et l’utilisateur. Les identifiants choisis (ici *symfony/password*) sont reportés dans la variable **DATABASE_URL** du conteneur Symfony pour que l’application puisse s’y connecter. Un volume nommé `db_data` est déclaré pour persister les données MySQL entre les arrêts/relances du conteneur.  
- Le service **redis** (optionnel) utilise l’image Redis. Dans Symfony, vous pourriez configurer le cache ou la session sur `redis://redis:6379` grâce à ce service. (Ici pas de volume, les données Redis sont en mémoire ou dans le conteneur seulement, ce qui est acceptable en dev).

Lancez cet ensemble avec :  
```bash
docker compose up -d
```  
La première fois, Docker va construire l’image Symfony app (ce qui peut prendre du temps). Une fois up, vous devriez accéder à Symfony en local. En dev, Symfony dispose aussi du serveur local `bin/console server:run` ou `symfony serve`, mais ici nous utilisons Apache dans le conteneur pour rester cohérent. Assurez-vous que votre application fonctionne bien (exécutez des commandes comme `docker compose exec symfony php bin/console doctrine:migrations:migrate` si vous avez des migrations, etc.).

**Gestion des variables et du .env :**  
En dev, Symfony utilise généralement le fichier `.env` à la racine. Dans le conteneur, vous pouvez override ces variables via `docker-compose.yml` (comme on l’a fait pour `APP_ENV` et `DATABASE_URL`). Veillez à ne **pas committer vos secrets** (mots de passe DB, JWT keys, etc.) dans le dépôt. En production, on utilisera d’autres valeurs (et souvent on passera par des variables d’environnement du conteneur).

**Vérification de Symfony en local :**  
Après avoir lancé les conteneurs, ouvrez `http://localhost:8080` – vous devriez voir soit la page d’accueil de votre app Symfony, soit la page de **debug Symfony** si quelque chose manque. Corrigez les éventuels problèmes (par exemple installer les packages manquants, ajuster les droits d’écriture sur `var/`...). Quand l’application tourne correctement dans Docker en local, vous êtes prêt pour la suite.

## 3. Configuration du dépôt Git et intégration avec DockerHub

L’étape suivante consiste à préparer votre dépôt de code pour qu’il s’intègre à un flux CI/CD avec Docker. Cela implique de structurer le dépôt, de créer un registre sur DockerHub et de mettre en place l’automatisation de build/push de l’image Docker.

### Création et structuration du repository

**Initialisation du dépôt Git :**  
Si ce n’est pas déjà fait, initialisez un dépôt Git dans votre projet Symfony 7 :  
```bash
git init
git add .
git commit -m "Initial commit"
```  
Créez ensuite un repository distant sur une plateforme Git (GitHub, GitLab, Bitbucket, etc.) et poussez-y votre code (`git remote add origin ... && git push -u origin main`).

**Organisation des fichiers :**  
Dans votre dépôt, assurez-vous d’inclure les fichiers liés à Docker : votre `Dockerfile`, le(s) fichier(s) `docker-compose.yml` (au moins celui de production, éventuellement un spécifique pour dev s’il diffère), et éventuellement des scripts de déploiement. Ajoutez un fichier `.dockerignore` pour exclure du contexte de build Docker tout ce qui n’est pas nécessaire (par ex: `.git`, `node_modules`, `var/log`, etc.), afin d’accélérer les builds et éviter d’embarquer des données inutiles dans l’image.

Votre repository pourrait ressembler à ceci : 

```
├── app/ (code Symfony: config/, src/, etc.)
├── public/ (front controller index.php, assets)
├── var/
├── Dockerfile
├── docker-compose.yml
├── docker-compose.prod.yml (éventuellement, pour la prod)
├── .dockerignore
├── .gitignore
└── ... etc.
```

**Docker Compose de production :**  
Il est courant d’avoir un fichier de composition spécifique pour la production. Par exemple `docker-compose.prod.yml` qui diffère du compose de dev. Dans ce fichier de prod, vous n’allez **pas** monter les volumes de code (puisque en prod on utilise l’image construite avec le code intégré) et vous utiliserez l’image depuis DockerHub au lieu de construire en local. Un exemple minimal (supposons un seul conteneur web + la DB) : 

```yaml
services:
  symfony:
    image: dockerhub_user/monapp:latest   # on tirera l'image depuis DockerHub
    container_name: symfony_app
    environment:
      - APP_ENV=prod 
      - DATABASE_URL=mysql://symfony:password@db:3306/app_db
    depends_on:
      - db

  db:
    image: mysql:8.0
    container_name: symfony_db
    environment:
      - MYSQL_DATABASE=app_db
      - MYSQL_USER=symfony
      - MYSQL_PASSWORD=password
      - MYSQL_ROOT_PASSWORD=rootpass
    volumes:
      - db_data_prod:/var/lib/mysql

volumes:
  db_data_prod:
```

Ici, on utilise l’image DockerHub `dockerhub_user/monapp:latest` que l’on va maintenir à jour via CI/CD. Ce compose de prod n’ouvre pas de ports vers l’extérieur pour le conteneur Symfony, car on passera par Nginx Proxy Manager pour y accéder. Veillez à adapter APP_ENV à “prod” et d’autres variables (par exemple APP_SECRET, etc. via un fichier `.env.prod` ou directement dans l’environnement du conteneur, éventuellement via Portainer plus tard). Commitez ce fichier de prod également.

### Automatisation du build et du push de l’image Docker sur DockerHub

Pour automatiser la création de l’image Docker et son envoi sur DockerHub à chaque modification, nous allons mettre en place un workflow CI (avec GitHub Actions dans cet exemple).  

**Création d’un repository sur DockerHub :**  
Connectez-vous à DockerHub (créez un compte si ce n’est pas fait) et créez un **nouveau repository** (public ou privé) pour votre application. Par exemple “monapp”. Notez le nom complet de l’image, qui sera de la forme `votre_utilisateur/monapp`. 

**Configuration des secrets (identifiants DockerHub) :**  
Dans votre plateforme CI (ici GitHub), ajoutez dans les **Secrets du repository** deux valeurs : 
- `DOCKERHUB_USERNAME` (votre nom d’utilisateur DockerHub)  
- `DOCKERHUB_TOKEN` (un token d’accès ou votre mot de passe DockerHub – privilégiez un token que vous pouvez générer dans DockerHub > Account Settings > Security).

Ces secrets seront utilisés par le workflow pour se connecter à DockerHub.

**Création du workflow GitHub Actions :**  
Dans votre dépôt, créez le dossier `.github/workflows` puis un fichier YAML (ex: `ci-cd.yml`). Voici un exemple de workflow minimal : 

```yaml
name: CI/CD Docker

on:
  push:
    branches: ["main"]   # déclenchement sur push sur la branche main (adapter selon votre branching model)

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Log in to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v3
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/monapp:latest
```

Explications : 
- Ce workflow se déclenche sur chaque push sur la branche principale. (On pourrait affiner pour ne builder que sur un tag ou sur merge vers main, etc.)  
- Il utilise l’action officielle `docker/build-push-action` qui construit et pousse l’image Docker en une étape. Avant cela, on utilise `actions/checkout` pour récupérer le code, puis `docker/login-action` pour se connecter à DockerHub en utilisant les secrets fournis ([Automate Docker Image Builds and Push to Docker Hub Using GitHub Actions  - DEV Community](https://dev.to/ken_mwaura1/automate-docker-image-builds-and-push-to-docker-hub-using-github-actions-32g5#:~:text=,secrets.DOCKER_PASSWORD)) ([Automate Docker Image Builds and Push to Docker Hub Using GitHub Actions  - DEV Community](https://dev.to/ken_mwaura1/automate-docker-image-builds-and-push-to-docker-hub-using-github-actions-32g5#:~:text=,steps.meta.outputs.labels)).  
- Le build se fait dans le contexte du repo (`context: .`) et utilise le Dockerfile à la racine. Le tag appliqué sera `votre_user/monapp:latest`. Vous pouvez ajouter d’autres tags (par exemple `${{ github.sha }` pour taguer avec le hash du commit ou une version).  

Enregistrez ce fichier et poussez-le sur le repo. Sur GitHub, allez dans l’onglet **Actions** pour voir le pipeline se déclencher. S’il réussit, votre image Docker sera disponible sur DockerHub. 🔄 Chaque nouveau push mettra à jour l’image. (Pour éviter de toujours tagger `latest`, vous pouvez adopter une stratégie de versionnement, par exemple tagger avec le numéro de version de l’application ou la date, et faire pointer `latest` vers le plus récent.)

**Option alternative – build auto DockerHub :** DockerHub offre aussi la fonctionnalité *Automated Builds* qui peut construire l’image à chaque push sur GitHub. Cependant, cette fonctionnalité est limitée et moins flexible que GitHub Actions. Le guide se concentre sur GitHub Actions pour plus de contrôle.

## 4. Configuration de Portainer et Nginx Proxy Manager sur le VPS

À présent que notre serveur a Docker/Compose, Portainer et NPM installés, et que notre pipeline CI peut fournir des images Docker à jour sur DockerHub, on va configurer Portainer et Nginx Proxy Manager pour préparer le déploiement de l’application Symfony.

### Configuration de Portainer pour gérer les conteneurs Docker

Nous avons déjà installé Portainer précédemment et configuré l’accès admi ([CrownCloud Wiki - How To Install Docker Portainer On Ubuntu 24 04](https://wiki.crowncloud.net/?How_to_Install_Docker_Portainer_on_Ubuntu_24_04#:~:text=Open%20a%20web%20browser%20and,navigate%20to%3A%20https%3A%2F%2F%3A9443))】. Portainer permet de gérer graphiquement vos conteneurs, stacks et volumes. Nous allons l’utiliser notamment pour déployer notre *stack* (ensemble de conteneurs) en production, et éventuellement pour faciliter les mises à jour via son interface ou ses webhooks.

**Création d’une Stack dans Portainer :**  
Au lieu de lancer manuellement `docker compose` en CLI sur le serveur, on peut utiliser Portainer pour déployer notre application Symfony via l’outil *Stacks*. Une stack dans Portainer correspond à un ensemble de services définis par un fichier Compose. 

1. Connectez-vous à Portainer (https://votre-ip:9443). Dans le menu de gauche, cliquez sur **Stacks**. Puis sur **Add Stack** (Ajouter une stack).  
2. Donnez un nom à la stack, par ex *symfony-app*.  
3. Dans le champ **Web editor**, collez le contenu de votre fichier `docker-compose.prod.yml` (ou équivalent) qui définit les services de prod (le conteneur Symfony + DB, etc.). Assurez-vous qu’il référence bien l’image DockerHub (par ex `image: votre_user/monapp:latest`).  
4. Définissez les variables d’environnement nécessaires. Portainer propose un onglet pour les définir (ou vous pouvez les laisser en dur dans le compose, mais par sécurité vous pouvez mettre les secrets comme DB_PASSWORD via l’UI). Par exemple, configurez `APP_SECRET`, `APP_ENV=prod`, `DATABASE_URL` avec les bonnes valeurs de prod (pointant vers la DB de la stack).  
5. Cliquez sur **Deploy the stack**. Portainer va alors traduire cela en création des conteneurs Docker correspondants.

Votre application Symfony devrait maintenant tourner sur le VPS dans un conteneur. Cependant, pour l’instant, elle n’est pas accessible de l’extérieur, car nous n’avons pas exposé de port directement (et on ne le souhaite pas). C’est ici que Nginx Proxy Manager entre en jeu.

### Mise en place de Nginx Proxy Manager (routage et SSL)

Nginx Proxy Manager va jouer le rôle de reverse proxy frontal : il écoute sur le port 80/443 du serveur, et redirige les requêtes vers le conteneur Symfony approprié. Il gère aussi l’obtention de certificats SSL **Let’s Encrypt** en quelques clics.

**Configurer le host proxy pour Symfony :** 

1. Assurez-vous d’avoir un nom de domaine ou au moins un sous-domaine pointant vers l’adresse IP de votre VPS (via un enregistrement DNS A par ex). Ici nous supposerons que vous avez un domaine (ex *monapp.mondomaine.com*).  
2. Dans l’interface de Nginx Proxy Manager (http://votre-ip:81, avec vos identifiants admin), allez dans **Proxy Hosts** > **Add Proxy Host**. Vous verrez un formulaire en plusieurs onglets. 
 ([Screenshots | Nginx Proxy Manager](https://nginxproxymanager.com/screenshots/))】 *Formulaire “New Proxy Host” de Nginx Proxy Manager permettant de créer un hôte proxy.* 

   - **Tab Details (Détails)** : Saisissez le(s) nom(s) de domaine dans “Domain Names” (ex: `monapp.mondomaine.com`). Choisissez `http` comme scheme si votre appli Symfony écoute en HTTP (ce qui est le cas de notre conteneur Apache). Dans “Forward Hostname / IP”, indiquez l’**adresse interne** de la cible. Comme tout tourne sur la même machine Docker, le plus simple est de mettre le nom du conteneur Symfony tel que défini dans Docker Compose (par ex `symfony_app` ou le nom du service dans la stack) – Docker et NPM étant sur le même réseau par défaut, ils peuvent se résoudre. Sinon, utilisez `localhost`. Pour “Forward Port”, entrez le port interne du service Symfony (80 si on a utilisé l’image apache). Par exemple : *Forward Hostname* = `symfony_app` et *Forward Port* = `80`.  
   - Activez l’option **Cache Assets** (cache des assets statiques) si vous voulez que NPM cache les fichiers statiques (CSS/JS) – optionnel. Laissez “Websockets Support” désactivé sauf si votre appli utilise des websockets. L’option “Block Common Exploits” peut être activée pour un léger plus de sécu.  
   - **Tab SSL** : c’est ici qu’on va obtenir un certificat. Cochez “Enable SSL”, puis “Request a new SSL Certificate”. Renseignez votre email pour Let’s Encrypt, et cochez “Force SSL” (pour rediriger HTTP vers HTTPS automatiquement). Validez. Nginx Proxy Manager va alors communiquer avec Let’s Encrypt pour générer un certificat pour votre domaine (assurez-vous que le DNS du domaine pointe bien vers le VPS, et que le port 80 n’était pas bloqué, car LE va vérifier via HTTP). Si tout va bien, vous devriez voir le statut **Online** pour votre nouvel hôte proxy.  

Une fois cette configuration faite, votre application Symfony est disponible publiquement à l’adresse **https://monapp.mondomaine.com** 🎉. Nginx Proxy Manager va réceptionner les requêtes HTTPS, les déchiffrer, puis les transmettre en HTTP au conteneur Symfony. Celui-ci (Apache + PHP) génère la page, renvoie à NPM, qui la transmet au client. 

**Remarque sur la base de données en prod :** Si vous utilisez MySQL/PostgreSQL sur le même VPS, vous pouvez soit l’exposer via NPM (peu utile pour un DB), soit simplement laisser le conteneur DB accessible uniquement en interne (ce que Docker fait par défaut). Par exemple, votre conteneur Symfony “voit” le conteneur `db` sur le réseau Docker, et NPM n’a pas à s’en soucier. Assurez-vous de sauvegarder régulièrement la base de données de production (via un volume ou via Portainer).

**Gestion des logs et erreurs :** En cas de souci, vous pouvez voir les logs Nginx Proxy Manager via son interface (onglet Logs), et les logs des conteneurs Symfony/DB via Portainer (en sélectionnant le conteneur puis “Logs”). Cela aide à débugger d’éventuels erreurs 502 Bad Gateway (souvent dû à un mauvais host/port), ou erreurs applicatives.

## 5. Mise en place du pipeline CI/CD automatisé

Maintenant que tout est en place, nous allons relier les pièces entre elles pour l’automatisation complète du déploiement. L’objectif : à chaque mise à jour du code sur le dépôt, une nouvelle image Docker est construite et poussée (ce qu’on a fait en section 3), **puis automatiquement déployée sur le VPS** sans intervention manuelle.

Plusieurs approches existent pour la partie déploiement continu (CD). Nous décrirons deux options : via GitHub Actions (avec connexion SSH ou webhook) et via un script bash déclenché par webhook DockerHub. Vous pourrez choisir celle qui convient le mieux.

### Automatisation du déploiement (via GitHub Actions ou script Bash)

**Option 1 : GitHub Actions avec déploiement SSH sur le VPS**  
On peut étendre le workflow GitHub Actions créé précédemment en ajoutant une étape qui, une fois l’image poussée, se connecte au VPS et exécute les commandes Docker pour mettre à jour. Pour cela, on utilise un action de type SSH. Par exemple, ajoutez un job au workflow : 

```yaml
  deploy:
    needs: build-and-push   # attend que le job précédent soit fini
    runs-on: ubuntu-latest
    steps:
      - name: Déployer sur VPS
        uses: appleboy/ssh-action@v0.1.7
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /path/vers/stack
            docker compose pull symfony       # récupère la nouvelle image pour le service symfony
            docker compose up -d --remove-orphans
```

Dans cet exemple, on utilise l’action `appleboy/ssh-action` qui exécute un script sur le serveur via SSH. Il faut au préalable ajouter des secrets pour l’IP/hostname du VPS (`VPS_HOST`), le user SSH (`VPS_USER`, par ex “ubuntu” ou autre), et la clé privée SSH (`VPS_SSH_KEY`). La clé privée correspond à une clé publique installée sur le VPS (on recommande de créer une clé dédiée au CI avec des droits limités)7】 L’action va ouvrir une connexion SSH et exécuter les commandes fournies dans `script`. Ici on se place dans le répertoire où se trouve le fichier docker-compose de production (celui qu’on a utilisé dans Portainer). On exécute `docker compose pull` pour récupérer la dernière image depuis DockerHub, puis `docker compose up -d` pour recréer le conteneur Symfony avec la nouvelle image (sans interrompre la DB ni autre service). L’option `--remove-orphans` est là pour nettoyer d’anciens conteneurs qui ne seraient plus dans le compose.

Avec ce setup, le pipeline complet serait : **commit** -> **push Git** -> *GH Actions build+push image* -> *GH Actions déploie* -> conteneur mis à jour. Le tout prenant quelques minutes (selon le temps de build de l’image).

**Option 2 : Script Bash + Webhook DockerHub**  
Si vous ne souhaitez pas utiliser GitHub Actions pour déployer, vous pouvez mettre en place un webhook depuis DockerHub ou un script qui tourne en tâche cron sur le VPS.  
- DockerHub peut appeler une URL à chaque push d’image (paramètre “Webhooks” dans DockerHub). Vous pourriez configurer un petit service web sur le VPS qui, à la réception du webhook, exécute un script de pull. Cependant, cela nécessite de développer ce récepteur de webhook (ou d’utiliser Portainer, voir option 3).  
- Plus simplement, vous pouvez mettre un **script Bash** sur le VPS, par exemple `/usr/local/bin/update_symfony.sh` qui fait les mêmes commandes que ci-dessus (docker compose pull && up). Vous pourriez le lancer manuellement à chaque déploiement, ou l’automatiser via **cron** toutes les x minutes. Par exemple, un cronjob toutes les 5 minutes qui exécute `docker compose pull` – cela introduit un petit délai mais assure de récupérer les images fraîchement publiées. Cependant, ce n’est pas instantané ni très efficace (DockerHub pourrait être interrogé pour rien). 

**Option 3 : Webhook via Portainer (rolling update)**  
Portainer (en version >= 2.11) offre la possibilité de créer un webhook de stack. Dans l’UI Portainer, si vous éditez la stack de votre app, vous verrez une option “Enable webhook”. En l’activant, Portainer génère une URL unique. En appelant cette URL (via `curl` par exemple), Portainer va automatiquement **redeployer la stack** en effectuant un `docker compose pull` + `up` en interne. Vous pouvez utiliser cette approche combinée avec DockerHub ou GitHub : par exemple, à la fin du job GitHub Actions, au lieu d’SSH, faites un appel HTTP (`curl`) vers le webhook de Portainer. (Veillez à restreindre l’accès à cette URL, car quiconque la connaît pourrait redeployer votre app. Elle est longue et secrète, mais ne la divulguez pas.) Cette méthode est élégante car vous n’exposez pas SSH et vous laissez Portainer gérer l’update. Notez cependant qu’il peut faire un redémarrage “rolling” (progressif) si configuré, pour minimiser le downti ([Webhooks | Portainer Documentation](https://docs.portainer.io/user/kubernetes/applications/webhooks#:~:text=Webhooks%20,restart%20of%20the%20application))1】.

Pour rester simple, la méthode **SSH + docker compose** fonctionne très bien et est facile à comprendre.

### Pull des images et rechargement des conteneurs

Que ce soit via l’action CI ou via Portainer, la logique est : **récupérer l’image mise à jour depuis DockerHub** puis **recréer les conteneurs** utilisant cette image. Docker Compose gère cela intelligemment : 
- `docker compose pull <service>` va télécharger la nouvelle version de l’image (tag “latest” par exemple) du service indiqué.  
- `docker compose up -d` va détecter que l’image du service a changé et va recréer le conteneur avec la nouvelle image. Il conserve les mêmes paramètres (volumes, variables, liens réseau). Les conteneurs des autres services (DB, etc.) ne sont pas recréés car leurs images n’ont pas changé. Ainsi, la base de données et autres restent en place, seules l’application Symfony est relancée. En général, ce redémarrage est très rapide (quelques secondes). 

**Note:** Si votre application a besoin de migrations de base de données, pensez à inclure cette étape soit dans le démarrage du conteneur (ex: une commande d’entrée qui exécute `bin/console doctrine:migrations:migrate`), soit manuellement via Portainer/SSH après déploiement d’une nouvelle version qui en nécessite. Vous pouvez automatiser cela via un script dans l’image Docker qui se lance à chaque container start (point d’entrée custom dans Dockerfile).

Après le `docker compose up -d`, le conteneur Symfony nouvellement lancé va commencer à servir le trafic. Nginx Proxy Manager n’a même pas besoin d’être au courant, il pointe toujours vers le même host/port Docker ; la transition se fait en coulisses. 

## 6. Tests de déploiement et bonnes pratiques

Une fois le pipeline mis en place, il est crucial de tester le bon fonctionnement et d’adopter des pratiques saines pour la maintenance.

### Vérifications post-déploiement

Après un déploiement (automatique ou manuel), effectuez ces vérifications : 
- **Accessibilité** : Rendez-vous sur l’URL publique (ex: https://monapp.mondomaine.com) et assurez-vous que la page se charge correctement via HTTPS. Vérifiez aussi que la redirection HTTP->HTTPS fonctionne en testant http://monapp.mondomaine.com.  
- **Fonctionnalités** : Naviguez dans l’application pour vérifier que tout est en ordre (pas d’erreur 500, pas de fonctionnalité cassée). Si possible, ayez une suite de tests ou au moins un petit script de **smoke testing** pour valider les fonctionnalités clés après chaque déploiement.  
- **Logs des conteneurs** : via Portainer ou en SSH (`docker compose logs -f symfony`), inspectez les journaux de l’app Symfony juste après le déploiement. Assurez-vous qu’il n’y a pas d’exception au startup, que la connexion à la base de données se fait bien, etc. Idem pour le conteneur de DB, vérifiez qu’il n’a pas redémarré inopinément.  
- **Certificat SSL** : Dans NPM, vérifiez que le certificat Let’s Encrypt est bien valide (section **SSL Certificates** de NPM) et notez sa date d’expiration. Nginx Proxy Manager renouvelle automatiquement les certificats avant expiration, mais c’est bien de savoir quand.  
- **Portainer stack** : Allez dans Portainer > Stacks > symfony-app (ou nom choisi) et voyez l’état. Portainer signale s’il y a eu des erreurs au déploiement. S’il y a un souci (ex: l’image n’a pas pu être pullée), vous aurez un message. Vous pouvez aussi voir l’historique des déploiements.

### Bonnes pratiques de maintenance et mises à jour du pipeline

Pour finir, voici quelques bonnes pratiques pour assurer la pérennité et la fiabilité de votre pipeline CI/CD :

- **Versionner vos images** : Utiliser uniquement le tag “latest” est simple mais peut compliquer les retours arrière. Envisagez d’ajouter un tag versionné (par exemple `monapp:v1.2.3` en plus de `latest`). Ainsi, en cas de problème en production avec la dernière version, vous pouvez facilement redéployer l’image précédente via Portainer en changeant le tag de l’image dans la stack (ou en faisant `docker compose up -d` avec l’ancien tag). Docker Hub conservera les tags tant que vous ne les supprimez pas.  
- **Sauvegardes** : Mettez en place des sauvegardes régulières de vos données critiques : base de données (dump SQL régulier stocké hors du VPS), et éventuellement volumes de fichiers si votre app en utilise (par ex, si l’app stocke des fichiers uploadés sur le disque, assurez-vous que ce soit dans un volume Docker et sauvegardez-le ou montez-le depuis le host pour l’inclure dans vos backups).  
- **Surveillance** : Utilisez des outils de monitoring pour être alerté si votre site tombe ou si des erreurs surviennent. Par exemple, des services externes ou un script simple qui fait un ping régulier sur une URL de santé (`/health` route par ex) de votre app. En cas de non-réponse, vous pouvez recevoir une alerte pour intervenir rapidement.  
- **Logs et debug** : Configurez un agrégateur de logs si possible (par exemple, envoyez les logs Symfony vers syslog ou vers un service externe type Loggly) pour garder un historique, surtout si vos conteneurs redémarrent (les logs in-container pourraient être perdus). Symfony en prod doit généralement avoir `APP_DEBUG=false` – assurez-vous de ne pas laisser le mode debug activé en prod pour des raisons de performance et de sécurité.  
- **Mises à jour de sécurité** : Maintenez à jour vos images Docker de base. Par exemple, si une nouvelle version de `php:8.2-apache` sort avec des correctifs, rebuildez votre image. Vous pouvez utiliser des outils comme Dependabot qui ouvrent des PR automatiques pour bump la version de base dans votre Dockerfile. De même, surveillez les mises à jour de Portainer et Nginx Proxy Manager. Portainer peut être mis à jour en tirant la nouvelle image et en redémarrant le conteneur (procédure relativement simple via Portainer lui-même ou CLI). Nginx Proxy Manager pareil (nouvelle image, `docker compose pull && up -d` dans son dossier).  
- **Sécurisation continue** : Changez régulièrement vos mots de passe admin (Portainer, NPM). Veillez à ce que le VPS lui-même soit sécurisé (fail2ban peut être utile pour bannir IP en cas de tentatives SSH multiples, etc.). Évitez d’ouvrir des ports non nécessaires au public.  
- **Documentation** : Documentez votre pipeline et l’infrastructure (par exemple, ce guide peut servir de base de documentation 😊). Ainsi, en cas de changement d’équipe ou si vous revenez sur le projet après plusieurs mois, vous saurez comment tout est branché.  

