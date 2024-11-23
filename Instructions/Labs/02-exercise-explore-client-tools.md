---
lab:
  title: Explorer PostgreSQL avec des outils clients
  module: Understand client-server communication in PostgreSQL
---

# Explorer PostgreSQL avec des outils clients

Dans cet exercice, vous allez télécharger et installer psql et Azure Data Studio. Si vous avez déjà installé Azure Data Studio sur votre ordinateur, vous pouvez passer directement à la section « Se connecter au serveur flexible Azure Database pour PostgreSQL ».

## Avant de commencer

Pour effectuer cet exercice, vous avez besoin de votre propre abonnement Azure. Si vous n’avez pas d’abonnement Azure, vous pouvez demander un [essai gratuit d’Azure](https://azure.microsoft.com/free).

## Créer l’environnement de l’exercice

Dans cet exercice et dans tous les exercices ultérieurs, vous allez utiliser Bicep dans Azure Cloud Shell pour déployer votre serveur PostgreSQL.

### Déployer des ressources dans votre abonnement Azure

Cette étape vous guide tout au long de l’utilisation de commandes Azure CLI à partir d’Azure Cloud Shell pour créer un groupe de ressources et exécuter un script Bicep pour déployer les services Azure nécessaires pour effectuer cet exercice dans votre abonnement Azure.

> Remarque
>
> Si vous effectuez plusieurs modules dans ce parcours d’apprentissage, vous pouvez partager l’environnement Azure entre eux. Dans ce cas, vous devez effectuer cette étape de déploiement de ressources une seule fois.

1. Ouvrez un navigateur web et accédez au [portail Azure](https://portal.azure.com/).

2. Dans la barre d’outils du portail Azure, sélectionnez l’icône **Cloud Shell** pour ouvrir un nouveau volet [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) en bas de la fenêtre de votre navigateur.

    ![Capture d’écran de la barre d’outils du portail Azure, avec l’icône Cloud Shell encadrée en rouge.](media/02-portal-toolbar-cloud-shell.png)

3. Si vous y êtes invité, sélectionnez les options requises pour ouvrir un interpréteur de commandes *Bash*. Si vous avez déjà utilisé une console *PowerShell*, remplacez-la par un interpréteur de commandes *Bash*.

4. À l’invite Cloud Shell, entrez ce qui suit pour cloner le référentiel GitHub contenant des ressources d’exercice :

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

5. Exécutez ensuite trois commandes pour définir des variables afin de réduire les saisies redondantes lors de l’utilisation des commandes Azure CLI pour créer des ressources Azure. Les variables représentent le nom à affecter à votre groupe de ressources (`RG_NAME`), la région Azure (`REGION`) dans laquelle les ressources seront déployées et un mot de passe généré de manière aléatoire pour la connexion de l’administrateur PostgreSQL (`ADMIN_PASSWORD`).

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

6. Si vous avez accès à plusieurs abonnements Azure et que votre abonnement par défaut n’est pas celui dans lequel vous souhaitez créer le groupe de ressources et d’autres ressources pour cet exercice, exécutez cette commande pour définir l’abonnement approprié, en remplaçant le jeton `<subscriptionName|subscriptionId>` par le nom ou l’ID de l’abonnement que vous souhaitez utiliser :

    ```azurecli
    az account set --subscription <subscriptionName|subscriptionId>
    ```

7. Exécutez la commande Azure CLI suivante pour créer un groupe de ressources :

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

8. Enfin, utilisez Azure CLI pour exécuter un script de déploiement Bicep pour approvisionner des ressources Azure dans votre groupe de ressources :

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
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

## Outils clients pour se connecter à PostgreSQL

### Se connecter à Azure Database pour PostgreSQL avec psql

Vous pouvez installer psql localement ou vous connecter à partir du portail Azure, ce qui ouvre Cloud Shell et vous invite à entrer le mot de passe du compte d’administrateur.

#### Connexion locale

1. Installez pymssql à partir [d’ici](https://sbp.enterprisedb.com/getfile.jsp?fileid=1258893).
    1. Dans l’Assistant Installation, lorsque vous atteignez la boîte de dialogue **Sélectionner des composants**, sélectionnez **Outils en ligne de commande**.
    > Remarque
    >
    > Pour vérifier si **psql** est déjà installé dans votre environnement, ouvrez une ligne de commande ou un terminal et exécutez la commande ***psql***. Si un message tel que « *psql : erreur : connexion au serveur sur socket...*  » est renvoyé, cela signifie que l’outil **psql** est déjà installé dans votre environnement et qu’il n’est pas nécessaire de le réinstaller.



1. Affichez une ligne de commande.
1. La syntaxe de connexion au serveur est la suivante :

    ```sql
    psql --h <servername> --p <port> -U <username> <dbname>
    ```

1. À l’invite de commandes, entrez **`--host=<servername>.postgres.database.azure.com`** où `<servername>` est le nom du serveur Azure Database pour PostgreSQL créé précédemment.
    1. Vous trouverez le nom du serveur dans **Vue d’ensemble** dans le portail Azure ou en tant que sortie du script bicep.

    ```sql
   psql -h <servername>.postgres.database.azure.com -p 5432 -U pgAdmin postgres
    ```

    1. Le mot de passe du compte Administrateur que vous venez de copier vous sera alors demandé.

1. Pour créer une base de données vide à l’invite, tapez :

    ```sql
    CREATE DATABASE mypgsqldb;
    ```

1. À l’invite, exécutez la commande suivante pour basculer la connexion sur la base de données **mypgsqldb** nouvellement créée :

    ```sql
    \c mypgsqldb
    ```

1. Maintenant que vous êtes connecté au serveur et que vous avez créé une base de données, vous pouvez exécuter des requêtes SQL courantes, par exemple pour créer des tables dans la base de données :

    ```sql
    CREATE TABLE inventory (
        id serial PRIMARY KEY,
        name VARCHAR(50),
        quantity INTEGER
        );
    ```

1. Charger des données dans les tables

    ```sql
    INSERT INTO inventory (id, name, quantity) VALUES (1, 'banana', 150);
    INSERT INTO inventory (id, name, quantity) VALUES (2, 'orange', 154);
    ```

1. Interroger et mettre à jour les données des tables

    ```sql
    SELECT * FROM inventory;
    ```

1. Mettez à jour les données dans les tables.

    ```sql
    UPDATE inventory SET quantity = 200 WHERE name = 'banana';
    ```

## Installer Azure Data Studio

> Remarque
>
> Si Azure Data Studio est déjà installé, passez à l’étape *Installer l’extension PostgreSQL*.

Pour installer Azure Data Studio afin de l’utiliser avec Azure Database pour PostgreSQL :

1. Dans un navigateur, accédez à [Télécharger et installer Azure Data Studio](https://go.microsoft.com/fwlink/?linkid=2282284) et, sous la plateforme Windows, sélectionnez **Programme d’installation utilisateur (recommandé).** Le fichier exécutable est téléchargé dans votre dossier Téléchargements.
1. Sélectionnez **Ouvrir le fichier**.
1. Le contrat de licence s’affiche. Lisez et **acceptez le contrat**, puis sélectionnez **Suivant**.
1. Dans **Sélectionner des tâches supplémentaires**, sélectionnez **Ajouter à PATH** et tout autre ajout nécessaire. Cliquez sur **Suivant**.
1. La boîte de dialogue **Prêt à installer** s’affiche. Passez vos paramètres en revue. Sélectionnez **Précédent** pour apporter des modifications, ou sélectionnez **Installer**.
1. La boîte de dialogue **Fin de l’Assistant Installation d’Azure Data Studio** s’affiche. Sélectionnez **Terminer**. Azure Data Studio démarre.

## Installer l’extension PostgreSQL

1. Ouvrez Azure Data Studio s’il n’est pas déjà ouvert.
2. Dans le menu de gauche, sélectionnez **Extensions** pour afficher le panneau Extensions.
3. Dans la barre de recherche, entrez **PostgreSQL**. L’icône de l’extension PostgreSQL pour Azure Data Studio s’affiche.
   
![Capture d’écran de l’extension PostgreSQL pour Azure Data Studio](media/02-postgresql-extension.png)
   
4. Sélectionnez **Installer**. L’extension s’installe.

## Se connecter au serveur flexible Azure Database pour PostgreSQL

1. Ouvrez Azure Data Studio s’il n’est pas déjà ouvert.
2. Dans le menu de gauche, sélectionnez **Connexions**.
   
![Capture d’écran montrant les connexions dans Azure Data Studio](media/02-connections.png)

3. Sélectionnez **Nouvelle connexion**.
   
![Capture d’écran montrant comment créer une connexion dans Azure Data Studio](media/02-create-connection.png)

4. Sous **Détails de la connexion**, dans **Type de connexion**, sélectionnez **PostgreSQL** dans la liste déroulante.
5. Dans **Nom du serveur**, entrez le nom complet du serveur tel qu’il apparaît sur le portail Azure.
6. Dans **Type d’authentification**, laissez Mot de passe.
7. Dans Nom d’utilisateur et Mot de passe, entrez le nom d’utilisateur **pgAdmin** et le **mot de passe administrateur aléatoire** que vous avez créé ci-dessus
8. Sélectionnez [ x ] Mémoriser le mot de passe.
9. Les champs restants sont facultatifs.
10. Sélectionnez **Connecter**. Vous êtes maintenant connecté au serveur Azure Database pour PostgreSQL.
11. Une liste des bases de données serveur s’affiche. Elle inclut les bases de données système et les bases de données utilisateur.

## Créer la base de données zoo

1. Accédez au dossier avec vos fichiers de script d’exercice ou téléchargez **Lab2_ZooDb.sql** à partir de [MSLearn PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql/tree/main/Allfiles/Labs/02).
1. Ouvrez Azure Data Studio s’il n’est pas déjà ouvert.
1. Sélectionnez **Fichier**, **Ouvrir le fichier**, puis accédez au dossier dans lequel vous avez enregistré les scripts. Sélectionnez **../Allfiles/Labs/02/Lab2_ZooDb.sql** et **Ouvri**.
   1. Mettez en surbrillance les instructions **DROP** et **CREATE**, puis exécutez-les.
   1. En haut de l’écran, utilisez la flèche déroulante pour voir les bases de données sur le serveur, comme zoodb et les bases de données système. Sélectionnez la base de données **zoodb**.
   1. Mettez en surbrillance les sections **Créer des tables**, **Créer des clés étrangères** et **Remplir des tables** et exécutez-les.
   1. Mettez en surbrillance les 3 instructions **SELECT** à la fin du script et exécutez-les pour vérifier que les tables ont été créées et remplies.
