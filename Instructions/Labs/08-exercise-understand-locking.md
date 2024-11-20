---
lab:
  title: Comprendre le verrouillage
  module: Understand concurrency in PostgreSQL
---

# Comprendre le verrouillage

Dans cet exercice, vous examinez les paramètres système et les métadonnées dans PostgreSQL.

## Avant de commencer

Vous devez disposer de votre propre abonnement Azure pour effectuer les exercices de ce module. Si vous n’avez pas d’abonnement Azure, vous pouvez configurer un compte d’essai gratuit en consultant [Créer dans le cloud avec un compte gratuit Azure](https://azure.microsoft.com/free/).

## Créer l’environnement de l’exercice

### Déployer des ressources dans votre abonnement Azure

Cette étape vous guide tout au long de l’utilisation de commandes Azure CLI à partir d’Azure Cloud Shell pour créer un groupe de ressources et exécuter un script Bicep pour déployer les services Azure nécessaires pour effectuer cet exercice dans votre abonnement Azure.

1. Ouvrez un navigateur web et accédez au [portail Azure](https://portal.azure.com/).

2. Dans la barre d’outils du portail Azure, sélectionnez l’icône **Cloud Shell** pour ouvrir un nouveau volet [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) en bas de la fenêtre de votre navigateur.

    ![Capture d’écran de la barre d’outils du portail Azure, avec l’icône Cloud Shell encadrée en rouge.](media/08-portal-toolbar-cloud-shell.png)

    Si vous y êtes invité, sélectionnez les options requises pour ouvrir un interpréteur de commandes *Bash*. Si vous avez déjà utilisé une console *PowerShell*, remplacez-la par un interpréteur de commandes *Bash*.

3. À l’invite Cloud Shell, entrez ce qui suit pour cloner le référentiel GitHub contenant des ressources d’exercice :

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

4. Exécutez ensuite trois commandes pour définir des variables afin de réduire les saisies redondantes lors de l’utilisation des commandes Azure CLI pour créer des ressources Azure. Les variables représentent le nom à affecter à votre groupe de ressources (`RG_NAME`), la région Azure (`REGION`) dans laquelle les ressources seront déployées et un mot de passe généré de manière aléatoire pour la connexion de l’administrateur PostgreSQL (`ADMIN_PASSWORD`).

    Dans la première commande, la région affectée à la variable correspondante est `eastus`, mais vous pouvez également la remplacer par un emplacement de votre choix.

    ```bash
    REGION=eastus
    ```

    La commande suivante attribue le nom à utiliser pour le groupe de ressources qui hébergera toutes les ressources utilisées dans cet exercice. Le nom du groupe de ressources affecté à la variable correspondante est `rg-learn-work-with-postgresql-$REGION`, où `$REGION` est l’emplacement que vous avez spécifié ci-dessus. Toutefois, vous pouvez le remplacer par tout autre nom de groupe de ressources de votre choix.

    ```bash
    RG_NAME=rg-learn-work-with-postgresql-$REGION
    ```

    La commande finale génère de façon aléatoire un mot de passe pour la connexion d’administrateur PostgreSQL. Copiez-le dans un endroit sécurisé pour pouvoir l’utiliser ultérieurement et vous connecter à votre serveur flexible PostgreSQL.

    ```bash
    a=()
    for i in {a..z} {A..Z} {0..9}; 
       do
       a[$RANDOM]=$i
    done
    ADMIN_PASSWORD=$(IFS=; echo "${a[*]::18}")
    echo "Your randomly generated PostgreSQL admin user's password is:"
    echo $ADMIN_PASSWORD
    ```

5. Si vous avez accès à plusieurs abonnements Azure et que votre abonnement par défaut n’est pas celui dans lequel vous souhaitez créer le groupe de ressources et d’autres ressources pour cet exercice, exécutez cette commande pour définir l’abonnement approprié, en remplaçant le jeton `<subscriptionName|subscriptionId>` par le nom ou l’ID de l’abonnement que vous souhaitez utiliser :

    ```azurecli
    az account set --subscription <subscriptionName|subscriptionId>
    ```

6. Exécutez la commande Azure CLI suivante pour créer un groupe de ressources :

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

7. Enfin, utilisez Azure CLI pour exécuter un script de déploiement Bicep pour approvisionner des ressources Azure dans votre groupe de ressources :

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD databaseName=adventureworks
    ```

    Le script de déploiement Bicep approvisionne les services Azure requis pour effectuer cet exercice dans votre groupe de ressources. Les ressources déployées sont un serveur flexible Azure Database pour PostgreSQL. Le script Bicep crée également une base de données qui peut être configurée sur la ligne de commande en tant que paramètre.

    Le déploiement prend généralement plusieurs minutes. Vous pouvez le surveiller à partir de Cloud Shell ou accéder à la page **Déploiements** du groupe de ressources que vous avez créé ci-dessus et observer la progression du déploiement.

8. Fermez le volet Cloud Shell une fois votre déploiement de ressources terminé.

### Résoudre les erreurs de déploiement

Vous pouvez rencontrer quelques erreurs lors de l’exécution du script de déploiement Bicep. Les messages les plus courants et les étapes à suivre pour les résoudre sont les suivants :

- Si vous avez précédemment exécuté le script de déploiement Bicep pour ce parcours d’apprentissage et supprimé par la suite les ressources, vous pouvez recevoir un message d’erreur semblable à ce qui suit si vous tentez de réexécuter le script dans les 48 heures suivant la suppression des ressources :

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is '4e87a33d-a0ac-4aec-88d8-177b04c1d752'. See inner errors for details."}
    
    Inner Errors:
    {"code": "FlagMustBeSetForRestore", "message": "An existing resource with ID '/subscriptions/{subscriptionId}/resourceGroups/rg-learn-postgresql-ai-eastus/providers/Microsoft.CognitiveServices/accounts/{accountName}' has been soft-deleted. To restore the resource, you must specify 'restore' to be 'true' in the property. If you don't want to restore existing resource, please purge it first."}
    ```

    Si vous recevez ce message, modifiez la commande `azure deployment group create` ci-dessus pour définir le paramètre `restore` égal à `true` et réexécutez-la.

- Si la région sélectionnée est limitée à l’approvisionnement de ressources spécifiques, vous devez définir la variable `REGION` à un autre emplacement et réexécuter les commandes pour créer le groupe de ressources et exécuter le script de déploiement Bicep.

    ```bash
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- Si le script ne parvient pas à créer une ressource IA en raison de la nécessité d’accepter le contrat d’IA responsable, vous pouvez rencontrer l’erreur suivante. Dans ce cas, utilisez l’interface utilisateur du portail Azure pour créer une ressource Azure AI Services, puis réexécutez le script de déploiement.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## Se connecter à votre base de données avec psql dans Azure Cloud Shell

Dans cette tâche, vous allez vous connecter à la base de données `adventureworks` sur votre serveur Azure Database pour PostgreSQL à l’aide de l’[utilitaire de ligne de commande psql](https://www.postgresql.org/docs/current/app-psql.html) à partir d’[Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview).

1. Dans le [portail Azure](https://portal.azure.com/), accédez à votre serveur flexible Azure Database pour PostgreSQL nouvellement créé.

2. Dans le menu de ressource, dans **Paramètres**, sélectionnez **Bases de données**, puis **Connecter** pour la base de données `adventureworks`.

    ![Capture d’écran de la page Bases de données d’Azure SQL Database pour PostgreSQL. Le paramètre Bases de données et l’élément Connecter de la base de données AdventureWorks sont encadrés en rouge.](media/08-postgresql-adventureworks-database-connect.png)

3. À l’invite « Mot de passe pour l’utilisateur pgAdmin » dans Cloud Shell, entrez le mot de passe généré de manière aléatoire pour la connexion **pgAdmin**.

    Une fois connecté, l’invite `psql` de la base de données `adventureworks` s’affiche.

4. Tout au long de cet exercice, vous continuez à travailler dans Cloud Shell. Il peut donc être utile d’étendre le volet dans la fenêtre de votre navigateur en sélectionnant le bouton **Agrandir** en haut à droite du volet.

    ![Capture d’écran du volet Azure Cloud Shell avec le bouton Agrandir encadré en rouge.](media/08-azure-cloud-shell-pane-maximize.png)

### Remplir la base de données avec des données

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

1. Ensuite, utilisez la commande `COPY` pour charger des données à partir de fichiers CSV dans la table que vous avez créée ci-dessus. Commencez par exécuter la commande suivante pour remplir la table `production.workorder`.

    ```sql
    \COPY production.workorder FROM 'mslearn-postgresql/Allfiles/Labs/08/Lab8_workorder.csv' CSV HEADER
    ```

    La sortie de commande doit être `COPY 72591`, indiquant que 72 591 lignes ont été écrites dans la table à partir du fichier CSV.

1. Fermez le volet Cloud Shell une fois les données chargées.

### Se connecter à la base de données avec Azure Data Studio

1. Si vous n’avez pas encore installé Azure Data Studio, [téléchargez et installez ***Azure Data Studio***](https://go.microsoft.com/fwlink/?linkid=2282284).
1. Démarrez Azure Data Studio.
1. Si vous n’avez pas installé l’**extension PostgreSQL** dans Azure Data Studio, installez-la maintenant.
1. Sélectionnez **Serveurs**, puis **Nouvelle connexion**.
1. Dans **Type de connexion**, sélectionnez **PostgreSQL**.
1. Dans **Nom d’u serveur**, tapez la valeur que vous avez spécifiée quand vous avez déployé le serveur.
1. Dans **Nom d’utilisateur**, tapez **pgAdmin**.
1. Dans **Mot de passe**, entrez le mot de passe généré de manière aléatoire pour la connexion **pgAdmin** que vous avez générée.
1. Sélectionnez **Mémoriser le mot de passe**.
1. Cliquez sur **Connexion**

## Tâche 1 : Examiner le comportement de verrouillage par défaut

1. Ouvrez Azure Data Studio.
1. Développez **Bases de données**, cliquez avec le bouton droit sur **adventureworks**, puis sélectionnez **Nouvelle requête**.
   
    ![Capture d’écran de base de données AdventureWorks mettant en évidence l’élément de menu contextuel Nouvelle requête.](media/08-new-query.png)

1. Accédez à **Fichier** et **Nouvelle requête**. Vous devez maintenant avoir un onglet de requête dont le nom commence par **SQL_Query_1** et un autre onglet de requête dont le nom commence par **SQL_Query_2**.
1. Sélectionnez l’onglet **SQLQuery_1** , tapez la requête suivante, puis sélectionnez **Exécuter**.

    ```sql
    SELECT * FROM production.workorder
    ORDER BY scrappedqty DESC;
    ```

1. Notez que la valeur **scrappedqty** pour la première ligne est **673**.
1. Sélectionnez l’onglet **SQLQuery_2** , tapez la requête suivante, puis sélectionnez **Exécuter**.

    ```sql
    BEGIN TRANSACTION;
    UPDATE production.workorder
        SET scrappedqty=scrappedqty+1;
    ```

1. Notez que la deuxième requête commence une transaction, mais ne la valide pas.
1. Revenez à **SQLQuery_1** et réexécutez la requête.
1. Notez que la valeur **stockedqty** pour la première ligne est toujours **673**. La requête utilise un instantané des données et ne voit pas les mises à jour de l’autre transaction.
1. Sélectionnez l’onglet **SQLQuery_2** , supprimez la requête existante, tapez la requête suivante, puis sélectionnez **Exécuter**.

    ```sql
    ROLLBACK TRANSACTION;
    ```

## Tâche 2 : Appliquer des verrous de table à une transaction

1. Sélectionnez l’onglet **SQLQuery_2** , tapez la requête suivante, puis sélectionnez **Exécuter**.

    ```sql
    BEGIN TRANSACTION;
    LOCK TABLE production.workorder IN ACCESS EXCLUSIVE MODE;
    UPDATE production.workorder
        SET scrappedqty=scrappedqty+1;
    ```

1. Notez que la deuxième requête commence une transaction, mais ne la valide pas.
1. Revenez à **SQLQuery_1** et réexécutez la requête.
1. Notez que la transaction est bloquée et ne se termine pas, peu importe combien de temps vous attendez.
1. Sélectionnez l’onglet **SQLQuery_2** , supprimez la requête existante, tapez la requête suivante, puis sélectionnez **Exécuter**.

    ```sql
    ROLLBACK TRANSACTION;
    ```

1. Revenez à **SQLQuery_1**, attendez quelques secondes, et notez que la requête s’est terminée une fois que le verrou a été supprimé.

Dans cet exercice, nous avons vu le comportement de verrouillage par défaut. Nous avons ensuite appliqué explicitement des verrous et vu que, bien que certains verrous fournissent des niveaux de protection très élevés, ces verrous peuvent également avoir des effets sur les performances.

## Nettoyage de l’exercice

Le serveur Azure Database pour PostgreSQL que nous avons déployé dans cet exercice entraîne des frais, vous pouvez supprimer le serveur après cet exercice. Vous pouvez également supprimer le groupe de ressources **rg-learn-work-with-postgresql-eastus** pour supprimer toutes les ressources que nous avons déployées dans le cadre de cet exercice.
