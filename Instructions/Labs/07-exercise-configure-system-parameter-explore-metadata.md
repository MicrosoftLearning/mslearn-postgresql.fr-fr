---
lab:
  title: Configurer les paramètres système et explorer les métadonnées avec des catalogues et des vues système
  module: Configure and manage Azure Database for PostgreSQL
---

# Configurer les paramètres système et explorer les métadonnées avec des catalogues et des vues système

Dans cet exercice, vous examinez les paramètres et les métadonnées système dans PostgreSQL.

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

### Déployer des ressources dans votre abonnement Azure

Si vos ressources Azure sont déjà installées, *ignorez le déploiement des ressources*.

Cette étape vous guide tout au long de l’utilisation de commandes Azure CLI à partir de Visual Studio Code pour créer un groupe de ressources et exécuter un script Bicep pour déployer les services Azure nécessaires pour effectuer cet exercice dans votre abonnement Azure.

> &#128221; Si vous effectuez plusieurs modules dans ce parcours d’apprentissage, vous pouvez partager l’environnement Azure entre eux. Dans ce cas, vous devez effectuer cette étape de déploiement de ressources une seule fois.

1. Ouvrez Visual Studio Code s’il n’est pas déjà ouvert et ouvrez le dossier du référentiel GitHun cloné.

1. Développez le dossier **mslearn-postgresql** dans le volet Explorateur.

1. Développez le dossier **Allfiles/Labs/Shared**.

1. Cliquez avec le bouton droit sur le dossier **Allfiles/Labs/Shared**, puis sélectionnez **Ouvrir dans le terminal intégré**. Cette sélection ouvre une fenêtre de terminal dans la fenêtre Visual Studio Code.

1. Le terminal peut ouvrir une fenêtre **PowerShell** par défaut. Pour cette section du labo, vous souhaitez utiliser l’**interpréteur de commandes Bash**. En plus de l’icône **+**, il existe une flèche déroulante. Sélectionnez-la et sélectionnez **Git Bash** ou **Bash** dans la liste des profils disponibles. Cette sélection ouvre une nouvelle fenêtre de terminal avec l’**interpréteur de commandes Bash**.

    > &#128221; Vous pouvez fermer la fenêtre de terminal **PowerShell** si vous le souhaitez, mais ce n’est pas nécessaire. Vous pouvez ouvrir plusieurs fenêtres de terminal en même temps.

1. Dans la fenêtre de terminal, exécutez la commande suivante pour vous connecter à votre compte Azure :

    ```bash
    az login
    ```

    Ce script ouvre une nouvelle fenêtre de navigateur permettant de vous connecter à votre compte Azure. Après votre connexion, revenez à la fenêtre de terminal.

1. Exécutez ensuite trois commandes pour définir des variables afin de réduire les saisies redondantes lors de l’utilisation des commandes Azure CLI pour créer des ressources Azure. Les variables représentent le nom à affecter à votre groupe de ressources (`RG_NAME`), la région Azure (`REGION`) dans laquelle les ressources seront déployées et un mot de passe généré de manière aléatoire pour la connexion de l’administrateur ou de l’administratrice PostgreSQL (`ADMIN_PASSWORD`).

    Dans la première commande, la région affectée à la variable correspondante est `eastus`, mais vous pouvez également la remplacer par un emplacement de votre choix.

    ```bash
    REGION=eastus
    ```

    La commande suivante attribue le nom à utiliser pour le groupe de ressources qui hébergera toutes les ressources utilisées dans cet exercice. Le nom du groupe de ressources affecté à la variable correspondante est `rg-learn-work-with-postgresql-$REGION`, où `$REGION` est l’emplacement que vous avez spécifié précédemment. *Toutefois, vous pouvez le remplacer par tout autre nom de groupe de ressources de votre choix ou dont vous disposez déjà*.

    ```bash
    RG_NAME=rg-learn-work-with-postgresql-$REGION
    ```

    La commande finale génère de façon aléatoire un mot de passe pour la connexion d’administrateur ou d’administratrice PostgreSQL. Copiez-le dans un endroit sécurisé pour pouvoir l’utiliser ultérieurement et vous connecter à votre serveur flexible PostgreSQL.

    ```bash
    #!/bin/bash
    
    # Define array of allowed characters explicitly
    chars=( {a..z} {A..Z} {0..9} '!' '@' '#' '$' '%' '^' '&' '*' '(' ')' '_' '+' )
    
    a=()
    for ((i = 0; i < 100; i++)); do
        rand_char=${chars[$RANDOM % ${#chars[@]}]}
        a+=("$rand_char")
    done
    
    # Join first 18 characters without delimiter
    ADMIN_PASSWORD=$(IFS=; echo "${a[*]:0:18}")
    
    echo "Your randomly generated PostgreSQL admin user's password is:"
    echo "$ADMIN_PASSWORD"
    echo "Please copy it to a safe place, as you will need it later to connect to your PostgreSQL flexible server."
    ```

