# Correction globale
voici une version aboutie du fichier `gitlab-ci.yml` :

```yml
image: php:8.3-cli

stages:
  - build
  - test
  - push_preprod
  - deploy_preprod
  - push_prod
  - deploy_prod

variables:
  PHP_VERSION: "8.3"

cache:
  paths:
    - vendor/
    - ~/.composer/cache

.php_setup: &php_setup
  before_script:
    - apt-get update && apt-get install -y curl git unzip zip libzip-dev zlib1g-dev
    - docker-php-ext-install zip
    - pecl install xdebug && docker-php-ext-enable xdebug
    - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
    - php -m | grep -E 'xml|mbstring'

build:
  <<: *php_setup
  stage: build
  script:
    - echo "Build en cours..."
    - composer install --prefer-dist --no-progress --no-suggest --no-interaction
  artifacts:
    paths:
      - vendor/
  only:
    - main

test:
  <<: *php_setup
  stage: test
  variables:
    XDEBUG_MODE: coverage
  coverage: '/Lines:\s*(\d+(\.\d+)?%?)/'
  script:
    - echo "Tests en cours..."
    - composer grumphp
    - composer coverage
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: var/log/coverage/cobertura.xml
  dependencies:
    - build
  only:
    - main

push_preprod:
  image: alpine:latest
  stage: push_preprod
  before_script:
    - apk add --no-cache git
  script:
    - |
      cat <<- EOF > .gitignore
      .editorconfig
      *.xml
      *.lock
      *.disabled
      *.json
      rector.php
      codeception.yml
      grumphp.yml

      src/Tests/
      tests/

      vendor/
      var/
      vendor-bin/

      EOF

    - git config --global user.email "robot@cicd.gitlab"
    - git config --global user.name "Robot CICD"

    - git rm -r --cached .
    - git add .
    - git commit -m "Update .gitignore" -n || echo "No changes to commit"

    - git push -u --force "https://gitlab-ci-token:${CICD_ROBOT_PWD}@gitlab.com/unsiteavous/cicd.git" HEAD:refs/heads/preprod
    # - git push -u --force "https://gitlab-ci-token:${CICD_ROBOT_PWD}@git.captp.fr/theophile/tests-et-ci-cd.git" HEAD:refs/heads/preprod

  only:
    - main

deploy_preprod:
  image: alpine:latest
  stage: deploy_preprod
  before_script:
    - apk add --no-cache lftp openssh-client

  script:
    - echo "Déploiement en cours..."

    - WORKDIR=$(pwd)
    - echo "Répertoire de travail :" . $WORKDIR
    - lftp sftp://${FTP_USER}:${FTP_PWD}@${FTP_HOST} -e "set sftp:auto-confirm yes; set net:timeout 300; set net:max-retries 3; cd ./deploy_test/; mirror -R -v --only-newer --delete $WORKDIR/ ./; quit"

  only:
    - preprod

push_prod:
  image: alpine:latest
  stage: push_prod
  before_script:
    - apk add --no-cache git
  script:
    - git config --global user.email "robot2@cicd.gitlab"
    - git config --global user.name "Robot2 CICD"
    - git push -u --force "https://gitlab-ci-token:${CICD_ROBOT_PWD2}@gitlab.com/unsiteavous/cicd.git" HEAD:refs/heads/prod
    # - git push -u --force "https://gitlab-ci-token:${CICD_ROBOT_PWD2}@git.captp.fr/theophile/tests-et-ci-cd.git" HEAD:refs/heads/prod

  only: 
    - preprod
  when: on_success

deploy_prod:
  image: alpine:latest
  stage: deploy_prod
  before_script:
    - apk add --no-cache lftp openssh-client
  script:
    - echo "Déploiement en cours..."

    - WORKDIR=$(pwd)
    - echo "Répertoire de travail :" . $WORKDIR

    - lftp sftp://${FTP_USER}:${FTP_PWD}@${FTP_HOST} -e "set sftp:auto-confirm yes; set net:timeout 300; set net:max-retries 3; cd ./deploy_test/; mirror -R -v --only-newer --delete $WORKDIR/ ./; quit"

    # - lftp sftp://${FTP_USER}:${FTP_PWD}@${FTP_HOST} -e "set sftp:auto-confirm yes; cd ./deploy_test/; mrm -r *; mirror -R -v --exclude .git/ --exclude .gitignore --exclude-glob '*.yml' --exclude vendor/ --exclude composer.json --exclude composer.lock --exclude .editorconfig --exclude rector.php --exclude tests/ --exclude-glob '*.xml' --exclude src/Tests $WORKDIR/ ./; quit"

  only:
    - prod
  when: manual
```

