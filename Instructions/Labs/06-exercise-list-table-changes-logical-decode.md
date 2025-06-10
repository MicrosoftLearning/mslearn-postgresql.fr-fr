---
lab:
  title: Répertorier les modifications apportées à la table avec le décodage logique
  module: Understand write-ahead logging
---

# Répertorier les modifications apportées à la table avec le décodage logique

Dans cet exercice, vous allez configurer la réplication logique, native dans PostgreSQL. Vous créez deux serveurs qui agissent en tant qu’éditeur et abonné. Les données dans zoodb sont répliquées entre les deux.

## Avant de commencer

Pour effectuer cet exercice, vous avez besoin de votre propre abonnement Azure. Si vous n’avez pas d’abonnement Azure, vous pouvez demander un [essai gratuit d’Azure](https://azure.microsoft.com/free).

De plus, les composants suivants doivent être installés sur votre ordinateur :

- Visual Studio Code.
- Extension Microsoft Visual Studio Code Postgres
- Azure CLI.
- Git.

## Créer l’environnement de l’exercice

Dans ces exercices et les suivants, vous utilisez un script Bicep pour déployer la base de données Azure Database pour PostgreSQL - Serveur flexible et d’autres ressources dans votre abonnement Azure. Les scripts Bicep se trouvent dans le dossier `/Allfiles/Labs/Shared` du référentiel GitHub que vous avez cloné précédemment.

### Télécharger et installer Visual Studio Code et l’extension PostgreSQL

Si Visual Studio Code n’est pas installé :

1. Dans un navigateur, accédez à [Télécharger Visual Studio Code](https://code.visualstudio.com/download) et sélectionnez la version appropriée pour votre système d’exploitation.

1. Suivez les instructions d’installation pour votre système d’exploitation.

1. Ouvrez Visual Studio Code.

1. Dans le menu de gauche, sélectionnez **Extensions** pour afficher le panneau Extensions.

1. Dans la barre de recherche, entrez **PostgreSQL**. L’icône de l’extension PostgreSQL pour Visual Studio Code s’affiche. Veillez à sélectionner celle de Microsoft.

1. Sélectionnez **Installer**. L’extension s’installe.

### Télécharger et installer Azure CLI et Git

Si Azure CLI ou Git ne sont pas installés :

1. Dans un navigateur, accédez à [Installer Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) et suivez les instructions pour votre système d’exploitation.

1. Dans un navigateur, accédez à [Télécharger et installer Git](https://git-scm.com/downloads) et suivez les instructions pour votre système d’exploitation.

### Télécharger les fichiers d’exercice

Si vous avez déjà cloné le référentiel GitHub contenant les fichiers d’exercice, *ignorez le téléchargement des fichiers d’exercice*.

Pour télécharger les fichiers d’exercice, vous clonez le référentiel GitHub contenant les fichiers d’exercice sur votre ordinateur local. Le référentiel contient tous les scripts et ressources dont vous avez besoin pour effectuer cet exercice.

1. Ouvrez Visual Studio Code s’il n’est pas déjà ouvert.

1. Sélectionnez **Afficher toutes les commandes** (Ctrl+Maj+P) pour ouvrir la palette de commandes.

1. Dans la palette de commandes, recherchez et sélectionnez **Git : cloner**.

1. Dans la palette de commandes, entrez les éléments suivants pour cloner le référentiel GitHub contenant des ressources d’exercice, puis appuyez sur **Entrée** :

    ```bash
    https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

1. Suivez les invites pour sélectionner un dossier dans lequel cloner le référentiel. Le référentiel est cloné dans un dossier nommé `mslearn-postgresql` à l’emplacement que vous avez sélectionné.

1. Quand il vous est demandé si vous voulez ouvrir le dépôt cloné, sélectionnez **Ouvrir**. Le référentiel s’ouvre dans Visual Studio Code.

## Créer un groupe de ressources

Dans cette section, vous allez créer un groupe de ressources pour contenir les serveurs Azure Database pour PostgreSQL. Un groupe de ressources est un conteneur logique qui contient les ressources associées d’une solution Azure.

1. Connectez-vous au portail Azure. Votre compte d’utilisateur doit être propriétaire ou contributeur pour l’abonnement Azure.

1. Sélectionnez **Groupes de ressources**, puis **+ Créer**.

1. Sélectionnez votre abonnement.

1. Dans le groupe de ressources, entrez **rg-PostgreSQL_Replication**.

1. Sélectionnez une région proche de votre emplacement.

1. Sélectionnez **Revoir + créer**.

1. Sélectionnez **Créer**.

## Créer un serveur de publication

Dans cette section, vous allez créer le serveur de publication. Le serveur de publication est la source des données à répliquer vers le serveur abonné.

1. Sous Services Azure, sélectionnez **+Créer une ressource**. Sous **Catégories**, sélectionnez **Bases de données**. Sous **Azure Database pour PostgreSQL**, sélectionnez **Créer**.

1. Sous l’onglet **Général** du serveur flexible, renseignez chaque champ comme suit :
    - **Abonnement** : votre abonnement.
    - **Groupe de ressources** - Sélectionnez **rg-PostgreSQL_Replication**.
    - **Nom du serveur** - *psql-postgresql-pub9999* (le nom doit être globalement unique, remplacez donc 9999 par quatre chiffres aléatoires).
    - **Région** - sélectionnez la même région que le groupe de ressources.
    - **Version de PostgreSQL** - Sélectionnez 16.
    - **Type de charge de travail** - *Développement*.
    - **Calcul + stockage** - *Burstable*. Sélectionnez **Configurer le serveur** et examinez les options de configuration. N’apportez aucune modification et fermez la section.
    - **Zone de disponibilité** - 1. Si les zones de disponibilité ne sont pas prises en charge, laissez Aucune préférence.
    - **Haute disponibilité** - Désactivée.
    - **Méthode d’authentification** - Authentification PostgreSQL uniquement.
    - Dans **Nom d’utilisateur administrateur**, entrez **`pgAdmin`**.
    - Dans **Mot de passe**, entrez un mot de passe complexe approprié.

1. Sélectionnez **Suivant : Réseau >**.

1. Sous l’onglet **Mise en réseau** du serveur flexible, renseignez chaque champ comme suit :
    - **Méthode de connectivité** : (o) Accès public (adresses IP autorisées).
    - **Autoriser l’accès public à partir d’un service Azure dans Azure à ce serveur** : coché. Cela doit être coché, afin que les bases de données de publication et abonnée puissent communiquer entre elles.
    - **Règles de pare-feu** : sélectionnez **+ Ajouter l’adresse IP client actuelle**. Cette option ajoute votre adresse IP actuelle en tant que règle de pare-feu. Vous pouvez éventuellement nommer cette règle de pare-feu en quelque chose de significatif.

1. Sélectionnez **Revoir + créer**. Sélectionnez ensuite **Créer**.

1. Étant donné que la création d’un serveur Azure Database pour PostgreSQL peut prendre quelques minutes, commencez par l’étape suivante dès que ce déploiement est en cours. N’oubliez pas d’ouvrir une nouvelle fenêtre ou un nouvel onglet de navigateur pour continuer.

## Créer un serveur abonné

1. Sous Services Azure, sélectionnez **+Créer une ressource**. Sous **Catégories**, sélectionnez **Bases de données**. Sous **Azure Database pour PostgreSQL**, sélectionnez **Créer**.

1. Sous l’onglet **Général** du serveur flexible, renseignez chaque champ comme suit :
    - **Abonnement** : votre abonnement.
    - **Groupe de ressources** - Sélectionnez **rg-PostgreSQL_Replication**.
    - **Nom du serveur** - *psql-postgresql-sub9999* (le nom doit être unique au niveau mondial, remplacez donc 9999 par quatre chiffres aléatoires).
    - **Région** - sélectionnez la même région que le groupe de ressources.
    - **Version de PostgreSQL** - Sélectionnez 16.
    - **Type de charge de travail** - *Développement*.
    - **Calcul + stockage** - *Burstable*. Sélectionnez **Configurer le serveur** et examinez les options de configuration. N’apportez aucune modification et fermez la section.
    -  **Zone de disponibilité** - 2. Si les zones de disponibilité ne sont pas prises en charge, laissez Aucune préférence.
    - **Haute disponibilité** - Désactivée.
    - **Méthode d’authentification** - Authentification PostgreSQL uniquement.
    - Dans **Nom d’utilisateur administrateur**, entrez **`pgAdmin`**.
    - Dans **Mot de passe**, entrez un mot de passe complexe approprié.

1. Sélectionnez **Suivant : Réseau >**.

1. Sous l’onglet **Mise en réseau** du serveur flexible, renseignez chaque champ comme suit :
    - **Méthode de connectivité** : (o) Accès public (adresses IP autorisées).
    - **Autoriser l’accès public à partir d’un service Azure dans Azure à ce serveur** : coché. Cela doit être coché, afin que les bases de données de publication et abonnée puissent communiquer entre elles.
    - **Règles de pare-feu** : sélectionnez **+ Ajouter l’adresse IP client actuelle**. Cette option ajoute votre adresse IP actuelle en tant que règle de pare-feu. Vous pouvez éventuellement nommer cette règle de pare-feu en quelque chose de significatif.

1. Sélectionnez **Revoir + créer**. Sélectionnez ensuite **Créer**.

1. Attendez que les deux serveurs Azure Database pour PostgreSQL soient déployés.

## Configurer la réplication

Pour les serveurs éditeur *et* abonné :

1. Dans le portail Azure, accédez au serveur, puis, sous Paramètres, sélectionnez **Paramètres du serveur**.

1. À l’aide de la barre de recherche, recherchez chaque paramètre et apportez les modifications suivantes :
    - `wal_level` = LOGICAL
    - `max_worker_processes` = 24

1. Cliquez sur **Enregistrer**. Sélectionnez ensuite **Enregistrer et redémarrer**.

1. Attendez que les deux serveurs redémarrent.

    > Une fois les serveurs redéployés, vous devrez peut-être actualiser vos fenêtres de navigateur pour constater que les serveurs ont redémarré.

## Configurer l’éditeur

Dans cette section, vous allez configurer le serveur de publication. Le serveur de publication est la source des données à répliquer vers le serveur abonné.

1. Ouvrez la première instance de Visual Studio Code pour vous connecter au serveur de publication.

1. Ouvrez le dossier dans lequel vous avez cloné le référentiel GitHub.

1. Sélectionnez l’icône **PostgreSQL** dans le menu de gauche.

    > &#128221; Si vous ne voyez pas l’icône PostgreSQL, sélectionnez l’icône **Extensions** et recherchez **PostgreSQL**. Sélectionnez l’extension **PostgreSQL** Microsoft, puis sélectionnez **Installer**.

1. Si vous avez déjà créé une connexion à votre serveur de *publication* PostgreSQL, passez à l’étape suivante. Pour créer une connexion :

    1. Dans l’extension **PostgreSQL**, sélectionnez **+ Ajouter une connexion** pour ajouter une nouvelle connexion.

    1. Dans la boîte de dialogue **NOUVELLE CONNEXION**, entrez les informations suivantes :

        - **Nom de serveur** : `<your-publisher-server-name>`.postgres.database.azure.com
        - **Type d’authentification** : Mot de passe
        - **Nom d’utilisateur ou d’utilisatrice** : pgAdmin
        - **Mot de passe** : mot de passe aléatoire que vous avez généré précédemment.
        - Cochez la case **Enregistrer le mot de passe**.
        - **Nom de la connexion** : `<your-publisher-server-name>`

    1. Sélectionnez **Tester la connexion** pour tester la connexion. Si la connexion réussit, sélectionnez **Enregistrer et se connecter** pour enregistrer la connexion, sinon passez en revue les informations de connexion, puis réessayez.

1. Sla connexion n’est pas déjà effectuée, sélectionnez **Se connecter** pour votre serveur PostgreSQL. Votre connexion au serveur Azure Database pour PostgreSQL est effective.

1. Développez le nœud serveur et ses bases de données. Les bases de données existantes sont répertoriées.

1. Dans Visual Studio Code, sélectionnez **Fichier**, **Ouvrir le fichier**, puis accédez au dossier dans lequel vous avez enregistré les scripts. Sélectionnez **../Allfiles/Labs/06/Lab6_Replication.sql**, puis **Ouvrir**.

1. En bas à droite de Visual Studio Code, vérifiez que la connexion est verte. Si ce n’est pas le cas, un message **PGSQL déconnecté** doit être affiché. Sélectionnez le texte **PGSQL déconnecté**, puis sélectionnez votre connexion de serveur PostgreSQL dans la liste dans la palette de commandes. Si un mot de passe est demandé, entrez le mot de passe que vous avez généré précédemment.

1. Nous allons maintenant configurer le serveur de *publication*.

    1. Mettez en surbrillance et exécutez la section **Accorder l’autorisation de réplication de l’utilisateur administrateur**.

    1. Mettez en surbrillance et exécutez la section **Créer une base de données zoodb**.

    1. Si vous mettez en surbrillance uniquement l’instruction **SELECT current_database()** et que vous l’exécutez, vous remarquez que la base de données est actuellement définie sur `postgres`. Vous devez remplacer cette valeur par `zoodb`.

    1. Sélectionnez les points de suspension dans la barre de menus avec l’icône d’*exécution* et sélectionnez **Modifier la base de données PostgreSQL**. Sélectionnez `zoodb` dans la liste des bases de données.

        > &#128221; Vous pouvez également modifier la base de données dans le volet de requête. Vous pouvez noter le nom du serveur et le nom de la base de données sous l’onglet requête lui-même. Sélectionner le nom de la base de données affiche une liste des bases de données. Sélectionnez la base de données `zoodb` dans la liste.

    1. Mettez en surbrillance et exécutez la section **Créer des tables** et des **contraintes de clé étrangère** dans zoodb.

    1. Mettez en surbrillance et exécutez la section **Remplir les tables dans zoodb**.

    1. Mettez en surbrillance et exécutez la section **Créer une publication**. Lorsque vous exécutez l’instruction SELECT, elle ne répertorie rien, car la réplication n’est pas encore active.

    1. N’exécutez PAS la section **CREATE SUBSCRIPTION**. Ce script est exécuté sur le serveur abonné.

    1. Ne fermez PAS cette instance de Visual Studio Code, réduisez-la. Vous y reviendrez une fois que vous aurez configuré le serveur abonné.

Vous avez maintenant créé le serveur de publication et la base de données zoodb. La base de données contient les tables et les données répliquées sur le serveur abonné.

## Configurer l’abonné

Dans cette section, vous allez configurer le serveur abonné. Le serveur abonné est la destination des données répliquées à partir du serveur de publication. Vous créez une base de données sur le serveur abonné, qui est remplie avec les données du serveur de publication.

1. Ouvrez une *deuxième instance* de Visual Studio Code et connectez-vous au serveur abonné.

1. Ouvrez le dossier dans lequel vous avez cloné le référentiel GitHub.

1. Sélectionnez l’icône **PostgreSQL** dans le menu de gauche.

    > &#128221; Si vous ne voyez pas l’icône PostgreSQL, sélectionnez l’icône **Extensions** et recherchez **PostgreSQL**. Sélectionnez l’extension **PostgreSQL** Microsoft, puis sélectionnez **Installer**.

1. Si vous avez déjà créé une connexion à votre serveur *abonné* PostgreSQL, passez à l’étape suivante. Pour créer une connexion :

    1. Dans l’extension **PostgreSQL**, sélectionnez **+ Ajouter une connexion** pour ajouter une nouvelle connexion.

    1. Dans la boîte de dialogue **NOUVELLE CONNEXION**, entrez les informations suivantes :

        - **Nom de serveur** : `<your-subscriber-server-name>`.postgres.database.azure.com
        - **Type d’authentification** : Mot de passe
        - **Nom d’utilisateur ou d’utilisatrice** : pgAdmin
        - **Mot de passe** : mot de passe aléatoire que vous avez généré précédemment.
        - Cochez la case **Enregistrer le mot de passe**.
        - **Nom de la connexion** : `<your-subscriber-server-name>`

    1. Sélectionnez **Tester la connexion** pour tester la connexion. Si la connexion réussit, sélectionnez **Enregistrer et se connecter** pour enregistrer la connexion, sinon passez en revue les informations de connexion, puis réessayez.

1. Sla connexion n’est pas déjà effectuée, sélectionnez **Se connecter** pour votre serveur PostgreSQL. Votre connexion au serveur Azure Database pour PostgreSQL est effective.

1. Développez le nœud serveur et ses bases de données. Les bases de données existantes sont répertoriées.

1. Dans Visual Studio Code, sélectionnez **Fichier**, **Ouvrir le fichier**, puis accédez au dossier dans lequel vous avez enregistré les scripts. Sélectionnez **../Allfiles/Labs/06/Lab6_Replication.sql**, puis **Ouvrir**.

1. En bas à droite de Visual Studio Code, vérifiez que la connexion est verte. Si ce n’est pas le cas, un message **PGSQL déconnecté** doit être affiché. Sélectionnez le texte **PGSQL déconnecté**, puis sélectionnez votre connexion de serveur PostgreSQL dans la liste dans la palette de commandes. Si un mot de passe est demandé, entrez le mot de passe que vous avez généré précédemment.

1. Nous allons maintenant configurer le serveur *abonné*.

    1. Mettez en surbrillance et exécutez la section **Accorder l’autorisation de réplication de l’utilisateur administrateur**.

    1. Mettez en surbrillance et exécutez la section **Créer une base de données zoodb**.

    1. Si vous mettez en surbrillance uniquement l’instruction **SELECT current_database()** et que vous l’exécutez, vous remarquez que la base de données est actuellement définie sur `postgres`. Vous devez remplacer cette valeur par `zoodb`.

    1. Sélectionnez les points de suspension dans la barre de menus avec l’icône d’*exécution* et sélectionnez **Modifier la base de données PostgreSQL**. Sélectionnez `zoodb` dans la liste des bases de données.

        > &#128221; Vous pouvez également modifier la base de données dans le volet de requête. Vous pouvez noter le nom du serveur et le nom de la base de données sous l’onglet requête lui-même. Sélectionner le nom de la base de données affiche une liste des bases de données. Sélectionnez la base de données `zoodb` dans la liste.

    1. Mettez en surbrillance et exécutez la section **Créer des tables** et des **contraintes de clé étrangère** dans `zoodb`.

    1. N’exécutez PAS la section **Créer une publication**, cette instruction a déjà été exécutée sur le serveur de publication.

    1. Faites défiler jusqu’à la section **Créer un abonnement**.

        1. Modifiez l’instruction **CREATE SUBSCRIPTION** afin qu’elle ait le nom correct du serveur de publication et le mot de passe fort de l’éditeur. Mettez en surbrillance et exécutez l’instruction.

        1. Mettez en surbrillance et exécutez l’instruction **SELECT**. Cela montre l’abonnement « sub » précédemment créé.

    1. Dans la section **Afficher les tables**, mettez en surbrillance et exécutez chaque instruction **SELECT**. Le serveur de publication a rempli ces tables à l’aide de la réplication.

Vous avez créé le serveur abonné et la base de données zoodb. La base de données contient les tables et les données qui ont été répliquées à partir du serveur de publication.

## Apporter des modifications à la base de données de l’éditeur

- Dans la première instance de Visual Studio Code (*votre instance de publication*), dans **Insérer plus d’animaux**, mettez en surbrillance et exécutez l’instruction **INSERT**. *Veillez à **ne pas** exécuter cette instruction INSERT au niveau de l’abonné*.

## Afficher les modifications apportées à la base de données de l’abonné

- Dans la deuxième instance de Visual Studio Code (abonné), dans **Afficher les tables d’animaux**, mettez en surbrillance et exécutez l’instruction **SELECT**.

Vous avez maintenant créé deux serveurs flexibles Azure Database pour PostgreSQL et configuré l’un en tant que serveur de publication, et l’autre en tant qu’abonné. Dans la base de données du serveur de publication, vous avez créé et rempli la base de données zoo. Dans la base de données d’abonné, vous avez créé une base de données vide, qui a ensuite été remplie par la réplication de diffusion en continu.

## Nettoyage

1. Si vous n’avez plus besoin de ces serveurs PostgreSQL pour d’autres exercices, pour éviter d’entraîner des coûts Azure inutiles, supprimez le groupe de ressources créé au cours de cet exercice.

1. Si vous souhaitez laisser les serveurs PostgreSQL en cours d’exécution, vous pouvez le faire. Sinon, vous pouvez arrêter les serveurs pour éviter d’entraîner des coûts inutiles via le terminal Bash. Pour arrêter les serveurs, exécutez la commande suivante pour chaque serveur :

    ```bash

    ```azurecli
    az postgres flexible-server stop --name <your-server-name> --resource-group $RG_NAME
    ```

    Remplacez `<your-server-name>` par le nom de vos serveurs PostreSQL.

    > &#128221; Vous pouvez également arrêter le serveur depuis le portail Azure. Dans le portail Azure, accédez aux **groupes de ressources** et sélectionnez le groupe de ressources que vous avez créé précédemment. Sélectionnez le serveur PostgreSQL, puis sélectionnez **Arrêter** dans le menu. Effectuez ceci pour le serveur de publication et le serveur abonné.

1. Si nécessaire, supprimez le référentiel Git que vous avez cloné précédemment.

Vous avez correctement créé un serveur PostgreSQL et l’avez configuré pour la réplication logique. Vous avez créé un serveur de publication et un serveur abonné et vous avez configuré la réplication entre eux. Vous avez également apporté des modifications à la base de données de publication et consulté les modifications dans la base de données abonnée. Vous avez maintenant acquis une bonne compréhension de la configuration de la réplication logique dans PostgreSQL.