1. (Ignorez si vous utilisez votre abonnement par défaut.) Si vous avez accès à plusieurs abonnements Azure et que votre abonnement par défaut *n’est pas* celui dans lequel vous souhaitez créer le groupe de ressources et d’autres ressources pour cet exercice, exécutez cette commande pour définir l’abonnement approprié, en remplaçant le jeton `<subscriptionName|subscriptionId>` par le nom ou l’ID de l’abonnement que vous souhaitez utiliser :

    ```azurecli
    az account set --subscription 16b3c013-d300-468d-ac64-7eda0820b6d3
    ```

1. (Ignorez si vous utilisez un groupe de ressources existant.) Exécutez la commande Azure CLI suivante pour créer votre groupe de ressources :

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

1. Enfin, utilisez Azure CLI pour exécuter un script de déploiement Bicep afin d’approvisionner des ressources Azure dans votre groupe de ressources :

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Le script de déploiement Bicep approvisionne les services Azure requis pour effectuer cet exercice dans votre groupe de ressources. Les ressources déployées sont un serveur flexible Azure Database pour PostgreSQL. Le script Bicep crée également une base de données qui peut être configurée sur la ligne de commande en tant que paramètre.

    Le déploiement prend généralement plusieurs minutes. Vous pouvez le surveiller à partir du terminal Bash ou accéder à la page **Déploiements** du groupe de ressources que vous avez créé ci-dessus et observer la progression du déploiement.

1. Étant donné que le script crée un nom aléatoire pour le serveur PostgreSQL, vous pouvez trouver le nom du serveur en exécutant la commande suivante :

    ```azurecli
    az postgres flexible-server list --query "[].{Name:name, ResourceGroup:resourceGroup, Location:location}" --output table
    ```

    Notez le nom du serveur, car vous en avez besoin pour vous connecter au serveur plus loin dans cet exercice.

    > &#128221; Vous pouvez également trouver le nom du serveur dans le portail Azure. Dans le portail Azure, accédez aux **groupes de ressources** et sélectionnez le groupe de ressources que vous avez créé précédemment. Le serveur PostgreSQL est répertorié dans le groupe de ressources.

### Résoudre les erreurs de déploiement

Vous pouvez rencontrer quelques erreurs lors de l’exécution du script de déploiement Bicep. Les messages les plus courants et les étapes à suivre pour les résoudre sont les suivants :

- Si vous avez précédemment exécuté le script de déploiement Bicep pour ce parcours d’apprentissage et que vous avez supprimé par la suite les ressources, vous pouvez recevoir un message d’erreur semblable à ce qui suit si vous tentez de réexécuter le script dans les 48 heures suivant la suppression des ressources :

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is '4e87a33d-a0ac-4aec-88d8-177b04c1d752'. See inner errors for details."}
    
    Inner Errors:
    {"code": "FlagMustBeSetForRestore", "message": "An existing resource with ID '/subscriptions/{subscriptionId}/resourceGroups/rg-learn-postgresql-ai-eastus/providers/Microsoft.CognitiveServices/accounts/{accountName}' has been soft-deleted. To restore the resource, you must specify 'restore' to be 'true' in the property. If you don't want to restore existing resource, please purge it first."}
    ```

    Si vous recevez ce message, modifiez la commande `azure deployment group create` ci-dessus pour que le paramètre `restore` soit défini sur `true` et réexécutez-la.

- Si la région sélectionnée est limitée à l’approvisionnement de ressources spécifiques, vous devez définir la variable `REGION` à un autre emplacement et réexécuter les commandes pour créer le groupe de ressources et exécuter le script de déploiement Bicep.

    ```bash
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- Si le labo nécessite des ressources IA, vous pouvez obtenir l’erreur suivante. Cette erreur se produit lorsque le script ne parvient pas à créer une ressource IA en raison de la nécessité d’accepter le contrat IA responsable. Si c’est le cas, utilisez l’interface d’utilisation du portail Azure pour créer une ressource Azure AI Services, puis réexécutez le script de déploiement.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## Connecter l’extension PostreSQL dans Visual Studio Code