Dans ce fichier, beaucoup de choses sont dûes à des réflexions abordées en cours. Je m'efforcerai de les retranscrire ici de manière synthétique, mais nous pourrons les réaborder en cours au besoin.

## Les **stages**
Ce fichier pousse la CI/CD à son apogée : une étape de build, de test, puis un changement de branche, un premier déploiement, si c'est réussi, un second changement de branche et un déploiement manuel.

On constate que pour chacune de ces étapes, les paramétrages donnés à la racine (au niveau 0 d'indentation) sont appliqués à chaque fois, ce qui peut ralentir l'exécution des tests.

Pour résoudre ce problème, on peut redéfinir des choses directement dans les différentes étapes. C'est le cas des push et des deploy, où nous n'avons pas besoin d'une image complète de PHP puisqu'on veut juste basculer des fichiers d'une branche à l'autre, ou d'un serveur à un autre. À ce moment-là, on a juste besoin de l'image la plus légère qui soit : **alpine:latest**.

De la même manière, pour s'assurer que le before_script s'exécute seulement dans les parties build et test, j'ai créé ici des ancres yaml, pour préparer un bout de script qui sera appelé ensuite à différents endroits : `.php_setup: &php_setup` crée l'ancre, `<<: *php_setup` l'appelle.

Vous remarquerez également que l'installation de git n'est mise en place que dans les branches push, et celle de lftp que pour les deploy. On accélère grandement l'exécution de chaque étape de cette manière.

## Variables protégées
Lorsque vous définissez des variables dans gitlab, attention à la case "protéger la variable" : En effet, si vous la cochez, la variable ne sera utilisable que depuis des branches ou des étiquettes **protégées**. Cela veut dire que si votre branche n'est pas protégée, vous ne pourrez pas appeler cette variable.

Deux possibilités s'offrent à vous : 
- Ne pas marquer comme protégées vos variables
- protéger vos branches (depuis paramètres > dépôt > branches protégées)

## .gitignore
Pour éviter de stocker dans les branches de preprod et de prod des fichiers qui n'ont pas à être en production (tout ce qui concerne les tests par exemple), jusqu'à présent nous excluions les fichiers lors du transfert SFTP.
Sauf que tous ces fichiers étaient tout de même dans le dépot. 

Ici, nous choisissons d'exclure grâce au .gitignore les fichiers directement des dépôts de preprod et de prod.
Avec `cat <<- EOF > .gitignore`, nous ecrivons les lignes nécessaires dans le .gitignore, jusqu'au `EOF` suivant.

Ensuite, nous modifions la mémoire de git pour qu'il revoit son exclusion de fichiers avec ces lignes : 
```yml
  - git rm -r --cached .
  - git add .
  - git commit -m "Update .gitignore" -n || echo "No changes to commit"
```
Cela nous permet aussi de simplifier nos lignes de SFTP, car les fichiers à exclure n'existant plus, il n'y a plus besoin de les exclure !

## transfert sftp
Lors du transfert, vous aurez la possibilité de faire du SFTP si votre utilisateur est créé avec SSH. 

C'est très bien pour la sécurité. Cependant, ça peut ajouter une petite complexité dans la commande : Lors d'une connexion au serveur, on nous demande normalement si on veut enregistrer l'hôte distant dans les known-hosts. Or là comme c'est automatisé, on ne peut pas répondre oui, et ça fait échouer la commande. On va donc rajouter un paramètre pour le faire automatiquement. De même, si la commande est trop longue, ça peut poser des problèmes. Pour éviter d'attendre 10 minutes qu'on ait une erreur, vous pouvez imposer un délai maximum, avec un timeout et un max-retries. 

`set sftp:auto-confirm yes; set net:timeout 300; set net:max-retries 3;`

Enfin, nous avions précédemment vu le drapeau `only-newer` pour ne pousser lors du déploiement que les fichiers modifiés ou créés. Mais cette commande ne supprime pas les fichiers qui sont sur le serveur et qui ne servent plus à rien. Pour cela, il faut rajouter le drapeau `--delete`. 

