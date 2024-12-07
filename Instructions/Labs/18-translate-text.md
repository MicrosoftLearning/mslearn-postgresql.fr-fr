---
lab:
  title: Traduire du texte à l’aide d’Azure AI Traducteur
  module: Translate Text using the Azure AI Translator and Azure Database for PostgreSQL
---

# Traduire du texte à l’aide d’Azure AI Traducteur

En tant que développeur principal pour Margie’s Travel, vous avez été invité à participer à un effort d’internationalisation. Aujourd’hui, toutes les annonces de location pour le service de location à court terme de l’entreprise sont en anglais. Vous souhaitez traduire ces annonces en plusieurs langues sans devoir déployer d’importants efforts de développement. Toutes vos données sont hébergées dans un serveur flexible Azure Database pour PostgreSQL et vous souhaitez utiliser Azure AI Services pour effectuer la traduction.

Dans cet exercice, vous allez traduire du texte en langue anglaise en différentes langues à l’aide du service Azure AI Translator via une base de données de serveur flexible Azure Database pour PostgreSQL.

## Avant de commencer

Vous avez besoin d’un [abonnement Azure](https://azure.microsoft.com/free) avec des droits d’administration.

### Déployer des ressources dans votre abonnement Azure

Cette étape vous guidera tout au long de l’utilisation des commandes Azure CLI à partir d’Azure Cloud Shell pour créer un groupe de ressources et exécuter un script Bicep afin de déployer les services Azure nécessaires pour effectuer cet exercice dans votre abonnement Azure.

1. Ouvrez un navigateur web et accédez au [portail Azure](https://portal.azure.com/).

2. Dans la barre d’outils du portail Azure, sélectionnez l’icône **Cloud Shell** pour ouvrir un nouveau volet [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) en bas de la fenêtre de votre navigateur.

    ![Capture d’écran de la barre d’outils du portail Azure, avec l’icône Cloud Shell encadrée en rouge.](media/11-portal-toolbar-cloud-shell.png)

    Si vous y êtes invité, sélectionnez les options requises pour ouvrir un interpréteur de commandes *Bash*. Si vous avez utilisé une console *PowerShell* auparavant, remplacez-la par un interpréteur de commandes *Bash*.

3. À l’invite Cloud Shell, entrez ce qui suit pour cloner le référentiel GitHub contenant des ressources d’exercice :

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

    Si vous avez déjà cloné ce référentiel GitHub dans un module précédent, il sera toujours disponible et le message d’erreur suivant pourrait s’afficher :

    ```bash
    fatal: destination path 'mslearn-postgresql' already exists and is not an empty directory.
    ```

    Si vous recevez ce message, vous pouvez passer en toute sécurité à l’étape suivante.

4. Exécutez ensuite trois commandes pour définir des variables afin de réduire les saisies redondantes lors de l’utilisation des commandes Azure CLI pour créer des ressources Azure. Les variables représentent le nom à affecter à votre groupe de ressources (`RG_NAME`), la région Azure (`REGION`) dans laquelle les ressources seront déployées et un mot de passe généré de manière aléatoire pour la connexion de l’administrateur PostgreSQL (`ADMIN_PASSWORD`).

    Dans la première commande, la région affectée à la variable correspondante est `eastus`, mais vous pouvez également la remplacer par un emplacement de votre choix. Toutefois, si vous remplacez la valeur par défaut, vous devez sélectionner une autre [région Azure qui prend en charge le résumé abstrait](https://learn.microsoft.com/azure/ai-services/language-service/summarization/region-support) pour être sûr de pouvoir effectuer toutes les tâches dans les modules de ce parcours d’apprentissage.

    ```bash
    REGION=eastus
    ```

    La commande suivante attribue le nom à utiliser pour le groupe de ressources qui hébergera toutes les ressources utilisées dans cet exercice. Le nom du groupe de ressources affecté à la variable correspondante est `rg-learn-postgresql-ai-$REGION`, où `$REGION` est l’emplacement que vous avez spécifié ci-dessus. Toutefois, vous pouvez le remplacer par tout autre nom de groupe de ressources de votre choix.

    ```bash
    RG_NAME=rg-learn-postgresql-ai-$REGION
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

7. Enfin, utilisez Azure CLI pour exécuter un script de déploiement Bicep afin d’approvisionner des ressources Azure dans votre groupe de ressources :

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-translate.bicep" --parameters restore=false adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Le script de déploiement Bicep approvisionne les services Azure requis pour effectuer cet exercice dans votre groupe de ressources. Les ressources déployées incluent un serveur flexible Azure Database pour PostgreSQL et un service Azure AI Translator. Le script Bicep effectue également certaines étapes de configuration, telles que l’ajout des extensions `azure_ai` et `vector` à la _liste d’autorisation_ du serveur PostgreSQL (via le paramètre de serveur azure.extensions), et la création d’une base de données nommée `rentals` sur le serveur. **Notez que le fichier Bicep diffère des autres modules de ce parcours d’apprentissage.**

    Le déploiement prend généralement plusieurs minutes. Vous pouvez le surveiller à partir de Cloud Shell ou accéder à la page **Déploiements** du groupe de ressources que vous avez créé ci-dessus et observer la progression du déploiement.

8. Fermez le volet Cloud Shell une fois votre déploiement de ressources terminé.

### Résoudre les erreurs de déploiement

Vous pouvez rencontrer quelques erreurs lors de l’exécution du script de déploiement Bicep.

- Si vous avez précédemment exécuté le script de déploiement Bicep pour ce parcours d’apprentissage et supprimé par la suite les ressources, vous pouvez recevoir un message d’erreur semblable à ce qui suit si vous tentez de réexécuter le script dans les 48 heures suivant la suppression des ressources :

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is '4e87a33d-a0ac-4aec-88d8-177b04c1d752'. See inner errors for details."}
    
    Inner Errors:
    {"code": "FlagMustBeSetForRestore", "message": "An existing resource with ID '/subscriptions/{subscriptionId}/resourceGroups/rg-learn-postgresql-ai-eastus/providers/Microsoft.CognitiveServices/accounts/{accountName}' has been soft-deleted. To restore the resource, you must specify 'restore' to be 'true' in the property. If you don't want to restore existing resource, please purge it first."}
    ```

    Si vous recevez ce message, modifiez la commande `azure deployment group create` ci-dessus pour que le paramètre `restore` soit défini sur `true` et réexécutez-la.

- Si la région sélectionnée est limitée à l’approvisionnement de ressources spécifiques, vous devez définir la variable `REGION` à un autre emplacement et réexécuter les commandes pour créer le groupe de ressources et exécuter le script de déploiement Bicep.

    ```bash
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- Si le script ne parvient pas à créer une ressource IA en raison de la nécessité d’accepter le contrat d’IA responsable, vous pouvez rencontrer l’erreur suivante. Dans ce cas, utilisez l’interface utilisateur du portail Azure pour créer une ressource Azure AI Services, puis réexécutez le script de déploiement.

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## Se connecter à votre base de données avec psql dans Azure Cloud Shell

Dans cette tâche, vous allez vous connecter à la base de données `rentals` sur votre serveur Azure Database pour PostgreSQL à l’aide de l’[utilitaire de ligne de commande psql](https://www.postgresql.org/docs/current/app-psql.html) à partir d’[Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview).

1. Dans le [portail Azure](https://portal.azure.com/), accédez à votre serveur flexible Azure Database pour PostgreSQL nouvellement créé.

2. Dans le menu de ressource, dans **Paramètres**, sélectionnez **Bases de données**, puis **Connecter** pour la base de données `rentals`.

    ![Capture d’écran de la page Bases de données d’Azure SQL Database pour PostgreSQL. Le paramètre Bases de données et l’élément Connecter de la base de données de location sont encadrés en rouge.](media/17-postgresql-rentals-database-connect.png)

3. À l’invite « Mot de passe pour l’utilisateur pgAdmin » dans Cloud Shell, entrez le mot de passe généré de manière aléatoire pour la connexion **pgAdmin**.

    Une fois connecté, l’invite `psql` de la base de données `rentals` s’affiche.

4. Tout au long de cet exercice, vous continuerez à travailler dans Cloud Shell. Il peut donc être utile d’étendre le volet dans la fenêtre de votre navigateur en sélectionnant le bouton **Agrandir** en haut à droite du volet.

    ![Capture d’écran du volet Azure Cloud Shell avec le bouton Agrandir encadré en rouge.](media/17-azure-cloud-shell-pane-maximize.png)

## Remplir la base de données avec des données d’annonces

Vous devez disposer de données d’annonces en langue anglaise pour les traduire. Si vous n’avez pas créé la table `listings` dans la base de données `rentals` dans un module précédent, suivez ces instructions pour la créer.

1. Exécutez les commandes suivantes pour créer la table `listings` et stocker les données d’annonces des locations :

    ```sql
    CREATE TABLE listings (
        id int,
        name varchar(100),
        description text,
        property_type varchar(25),
        room_type varchar(30),
        price numeric,
        weekly_price numeric
    );
    ```

2. Ensuite, utilisez la commande `COPY` pour charger des données à partir de fichiers CSV dans chaque table que vous avez créée ci-dessus. Commencez par exécuter la commande suivante pour remplir la table `listings` :

    ```sql
    \COPY listings FROM 'mslearn-postgresql/Allfiles/Labs/Shared/listings.csv' CSV HEADER
    ```

    La sortie de commande doit être `COPY 50`, indiquant que 50 lignes ont été écrites dans la table à partir du fichier CSV.

## Créer des tables supplémentaires pour la traduction

Les données `listings` sont en place, mais vous avez besoin de deux tables supplémentaires pour effectuer la traduction.

1. Exécutez les commandes suivantes pour créer les tables `languages` et `listing_translations`.

    ```sql
    CREATE TABLE languages (
        code VARCHAR(7) NOT NULL PRIMARY KEY
    );
    ```

    ```sql
    CREATE TABLE listing_translations(
        id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
        listing_id INT,
        language_code VARCHAR(7),
        description TEXT
    );
    ```

2. Ensuite, insérez une ligne par langue pour la traduction. Dans ce cas, vous allez créer des lignes pour cinq langues : allemand, chinois simplifié, hindi, hongrois et swahili.

    ```sql
    INSERT INTO languages(code)
    VALUES
        ('de'),
        ('zh-Hans'),
        ('hi'),
        ('hu'),
        ('sw');
    ```

    La sortie de commande doit être `INSERT 0 5`, indiquant que vous avez inséré cinq nouvelles lignes dans la table.

## Installer et configurer l’extension `azure_ai`

Avant d’utiliser l’extension `azure_ai`, vous devez l’installer dans votre base de données et la configurer pour vous connecter à vos ressources Azure AI Services. L’extension `azure_ai` vous permet d’intégrer les services Azure OpenAI et Azure AI Language dans votre base de données. Pour activer l’extension dans votre base de données, procédez comme suit :

1. Exécutez la commande suivante à l’invite `psql` pour vérifier que les extensions `azure_ai` et `vector` ont été correctement ajoutées à la _liste d’autorisation_ de votre serveur par le script de déploiement Bicep que vous avez exécuté lors de la configuration de votre environnement :

    ```sql
    SHOW azure.extensions;
    ```

    La commande affiche la liste des extensions sur la _liste d’autorisation_ du serveur. Si tout a été correctement installé, votre sortie doit inclure `azure_ai` et `vector`, comme suit :

    ```sql
     azure.extensions 
    ------------------
     azure_ai,vector
    ```

    Avant qu’une extension ne puisse être installée et utilisée dans un serveur flexible Azure Database pour PostgreSQL, elle doit être ajoutée à la _liste d’autorisation_ du serveur, comme décrit dans [Guide pratique pour utiliser les extensions PostgreSQL](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions).

2. Vous pouvez désormais installer l’extension `azure_ai` à l’aide de la commande [CREATE EXTENSION](https://www.postgresql.org/docs/current/sql-createextension.html).

    ```sql
    CREATE EXTENSION IF NOT EXISTS azure_ai;
    ```

    `CREATE EXTENSION` charge une nouvelle extension dans la base de données en exécutant son fichier de script. Ce script crée généralement de nouveaux objets SQL tels que des fonctions, des types de données et des schémas. Une erreur est générée si une extension du même nom existe déjà. L’ajout de `IF NOT EXISTS` permet à la commande de s’exécuter sans générer d’erreur si elle est déjà installée.

3. Vous devez ensuite utiliser la fonction `azure_ai.set_setting()` pour configurer la connexion à votre service Azure AI Traducteur. À l’aide de l’onglet du navigateur dans lequel Cloud Shell est ouvert, réduisez ou restaurez le volet Cloud Shell, puis accédez à votre ressource Azure AI Traducteur dans le [portail Azure](https://portal.azure.com/). Une fois que vous êtes sur la page de ressources Azure AI Traducteur, dans le menu des ressources, dans la section **Gestion des ressources**, sélectionnez **Clés et point de terminaison**, puis copiez l’une des clés disponibles, votre région et votre point de terminaison de traduction de documents.

    ![Capture d’écran de la page Clés et points de terminaison du service Azure AI Traducteur. Les boutons de copie de la clé 1, de la région et du point de terminaison de la traduction de documents sont encadrés en rouge.](media/18-azure-ai-translator-keys-and-endpoint.png)

    Vous pouvez utiliser soit `KEY 1`, soit `KEY 2`. Avoir toujours deux clés vous permet de faire pivoter et de régénérer en toute sécurité les clés sans provoquer d’interruption de service.

4. Configurez les paramètres `azure_cognitive` pour pointer vers votre point de terminaison AI Traducteur, votre clé d’abonnement et votre région. La valeur de `azure_cognitive.endpoint` correspondra à l’URL de traduction de documents de votre service. La valeur de `azure_cognitive.subscription_key` correspondra à la clé 1 ou à la clé 2. La valeur de `azure_cognitive.region` correspondra à la région de l’instance Azure AI Traducteur.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint','https://<YOUR_ENDPOINT>.cognitiveservices.azure.com/');
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '<YOUR_KEY>');
    SELECT azure_ai.set_setting('azure_cognitive.region', '<YOUR_REGION>');
    ```

## Créer une procédure stockée pour traduire les données d’annonces

Pour remplir la table de traduction linguistique, vous allez créer une procédure stockée pour charger des données par lots.

1. Exécutez la commande suivante à l’invite `psql` pour créer une procédure stockée nommée `translate_listing_descriptions`.

    ```sql
    CREATE OR REPLACE PROCEDURE translate_listing_descriptions(max_num_listings INT DEFAULT 10)
    LANGUAGE plpgsql
    AS $$
    BEGIN
        WITH batch_to_load(id, description) AS
        (
            SELECT id, description
            FROM listings l
            WHERE NOT EXISTS (SELECT * FROM listing_translations ll WHERE ll.listing_id = l.id)
            LIMIT max_num_listings
        )
        INSERT INTO listing_translations(listing_id, language_code, description)
        SELECT b.id, l.code, (unnest(tr.translations)).TEXT
        FROM batch_to_load b
            CROSS JOIN languages l
            CROSS JOIN LATERAL azure_cognitive.translate(b.description, l.code) tr;
    END;
    $$;
    ```

    Cette procédure stockée chargera un lot de 5 enregistrements, traduira la description dans chaque langue que vous sélectionnez, puis insèrera les descriptions traduites dans la table `listing_translations`.

2. Exécutez la procédure stockée à l’aide de la commande SQL suivante :

    ```sql
    CALL translate_listing_descriptions(10);
    ```

    Cet appel prendra environ une seconde par annonce de location pour effectuer la traduction en cinq langues, de sorte que chaque exécution doit prendre environ 10 secondes. La sortie de commande doit être `CALL`, indiquant que l’appel de procédure stockée a réussi.

3. Appelez la procédure stockée quatre fois de plus, vous aurez alors appelé cette procédure cinq fois. Cela génère des traductions pour chaque annonce dans la table.

4. Exécutez le script suivant pour obtenir le nombre de traductions d’annonces.

    ```sql
    SELECT COUNT(*) FROM listing_translations;
    ```

    L’appel doit retourner une valeur de 250, indiquant que chaque annonce a été traduite en cinq langues. Vous pouvez analyser davantage les données en interrogeant la table `listing_translations`.

## Créer une procédure pour ajouter une nouvelle annonce avec des traductions

Vous disposez d’une procédure stockée pour traduire les annonces existantes, mais vos plans d’internationalisation nécessitent également la traduction de nouvelles annocnes au fur et à mesure qu’elles sont entrés. Pour ce faire, vous allez créer une autre procédure stockée.

1. Exécutez la commande suivante à l’invite `psql` pour créer une procédure stockée nommée `add_listing`.

    ```sql
    CREATE OR REPLACE PROCEDURE add_listing(id INT, name VARCHAR(255), description TEXT)
    LANGUAGE plpgsql
    AS $$
    DECLARE
    listing_id INT;
    BEGIN
        INSERT INTO listings(id, name, description)
        VALUES(id, name, description);

        INSERT INTO listing_translations(listing_id, language_code, description)
        SELECT id, l.code, (unnest(tr.translations)).TEXT
        FROM languages l
            CROSS JOIN LATERAL azure_cognitive.translate(description, l.code) tr;
    END;
    $$;
    ```

    Cette procédure stockée insère une ligne dans la table `listings`. Ensuite, elle traduit la description pour chaque langue de la table `languages` et insère ces traductions dans la table `listing_translations`.

2. Exécutez la procédure stockée à l’aide de la commande SQL suivante :

    ```sql
    CALL add_listing(51, 'A Beautiful Home', 'This is a beautiful home in a great location.');
    ```

    La sortie de commande doit être `CALL`, indiquant que l’appel de procédure stockée a réussi.

3. Exécutez le script suivant pour obtenir les traductions de votre nouvelle annonce.

    ```sql
    SELECT l.id, l.name, l.description, lt.language_code, lt.description AS translated_description
    FROM listing_translations lt
        INNER JOIN listings l ON lt.listing_id = l.id
    WHERE l.name = 'A Beautiful Home';
    ```

    L’appel doit retourner cinq lignes, avec des valeurs similaires à la table suivante.

    ```sql
     id  | listing_id | language_code |                    description                     
    -----+------------+---------------+------------------------------------------------------
     126 |          2 | de            | Dies ist ein schönes Haus in einer großartigen Lage.
     127 |          2 | zh-Hans       | 这是一个美丽的家，地理位置优越。
     128 |          2 | hi            | यह एक महान स्थान में एक सुंदर घर है।
     129 |          2 | hu            | Ez egy gyönyörű otthon egy nagyszerű helyen.
     130 |          2 | sw            | Hii ni nyumba nzuri katika eneo kubwa.
    ```

## Nettoyage

Une fois cet exercice terminé, supprimez les ressources Azure que vous avez créées. La capacité configurée vous est facturée, et non la quantité de base de données utilisée. Suivez ces instructions pour supprimer votre groupe de ressources et toutes les ressources que vous avez créées pour ce labo.

1. Ouvrez un navigateur web et accédez au [portail Azure](https://portal.azure.com/), puis, dans la page d’accueil, dans Services Azure, sélectionnez **Groupes de ressources**.

    ![Capture d’écran du service Groupes de ressources encadré en rouge dans la section Services Azure du portail Azure.](media/17-azure-portal-home-azure-services-resource-groups.png)

2. Dans le filtre de n’importe quelle zone de recherche de champ, entrez le nom du groupe de ressources que vous avez créé pour ce labo, puis sélectionnez le groupe de ressources dans la liste.

3. Dans la page **Vue d’ensemble** de votre groupe de ressources, sélectionnez **Supprimer le groupe de ressources**.

    ![Capture d’écran du panneau Vue d’ensemble du groupe de ressources avec le bouton Supprimer le groupe de ressources encadré en rouge.](media/17-resource-group-delete.png)

4. Dans la boîte de dialogue de confirmation, entrez le nom de votre groupe de ressources pour confirmer, puis sélectionnez **Supprimer**.
