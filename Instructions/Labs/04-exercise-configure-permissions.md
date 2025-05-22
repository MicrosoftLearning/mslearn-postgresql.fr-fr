---
lab:
  title: Configurer les autorisations dans Azure Database pour PostgreSQL
  module: Secure Azure Database for PostgreSQL
---

# Configurer les autorisations dans Azure Database pour PostgreSQL

Dans ces exercices de labo, vous attribuez des rôles RBAC (contrôle d’accès en fonction du rôle) afin de contrôler l’accès aux ressources Azure Database pour PostgreSQL, mais également PostgreSQL GRANTS pour contrôler l’accès aux opérations de base de données.

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

## Créer un compte d’utilisateur dans Microsoft Entra ID

> &#128221; Dans la plupart des environnements de production ou de développement, il est très possible que vous n’ayez pas les privilèges de compte d’abonnement pour créer des comptes sur votre service Microsoft Entra ID. Dans ce cas, si votre organisation l’autorise, essayez de demander à votre administrateur Microsoft Entra ID de créer un compte de test pour vous. *Si vous ne parvenez pas à obtenir le compte de test Microsoft Entra, ignorez cette section et passez à la section **Accorder l’accès GRANT à Azure Database pour PostgreSQL***.

1. Dans le [portail Azure](https://portal.azure.com), connectez-vous en utilisant un compte Propriétaire et accédez à Microsoft Entra ID.

1. Sous **Gérer**, sélectionnez **Utilisateurs**.

1. En haut à gauche, sélectionnez **Nouvel utilisateur**, puis sélectionnez **Créer un utilisateur**.

1. Dans la page **Nouvel utilisateur**, entrez ces détails, puis sélectionnez **Créer** :
    - **Nom d’utilisateur principal :** choisissez un nom de principe.
    - **Nom d’affichage :** choisissez un nom d’affichage.
    - **Mot de passe :** décochez **Générer automatiquement un mot de passe**, puis entrez un mot de passe fort. Prenez note du nom de principal et du mot de passe.
    - Sélectionner **Vérifier + créer**

    > &#128161; Une fois la création de l’utilisateur ou de l’utilisatrice effectuée, notez le **Nom d’utilisateur principal ou d’utilisatrice principale** complet afin de pouvoir l’utiliser ultérieurement pour vous connecter.

### Attribuer le rôle Lecteur

1. Dans le portail Azure, sélectionnez **Toutes les ressources**, puis sélectionnez votre ressource Azure Database pour PostgreSQL.

1. Sélectionnez **Contrôle d’accès (IAM)**, puis sélectionnez **Attributions de rôles**. Le nouveau compte ne s’affiche pas dans la liste.

1. Sélectionnez **+ Ajouter**, puis **Ajouter une attribution de rôle**.

1. Sélectionnez le rôle **Lecteur**, puis sélectionnez **Suivant**.

1. Choisissez **+ Sélectionner des membres**, ajoutez le nouveau compte que vous avez ajouté à l’étape précédente à la liste des membres, puis sélectionnez **Suivant**.

1. Sélectionnez **Vérifier + attribuer**.

### Testez le rôle Lecteur

1. En haut à droite du portail Azure, sélectionnez votre compte d’utilisateur, puis sélectionnez **Se déconnecter**.

1. Connectez-vous à l’aide du nouvel utilisateur, avec le nom d’utilisateur principal et le mot de passe que vous avez notés. Remplacez le mot de passe par défaut si vous y êtes invité et notez le nouveau mot de passe.

1. Choisissez **Me demander plus tard** si vous recevez une invitation à utiliser l’authentification multifacteur.

1. Dans la page d’accueil du portail, sélectionnez **Toutes les ressources**, puis sélectionnez votre ressource Azure Database pour PostgreSQL.

1. Sélectionnez **Arrêter**. Une erreur s’affiche, car le rôle Lecteur permet d’afficher la ressource, mais pas de la modifier.

### Attribuer le rôle Contributeur

1. En haut à droite du portail Azure, sélectionnez le nouveau compte d’utilisateur, puis **Se déconnecter**.

1. Connectez-vous à l’aide de votre compte Propriétaire d’origine.

1. Accédez à votre ressource Azure Database pour PostgreSQL, puis sélectionnez **Access Control (IAM)**.

1. Sélectionnez **+ Ajouter**, puis **Ajouter une attribution de rôle**.

1. Sélectionnez **Rôles Administrateur privilégié**.

1. Sélectionnez le rôle **Contributeur**, puis sélectionnez **Suivant**.

1. Ajoutez le nouveau compte que vous avez précédemment ajouté à la liste des membres, puis sélectionnez **Suivant**.

1. Sélectionnez **Vérifier + attribuer**.

1. Sélectionnez **Attributions de rôles**. Le nouveau compte bénéficie désormais de deux attributions de rôles : Lecteur et Contributeur.

## Tester le rôle Contributeur

1. En haut à droite du portail Azure, sélectionnez votre compte d’utilisateur, puis sélectionnez **Se déconnecter**.

1. Connectez-vous à l’aide du nouveau compte, avec le nom d’utilisateur principal et le mot de passe que vous avez notés.

1. Dans la page d’accueil du portail, sélectionnez **Toutes les ressources**, puis sélectionnez votre ressource Azure Database pour MySQL.

1. Sélectionnez **Arrêter**, puis Sélectionnez **Oui**. Le serveur s’arrête cette fois sans erreur, car le nouveau compte dispose du rôle nécessaire.

1. Sélectionnez **Démarrer** pour vous assurer que le serveur PostgreSQL est prêt à passer aux étapes suivantes.

1. En haut à droite du portail Azure, sélectionnez le nouveau compte d’utilisateur, puis **Se déconnecter**.

1. Connectez-vous à l’aide de votre compte Propriétaire d’origine.

## Accorder l’accès GRANT à Azure Database pour PostgreSQL

Dans cette section, vous allez créer un rôle dans la base de données PostgreSQL et lui attribuer des autorisations pour accéder à la base de données. Vous testez également le nouveau rôle pour vous assurer qu’il dispose des autorisations appropriées.

1. Ouvrez Visual Studio Code s’il n’est pas déjà ouvert.

1. Affichez la palette de commandes (Ctrl+Maj+P) et sélectionnez **PGSQL : nouvelle requête**.

1. Sélectionnez votre connexion de serveur PostgreSQL dans la liste dans la palette de commandes. Si un mot de passe est demandé, entrez le mot de passe que vous avez généré précédemment.

1. Dans la fenêtre **Nouvelle requête**, remplacez la base de données par **zoodb**. Pour modifier la base de données, sélectionnez les points de suspension dans la barre de menus avec l’icône d’*exécution* et sélectionnez **Modifier la base de données PostgreSQL**. Sélectionnez `zoodb` dans la liste des bases de données. Vérifiez que la base de données est désormais définie sur `zoodb` en exécutant l’instruction **SELECT current_database();**.

1. Dans le volet de requête, copiez, mettez en surbrillance et exécutez l’instruction SQL suivante sur la base de données Postgres. Plusieurs rôles d’utilisation doivent être retournés, notamment le rôle **pgAdmin** que vous utilisez pour vous connecter :

    ```SQL
    SELECT rolname FROM pg_catalog.pg_roles;
    ```

1. Pour créer un rôle, exécutez ce code

    ```SQL
    -- Make sure to change the password to a complex one
    -- and replace the password in the script below
    CREATE ROLE dbuser WITH LOGIN NOSUPERUSER INHERIT CREATEDB NOCREATEROLE NOREPLICATION PASSWORD 'R3placeWithAComplexPW!';
    GRANT CONNECT ON DATABASE zoodb TO dbuser;
    ```

    > &#128221; Veillez à remplacer le mot de passe dans le script précédent par un mot de passe complexe.

1. Pour répertorier le nouveau rôle, réexécutez la requête SELECT dans **pg_catalog.pg_roles**. Le rôle **dbuser** doit être répertorié.

1. Pour permettre au nouveau rôle d’interroger et de modifier les données contenues dans la table **animal** de la base de données **zoodb**, procédez comme suit : 

    1. Dans la fenêtre **Nouvelle requête**, remplacez la base de données par **zoodb**. Pour modifier la base de données, sélectionnez les points de suspension dans la barre de menus avec l’icône d’*exécution* et sélectionnez **Modifier la base de données PostgreSQL**. Sélectionnez `zoodb` dans la liste des bases de données. Vérifiez que la base de données est désormais définie sur `zoodb` en exécutant l’instruction **SELECT current_database();**.

    1. exécutez ce code sur la base de données `zoodb` :

        ```SQL
        GRANT SELECT, INSERT, UPDATE, DELETE ON animal TO dbuser;
        ```

## Tester le nouveau rôle

Testons le nouveau rôle pour s’assurer qu’il dispose des autorisations appropriées.

1. Dans l’extension **PostgreSQL**, sélectionnez **+ Ajouter une connexion** pour ajouter une nouvelle connexion.

1. Dans la boîte de dialogue **NOUVELLE CONNEXION**, entrez les informations suivantes :

    - **Nom du serveur** : <your-server-name>.postgres.database.azure.com
    - **Type d’authentification** : Mot de passe
    - **Nom d’utilisateur ou d’utilisatrice** : dbuser
    - **Mot de passe** : mot de passe utilisé lors de la création du rôle.
    - Cochez la case **Enregistrer le mot de passe**.
    - **Nom de la base de données** : zoodb
    - **Nom de connexion** : <your-server-name> + « -dbuser »

1. Sélectionnez **Tester la connexion** pour tester la connexion. Si la connexion réussit, sélectionnez **Enregistrer et se connecter** pour enregistrer la connexion, sinon passez en revue les informations de connexion, puis réessayez.

1. Affichez la palette de commandes (Ctrl+Maj+P) et sélectionnez **PGSQL : nouvelle requête**. Sélectionnez la nouvelle connexion que vous avez créée dans la liste dans la palette de commandes. Si on vous demande un mot de passe, entrez le mot de passe que vous avez créé pour le nouveau rôle.

1. La connexion doit afficher la base de données **zoodb** par défaut. Si ce n’est pas le cas, vous pouvez modifier la base de données en **zoodb**. Pour modifier la base de données, sélectionnez les points de suspension dans la barre de menus avec l’icône d’*exécution* et sélectionnez **Modifier la base de données PostgreSQL**. Sélectionnez `zoodb` dans la liste des bases de données. Vérifiez que la base de données est désormais définie sur `zoodb` en exécutant l’instruction **SELECT current_database();**.

1. Dans la fenêtre **Nouvelle requête**, copiez, mettez en surbrillance et exécutez l’instruction SQL suivante sur la base de données **zoodb** :

    ```SQL
    SELECT * FROM animal;
    ```

1. Pour effectuer un test afin de déterminer si vous disposez du privilège UPDATE, copiez, mettez en surbrillance et exécutez ce code :

    ```SQL
    UPDATE animal SET name = 'Linda Lioness' WHERE ani_id = 7;
    SELECT * FROM animal;
    ```

1. Pour effectuer un test afin de déterminer si vous disposez du privilège DROP, exécutez ce code. En cas d’erreur, examinez le code d’erreur :

    ```SQL
    DROP TABLE animal;
    ```

1. Pour effectuer un test afin de déterminer si vous disposez du privilège GRANT, exécutez ce code :

    ```SQL
    GRANT ALL PRIVILEGES ON animal TO dbuser;
    ```

Ces tests montrent que le nouvel utilisateur ou la nouvelle utilisatrice peut exécuter des commandes DML (langage de manipulation de données) pour interroger et modifier les données. Toutefois, le nouvel utilisateur ou la nouvelle utilisatrice ne peut pas utiliser les commandes DDL (langage de définition de données) pour modifier le schéma. En outre, le nouvel utilisateur ou la nouvelle utilisatrice ne peut pas accorder de nouveaux privilèges (GRANT) pour contourner les autorisations.

## Nettoyage

1. Si vous n’avez plus besoin de ce serveur PostgreSQL pour d’autres exercices, pour éviter d’entraîner des coûts Azure inutiles, supprimez le groupe de ressources créé dans cet exercice.

1. Si vous souhaitez conserver le serveur PostgreSQL en cours d’exécution, vous pouvez le laisser en cours d’exécution. Sinon, vous pouvez arrêter le serveur pour éviter d’entraîner des coûts inutiles dans le terminal Bash. Exécutez la commande suivante pour arrêter le serveur :

    ```azurecli
    az postgres flexible-server stop --name <your-server-name> --resource-group $RG_NAME
    ```

    Remplacez `<your-server-name>` par le nom de votre serveur PostreSQL.

    > &#128221; Vous pouvez également arrêter le serveur depuis le portail Azure. Dans le portail Azure, accédez aux **groupes de ressources** et sélectionnez le groupe de ressources que vous avez créé précédemment. Sélectionnez le serveur PostgreSQL, puis sélectionnez **Arrêter** dans le menu.

1. Si nécessaire, supprimez le référentiel Git que vous avez cloné précédemment.

Vous avez terminé cet exercice. Vous avez appris à attribuer des rôles RBAC afin de contrôler l’accès aux ressources Azure Database pour PostgreSQL et PostgreSQL GRANTS pour contrôler l’accès aux opérations de base de données.

Vous avez également appris à créer un compte d’utilisateur dans Microsoft Entra ID et à lui attribuer les rôles Lecteur et Contributeur. Enfin, vous avez créé un rôle dans PostgreSQL et lui avez attribué des autorisations pour accéder à la base de données.