Dans cette section, vous vous connectez au serveur PostgreSQL à l’aide de l’extension PostgreSQL dans Visual Studio Code. Vous utilisez l’extension PostgreSQL pour exécuter des scripts SQL sur le serveur PostgreSQL.

1. Ouvrez Visual Studio Code s’il n’est pas déjà ouvert et ouvrez le dossier dans lequel vous avez cloné le référentiel GitHub.

1. Sélectionnez l’icône **PostgreSQL** dans le menu de gauche.

    > &#128221; Si vous ne voyez pas l’icône PostgreSQL, sélectionnez l’icône **Extensions** et recherchez **PostgreSQL**. Sélectionnez l’extension **PostgreSQL** Microsoft, puis sélectionnez **Installer**.

1. Si vous avez déjà créé une connexion à votre serveur PostgreSQL, passez à l’étape suivante. Pour créer une connexion :

    1. Dans l’extension **PostgreSQL**, sélectionnez **+ Ajouter une connexion** pour ajouter une nouvelle connexion.

    1. Dans la boîte de dialogue **NOUVELLE CONNEXION**, entrez les informations suivantes :

        - **Nom de serveur** : `<your-server-name>`.postgres.database.azure.com
        - **Type d’authentification** : Mot de passe
        - **Nom d’utilisateur ou d’utilisatrice** : pgAdmin
        - **Mot de passe** : mot de passe aléatoire que vous avez généré précédemment.
        - Cochez la case **Enregistrer le mot de passe**.
        - **Nom de la connexion** : `<your-server-name>`

    1. Sélectionnez **Tester la connexion** pour tester la connexion. Si la connexion réussit, sélectionnez **Enregistrer et se connecter** pour enregistrer la connexion, sinon passez en revue les informations de connexion, puis réessayez.

1. Sla connexion n’est pas déjà effectuée, sélectionnez **Se connecter** pour votre serveur PostgreSQL. Votre connexion au serveur Azure Database pour PostgreSQL est effective.

1. Développez le nœud serveur et ses bases de données. Les bases de données existantes sont répertoriées.

1. Si vous n’avez pas encore créé la base de données zoodb, sélectionnez **Fichier**, **Ouvrir un fichier**, puis accédez au dossier dans lequel vous avez enregistré les scripts. Sélectionnez **../Allfiles/Labs/02/Lab2_ZooDb.sql** et **Ouvri**.

1. En bas à droite de Visual Studio Code, vérifiez que la connexion est verte. Si ce n’est pas le cas, un message **PGSQL déconnecté** doit être affiché. Sélectionnez le texte **PGSQL déconnecté**, puis sélectionnez votre connexion de serveur PostgreSQL dans la liste dans la palette de commandes. Si un mot de passe est demandé, entrez le mot de passe que vous avez généré précédemment.

1. C’est le moment de créer la base de données.

    1. Mettez en surbrillance les instructions **DROP** et **CREATE**, puis exécutez-les.

    1. Si vous mettez en surbrillance uniquement l’instruction **SELECT current_database()** et que vous l’exécutez, vous remarquez que la base de données est actuellement définie sur `postgres`. Vous devez remplacer cette valeur par `zoodb`.

    1. Sélectionnez les points de suspension dans la barre de menus avec l’icône d’*exécution* et sélectionnez **Modifier la base de données PostgreSQL**. Sélectionnez `zoodb` dans la liste des bases de données.

        > &#128221; Vous pouvez également modifier la base de données dans le volet de requête. Vous pouvez noter le nom du serveur et le nom de la base de données sous l’onglet requête lui-même. Sélectionner le nom de la base de données affiche une liste des bases de données. Sélectionnez la base de données `zoodb` dans la liste.

    1. Réexécutez l’instruction **SELECT current_database()** pour confirmer que la base de données est maintenant définie sur `zoodb`.

    1. Mettez en surbrillance les sections **Créer des tables**, **Créer des clés étrangères** et **Remplir des tables** et exécutez-les.

    1. Mettez en surbrillance les 3 instructions **SELECT** à la fin du script et exécutez-les pour vérifier que les tables ont été créées et remplies.

