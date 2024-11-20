---
lab:
  title: Répertorier les modifications apportées à la table avec le décodage logique
  module: Understand write-ahead logging
---

# Répertorier les modifications apportées à la table avec le décodage logique

Dans cet exercice, vous allez configurer la réplication logique, native dans PostgreSQL. Vous allez créer deux serveurs, qui agissent en tant qu’éditeur et abonné. Les données dans zoodb seront répliquées entre les deux.

## Avant de commencer

Pour effectuer cet exercice, vous avez besoin de votre propre abonnement Azure. Si vous n’avez pas d’abonnement Azure, vous pouvez demander un [essai gratuit d’Azure](https://azure.microsoft.com/free).

## Créer un groupe de ressources

1. Connectez-vous au portail Azure. Votre compte d’utilisateur doit être propriétaire ou contributeur pour l’abonnement Azure.
1. Sélectionnez **Groupes de ressources**, puis **+ Créer**.
1. Sélectionnez votre abonnement.
1. Dans le groupe de ressources, entrez **rg-PostgreSQL_Replication**.
1. Sélectionnez une région proche de votre emplacement.
1. Sélectionnez **Revoir + créer**.
1. Sélectionnez **Créer**.

## Créer un serveur de publication

1. Sous Services Azure, sélectionnez **+Créer une ressource**. Sous **Catégories**, sélectionnez **Bases de données**. Sous **Azure Database pour PostgreSQL**, sélectionnez **Créer**.
1. Sous l’onglet **Général** du serveur flexible, renseignez chaque champ comme suit :
    1. Abonnement : votre abonnement.
    1. Groupe de ressources : sélectionnez **rg-PostgreSQL_Replication**.
    1. Nom du serveur : **psql-postgresql-pub9999** (le nom doit être globalement unique, remplacez donc 9999 par quatre chiffres aléatoires).
    1. Région - sélectionnez la même région que le groupe de ressources.
    1. Version de PostgreSQL : sélectionnez 16.
    1. Type de charge de travail - **Développement**.
    1. Calcul + stockage - **Burstable**. Sélectionnez **Configurer le serveur** et examinez les options de configuration. N’apportez aucune modification et fermez la section.
    1. Zone de disponibilité - 1. Si les zones de disponibilité ne sont pas prises en charge, laissez Aucune préférence.
    1. Haute disponibilité : activée.
    1. Méthode d’authentification : authentification PostgreSQL uniquement
    1. Dans **Nom d’utilisateur administrateur**, entrez **`demo`**.
    1. Dans **Mot de passe**, entrez un mot de passe complexe approprié.
    1. Sélectionnez **Suivant : Réseau >**.
1. Sous l’onglet **Mise en réseau** du serveur flexible, renseignez chaque champ comme suit :
    1. Méthode de connectivité : (o) Accès public (adresses IP autorisées).
    1. **Autoriser l’accès public à partir d’un service Azure dans Azure sur ce serveur** - coché. Cela doit être coché, afin que les bases de données d’éditeur et d’abonné puissent communiquer entre elles.
    1. Sous Règles de pare-feu, sélectionnez **Ajouter l’adresse IP actuelle du client**. Cela ajoute votre adresse IP actuelle en tant que règle de pare-feu. Vous pouvez éventuellement nommer cette règle de pare-feu en quelque chose de significatif.
1. Sélectionnez **Revoir + créer**. Sélectionnez ensuite **Créer**.
1. Étant donné que la création d’un serveur Azure Database pour PostgreSQL peut prendre quelques minutes, commencez par l’étape suivante dès que ce déploiement est en cours. N’oubliez pas d’ouvrir une nouvelle fenêtre ou un nouvel onglet de navigateur pour continuer.

## Créer un serveur abonné

1. Sous Services Azure, sélectionnez **+Créer une ressource**. Sous **Catégories**, sélectionnez **Bases de données**. Sous **Azure Database pour PostgreSQL**, sélectionnez **Créer**.
1. Sous l’onglet **Général** du serveur flexible, renseignez chaque champ comme suit :
    1. Abonnement : votre abonnement.
    1. Groupe de ressources : sélectionnez **rg-PostgreSQL_Replication**.
    1. Nom du serveur : **psql-postgresql-sub9999** (le nom doit être globalement unique, remplacez donc 9999 par des nombres aléatoires).
    1. Région - sélectionnez la même région que le groupe de ressources.
    1. Version de PostgreSQL : sélectionnez 16.
    1. Type de charge de travail - **Développement**.
    1. Calcul + stockage - **Burstable**. Sélectionnez **Configurer le serveur** et examinez les options de configuration. N’apportez aucune modification et fermez la section.
    1. Zone de disponibilité - 2. Si les zones de disponibilité ne sont pas prises en charge, laissez Aucune préférence.
    1. Haute disponibilité : activée.
    1. Méthode d’authentification : authentification PostgreSQL uniquement
    1. Dans **Nom d’utilisateur administrateur**, entrez **`demo`**.
    1. Dans **Mot de passe**, entrez un mot de passe complexe approprié.
    1. Sélectionnez **Suivant : Réseau >**.
1. Sous l’onglet **Mise en réseau** du serveur flexible, renseignez chaque champ comme suit :
    1. Méthode de connectivité : (o) Accès public (adresses IP autorisées)
    1. **Autoriser l’accès public à partir d’un service Azure dans Azure sur ce serveur** - coché. Cela doit être coché, afin que les bases de données d’éditeur et d’abonné puissent communiquer entre elles.
    1. Sous Règles de pare-feu, sélectionnez **Ajouter l’adresse IP actuelle du client**. Cela ajoute votre adresse IP actuelle en tant que règle de pare-feu. Vous pouvez éventuellement nommer cette règle de pare-feu en quelque chose de significatif.
1. Sélectionnez **Revoir + créer**. Sélectionnez ensuite **Créer**.
1. Attendez que les deux serveurs Azure Database pour PostgreSQL soient déployés.

## Configurer la réplication

Pour les serveurs éditeur *et* abonné :

1. Dans le portail Azure, accédez au serveur, puis, sous Paramètres, sélectionnez **Paramètres du serveur**.
1. À l’aide de la barre de recherche, recherchez chaque paramètre et apportez les modifications suivantes :
    1. `wal_level` = LOGICAL
    1. `max_worker_processes` = 24
1. Cliquez sur **Enregistrer**. Sélectionnez ensuite **Enregistrer et redémarrer**.
1. Attendez que les deux serveurs redémarrent.

    > Remarque
    >
    > Une fois les serveurs redéployés, vous devrez peut-être actualiser vos fenêtres de navigateur pour remarquer que les serveurs ont redémarré.

## Avant de continuer

Assurez-vous d’avoir effectué ce qui suit :

1. Vous avez installé et démarré les deux serveurs flexibles Azure Database pour PostgreSQL.
1. Vous avez déjà cloné les scripts du labo à partir de [PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql.git). Si vous ne l’avez pas déjà fait, clonez ce référentiel localement :
    1. Ouvrez une ligne de commande/un terminal.
    1. Exécutez la commande suivante :
       ```bash
       md .\DP3021Lab
       git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git .\DP3021Lab
       ```
       > REMARQUE
       > 
       > Si **Git** n’est pas installé, [téléchargez et installez l’application ***Git***](https://git-scm.com/download) et réessayez d’exécuter les commandes précédentes.
1. Vous avez installé Azure Data Studio. Si ce n’est pas le cas, [téléchargez et installez ***Azure Data Studio***](https://go.microsoft.com/fwlink/?linkid=2282284).
1. Installez l’extension **PostgreSQL** dans Azure Data Studio.

## Configurer l’éditeur

1. Ouvrez Azure Data Studio et connectez-vous au serveur d’édition. (Copiez le nom du serveur dans la section Vue d’ensemble.)
1. Sélectionnez **Fichier**, **Ouvrir le fichier**, puis accédez au dossier dans lequel vous avez enregistré les scripts.
1. Ouvrez le script **../Allfiles/Labs/06/Lab6_Replication.sql** et connectez-vous au serveur.
1. Mettez en surbrillance et exécutez la section **Accorder l’autorisation de réplication de l’utilisateur administrateur**.
1. Mettez en surbrillance et exécutez la section **Créer une base de données zoodb**.
1. Sélectionnez zoodb comme base de données actuelle à l’aide de la liste déroulante dans la barre d’outils. Vérifiez que zoodb est la base de données actuelle en exécutant l’instruction **SELECT**.
1. Mettez en surbrillance et exécutez la section **Créer des tables** et des **contraintes de clé étrangère** dans zoodb.
1. Mettez en surbrillance et exécutez la section **Remplir les tables dans zoodb**.
1. Mettez en surbrillance et exécutez la section **Créer une publication**. Lorsque vous exécutez l’instruction SELECT, elle ne répertorie rien, car la réplication n’est pas encore active.

## Configurer l’abonné

1. Ouvrez une deuxième instance d’Azure Data Studio et connectez-vous au serveur abonné.
1. Sélectionnez **Fichier**, **Ouvrir le fichier**, puis accédez au dossier dans lequel vous avez enregistré les scripts.
1. Ouvrez le script **../Allfiles/Labs/06/Lab6_Replication.sql** et connectez-vous au serveur abonné. (Copiez le nom du serveur dans la section Vue d’ensemble.)
1. Mettez en surbrillance et exécutez la section **Accorder l’autorisation de réplication de l’utilisateur administrateur**.
1. Mettez en surbrillance et exécutez la section **Créer une base de données zoodb**.
1. Sélectionnez zoodb comme base de données actuelle à l’aide de la liste déroulante dans la barre d’outils. Vérifiez que zoodb est la base de données actuelle en exécutant l’instruction **SELECT**.
1. Mettez en surbrillance et exécutez la section **Créer des tables** et des **contraintes de clé étrangère** dans zoodb.
1. Faites défiler jusqu’à la section **Créer un abonnement**.
    1. Modifiez l’instruction **CREATE SUBSCRIPTION** afin qu’elle ait le nom correct du serveur de publication et le mot de passe fort de l’éditeur. Mettez en surbrillance et exécutez l’instruction.
    1. Mettez en surbrillance et exécutez l’instruction **SELECT**. Cela montre l’abonnement « sub » que vous avez créé.
1. Sous la section **Afficher les tables**, mettez en surbrillance et exécutez chaque instruction **SELECT**. Les tables ont été remplies par réplication à partir du serveur éditeur.

## Apporter des modifications à la base de données de l’éditeur

- Dans la première instance d’Azure Data Studio (*votre instance d’éditeur*), dans **Insérer plus d’animaux**, mettez en surbrillance et exécutez l’instruction **INSERT**. *Veillez à **ne pas** exécuter cette instruction INSERT au niveau de l’abonné*.

## Afficher les modifications apportées à la base de données de l’abonné

- Dans la deuxième instance d’Azure Data Studio (abonné), sous **Afficher les tables d’animaux**, mettez en surbrillance et exécutez l’instruction **SELECT**.

Vous avez maintenant créé deux serveurs flexibles Azure Database pour PostgreSQL et configuré l’un en tant qu’éditeur, et l’autre en tant qu’abonné. Dans la base de données d’éditeur, vous avez créé et renseigné la base de données de zoo. Dans la base de données d’abonné, vous avez créé une base de données vide, qui a ensuite été remplie par la réplication de diffusion en continu.

## Nettoyage

1. Une fois l’exercice terminé, supprimez le groupe de ressources contenant les deux serveurs. Vous serez facturé pour les serveurs, sauf si vous les arrêtez ou supprimez-les.
1. Si nécessaire, supprimez le dossier .\DP3021Lab.
