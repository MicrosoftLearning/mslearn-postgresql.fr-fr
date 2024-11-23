---
lab:
  title: Migration de base de données PostgreSQL hors connexion
  module: Migrate to Azure Database for PostgreSQL Flexible Server
---

# Migration de base de données PostgreSQL hors connexion

Dans cet exercice, vous allez créer un serveur flexible Azure Database pour PostgreSQL et effectuer une migration de base de données hors connexion à partir d’un serveur PostgreSQL local ou d’un serveur Azure Database pour PostgreSQL à l’aide de la fonctionnalité Migration dans le serveur flexible Azure Database pour PostgreSQL.

## Avant de commencer

Pour effectuer cet exercice, vous avez besoin de votre propre abonnement Azure. Si vous n’avez pas d’abonnement Azure, vous pouvez demander un [essai gratuit d’Azure](https://azure.microsoft.com/free).

### Modifiez le fichier pg_hba.conf pour autoriser la connectivité à partir d’Azure (à ignorer en cas de migration à partir d’un serveur autre qu’un serveur PostgreSQL externe)

> [!NOTE]
> Ce labo crée deux serveurs Azure Database pour PostgreSQL à utiliser comme source et destination pour la migration. Toutefois, si vous utilisez votre propre environnement, pour effectuer cet exercice, vous devez avoir accès à un serveur PostgreSQL existant avec une base de données, des autorisations appropriées et un accès réseau.
> 
> Si vous utilisez votre propre environnement, cet exercice nécessite que le serveur que vous utilisez comme source de la migration soit accessible au serveur flexible Azure Database pour PostgreSQL afin qu’il puisse connecter et migrer les bases de données. Cela nécessite que le serveur source soit accessible via une adresse IP et un port publics. Une liste d’adresses IP de région Azure peut être téléchargée à partir de [Plages d’adresses IP Azure et étiquettes de service : cloud public](https://www.microsoft.com/en-gb/download/details.aspx?id=56519) pour réduire les plages autorisées d’adresses IP dans vos règles de pare-feu baen fonction de la région Azure utilisée. Ouvrez le pare-feu de vos serveurs pour permettre à la fonctionnalité Migration dans le serveur flexible Azure Database pour PostgreSQL d’accéder au serveur PostgreSQL source, par défaut le port TCP **5432**.
>
> Lorsque vous utilisez une appliance de pare-feu devant vos bases de données sources, vous devrez peut-être ajouter des règles de pare-feu pour permettre à la fonctionnalité Migration dans le serveur flexible Azure Database pour PostgreSQL d’accéder aux bases de données sources pour la migration.
>
> La version maximale prise en charge de PostgreSQL pour la migration est la version 16.

Le serveur PostgreSQL source doit mettre à jour le fichier pg_hba.conf pour vous assurer que l’instance autorise la connectivité à partir du serveur flexible Azure Database pour PostgreSQL.

1. Vous allez ajouter des entrées à pg_hba.conf pour autoriser les connexions à partir des plages d’adresses IP Azure. Les entrées dans pg_hba.conf déterminent quels hôtes peuvent se connecter, quelles bases de données, quels utilisateurs et quelles méthodes d’authentification peuvent être utilisés.
1. Par exemple, vos services Azure se trouvent dans la plage d’adresses IP 104.45.0.0/16. Pour permettre à tous les utilisateurs de se connecter à toutes les bases de données de cette plage à l’aide de l’authentification par mot de passe, vous devez ajouter :

``` bash
host    all    all    104.45.0.0/16    md5
```

1. Lorsque vous autorisez des connexions via Internet, notamment à partir d’Azure, assurez-vous de disposer de mécanismes d’authentification forts.

- Utilisez des mots de passe forts.
- Limitez l’accès à un nombre d’adresses IP aussi réduit que possible.
- Utilisez un VPN ou un VNet : si possible, configurez un réseau privé virtuel (VPN) ou un réseau virtuel (VNet) Azure pour fournir un tunnel sécurisé entre Azure et votre serveur PostgreSQL.

1. Après avoir enregistré les modifications apportées à pg_hba.conf, vous devez recharger la configuration PostgreSQL pour que les modifications prennent effet à l’aide d’une commande SQL dans une session psql :

```sql
SELECT pg_reload_conf();
```

1. Testez la connexion entre Azure et votre serveur PostgreSQL local pour vous assurer que la configuration fonctionne comme prévu. Vous pouvez le faire à partir d’une machine virtuelle Azure ou d’un service qui prend en charge les connexions aux base de données sortantes.

### Déployer des ressources dans votre abonnement Azure

Cette étape vous guide tout au long de l’utilisation de commandes Azure CLI à partir d’Azure Cloud Shell pour créer un groupe de ressources et exécuter un script Bicep pour déployer les services Azure nécessaires pour effectuer cet exercice dans votre abonnement Azure.

1. Ouvrez un navigateur web et accédez au [portail Azure](https://portal.azure.com/).

1. Dans la barre d’outils du portail Azure, sélectionnez l’icône **Cloud Shell** pour ouvrir un nouveau volet [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) en bas de la fenêtre de votre navigateur.

    ![Capture d’écran de la barre d’outils du portail Azure, avec l’icône Cloud Shell encadrée en rouge.](media/11-portal-toolbar-cloud-shell.png)

    Si vous y êtes invité, sélectionnez les options requises pour ouvrir un interpréteur de commandes *Bash*. Si vous avez déjà utilisé une console *PowerShell*, remplacez-la par un interpréteur de commandes *Bash*.

1. À l’invite Cloud Shell, entrez ce qui suit pour cloner le référentiel GitHub contenant des ressources d’exercice :

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

1. Exécutez ensuite trois commandes pour définir des variables afin de réduire les saisies redondantes lors de l’utilisation des commandes Azure CLI pour créer des ressources Azure. Les variables représentent le nom à affecter à votre groupe de ressources (`RG_NAME`), la région Azure (`REGION`) dans laquelle les ressources seront déployées et un mot de passe généré de manière aléatoire pour la connexion de l’administrateur PostgreSQL (`ADMIN_PASSWORD`).

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

1. Si vous avez accès à plusieurs abonnements Azure et que votre abonnement par défaut n’est pas celui dans lequel vous souhaitez créer le groupe de ressources et d’autres ressources pour cet exercice, exécutez cette commande pour définir l’abonnement approprié, en remplaçant le jeton `<subscriptionName|subscriptionId>` par le nom ou l’ID de l’abonnement que vous souhaitez utiliser :

    ```azurecli
    az account set --subscription <subscriptionName|subscriptionId>
    ```

1. Exécutez la commande Azure CLI suivante pour créer un groupe de ressources :

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

1. Enfin, utilisez Azure CLI pour exécuter un script de déploiement Bicep pour approvisionner des ressources Azure dans votre groupe de ressources :

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-postgresql-server-migration.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD databaseName=adventureworks
    ```

    Le script de déploiement Bicep approvisionne les services Azure requis pour effectuer cet exercice dans votre groupe de ressources. Les ressources déployées sont deux serveurs flexibles Azure Database pour PostgreSQL. Une source et un serveur de destination pour la migration.

    Le déploiement prend généralement plusieurs minutes (5 à 10 minutes ou plus). Vous pouvez le surveiller à partir de Cloud Shell ou accéder à la page **Déploiements** du groupe de ressources que vous avez créé ci-dessus et observer la progression du déploiement.

1. Fermez le volet Cloud Shell une fois votre déploiement de ressources terminé.

1. Sur le portail Azure, passez en revue les noms des deux nouveaux serveurs Azure Database pour PostgreSQL. Notez que la liste des bases de données du serveur source inclut la base de données **AdventureWorks**, mais pas la destination.

1. Dans la section **Mise en réseau** des *deux* serveurs :
    1. Sélectionnez **+ Ajouter l’adresse IP actuelle (xxx.xxx.xxx)** et **Enregistrer**.
    1. Activez la case à cocher **Autoriser l’accès public à partir d’un service Azure dans Azure sur ce serveur**.
    1. Cochez la case **Autoriser l’accès public à cette ressource via Internet à l’aide d’une adresse IP publique**.

> [!NOTE]
> Dans un environnement de production, sélectionnez uniquement les options, réseaux et adresses IP qui, selon vous, devraient avoir accès à vos serveurs Azure Database pour PostgreSQL. 

> [!NOTE]
> Comme indiqué précédemment, ce script Bicep crée deux serveurs Azure Database pour PostgreSQL, un serveur source et un serveur de destination.  ***Si vous utilisez un serveur PostgreSQL local dans votre environnement comme serveur source pour ce labo, remplacez les informations de connexion du serveur source dans toutes les instructions suivantes par les informations de connexion de votre serveur local dans votre environnement***.  Veillez à activer les règles de pare-feu nécessaires dans votre environnement et dans Azure.
    
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
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- Si le script ne parvient pas à créer une ressource IA en raison de la nécessité d’accepter le contrat d’IA responsable, vous pouvez rencontrer l’erreur suivante. Dans ce cas, utilisez l’interface utilisateur du portail Azure pour créer une ressource Azure AI Services, puis réexécutez le script de déploiement.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## Créer une base de données, une table et des données pour la migration

Ce labo vous permet d’effectuer la migration à partir d’un serveur PostreSQL local ou d’un serveur Azure Database pour PostgreSQL. Suivez les instructions pour le type de serveur à partir duquel vous effectuez la migration.

### Créer une base de données sur le serveur PostgreSQL local (ignorez si vous effectuez la migration à partir d’un serveur Azure Database pour PostgreSQL)

À présent, nous devons configurer la base de données, que vous allez migrer vers le serveur flexible Azure Database pour PostgreSQL. Cette étape doit être effectuée sur votre instance de serveur PostgreSQL source, qui doit être accessible au serveur flexible Azure Database pour PostgreSQL pour terminer ce labo.

Tout d’abord, nous devons créer une base de données vide, dans laquelle nous allons créer une table, puis la charger avec des données. Tout d’abord, vous devez télécharger les fichiers ***Lab10_setupTable.sql*** et ***Lab10_workorder.csv*** depuis le [référentiel](https://github.com/MicrosoftLearning/mslearn-postgresql/tree/main/Allfiles/Labs/10) vers votre lecteur local (par exemple, **C:\\**).
Une fois que vous avez ces fichiers, nous pouvons créer la base de données à l’aide de la commande suivante, **remplacer les valeurs de l’hôte, du port et du nom d’utilisateur comme requis pour votre serveur PostgreSQL.**

```bash
psql --host=localhost --port=5432 --username=pgadmin --command="CREATE DATABASE adventureworks;"
```

Exécutez la commande suivante pour créer la table `production.workorder` et charger les données :

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
    ALTER TABLE production.workorder OWNER to pgAdmin;
```

```sql
psql --host=localhost --port=5432 --username=postgres --dbname=adventureworks --command="\COPY production.workorder FROM 'C:\Lab10_workorder.csv' CSV HEADER"
```

La sortie de commande doit être `COPY 72101`, indiquant que 72 101 lignes ont été écrites dans la table à partir du fichier CSV.

## Pré-migration (à ignorer en cas de migration à partir d’un serveur Azure Database pour PostgreSQL)

Avant de démarrer la migration hors connexion de la base de données à partir du serveur source, nous devons nous assurer que le serveur cible est configuré et prêt.

1. Migrez les utilisateurs et les rôles du serveur source vers le nouveau serveur flexible. Pour ce faire, utilisez l’outil pg_dumpall avec le code suivant.
    1. Les rôles de superutilisateur ne sont pas pris en charge sur Azure Database pour PostgreSQL. Les utilisateurs disposant de ces privilèges doivent donc les supprimer avant la migration.

```bash
pg_dumpall --globals-only -U <<username>> -f <<filename>>.sql
```

1. Faites correspondre les valeurs des paramètres du serveur à partir du serveur source sur le serveur cible.
1. Désactivez la haute disponibilité et les réplicas en lecture sur la cible

### Créer une base de données sur le serveur Azure Database pour PostgreSQL (à ignorer en cas de migration à partir d’un serveur PostgreSQL local)

À présent, nous devons configurer la base de données, que vous allez migrer vers le serveur flexible Azure Database pour PostgreSQL. Cette étape doit être effectuée sur votre instance de serveur PostgreSQL source, qui doit être accessible au serveur flexible Azure Database pour PostgreSQL pour terminer ce labo.

Tout d’abord, nous devons créer une base de données vide, dans laquelle nous allons créer une table, puis la charger avec des données. 

1. Dans le [portail Azure](https://portal.azure.com/), accédez au serveur Azure Database pour PostgreSQL source nouvellement créé (_**psql-learn-source**_-location-uniquevalue).

1. Dans le menu de ressource, dans **Paramètres**, sélectionnez **Bases de données**, puis **Connecter** pour la base de données `adventureworks`.

    ![Capture d’écran de la page Bases de données d’Azure SQL Database pour PostgreSQL. Le paramètre Bases de données et l’élément Connecter de la base de données AdventureWorks sont encadrés en rouge.](media/08-postgresql-adventureworks-database-connect.png)

1. À l’invite « Mot de passe pour l’utilisateur pgAdmin » dans Cloud Shell, entrez le mot de passe généré de manière aléatoire pour la connexion **pgAdmin**.

    Une fois connecté, l’invite `psql` de la base de données `adventureworks` s’affiche.

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

    ```sql
    \COPY production.workorder FROM 'mslearn-postgresql/Allfiles/Labs/10/Lab10_workorder.csv' CSV HEADER
    ```

    La sortie de commande doit être `COPY 72101`, indiquant que 72 101 lignes ont été écrites dans la table à partir du fichier CSV.

1. Fermez le **Cloud Shell**.

## Créer un projet de migration de base de données dans le serveur flexible Azure Database pour PostgreSQL

1. Sur le serveur de destination, sélectionnez **Migration** dans le menu situé à gauche du panneau serveur flexible.

   ![Option de migration du serveur flexible Azure Database pour PostgreSQL.](./media/10-pgflex-migation.png)

1. Cliquez sur l’option **+ Créer** en haut du panneau **Migration**.
   > **Remarque** : si l’option **+ Créer** n’est pas disponible, sélectionnez **Calcul + stockage** et remplacez le niveau de calcul par **Usage général** ou **Mémoire optimisée**, puis réessayez de créer le processus de migration. Une fois la migration réussie, vous pouvez rétablir le niveau de calcul sur **Burstable**.
1. Dans l’onglet **Configuration**, renseignez chaque champ comme suit :
    1. Nom de la migration : **`Migration-AdventureWorks`**.
    1. Type de serveur source : pour ce labo, peu importe si vous effectuez une migration à partir d’un serveur local ou d’un serveur Azure Database pour PostgreSQL, sélectionnez **Serveur local**. Dans un environnement de production, choisissez le type de serveur source approprié.
    1. Option de migration : **Valider et migrer**.
    1. Mode de migration : **Hors connexion**. 
    1. Sélectionnez le bouton **Suivant : Sélectionner un serveur d’exécution >**.
    1. Sélectionnez **Non** pour *Utiliser un serveur d’exécution*.
    1. Sélectionnez **Suivant : Se connecter à la source**.

    ![Configurez la migration de base de données hors connextion pour le serveur flexible Azure Database pour PostgreSQL.](./media/10-pgflex-migation-setup.png)

1. Migration à partir d’un serveur Azure Database pour PostgreSQ : dans l’onglet **Se connecter à la source**, entrez chaque champ comme suit :
    1. Nom du serveur : adresse du serveur que vous utilisez comme source.
    1. Port : port utilisé par votre instance de PostgreSQL sur votre serveur source (valeur par défaut 5432).
    1. Nom de connexion de l’administrateur du serveur : nom d’un utilisateur administrateur pour votre instance PostgreSQL (pgAdmin par défaut).
    1. Mot de passe : mot de passe de l’utilisateur administrateur PostgreSQL que vous avez spécifié à l’étape précédente.
    1. Mode SSL : Prefer.
    1. Cliquez sur l’option **Se connecter à la source** pour valider les détails de connectivité fournis.
    1. Cliquez sur le bouton **Suivant : Sélectionner la cible de migration** pour progresser.

1. Les détails de connectivité doivent être automatiquement terminés pour le serveur cible vers lequel nous procédons à la migration.
    1. Dans le champ mot de passe : entrez le mot de passe généré de manière aléatoire pour la connexion **pgAdmin** que vous avez créée avec le script bicep.
    1. Cliquez sur l’option **Se connecter à la cible** pour valider les détails de connectivité fournis.
    1. Cliquez sur le bouton **Suivant : Sélectionner une ou plusieurs bases de données pour la migration >** pour progresser.
1. Dans l’onglet **Sélectionner une ou plusieurs bases de données pour la migration**, sélectionnez **AdventureWorks** à partir du serveur source que vous souhaitez migrer vers le serveur flexible.

    ![Sélectionnez une ou plusieurs bases de données pour la migration de serveur flexible Azure Database pour PostgreSQL.](./media/10-pgflex-migation-dbSelection.png)

1. Cliquez sur le bouton **Suivant : Résumé >** pour progresser et passer en revue les données fournies.
1. Dans l’onglet **Résumé**, passez en revue les informations, puis cliquez sur le bouton **Démarrer la validation et la migration** pour démarrer la migration vers le serveur flexible.
1. Dans l’onglet **Migration**, vous pouvez surveiller la progression de la migration à l’aide du bouton **Actualiser** dans le menu supérieur pour afficher la progression dans le processus de validation et de migration.
    1. En cliquant sur l’activité **Migration-AdventureWorks**, vous pouvez afficher des informations détaillées sur la progression de l’activité de migration.
1. Une fois la migration terminée, vérifiez le serveur de destination. La base de données **AdventureWorks** doit également être répertoriée sous ce serveur.

Une fois le processus de migration terminé, nous pouvons effectuer des tâches post-migration telles que la validation des données dans la nouvelle base de données et la configuration de la haute disponibilité avant de pointer l’application vers la base de données et de l’activer à nouveau.

## Nettoyage de l’exercice

Le serveur Azure Database pour PostgreSQL que nous avons déployé dans cet exercice sera utilisé dans l’exercice suivant. Cela entraîne des frais, vous pouvez donc arrêter le serveur après cet exercice. Vous pouvez également supprimer le groupe de ressources **rg-learn-work-with-postgresql-eastus** pour supprimer toutes les ressources que nous avons déployées dans le cadre de cet exercice. Cela signifie que vous devez répéter les étapes de cet exercice pour effectuer l’exercice suivant.
