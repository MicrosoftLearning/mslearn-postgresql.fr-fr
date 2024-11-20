---
lab:
  title: Créer une procédure stockée dans Azure Database pour PostgreSQL
  module: Procedures and functions in PostgreSQL
---

# Créer une procédure stockée dans Azure Database pour PostgreSQL

Dans cet exercice, vous allez créer une procédure stockée.

## Avant de commencer

Pour effectuer cet exercice, vous avez besoin de votre propre abonnement Azure. Si vous n’avez pas d’abonnement Azure, vous pouvez demander un [essai gratuit d’Azure](https://azure.microsoft.com/free).

## Créer l’environnement de l’exercice

Dans cet exercice et dans tous les exercices ultérieurs, vous allez utiliser Bicep dans Azure Cloud Shell pour déployer votre serveur PostgreSQL.
Ignorez le déploiement des ressources et l’installation d’Azure Data Studio si vous lez avez déjà installés.

### Déployer des ressources dans votre abonnement Azure

Cette étape vous guide tout au long de l’utilisation de commandes Azure CLI à partir d’Azure Cloud Shell pour créer un groupe de ressources et exécuter un script Bicep pour déployer les services Azure nécessaires pour effectuer cet exercice dans votre abonnement Azure.

> Remarque
>
> Si vous effectuez plusieurs modules dans ce parcours d’apprentissage, vous pouvez partager l’environnement Azure entre eux. Dans ce cas, vous devez effectuer cette étape de déploiement de ressources une seule fois.

1. Ouvrez un navigateur web et accédez au [portail Azure](https://portal.azure.com/).

2. Dans la barre d’outils du portail Azure, sélectionnez l’icône **Cloud Shell** pour ouvrir un nouveau volet [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) en bas de la fenêtre de votre navigateur.

    ![Capture d’écran de la barre d’outils du portail Azure, avec l’icône Cloud Shell encadrée en rouge.](media/05-portal-toolbar-cloud-shell.png)

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

## Cloner le référentiel GitHub localement

Vérifiez que vous avez déjà cloné les scripts de labo à partir de [PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql.git). Si vous ne l’avez pas déjà fait, clonez ce référentiel localement de cette façon :

1. Ouvrez une ligne de commande/un terminal.
1. Exécutez la commande suivante :
    ```bash
    md .\DP3021Lab
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git .\DP3021Lab
    ```
    > REMARQUE
    > 
    > Si **Git** n’est pas installé, [téléchargez et installez l’application ***Git***](https://git-scm.com/download) et réessayez d’exécuter les commandes précédentes.

## Installer Azure Data Studio

Si Azure Data Studio n’est pas installé :

1. Dans un navigateur, accédez à [Télécharger et installer Azure Data Studio](/sql/azure-data-studio/download-azure-data-studio) et, sous la plateforme Windows, sélectionnez **Programme d’installation utilisateur (recommandé).** Le fichier exécutable est téléchargé dans votre dossier Téléchargements.
1. Sélectionnez **Ouvrir le fichier**.
1. Le contrat de licence s’affiche. Lisez et **acceptez le contrat**, puis sélectionnez **Suivant**.
1. Dans **Sélectionner des tâches supplémentaires**, sélectionnez **Ajouter à PATH** et tout autre ajout nécessaire. Cliquez sur **Suivant**.
1. La boîte de dialogue **Prêt à installer** s’affiche. Passez vos paramètres en revue. Sélectionnez **Précédent** pour apporter des modifications, ou sélectionnez **Installer**.
1. La boîte de dialogue **Fin de l’Assistant Installation d’Azure Data Studio** s’affiche. Sélectionnez **Terminer**. Azure Data Studio démarre.

## Installer l’extension PostgreSQL

Si vous n’avez pas installé l’extension PostgreSQL dans votre instance Azure Data Studio :

1. Ouvrez Azure Data Studio s’il n’est pas déjà ouvert.
1. Dans le menu de gauche, sélectionnez **Extensions** pour afficher le panneau Extensions.
1. Dans la barre de recherche, entrez **PostgreSQL**. L’icône de l’extension PostgreSQL pour Azure Data Studio s’affiche.
1. Sélectionnez **Installer**. L’extension s’installe.

## Se connecter au serveur flexible Azure Database pour PostgreSQL

1. Ouvrez Azure Data Studio s’il n’est pas déjà ouvert.
1. Dans le menu de gauche, sélectionnez **Connexions**.
1. Sélectionnez **Nouvelle connexion**.
1. Sous **Détails de la connexion**, dans **Type de connexion**, sélectionnez **PostgreSQL** dans la liste déroulante.
1. Dans **Nom du serveur**, entrez le nom complet du serveur tel qu’il apparaît sur le portail Azure.
1. Dans **Type d’authentification**, laissez Mot de passe.
1. Dans Nom d’utilisateur et Mot de passe, entrez le nom d’utilisateur **pgAdmin** et le **mot de passe administrateur aléatoire** que vous avez créé ci-dessus
1. Sélectionnez [ x ] Mémoriser le mot de passe.
1. Les champs restants sont facultatifs.
1. Sélectionnez **Connecter**. Vous êtes maintenant connecté au serveur Azure Database pour PostgreSQL.
1. Une liste des bases de données serveur s’affiche. Elle inclut les bases de données système et les bases de données utilisateur.
1. Si vous n’avez pas encore créé la base de données zoodb, sélectionnez **Fichier**, **Ouvrir un fichier**, puis accédez au dossier dans lequel vous avez enregistré les scripts. Sélectionnez **../Allfiles/Labs/02/Lab2_ZooDb.sql** et **Ouvri**.
   1. Mettez en surbrillance les instructions **DROP** et **CREATE**, puis exécutez-les.
   1. En haut de l’écran, utilisez la flèche déroulante pour voir les bases de données sur le serveur, comme zoodb et les bases de données système. Sélectionnez la base de données **zoodb**.
   1. Mettez en surbrillance les sections **Créer des tables**, **Créer des clés étrangères** et **Remplir des tables** et exécutez-les.
   1. Mettez en surbrillance les 3 instructions **SELECT** à la fin du script et exécutez-les pour vérifier que les tables ont été créées et remplies.

## Créer la procédure stockée repopulate_zoo()

1. En haut de l’écran, utilisez la flèche déroulante pour rendre la base de données zoodb active.
1. Dans Azure Data Studio, sélectionnez **Fichier**, **Ouvrir un fichier**, puis accédez aux scripts de labo. Sélectionnez **../Allfiles/Labs/03/Lab3_RepopulateZoo.sql**, puis **Ouvrir**. Reconnectez-vous au serveur si nécessaire.
1. Mettez en surbrillance la section dans **Créer une procédure stockée** de **CREATE PROCEDURE** à **END $$.**. Exécutez le texte mis en surbrillance.
1. Gardez Azure Data Studio ouvert avec le fichier ouvert, prêt pour l’exercice suivant.

## Créer la procédure stockée new_exhibit()

1. En haut de l’écran, utilisez la flèche déroulante pour rendre la base de données zoodb active.
1. Dans Azure Data Studio, sélectionnez **Fichier**, **Ouvrir un fichier**, puis accédez aux scripts de labo. Sélectionnez **../Allfiles/Labs/05/Lab5_StoredProcedure.sql**, puis **Ouvrir**. Reconnectez-vous au serveur si nécessaire.
1. Mettez en surbrillance la section dans **Créer une procédure stockée** de **CREATE PROCEDURE** à **END $$.**. Exécutez le texte mis en surbrillance. Lisez la procédure. Vous verrez qu’elle déclare certains paramètres d’entrée et les utilise pour insérer des lignes dans la table d’enclos et la table d’animaux.
1. Gardez Azure Data Studio ouvert avec le fichier ouvert, prêt pour l’exercice suivant.

## Appelez la procédure stockée

1. Mettez en surbrillance la section sous **Appeler la procédure stockée**. Exécutez le texte mis en surbrillance. Cela appelle la procédure stockée en transmettant des valeurs aux paramètres d’entrée.
1. Mettez en surbrillance et exécutez les deux instructions **SELECT**. Exécutez le texte mis en surbrillance. Vous pouvez voir qu’une nouvelle ligne a été insérée dans la table d’enclos, et cinq nouvelles lignes insérées dans la table d’animaux.

## Créer et appeler une fonction table

1. Dans Azure Data Studio, sélectionnez **Fichier**, **Ouvrir un fichier**, puis accédez aux scripts de labo. Sélectionnez **../Allfiles/Labs/05/Lab5_Table_Function.sql**, puis **Ouvrir**.
1. Mettez en surbrillance et exécutez la première instruction **SELECT** pour vérifier que la base de données zoodb est sélectionnée.
1. Mettez en surbrillance et exécutez la procédure stockée **repopulate_zoo()** pour commencer avec des données propres.
1. Mettez en surbrillance et exécutez la section sous **Créer une fonction table**. Cette fonction retourne une table appelée **enclosure_summary**. Lisez le code de fonction pour comprendre comment la table est remplie.
1. Mettez en surbrillance et exécutez les deux instructions SELECT, en passant un ID d’enclos différent chaque fois.
1. Mettez en surbrillance et exécutez la section sous **How to use a table valued function with a LATERAL join**. Cela montre la fonction table utilisée à la place d’un nom de table dans une jointure.

## Exercice facultatif - Fonctions intégrées

1. Dans Azure Data Studio, sélectionnez **Fichier**, **Ouvrir un fichier**, puis accédez aux scripts de labo. Sélectionnez **../Allfiles/Labs/05/Lab5_SimpleFunctions.sql**, puis **Ouvrir**.
1. Mettez en surbrillance et exécutez chaque fonction pour voir comment elle fonctionne. Reportez-vous à la [documentation en ligne](https://www.postgresql.org/docs/current/functions.html) pour plus d’informations sur chaque fonction.
1. Fermez Azure Data Studio sans enregistrer les scripts.
1. Arrêtez votre serveur Azure Database pour PostgreSQL afin de ne pas avoir à payer lorsque vous n’utilisez pas le serveur.

## Nettoyage

1. Supprimez le groupe de ressources créé dans cet exercice pour éviter des coûts Azure inutiles.
1. Si nécessaire, supprimez le dossier .\DP3021Lab.

