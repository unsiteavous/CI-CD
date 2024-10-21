# gitlab-ci.yml
L'integration continue commence vraiment ici. Pourtant, avec nos précédentes étapes, on a énormément défriché le terrain. Nos tests peuvent être exécutés en local, il ne nous reste plus qu'à demander à gitlab de les exécuter en ligne.

Pour cela, nous aurons besoin :
- d'un compte gitlab,
- d'un accès ftp vers un hébergement (pour la CD),
- d'un fichier `gitlab-ci.yml`,
- d'un runner associé à ce projet,
- de variables CI/CD 

### 1. Découverte de l'interface de gitlab
Rendez-vous sur gitlab, et créez un projet vierge.
Initialisez votre dépôt en local.

Créez un fichier `gitlab-ci.yml`. 

On va commencer par mettre en place un peu tous les trucs nécessaires au bon fonctionnement du CI.

```yml
image: php:8.3-cli

stages:
  - build
  - test
  - push
  - deploy

variables:
  PHP_VERSION: "8.3"

build:
  stage: build
  script:
    - echo "Build en cours..."
    - echo "La version de PHP est " . $PHP_VERSION

test:
  stage: test
  script:
    - echo "Tests en cours..."

push:
  stage: push
  script:
    - echo "On pousse sur une autre branche..."

deploy:
  stage: deploy
  script:
    - echo "Déploiement en cours..."
```

Actuellement, tout se passe bien, parce qu'on ne fait rien de foufou.
Si on veut exécuter nos tests avec grumphp, par exemple, ça va compléxifier un peu les choses. 
Tout d'abord, notre dossier vendor n'est pas chargé sur git. Donc il faur installer toutes les dépendances pour faire les tests. Pour cela, il nous faut composer. Or, composer n'est pas installé dans notre image ! Pour installer composer, il nous faut curl. Pas installé non plus. Pour que les tests de phpunit fonctionnent, il faut que Xdebug soit installé et activé par défaut dans le conteneur, ce qui n'est pas le cas non plus. Et ainsi de suite.

## Activité 3
Mettez en place tout le nécessaire pour pouvoir lancer votre commande habituelle pour faire fonctionner grumphp en CI.

## Activité 4 
Nos tests fonctionnent, poussons le code en prod.

## Activité 5
Le code est poussé sur la branche de production, exécutons un script pour transférer notre code vers le serveur de production, seulement quand on est sur la branche prod.

