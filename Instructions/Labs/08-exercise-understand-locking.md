---
lab:
  title: Comprendre le verrouillage
  module: Understand concurrency in PostgreSQL
---

# Comprendre le verrouillage

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
    az deployment group create --resource-group $RG_NAME --template-file "Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD databaseName=adventureworks
    ```

    Le script de déploiement Bicep approvisionne les services Azure requis pour effectuer cet exercice dans votre groupe de ressources. Les ressources déployées sont un serveur flexible Azure Database pour PostgreSQL. Le script Bicep crée également la base de données AdventureWorks.

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

### Installer psql

Étant donné que nous devons copier les données du fichier CSV dans la base de données PostgreSQL, il est nécessaire d’installer `psql` sur votre ordinateur local. *Si vous avez déjà installé `psql`, ignorez cette section.*

1. Pour vérifier si **psql** est déjà installé dans votre environnement, ouvrez une ligne de commande ou un terminal et exécutez la commande ***psql***. Si un message tel que « *psql : erreur : la connexion au serveur sur le socket...*  » est renvoyé, cela signifie que l’outil **psql** est déjà installé dans votre environnement et qu’il n’est pas nécessaire de le réinstaller. Vous pouvez donc ignorer cette section.

1. Installez [psql](https://sbp.enterprisedb.com/getfile.jsp?fileid=1258893).

1. Dans l’Assistant Installation, lorsque vous atteignez la boîte de dialogue **Sélectionner les composants**, sélectionnez **Outils de ligne de commande**. Vous pouvez décocher les autres composants si vous ne prévoyez pas de les utiliser. Suivez les instructions pour terminer l’installation.

1. Pour vérifier que l’installation a réussi, ouvrez une fenêtre de terminal ou d’invite de commandes et exécutez **psql** : Si vous voyez un message tel que «* psql : erreur : la connexion au serveur sur le socket...* », cela signifie que l’outil **psql** est correctement installé. Sinon, vous devrez peut-être ajouter le répertoire bin PostgreSQL à votre variable système Path.

    1. Si vous utilisez Windows, assurez-vous d’ajouter le répertoire bin PostgreSQL à votre variable système Path. L’emplacement du répertoire bin est généralement `C:\Program Files\PostgreSQL\<version>\bin`.
        1. Vous pouvez vérifier si le répertoire bin se trouve dans votre variable Path en exécutant la commande `echo %PATH%` dans une invite de commandes et en vérifiant si le répertoire bin PostgreSQL est répertorié. Si ce n’est pas le cas, vous pouvez l’ajouter manuellement :
        1. Pour l’ajouter manuellement, cliquez avec le bouton droit sur le bouton **Démarrer**.
        1. Sélectionnez **Système**, puis **Paramètres système avancés**.
        1. Sélectionnez le bouton **variables d’environnement**.
        1. Double-cliquez sur la variable **Path** dans la section **Variables système**.
        1. Sélectionnez **Nouveau**, puis ajoutez le chemin d’accès du répertoire bin PostgreSQL.
        1. Après l’avoir ajouté, fermez et ouvrez à nouveau l’invite de commandes pour que les modifications s’appliquent.

    1. Si vous utilisez macOS ou Linux, le répertoire `bin` PostgreSQL se trouve généralement à l’emplacement `/usr/local/pgsql/bin`.  
        1. Vous pouvez vérifier si ce répertoire se trouve dans votre variable d’environnement `PATH` en exécutant `echo $PATH` dans un terminal.  
        1. Si ce n’est pas le cas, vous pouvez l’ajouter en modifiant le fichier de configuration shell, généralement `.bash_profile`, `.bashrc` ou `.zshrc` en fonction de votre shell.

### Remplir la base de données avec des données

Une fois que vous avez vérifié que **psql** est installé, vous pouvez vous connecter à votre serveur PostgreSQL à l’aide de la ligne de commande, en ouvrant une fenêtre d’invite de commandes ou de terminal.

> Si vous utilisez Windows, vous pouvez utiliser **Windows PowerShell** ou **l’invite de** commandes. Si vous utilisez macOS ou Linux, vous pouvez utiliser l’application **Terminal**.

La syntaxe de connexion au serveur est la suivante :

```sql
psql -h <servername> -p <port> -U <username> <dbname>
```

1. Dans l’invite de commandes ou le terminal, saisissez **`--host=<servername>.postgres.database.azure.com`** où `<servername>` est le nom du serveur Azure Database pour PostgreSQL créé précédemment.

    Vous trouverez le nom du serveur dans **Vue d’ensemble** dans le portail Azure ou en tant que sortie du script Bicep.

    ```sql
   psql -h <servername>.postgres.database.azure.com -p 5432 -U pgAdmin adventureworks
    ```

    Le mot de passe de compte Administrateur que vous avez précédemment copié vous est alors demandé.

1. Vous devez créer une table dans la base de données et la remplir avec des exemples de données afin d’avoir des informations à utiliser lors de la révision du verrouillage dans cet exercice.

1. Exécutez la commande suivante pour créer la table `production.workorder` et charger les données :

    ```sql
    DROP SCHEMA IF EXISTS production CASCADE;
    CREATE SCHEMA production;
    
    DROP TABLE IF EXISTS production.workorder;
    CREATE TABLE production.workorder
    (
        workorderid integer NOT NULL,
        productid integer NOT NULL,
        orderqty integer NOT NULL,
        scrappedqty smallint NOT NULL,
        startdate timestamp without time zone NOT NULL,
        enddate timestamp without time zone,
        duedate timestamp without time zone NOT NULL,
        scrapreasonid smallint,
        modifieddate timestamp without time zone NOT NULL DEFAULT now()
    )
    WITH (
        OIDS = FALSE
    )
    TABLESPACE pg_default;
    ```

1. Ensuite, utilisez la commande `COPY` pour charger des données à partir de fichiers CSV dans la table que vous avez créée précédemment. Commencez par exécuter la commande suivante pour remplir la table `production.workorder` :

    ```sql
    \COPY production.workorder FROM 'Allfiles/Labs/08/Lab8_workorder.csv' CSV HEADER
    ```

    La sortie de commande doit être `COPY 72591`, indiquant que 72 591 lignes ont été écrites dans la table à partir du fichier CSV.

1. Fermez la fenêtre d’invite de commandes ou de terminal.

### Se connecter à la base de données avec Visual Studio Code

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

## Tâche 1 : Examiner le comportement de verrouillage par défaut

1. Ouvrez Visual Studio Code s’il n’est pas déjà ouvert.

1. Affichez la palette de commandes (Ctrl+Maj+P) et sélectionnez **PGSQL : nouvelle requête**. Sélectionnez la nouvelle connexion que vous avez créée dans la liste dans la palette de commandes. Si on vous demande un mot de passe, entrez le mot de passe que vous avez créé pour le nouveau rôle.

1. En bas à droite de la fenêtre **Nouvelle requête**, vérifiez que la connexion est verte. Si ce n’est pas le cas, un message **PGSQL déconnecté** doit être affiché. Sélectionnez le texte **PGSQL déconnecté**, puis sélectionnez votre connexion de serveur PostgreSQL dans la liste dans la palette de commandes. Si un mot de passe est demandé, entrez le mot de passe que vous avez généré précédemment.

1. Dans la fenêtre **Nouvelle requête**, copiez, mettez en surbrillance et exécutez l’instruction SQL suivante :

    ```sql
    SELECT current_database();
    ```

1. Si la base de données actuelle n’est pas définie sur **adventureworks**, vous devez remplacer la base de données par **adventureworks**. Pour modifier la base de données, sélectionnez les points de suspension dans la barre de menus avec l’icône d’*exécution* et sélectionnez **Modifier la base de données PostgreSQL**. Sélectionnez `adventureworks` dans la liste des bases de données. Vérifiez que la base de données est désormais définie sur `adventureworks` en exécutant l’instruction **SELECT current_database();**.

1. Dans la fenêtre **Nouvelle requête**, copiez, mettez en surbrillance et exécutez l’instruction SQL suivante :

    ```sql
    SELECT * FROM production.workorder
    ORDER BY scrappedqty DESC;
    ```

1. Notez que la valeur **scrappedqty** pour la première ligne est **673**.

1. Nous devons ouvrir une deuxième fenêtre de requête pour simuler une transaction qui met à jour les données dans la première fenêtre de requête. Affichez la palette de commandes (Ctrl+Maj+P) et sélectionnez **PGSQL : nouvelle requête**. Sélectionnez la nouvelle connexion que vous avez créée dans la liste dans la palette de commandes. Si on vous demande un mot de passe, entrez le mot de passe que vous avez créé pour le nouveau rôle.

1. En bas à droite de la fenêtre **Nouvelle requête**, vérifiez que la connexion est verte. Si ce n’est pas le cas, un message **PGSQL déconnecté** doit être affiché. Sélectionnez le texte **PGSQL déconnecté**, puis sélectionnez votre connexion de serveur PostgreSQL dans la liste dans la palette de commandes. Si un mot de passe est demandé, entrez le mot de passe que vous avez généré précédemment.

1. Dans la fenêtre **Nouvelle requête**, copiez, mettez en surbrillance et exécutez l’instruction SQL suivante :

    ```sql
    SELECT current_database();
    ```

1. Si la base de données actuelle n’est pas définie sur **adventureworks**, vous devez remplacer la base de données par **adventureworks**. Pour modifier la base de données, sélectionnez les points de suspension dans la barre de menus avec l’icône d’*exécution* et sélectionnez **Modifier la base de données PostgreSQL**. Sélectionnez `adventureworks` dans la liste des bases de données. Vérifiez que la base de données est désormais définie sur `adventureworks` en exécutant l’instruction **SELECT current_database();**.

1. Dans la fenêtre **Nouvelle requête**, copiez, mettez en surbrillance et exécutez l’instruction SQL suivante :

    ```sql
    BEGIN TRANSACTION;
    UPDATE production.workorder
        SET scrappedqty=scrappedqty+1;
    ```

1. Notez que la deuxième requête commence une transaction, mais ne la valide pas.

1. Revenez à la *première* fenêtre de requête et exécutez à nouveau la requête dans cette fenêtre.

1. Notez que la valeur de **scrappedqty** pour la première ligne est toujours **673**. La requête utilise un instantané des données et ne voit pas les mises à jour de l’autre transaction.

1. Sélectionnez l’onglet de la *deuxième* requête, supprimez la requête existante, saisissez la requête suivante, puis sélectionnez **Exécuter**.

    ```sql
    ROLLBACK TRANSACTION;
    ```

## Tâche 2 : Appliquer des verrous de table à une transaction

1. Sélectionnez l’onglet de la *deuxième* requête, saisissez la requête suivante, puis sélectionnez **Exécuter**.

    ```sql
    BEGIN TRANSACTION;
    LOCK TABLE production.workorder IN ACCESS EXCLUSIVE MODE;
    UPDATE production.workorder
        SET scrappedqty=scrappedqty+1;
    ```

1. Notez que la deuxième requête commence une transaction, mais ne la valide pas.

1. Revenez à la *première* requête et exécutez à nouveau la requête.

1. Notez que la transaction est bloquée et ne se termine pas, peu importe combien de temps vous attendez.

1. Sélectionnez l’onglet de la *deuxième* requête, supprimez la requête existante, saisissez la requête suivante, puis sélectionnez **Exécuter**.

    ```sql
    ROLLBACK TRANSACTION;
    ```

1. Revenez à la *première* requête, patientez quelques secondes, et notez que la requête s’est terminée une fois le blocage retiré.

## Nettoyage

1. Si vous n’avez plus besoin de ce serveur PostgreSQL pour d’autres exercices, pour éviter d’entraîner des coûts Azure inutiles, supprimez le groupe de ressources créé dans cet exercice.

1. Si vous souhaitez conserver le serveur PostgreSQL en cours d’exécution, vous pouvez le laisser en cours d’exécution. Sinon, vous pouvez arrêter le serveur pour éviter d’entraîner des coûts inutiles dans le terminal Bash. Exécutez la commande suivante pour arrêter le serveur :

    ```azurecli
    az postgres flexible-server stop --name <your-server-name> --resource-group $RG_NAME
    ```

    Remplacez `<your-server-name>` par le nom de votre serveur PostreSQL.

    > &#128221; Vous pouvez également arrêter le serveur depuis le portail Azure. Dans le portail Azure, accédez aux **groupes de ressources** et sélectionnez le groupe de ressources que vous avez créé précédemment. Sélectionnez le serveur PostgreSQL, puis sélectionnez **Arrêter** dans le menu.

1. Si nécessaire, supprimez le référentiel Git que vous avez cloné précédemment.

Dans cet exercice, nous avons examiné le comportement de blocage par défaut. Nous avons ensuite appliqué des blocages spécifiques, et bien que certains blocages fournissent des niveaux de protection très élevés, ces blocages peuvent également affecter les performances.
