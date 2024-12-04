---
lab:
  title: Extraire des insights à l’aide d’Azure AI Language
  module: Extract insights using the Azure AI Language service with Azure Database for PostgreSQL
---

# Extraire des insights à l’aide d’Azure AI Language et d’Azure Database pour PostgreSQL

Rappelez-vous que la société d’annonces souhaite analyser les tendances du marché, comme les phrases ou les endroits les plus populaires. L’équipe a également l’intention d’améliorer les protections pour les informations d’identification personnelle (PII). Les données actuelles sont stockées dans un serveur flexible Azure Database pour PostgreSQL. Le budget du projet est restreint. La réduction des coûts initiaux et des coûts continus pour maintenir des mots clés et des balises est donc essentielle. Les développeurs se méfient des nombreuses formes que peuvent prendre les PII. Ils préférent une solution rentable et approuvée par rapport à une correspondance d’expression régulière interne.

Vous allez intégrer la base de données aux services Azure AI Language à l’aide de l’extension `azure_ai` . L’extension fournit des API de fonction SQL définies par l’utilisateur à plusieurs API Azure Cognitive Service, notamment :

- à l’extraction de phrases clés
- reconnaissance d’entité
- Reconnaissance des PII

Cette approche permettra à l’équipe de science des données de comparer rapidement la liste des données de popularité pour déterminer les tendances du marché. Il permettra également aux développeurs d’applications de présenter un texte sécurisé pour les PII dans des situations qui ne nécessitent pas d’accès. Le stockage d’entités identifiées permet une révision humaine en cas de recherche ou de reconnaissance de faux positifs pour des PII (considérer un élément comme des PII alors que ce n’est pas le cas).

À la fin, vous aurez quatre nouvelles colonnes dans la table `listings` avec des insights extraits :

- `key_phrases`
- `recognized_entities`
- `pii_safe_description`
- `pii_entities`

## Avant de commencer

