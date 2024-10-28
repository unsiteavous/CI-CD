# Mise en place du *continuous deployment*
Dans ce cours nous allons voir comment mettre en place un déploiement continu avec un dépôt qui contient un code facilement portable : 
- des fichiers PHP
- des fichiers web (css, js, assets, ...)

Pour cela, nous aurons besoin simplement des codes d'accès FTP d'un espace de production.

La démarche sera très simple : 
1. Une fois que le code a passé les tests en CI,
2. on le pousse sur la branche de prod.
3. lorsque le code est poussé sur la branche prod, la dernière étape du gitlab-ci.yml s'exécute : la mise en production du code.
4. Cette étape consiste en la copie des fichiers qui nous intéressent sur le serveur via FTP, pour mettre en ligne le projet.

## Les variables
Dans gitlab, nous avons la possibilité de créer des variables secrètes depuis l'interface, qui seront appelées par le gitlab-ci.yml, lors de son exécution. Cela permet de garder secret les mots de passes et accès au FTP, à la base de données de production, au compte gitlab qui pourra pousser le code d'une branche à une autre, ... 

## Le LFTP
Le [LFTP](https://doc.ubuntu-fr.org/lftp) est un moyen de transférer des fichiers en ligne de commande. On va ainsi pouvoir travailler depuis notre conteneur de CD, et communiquer avec le serveur de production, en lui disant quoi faire. 

## Activité 1 : pousser sur la branche prod
Dans un premier temps, on va basculer le code validé sur la branche de production. Pour cela, créez un jeton d'accès (paramètres > access token) en lui accordant la possibilité d'écrire dans le dépôt. 
Ce jeton va nous permettre de pousser le code vers la branche de production, sans utiliser votre compte à vous. Cela permet de resteindre les droits et sécuriser votre compte. 

Ensuite, dans les variables (paramètres > CI/CD > variables), créez-en une pour stocker le jeton précédemment créé.

Puis, dans votre `gitlab-ci.yml`, ajoutez le code nécessaire pour pousser vers la branche prod. Ce code doit contenir : 
- une limitation d'exécution à la branche main
- une configuration git globale indiquant l'email et le nom de l'utilisateur qu'on va utiliser pour faire le push (en l'occurence, vous mettez ce que vous voulez, quelque chose de fictif qui permet de savoir qu'on va travailler avec un robot qui a un jeton d'accès).
- vous écrivez la ligne du push, avec le jeton d'accès, et vous poussez sur la branche main. Ajoutez le drapeau `--force` afin que le robot puisse toujours pousser votre code sur cette branche, quoi qu'il arrive à la branche prod.

## Activité 2 : Mettez en ligne le code
Maintenant qu'on a du code sur la branche de production, on va pouvoir ajouter la dernière étape, qui consiste en une copie par lftp des fichiers du dépôt vers le serveur.