## Tâche 1 : explorer le processus de nettoyage dans PostgreSQL

Dans cette section, vous allez explorer le processus de nettoyage dans PostgreSQL. Le processus de nettoyage est utilisé pour récupérer de l’espace de stockage et optimiser les performances de la base de données. Vous pouvez configurer le processus de nettoyage pour qu’il s’exécute automatiquement ou l’exécuter manuellement.

1. Dans la fenêtre Visual Studio Code, sélectionnez **Fichier**, **Ouvrir un fichier**, puis accédez aux scripts de labo. Sélectionnez **.. /Allfiles/Labs/07/Lab7_vacuum.sql**, puis sélectionnez **Ouvrir**. Si nécessaire, reconnectez-vous au serveur en sélectionnant le texte **PGSQLdéconnecté** puis en sélectionnant votre connexion de serveur PostgreSQL dans la liste dans la palette de commandes. Si un mot de passe est demandé, entrez le mot de passe que vous avez généré précédemment.

1. En bas à droite de Visual Studio Code, vérifiez que la connexion est verte. Si ce n’est pas le cas, un message **PGSQL déconnecté** doit être affiché. Sélectionnez le texte **PGSQL déconnecté**, puis sélectionnez votre connexion de serveur PostgreSQL dans la liste dans la palette de commandes. Si un mot de passe est demandé, entrez le mot de passe que vous avez généré précédemment.

1. Exécutez l’instruction **SELECT current_database()** pour vérifier votre base de données active. Vérifiez si la connexion est actuellement définie sur la base de données **zoodb**. Si ce n’est pas le cas, vous pouvez modifier la base de données en **zoodb**. Pour modifier la base de données, sélectionnez les points de suspension dans la barre de menus avec l’icône d’*exécution* et sélectionnez **Modifier la base de données PostgreSQL**. Sélectionnez `zoodb` dans la liste des bases de données. Vérifiez que la base de données est désormais définie sur `zoodb` en exécutant l’instruction **SELECT current_database();**.

1. Surlignez et exécutez la section **Afficher les tuples morts**. Cette requête affiche le nombre de tuples morts et vivants dans la base de données. Notez le nombre de tuples morts.

1. Surlignez et exécutez la section **Modifier le poids** dix fois de suite. Cette requête met à jour la colonne du poids pour tous les animaux.

1. Réexécutez la section sous **Afficher les tuples morts**. Notez le nombre de tuples morts une fois les mises à jour effectuées.

1. Exécutez la section sous **Exécuter manuellement VACUUM** pour exécuter le processus de nettoyage.

1. Réexécutez la section sous **Afficher les tuples morts**. Notez le nombre de tuples morts après l’exécution du processus de nettoyage.

## Tâche 2 : configurer les paramètres du serveur de nettoyage automatique

Dans cette section, vous allez configurer les paramètres serveur de nettoyage automatique. Le processus de nettoyage automatique est utilisé pour récupérer automatiquement l’espace de stockage et optimiser les performances de la base de données. Vous pouvez configurer le processus de nettoyage automatique pour qu’il s’exécute automatiquement en fonction de paramètres spécifiques.

