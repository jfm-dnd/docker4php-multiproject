# docker4php-multiproject

Après avoir passé de longues heures à configurer un environnement multi-projets avec Docker, il m'a semblé intéressant de partager cette expérience.  

La description ci-après est basée sur les ressources mises à disposition par https://github.com/wodby/docker4php et des recherches mentionnées dans le texte.  
L'[installation de Docker](https://hub.docker.com/) est bien entendu un pré-requis.

## Structure pas à pas

Tout d'abord, télécharger la [dernière version stable de Docker4php](https://github.com/wodby/docker4php/releases). Le dossier décompressé ressemble à quelque chose comme ceci :  

```
|- docker4php
|--| .env
|--| docker-compose.yml
|--| docker-sync.yml
|--| traefik.yml
```
[Version 1.5.12](https://github.com/wodby/docker4php/releases/tag/1.5.12) utilisée sur cet exemple.  
À ce stade il est possible de lancer la commande suivante depuis le dossier docker4php :

```
docker-compose up -d
```

Par défaut, **nginx** pointe vers le dossier `/public`, donc penser à le créer et y ajouter un `index.php`. 

```
|- docker4php
|--| .env
|--| docker-compose.yml
|--| docker-sync.yml
|--| traefik.yml
|--| public
|--|--| index.php
```
Le site est à présent accessible à http://php.docker.localhost:8000. les paramètres du projet (nom, url, versions php) sont modifiables via le fichier .env.  
Pour plus d'infos sur la configuration : https://github.com/wodby/docker4php et https://wodby.com/docs/stacks/php/local.

Cette configuration peut être répétée pour chacun de vos projets. Se posera alors le problème du port (8000 par défaut) utilisé par traefik qu'il faudra penser à modifier via les fichiers docker-compose.yml si vous souhaitez lancer plusieurs projets en même temps.  

### Heureusement il y a traefik

Pour répondre à cette problématique, voici une autre structure possible que docker4php nous propose (https://github.com/wodby/docker4drupal/issues/124).

````
|- docker4php
|--| traefik.yml
|--| site1
|--|---| .env
|--|---| docker-compose.yml
|--|---| public
|--| site2
|--|---| .env
|--|---| docker-compose.yml
|--|---| public
...
````
Chaque projet aura ainsi sa propre configuration via les fichiers docker-compose.yml et .env.

#### Fichier traefik.yml en racine

Comme très bien expliqué dans la doc https://github.com/wodby/php-docs/blob/master/docs/local/multiple-projects.md, voici le fichier traefik.yml modifié et correspondant à la structure donnée en exemple ci-dessus.

```
version: '3'

services:
  traefik:
    image: traefik:v2.0
    command: --api.insecure=true --providers.docker
    networks:
    - site1
    - site2
    ports:
    - '80:80'
    - '8080:8080'
    volumes:
    - /var/run/docker.sock:/var/run/docker.sock

networks:
  site1:
    external:
      name: site1_default
  site2:
    external:
      name: site2_default
```

*IMPORTANT*

- Vérifier l'unicité des url définies via la variable PROJECT_BASE_URL (fichier .env)
- Commenter la section traefik de chaque fichier docker-compose.yml

Étape suivante :  
Depuis chaque sous-répertoire (site1 et site2 dans l'exemple), lancer la commande `docker-compose up -d`.
 
```
site1 $ docker-compose up -d
site2 $ docker-compose up -d
```

Puis, depuis le répertoire parent (docker4php dans notre exemple) :

```
docker4php $ docker-compose -f traefik.yml up -d
```

Cette dernière commande ouvre le port 80 et donne accès aux sous-projets http://site1.docker.localhost et http://site2.docker.localhost.
Les différents services fonctionnent ensuite normalement.
Pour phpmyadmin par exemple :  
http://pma.site1.docker.localhost, http://pma.site2.docker.localhost.

### Plus loin dans la configuration
#### Phpmyadmin - privileges
Par défaut, l'accès au phpmyadmin est configuré avec un identifiant - USER - ne donnant pas accès à certaines fonctionnalités comme la création de bases de données notamment.

```
pma:
  environment:
    PMA_USER: $DB_USER
    PMA_PASSWORD: $DB_PASSWORD
```

Obtenez les privilèges root comme suit :

```
pma:
  image: phpmyadmin/phpmyadmin
  container_name: "${PROJECT_NAME}_pma"
  environment:
    PMA_HOST: $DB_HOST
    PMA_USER: root
    PMA_PASSWORD: $DB_ROOT_PASSWORD
```

#### php.ini pour Phpmyadmin - upload_max_filesize

Vous constaterez que l'importation des fichiers .sql en base de donnée est limitée à 2 048Kio.
Voici une solution pour surcharger le fichier php.ini de pma. Ajouter `./php-ini/php.ini:/usr/local/etc/php/php.ini` à la section volumes de pma.

```
  pma:
    image: phpmyadmin/phpmyadmin
    container_name: "${PROJECT_NAME}_pma"
    environment:
      PMA_HOST: $DB_HOST
      PMA_USER: root
      PMA_PASSWORD: $DB_ROOT_PASSWORD
      PHP_UPLOAD_MAX_FILESIZE: 2G
      PHP_MAX_INPUT_VARS: 2G
    labels:
      - "traefik.http.routers.${PROJECT_NAME}_pma.rule=Host(`pma.${PROJECT_BASE_URL}`)"
    volumes:
      - /sessions
      - ./php-ini/php.ini:/usr/local/etc/php/php.ini
```
Créer un répertoire /php-ini au même niveau que le docker-compose dans lequel vous placerez un fichier php.ini.
La structure ressemble à présent à ceci :

````
|- docker4php
|--| traefik.yml
|--| site1
|--|---| .env
|--|---| docker-compose.yml
|--|---| php-ini
|--|---|--| php.ini
|--|---| public
|--| site2
|--|---| .env
|--|---| docker-compose.yml
|--|---| php-ini
|--|---|--| php.ini
|--|---| public
````

Ajouter les configurations au fichier php.ini, exemple :

```
post_max_size=400M
```
La modification sera prise en compte au prochain redémarrage `site1 $ docker-compose restart`.

Le fichier php.ini ainsi créé peut également servir pour surcharger php en ajoutant la ligne `PHP_INI_SCAN_DIR: "/usr/local/etc/php/custom.d:/usr/local/etc/php/conf.d"`
```
  php:
    image: wodby/php:$PHP_TAG
    container_name: "${PROJECT_NAME}_php"
    environment:
      PHP_SENDMAIL_PATH: /usr/sbin/sendmail -t -i -S mailhog:1025
      DB_HOST: $DB_HOST
      DB_USER: $DB_USER
      DB_PASSWORD: $DB_PASSWORD
      DB_NAME: $DB_NAME
      PHP_FPM_USER: wodby
      PHP_FPM_GROUP: wodby
      PHP_INI_SCAN_DIR: "/usr/local/etc/php/custom.d:/usr/local/etc/php/conf.d"
```
Pour les explications sur le chemin *PHP_INI_SCAN_DIR*, je vous invite à voir cet article https://sheershoff.ru/custom-php-ini-docker-php-docker-compose-config/.
#### .htaccess

Ngninx n'autorise pas l'utilisation d'un fichier .htaccess. Pour cela il est possible d'activer apache via le fichier docker-compose en décommentant la section correspondante.
Pour les utilisateurs Mac, utilisez plutôt `- ./:/var/www/html:cached` afin d'avoir les modifications en temps réel (https://wodby.com/docs/stacks/php/local/#docker-for-mac).

#### Navigateurs

Enfin, penser à déclarer les noms de domaine dans l'environnement virtuel si vous voulez accéder aux projets avec d'autres navigateurs que Chrome.
Exemple sur Mac :

```bash
sudo pico /etc/hosts
```

```
127.0.0.1 site1.docker.localhost
127.0.0.1 www.site1.docker.localhost
```

La config en exemple est disponible sur ce dépôt dans le répertoire docker4php.  
N'hésitez pas à me laisser vos commentaires et remarques. Enjoy :)