Vous avez besoin d’un [abonnement Azure](https://azure.microsoft.com/free) avec des droits d’administration.

### Déployer des ressources dans votre abonnement Azure

Cette étape vous guide tout au long de l’utilisation de commandes Azure CLI à partir d’Azure Cloud Shell pour créer un groupe de ressources et exécuter un script Bicep pour déployer les services Azure nécessaires pour effectuer cet exercice dans votre abonnement Azure.

1. Ouvrez un navigateur web et accédez au [portail Azure](https://portal.azure.com/).

2. Dans la barre d’outils du portail Azure, sélectionnez l’icône **Cloud Shell** pour ouvrir un nouveau volet [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) en bas de la fenêtre de votre navigateur.

    ![Capture d’écran de la barre d’outils du portail Azure, avec l’icône Cloud Shell encadrée en rouge.](media/17-portal-toolbar-cloud-shell.png)

    Si vous y êtes invité, sélectionnez les options requises pour ouvrir un interpréteur de commandes *Bash*. Si vous avez utilisé une console *PowerShell* auparavant, remplacez-la par un interpréteur de commandes *Bash*.

3. À l’invite Cloud Shell, entrez ce qui suit pour cloner le référentiel GitHub contenant les ressources de l’exercice :

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

4. Exécutez ensuite trois commandes pour définir des variables afin de réduire les saisies redondantes lors de l’utilisation des commandes Azure CLI pour créer des ressources Azure. Les variables représentent le nom à affecter à votre groupe de ressources (`RG_NAME`), la région Azure (`REGION`) dans laquelle les ressources seront déployées et un mot de passe généré de manière aléatoire pour la connexion de l’administrateur PostgreSQL (`ADMIN_PASSWORD`).

    Dans la première commande, la région affectée à la variable correspondante est `eastus`, mais vous pouvez également la remplacer par un emplacement de votre choix. Toutefois, si vous remplacez la valeur par défaut, vous devez sélectionner une autre [région Azure qui prend en charge le résumé abstrait](https://learn.microsoft.com/azure/ai-services/language-service/summarization/region-support) pour être sûr de pouvoir effectuer toutes les tâches dans les modules de ce parcours d’apprentissage.

    ```bash
    REGION=eastus
    ```

    La commande suivante attribue le nom à utiliser pour le groupe de ressources qui hébergera toutes les ressources utilisées dans cet exercice. Le nom du groupe de ressources affecté à la variable correspondante est `rg-learn-postgresql-ai-$REGION`, où `$REGION` est l’emplacement que vous avez spécifié ci-dessus. Toutefois, vous pouvez le remplacer par tout autre nom de groupe de ressources de votre choix.

    ```bash
    RG_NAME=rg-learn-postgresql-ai-$REGION
    ```

    La commande finale génère de façon aléatoire un mot de passe pour la connexion d’administrateur PostgreSQL. **Copiez-le** dans un endroit sécurisé pour pouvoir l’utiliser ultérieurement et vous connecter à votre serveur flexible PostgreSQL.

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
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy.bicep" --parameters restore=false adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Le script de déploiement Bicep approvisionne les services Azure requis pour effectuer cet exercice dans votre groupe de ressources. Les ressources déployées incluent un serveur flexible Azure Database pour PostgreSQ, Azure OpenAI et un service Azure AI Language. Le script Bicep effectue également certaines étapes de configuration, telles que l’ajout des extensions `azure_ai` et `vector` à la _liste d’autorisation_ du serveur PostgreSQL (via le paramètre de serveur `azure.extensions`), la création d’une base de données nommée `rentals` sur le serveur et l’ajout d’un déploiement nommé `embedding` à l’aide du modèle `text-embedding-ada-002` à votre service Azure OpenAI. Notez que le fichier Bicep est partagé par tous les modules de ce parcours d’apprentissage. Vous pouvez donc utiliser uniquement certaines des ressources déployées dans certains exercices.

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

Dans cette tâche, vous allez vous connecter à la base de données `rentals` sur votre serveur Azure Database pour PostgreSQL à l’aide de l’[utilitaire de ligne de commande psql](https://www.postgresql.org/docs/current/app-psql.html) à partir d’[Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview).

1. Dans le [portail Azure](https://portal.azure.com/), accédez à votre serveur flexible Azure Database pour PostgreSQL nouvellement créé.

2. Dans le menu de ressource, dans **Paramètres**, sélectionnez **Bases de données**, puis **Connecter** pour la base de données `rentals`.

    ![Capture d’écran de la page Bases de données d’Azure SQL Database pour PostgreSQL. Le paramètre Bases de données et l’élément Connecter de la base de données de location sont encadrés en rouge.](media/17-postgresql-rentals-database-connect.png)

3. À l’invite « Mot de passe pour l’utilisateur pgAdmin » dans Cloud Shell, entrez le mot de passe généré de manière aléatoire pour la connexion **pgAdmin**.

    Une fois connecté, l’invite `psql` de la base de données `rentals` s’affiche.

4. Tout au long de cet exercice, vous continuerez à travailler dans Cloud Shell. Il peut donc être utile d’étendre le volet dans la fenêtre de votre navigateur en sélectionnant le bouton **Agrandir** en haut à droite du volet.

    ![Capture d’écran du volet Azure Cloud Shell avec le bouton Agrandir encadré en rouge.](media/17-azure-cloud-shell-pane-maximize.png)

## Configuration : configurer des extensions

Pour stocker et interroger des vecteurs et générer des incorporations, vous devez autoriser et activer deux extensions pour Azure Database pour PostgreSQL - Serveur flexible : `vector` et `azure_ai`.

1. Pour placer les deux extensions en liste d’autorisation, ajoutez `vector` et `azure_ai` au paramètre de serveur `azure.extensions`, conformément aux instructions fournies dans [Guide pratique pour utiliser les extensions PostgreSQL](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions).

2. Exécutez la commande SQL suivante pour installer l’extension `vector`. Pour obtenir des instructions détaillées, consultez [Comment activer et utiliser `pgvector` sur Azure Database pour PostgreSQL - Serveur flexible](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-use-pgvector#enable-extension).

    ```sql
    CREATE EXTENSION vector;
    ```

3. Pour installer l’extension `azure_ai`, exécutez la commande SQL suivante : Vous aurez besoin du point de terminaison et de la clé d’API pour la ressource ***Azure OpenAI***. Pour obtenir des instructions détaillées, consultez [Activer l’extension `azure_ai`](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/generative-ai-azure-overview#enable-the-azure_ai-extension).

    ```sql
    CREATE EXTENSION azure_ai;
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_openai.endpoint', 'https://<endpoint>.openai.azure.com');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_openai.subscription_key', '<API Key>');
    ```

4. Pour effectuer des appels sur votre service ***Azure AI Language*** à l’aide de l’extension `azure_ai`, vous devez fournir son point de terminaison et sa clé à l’extension. Depuis l’onglet du navigateur dans lequel Cloud Shell est ouvert, accédez à votre ressource de service de langage sur le [portail Azure](https://portal.azure.com/) et sélectionnez l’élément **Clés et point de terminaison** sous **Gestion des ressources** dans le menu de navigation de gauche.

5. Copiez les valeurs de point de terminaison et de clé d’accès, puis, dans les commandes ci-dessous, remplacez les jetons `{endpoint}` et `{api-key}` par les valeurs que vous avez copiées à partir du portail Azure. Exécutez les commandes à partir de l’invite de commandes `psql` dans Cloud Shell pour ajouter vos valeurs à la table `azure_ai.settings`.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '{api-key}');
    ```

## Remplissez la base de données avec des exemples de données

Avant d’explorer l’extension `azure_ai`, ajoutez quelques tables à la base de données `rentals` et remplissez-les avec des exemples de données afin d’avoir des informations à utiliser lorsque vous passez en revue les fonctionnalités de l’extension.

1. Exécutez les commandes suivantes afin de créer les tables `listings` et `reviews` pour stocker les données d’annonces de location et les données d’avis client :

    ```sql
    DROP TABLE IF EXISTS listings;
    
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

    ```sql
    DROP TABLE IF EXISTS reviews;
    
    CREATE TABLE reviews (
        id int,
        listing_id int, 
        date date,
        comments text
    );
    ```

2. Ensuite, utilisez la commande `COPY` pour charger des données à partir de fichiers CSV dans chaque table que vous avez créée ci-dessus. Commencez par exécuter la commande suivante pour remplir la table `listings` :

    ```sql
    \COPY listings FROM 'mslearn-postgresql/Allfiles/Labs/Shared/listings.csv' CSV HEADER
    ```

    La sortie de commande doit être `COPY 50`, indiquant que 50 lignes ont été écrites dans la table à partir du fichier CSV.

3. Enfin, exécutez la commande ci-dessous pour charger les avis des clients dans la table `reviews` :

    ```sql
    \COPY reviews FROM 'mslearn-postgresql/Allfiles/Labs/Shared/reviews.csv' CSV HEADER
    ```

    La sortie de commande doit être `COPY 354`, indiquant que 354 lignes ont été écrites dans la table à partir du fichier CSV.

Pour réinitialiser vos exemples de données, vous pouvez exécuter `DROP TABLE listings` et répéter la procédure.

## Extraire les phrases clés

1. Les phrases clés sont extraites en tant que `text[]`, comme indiqué par la fonction `pg_typeof` :

    ```sql
    SELECT pg_typeof(azure_cognitive.extract_key_phrases('The food was delicious and the staff were wonderful.', 'en-us'));
    ```

    Créez une colonne pour les résultats clés.

    ```sql
    ALTER TABLE listings ADD COLUMN key_phrases text[];
    ```

1. Remplissez la colonne par lots. Selon le quota, vous souhaiterez peut-être ajuster la valeur `LIMIT`. *N’hésitez pas à exécuter la commande autant de fois que vous le souhaitez*. Vous n’avez pas besoin de remplir toutes les lignes pour cet exercice.

    ```sql
    UPDATE listings
    SET key_phrases = azure_cognitive.extract_key_phrases(description)
    FROM (SELECT id FROM listings WHERE key_phrases IS NULL ORDER BY id LIMIT 100) subset
    WHERE listings.id = subset.id;
    ```

1. Interrogez les annonces selon des phrases clés :

    ```sql
    SELECT id, name FROM listings WHERE 'closet' = ANY(key_phrases);
    ```

    Vous obtiendrez des résultats semblables à ceux-ci, en fonction des annonces qui contiennent des phrases clés :

    ```sql
       id    |                name                
    ---------+-------------------------------------
      931154 | Met Tower in Belltown! MT2
      931758 | Hottest Downtown Address, Pool! MT2
     1084046 | Near Pike Place & Space Needle! MT2
     1084084 | The Best of the Best, Seattle! MT2
    ```

## Reconnaissance d’entité nommée

1. Les entités sont extraites en tant que `azure_cognitive.entity[]`, comme indiqué par la fonction `pg_typeof` :

    ```sql
    SELECT pg_typeof(azure_cognitive.recognize_entities('For more information, see Cognitive Services Compliance and Privacy notes.', 'en-us'));
    ```

    Créez une colonne pour les résultats clés.

    ```sql
    ALTER TABLE listings ADD COLUMN entities azure_cognitive.entity[];
    ```

2. Remplissez la colonne par lots. Ce processus peut prendre quelques minutes. Vous pouvez ajuster la valeur `LIMIT` en fonction du quota ou pour obtenir plus rapidement des résultats partiels. *N’hésitez pas à exécuter la commande autant de fois que vous le souhaitez*. Vous n’avez pas besoin de remplir toutes les lignes pour cet exercice.

    ```sql
    UPDATE listings
    SET entities = azure_cognitive.recognize_entities(description, 'en-us')
    FROM (SELECT id FROM listings WHERE entities IS NULL ORDER BY id LIMIT 500) subset
    WHERE listings.id = subset.id;
    ```

3. Vous pouvez maintenant interroger toutes les entités des annonces pour rechercher des propriétés avec sous-sol :

    ```sql
    SELECT id, name
    FROM listings, unnest(listings.entities) AS e
    WHERE e.text LIKE '%roof%deck%'
    LIMIT 10;
    ```

    Le résultat doit être semblable à celui-ci :

    ```sql
       id    |                name                
    ---------+-------------------------------------
      430610 | 3br/3ba. modern, roof deck, garage
      430610 | 3br/3ba. modern, roof deck, garage
     1214306 | Private Bed/bath in Home: green (A)
       74328 | Spacious Designer Condo
      938785 | Best Ocean Views By Pike Place! PA1
       23430 | 1 Bedroom Modern Water View Condo
      828298 | 2 Bedroom Sparkling City Oasis
      338043 | large modern unit & fab location
      872152 | Luxurious Local Lifestyle 2Bd/2+Bth
      116221 | Modern, Light-Filled Fremont Flat
    ```

## Reconnaissance des PII

1. Les entités sont extraites en tant que `azure_cognitive.pii_entity_recognition_result`, comme indiqué par la fonction `pg_typeof` :

    ```sql
    SELECT pg_typeof(azure_cognitive.recognize_pii_entities('For more information, see Cognitive Services Compliance and Privacy notes.', 'en-us'));
    ```

    Cette valeur est un type composite contenant du texte expurgé et un tableau d’entités PII, comme vérifié par :

    ```sql
    \d azure_cognitive.pii_entity_recognition_result
    ```

    Affichage :

    ```sql
         Composite type "azure_cognitive.pii_entity_recognition_result"
         Column    |           Type           | Collation | Nullable | Default 
    ---------------+--------------------------+-----------+----------+---------
     redacted_text | text                     |           |          | 
     entities      | azure_cognitive.entity[] |           |          | 
    ```

    Créez une colonne pour contenir le texte expurgé et un autre pour les entités reconnues :

    ```sql
    ALTER TABLE listings ADD COLUMN description_pii_safe text;
    ALTER TABLE listings ADD COLUMN pii_entities azure_cognitive.entity[];
    ```

2. Remplissez la colonne par lots. Ce processus peut prendre quelques minutes. Vous pouvez ajuster la valeur `LIMIT` en fonction du quota ou pour obtenir plus rapidement des résultats partiels. *N’hésitez pas à exécuter la commande autant de fois que vous le souhaitez*. Vous n’avez pas besoin de remplir toutes les lignes pour cet exercice.

    ```sql
    UPDATE listings
    SET
        description_pii_safe = pii.redacted_text,
        pii_entities = pii.entities
    FROM (SELECT id, description FROM listings WHERE description_pii_safe IS NULL OR pii_entities IS NULL ORDER BY id LIMIT 100) subset,
    LATERAL azure_cognitive.recognize_pii_entities(subset.description, 'en-us') as pii
    WHERE listings.id = subset.id;
    ```

3. Vous pouvez maintenant afficher des descriptions de référencement avec des PII potentielles expurgées :

    ```sql
    SELECT description_pii_safe
    FROM listings
    WHERE description_pii_safe IS NOT NULL
    LIMIT 1;
    ```

    Affichage :

    ```sql
    A lovely stone-tiled room with kitchenette. New full mattress futon bed. Fridge, microwave, kettle for coffee and tea. Separate entrance into book-lined mudroom. Large bathroom with Jacuzzi (shared occasionally with ***** to do laundry). Stone-tiled, radiant heated floor, 300 sq ft room with 3 large windows. The bed is queen-sized futon and has a full-sized mattress with topper. Bedside tables and reading lights on both sides. Also large leather couch with cushions. Kitchenette is off the side wing of the main room and has a microwave, and fridge, and an electric kettle for making coffee or tea. Kitchen table with two chairs to use for meals or as desk. Extra high-speed WiFi is also provided. Access to English Garden. The Ballard Neighborhood is a great place to visit: *10 minute walk to downtown Ballard with fabulous bars and restaurants, great ****** farmers market, nice three-screen cinema, and much more. *5 minute walk to the Ballard Locks, where ships enter and exit Puget Sound
    ```

4. Vous pouvez également identifier les entités reconnues dans les PII. Par exemple, en utilisant la même liste indiquée ci-dessus :

    ```sql
    SELECT entities
    FROM listings
    WHERE entities IS NOT NULL
    LIMIT 1;
    ```

    Affichage :

    ```sql
                            pii_entities                        
    -------------------------------------------------------------
    {"(hosts,PersonType,\"\",0.93)","(Sunday,DateTime,Date,1)"}
    ```

## Vérifier votre travail

Nous allons vérifier que les phrases clés extraites, les entités reconnues et les PII ont été remplies.

1. Vérifiez les phrases clés :

    ```sql
    SELECT COUNT(*) FROM listings WHERE key_phrases IS NOT NULL;
    ```

    Vous devriez voir quelque chose semblable à ceci, en fonction du nombre de lots que vous avez exécutés :

    ```sql
    count 
    -------
     100
    ```

2. Vérifiez les entités reconnues :

    ```sql
    SELECT COUNT(*) FROM listings WHERE entities IS NOT NULL;
    ```

    Un résultat semblable à celui-ci doit s’afficher :

    ```sql
    count 
    -------
     500
    ```

3. Vérifiez les PII expurgées :

    ```sql
    SELECT COUNT(*) FROM listings WHERE description_pii_safe IS NOT NULL;
    ```

    Si vous avez chargé un seul lot de 100, vous devriez voir :

    ```sql
    count 
    -------
     100
    ```

    Vous pouvez vérifier le nombre de référencements détectés par les PII :

    ```sql
    SELECT COUNT(*) FROM listings WHERE description != description_pii_safe;
    ```

    Un résultat semblable à celui-ci doit s’afficher :

    ```sql
    count 
    -------
        87
    ```

4. Vérifiez les entités PII détectées : conformément à ce qui précède, nous devrions en avoir 13 sans tableau PII vide.

    ```sql
    SELECT COUNT(*) FROM listings WHERE pii_entities IS NULL AND description_pii_safe IS NOT NULL;
    ```

    Résultat :

    ```sql
    count 
    -------
        13
    ```

## Nettoyage

Une fois cet exercice terminé, supprimez les ressources Azure que vous avez créées. La capacité configurée vous est facturée, et non la quantité de base de données utilisée. Suivez ces instructions pour supprimer votre groupe de ressources et toutes les ressources que vous avez créées pour ce labo.

1. Ouvrez un navigateur web et accédez au [portail Azure](https://portal.azure.com/), puis, dans la page d’accueil, dans Services Azure, sélectionnez **Groupes de ressources**.

    ![Capture d’écran du service Groupes de ressources encadré en rouge dans la section Services Azure du portail Azure.](media/17-azure-portal-home-azure-services-resource-groups.png)

2. Dans le filtre de n’importe quelle zone de recherche de champ, entrez le nom du groupe de ressources que vous avez créé pour ce labo, puis sélectionnez votre groupe de ressources dans la liste.

3. Dans la page **Vue d’ensemble** de votre groupe de ressources, sélectionnez **Supprimer le groupe de ressources**.

    ![Capture d’écran du panneau Vue d’ensemble du groupe de ressources avec le bouton Supprimer le groupe de ressources encadré en rouge.](media/17-resource-group-delete.png)

4. Dans la boîte de dialogue de confirmation, entrez le nom de votre groupe de ressources pour confirmer, puis sélectionnez **Supprimer**.