1. Si ce n’est pas déjà fait, accédez au [portail Azure](https://portal.azure.com) et connectez-vous.

1. Dans le portail Azure, accédez à votre serveur flexible Azure Database pour PostgreSQL.

1. Sous **Paramètres**, sélectionnez **Paramètres du serveur**.

1. Dans la barre de recherche, tapez **`vacuum`**. Recherchez les paramètres suivants et modifiez les valeurs comme suit :

    - autovacuum = ON (La valeur doit être ON par défaut.)
    - autovacuum_vacuum_scale_factor = 0,1
    - autovacuum_vacuum_threshold = 50

    Ces modifications reviennent à exécuter le processus de nettoyage automatique quand 10 % d’une table comporte des lignes marquées pour suppression ou lorsque 50 lignes sont mises à jour ou supprimées dans une seule table.

1. Cliquez sur **Enregistrer**. Le serveur est redémarré.

## Tâche 3 : afficher les métadonnées PostgreSQL dans le portail Azure

Dans cette section, vous allez afficher les métadonnées PostgreSQL dans le portail Azure. Le portail Azure fournit une interface graphique permettant de gérer et de surveiller votre serveur PostgreSQL.

1. Si ce n’est pas déjà fait, accédez au [portail Azure](https://portal.azure.com) et connectez-vous.

1. Recherchez et sélectionnez **Azure Database pour PostgreSQL**.

1. Sélectionnez le serveur flexible Azure Database pour PostgreSQL que vous avez créé pour cet exercice.

1. Dans **Supervision**, sélectionnez **Métriques**.

1. Sélectionnez **Métrique**, puis **Pourcentage d’UC**.

1. Notez que vous pouvez afficher différentes métriques sur vos bases de données.

## Tâche 4 : afficher des données dans les tables de catalogue système

Dans cette section, vous allez afficher les données dans les tables de catalogue système. Les tables de catalogue système sont utilisées pour stocker des métadonnées sur les objets de base de données dans PostgreSQL. Vous pouvez interroger ces tables pour récupérer des informations sur les objets de base de données.

1. Ouvrez Visual Studio Code s’il n’est pas déjà ouvert.

1. Affichez la palette de commandes (Ctrl+Maj+P) et sélectionnez **PGSQL : nouvelle requête**. Sélectionnez la nouvelle connexion que vous avez créée dans la liste dans la palette de commandes. Si on vous demande un mot de passe, entrez le mot de passe que vous avez créé pour le nouveau rôle.

1. Dans la fenêtre **Nouvelle requête**, copiez, mettez en surbrillance et exécutez l’instruction SQL suivante :

    ```sql
    SELECT datname, xact_commit, xact_rollback FROM pg_stat_database;
    ```

1. Notez que vous pouvez afficher les validations et les restaurations pour chaque base de données.

## Afficher une requête de métadonnées complexe à l’aide d’une vue système

Dans cette section, vous allez afficher une requête de métadonnées complexe à l’aide d’une vue système. Les vues système sont utilisées afin de fournir une interface simplifiée pour interroger les métadonnées d’objets de base de données dans PostgreSQL. 

1. Ouvrez Visual Studio Code s’il n’est pas déjà ouvert.

1. Affichez la palette de commandes (Ctrl+Maj+P) et sélectionnez **PGSQL : nouvelle requête**. Sélectionnez la nouvelle connexion que vous avez créée dans la liste dans la palette de commandes. Si on vous demande un mot de passe, entrez le mot de passe que vous avez créé pour le nouveau rôle.

1. Dans la fenêtre **Nouvelle requête**, copiez, mettez en surbrillance et exécutez l’instruction SQL suivante :

    ```sql
    SELECT *
    FROM pg_catalog.pg_stats;
    ```

1. Notez que vous pouvez afficher une grande quantité d’informations sur les statistiques.

1. En utilisant des vues système, vous pouvez réduire la complexité des requêtes SQL que vous devez écrire. La requête précédente aurait besoin du code suivant si vous n’utilisiez pas la vue **pg_stats**, exécutons donc ce code pour voir comment celui-ci fonctionne. Dans la fenêtre **Nouvelle requête**, copiez, mettez en surbrillance et exécutez l’instruction SQL suivante :

    ```sql
    SELECT n.nspname AS schemaname,
    c.relname AS tablename,
    a.attname,
    s.stainherit AS inherited,
    s.stanullfrac AS null_frac,
    s.stawidth AS avg_width,
    s.stadistinct AS n_distinct,
        CASE
            WHEN s.stakind1 = 1 THEN s.stavalues1
            WHEN s.stakind2 = 1 THEN s.stavalues2
            WHEN s.stakind3 = 1 THEN s.stavalues3
            WHEN s.stakind4 = 1 THEN s.stavalues4
            WHEN s.stakind5 = 1 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS most_common_vals,
        CASE
            WHEN s.stakind1 = 1 THEN s.stanumbers1
            WHEN s.stakind2 = 1 THEN s.stanumbers2
            WHEN s.stakind3 = 1 THEN s.stanumbers3
            WHEN s.stakind4 = 1 THEN s.stanumbers4
            WHEN s.stakind5 = 1 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS most_common_freqs,
        CASE
            WHEN s.stakind1 = 2 THEN s.stavalues1
            WHEN s.stakind2 = 2 THEN s.stavalues2
            WHEN s.stakind3 = 2 THEN s.stavalues3
            WHEN s.stakind4 = 2 THEN s.stavalues4
            WHEN s.stakind5 = 2 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS histogram_bounds,
        CASE
            WHEN s.stakind1 = 3 THEN s.stanumbers1[1]
            WHEN s.stakind2 = 3 THEN s.stanumbers2[1]
            WHEN s.stakind3 = 3 THEN s.stanumbers3[1]
            WHEN s.stakind4 = 3 THEN s.stanumbers4[1]
            WHEN s.stakind5 = 3 THEN s.stanumbers5[1]
            ELSE NULL::real
        END AS correlation,
        CASE
            WHEN s.stakind1 = 4 THEN s.stavalues1
            WHEN s.stakind2 = 4 THEN s.stavalues2
            WHEN s.stakind3 = 4 THEN s.stavalues3
            WHEN s.stakind4 = 4 THEN s.stavalues4
            WHEN s.stakind5 = 4 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS most_common_elems,
        CASE
            WHEN s.stakind1 = 4 THEN s.stanumbers1
            WHEN s.stakind2 = 4 THEN s.stanumbers2
            WHEN s.stakind3 = 4 THEN s.stanumbers3
            WHEN s.stakind4 = 4 THEN s.stanumbers4
            WHEN s.stakind5 = 4 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS most_common_elem_freqs,
        CASE
            WHEN s.stakind1 = 5 THEN s.stanumbers1
            WHEN s.stakind2 = 5 THEN s.stanumbers2
            WHEN s.stakind3 = 5 THEN s.stanumbers3
            WHEN s.stakind4 = 5 THEN s.stanumbers4
            WHEN s.stakind5 = 5 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS elem_count_histogram
    FROM pg_statistic s
     JOIN pg_class c ON c.oid = s.starelid
     JOIN pg_attribute a ON c.oid = a.attrelid AND a.attnum = s.staattnum
     LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
    WHERE NOT a.attisdropped AND has_column_privilege(c.oid, a.attnum, 'select'::text) AND (c.relrowsecurity = false OR NOT row_security_active(c.oid));
    ```

## Nettoyage

1. Si vous n’avez plus besoin de ce serveur PostgreSQL pour d’autres exercices, pour éviter d’entraîner des coûts Azure inutiles, supprimez le groupe de ressources créé dans cet exercice.

1. Si vous souhaitez conserver le serveur PostgreSQL en cours d’exécution, vous pouvez le laisser en cours d’exécution. Sinon, vous pouvez arrêter le serveur pour éviter d’entraîner des coûts inutiles dans le terminal Bash. Exécutez la commande suivante pour arrêter le serveur :

    ```azurecli
    az postgres flexible-server stop --name <your-server-name> --resource-group $RG_NAME
    ```

    Remplacez `<your-server-name>` par le nom de votre serveur PostreSQL.

    > &#128221; Vous pouvez également arrêter le serveur depuis le portail Azure. Dans le portail Azure, accédez aux **groupes de ressources** et sélectionnez le groupe de ressources que vous avez créé précédemment. Sélectionnez le serveur PostgreSQL, puis sélectionnez **Arrêter** dans le menu.

1. Si nécessaire, supprimez le référentiel Git que vous avez cloné précédemment.

Dans cet exercice, vous avez appris à configurer les paramètres système et à explorer les métadonnées dans PostgreSQL. Vous avez également appris à afficher les métadonnées PostgreSQL dans le portail Azure et à afficher les données dans les tables de catalogue système. En outre, vous avez appris à afficher une requête de métadonnées complexe à l’aide d’une vue système.
