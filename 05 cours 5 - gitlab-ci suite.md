# gitlab-ci.yml (suite)
Lors des activités précédentes, vous devez avoir approfondi votre `gitlab-ci.yml`. 
Voici une version expliquée, pour corriger votre travail :

```yaml
image: php:8.3-cli

stages:
  - build
  - test
  - push
  - deploy

before_script:
  # Installe PHP et les dépendances requises
  - apt-get update && apt-get install -y curl git unzip zip libzip-dev zlib1g-dev
  - docker-php-ext-install zip
  - pecl install xdebug && docker-php-ext-enable xdebug
  - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
  - php -m | grep -E 'xml|mbstring'
  - apt-get install -y lftp

cache:
  paths:
    - vendor/
    - ~/.composer/cache

build:
  stage: build
  script:
    - echo "Build en cours..."
    - composer install --prefer-dist --no-progress --no-suggest --no-interaction

  only:
    - main

test:
  stage: test
  script:
    - echo "Test en cours..."
    - composer grumphp
    - composer coverage
  variables:
    XDEBUG_MODE: coverage
  coverage: '/Lines:\s*(\d+(\.\d+)?%?)/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: var/log/coverage/cobertura.xml

  only:
    - main

push:
  stage: push
  script:
    - echo "Push en cours..."

deploy:
  stage: deploy
  script:
    - echo "Deploy en cours..."

  only:
    - prod
```
## Explications
- **image: php:8.3-cli**  
  Lors de la mise en route, le runner de gitlab va instancier un environnement docker pour opérer les différentes étapes du CI sur notre code. Pour cela, on lui précise le type d'image à installer.

- **stages**  
  Les "stages" (ou étapes), sont un sommaire de tout ce qu'il faudra faire dans le CI/CD. on les retrouve ensuite dans chaque job (au singulier).

- **before_script:**  
  Lorsqu'on installe notre image de base, on est vite coincés : par exemple, il n'y a pas composer d'installé par défaut dans cette image de PHP. Donc lorsqu'on veut lancer la commande `composer install`, et bien on a une erreur qui nous dit que la commande composer n'existe pas.

  Pour cela, on vient compléter notre image avec des installations à réaliser après avoir monté l'image, et avant de commencer les étapes du CI/CD.

  1. Dans les différentes étapes, on commence par mettre à jour et installer les librairies manquantes.
  2. docker-php-ect-install est une commande qui permet d'installer globalement un élément dans l'image docker, comme si c'était inclus de base. De même, docker-php-ext-enable permet d'activer quelque chose globalement dans le docker, pour qu'on puisse s'en servir de n'importe où ensuite.
  3. On installe évidemment composer, mais aussi xdegub, pour la couverture de tests.
  4. le lftp est un élément qu'on reverra plus tard, dans de CD.

- **cache:**  
  L'annotation cache permet de préciser à gitlab où stocker les dossier qu'il aura probablement à réutiliser plusieurs fois, pour gagner du temps lors de l'exécution des pipelines

- **les différentes étapes :**  
  Ensuite se déroulent les différentes étapes (build, test, push, deploy), avec toujours la même architecture :
    - l'intitulé de l'étape,
    - le nom de l'étape associée (stage),
    - les scripts à exécuter,
    - sur quelles branches s'exécuteront ces étapes (only)

  le build permet de mettre en place le projet (pas le docker), en chargeant avec composer le dossier vendor.

  La phase de test permet de lancer grumphp et d'écouter la couverture des tests. La ligne de regex change en fonction des langages et des frameworks de tests, vous retrouverez tout ça dans la [documentation](https://docs.gitlab.com/ee/ci/testing/code_coverage.html).

  Elle permet également d'enregistrer un **Artéfact** pour garder une trace du fichier de couverture même une fois que les tests sont terminés.

  Les phases de push et de deploy seront abordées lors du CD.

Pour chacune des phases, on peut choisir sur quelle branche elles s'exécuteront. Par exemple, on lance toutes les phases lors d'un push sur la branche main, sauf la mise en prod qui se lancera après un push sur la branche prod.

## Interface de gitlab
Depuis gitlab, pour un CI tel quel, nous n'avons pas grand chose à mettre en place. Il faut quand même vérifier qu'on a bien des runners qui vont exécuter les tests.
Allez dans Paramètres > CI/CD > runners et vérifiez que les runners d'instance sont bien activés.