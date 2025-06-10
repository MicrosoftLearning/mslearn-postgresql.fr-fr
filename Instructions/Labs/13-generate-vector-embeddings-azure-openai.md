---
lab:
  title: Générer des incorporations vectorielles avec Azure OpenAI
  module: Enable Semantic Search with Azure Database for PostgreSQL
---

# Générer des incorporations vectorielles avec Azure OpenAI

Pour effectuer des recherches sémantiques, vous devez d’abord générer des vecteurs d’incorporation à partir d’un modèle, les stocker dans une base de données vectorielle, puis interroger les incorporations. Vous allez créer une base de données, la remplir avec des exemples de données et exécuter des recherches sémantiques sur ces annonces.

À la fin de cet exercice, vous disposerez d’une instance de serveur flexible Azure Database pour PostgreSQL avec les extensions `vector` et `azure_ai` activées. Vous allez générer des incorporations pour la table `listings` du jeu de données [Seattle Airbnb Open Data](https://www.kaggle.com/datasets/airbnb/seattle?select=listings.csv). Vous allez également exécuter des recherches sémantiques sur ces annonces en générant le vecteur d’incorporation d’une requête et en effectuant une recherche de distance de cosinus vectorielle.

## Avant de commencer

Vous avez besoin d’un [abonnement Azure](https://azure.microsoft.com/free) avec des droits d’administration.

### Déployer des ressources dans votre abonnement Azure

Cette étape vous guide tout au long de l’utilisation de commandes Azure CLI à partir d’Azure Cloud Shell pour créer un groupe de ressources et exécuter un script Bicep pour déployer les services Azure nécessaires pour effectuer cet exercice dans votre abonnement Azure.

> **Remarque** : si vous effectuez plusieurs modules dans ce parcours d’apprentissage, vous pouvez partager l’environnement Azure entre eux. Dans ce cas, vous devez effectuer cette étape de déploiement de ressources une seule fois.

1. Ouvrez un navigateur web et accédez au [portail Azure](https://portal.azure.com/).

2. Dans la barre d’outils du portail Azure, sélectionnez l’icône **Cloud Shell** pour ouvrir un nouveau volet [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) en bas de la fenêtre de votre navigateur.

    ![Capture d’écran de la barre d’outils du portail Azure, avec l’icône Cloud Shell encadrée en rouge.](media/13-portal-toolbar-cloud-shell.png)

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

2. Dans le menu de ressource, dans **Paramètres**, sélectionnez **Bases de données**, puis **Connecter** pour la base de données `rentals`. Notez que la sélection de **Se connecter** ne vous connecte pas réellement à la base de données ; elle fournit simplement des instructions pour se connecter à la base de données en utilisant différentes méthodes. Consultez les instructions pour **Se connecter depuis le navigateur ou localement** et utilisez-les pour vous connecter en utilisant Azure Cloud Shell.

    ![Capture d’écran de la page Bases de données d’Azure SQL Database pour PostgreSQL. Le paramètre Bases de données et l’élément Connecter de la base de données de location sont encadrés en rouge.](media/13-postgresql-rentals-database-connect.png)

3. À l’invite « Mot de passe pour l’utilisateur pgAdmin » dans Cloud Shell, entrez le mot de passe généré de manière aléatoire pour la connexion **pgAdmin**.

    Une fois connecté, l’invite `psql` de la base de données `rentals` s’affiche.

4. Tout au long de cet exercice, vous continuerez à travailler dans Cloud Shell. Il peut donc être utile d’étendre le volet dans la fenêtre de votre navigateur en sélectionnant le bouton **Agrandir** en haut à droite du volet.

    ![Capture d’écran du volet Azure Cloud Shell avec le bouton Agrandir encadré en rouge.](media/13-azure-cloud-shell-pane-maximize.png)

## Configuration : configurer des extensions

Pour stocker et interroger des vecteurs et générer des incorporations, vous devez autoriser et activer deux extensions pour Azure Database pour PostgreSQL - Serveur flexible : `vector` et `azure_ai`.

1. Pour placer les deux extensions en liste d’autorisation, ajoutez `vector` et `azure_ai` au paramètre de serveur `azure.extensions`, conformément aux instructions fournies dans [Guide pratique pour utiliser les extensions PostgreSQL](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions).

2. Exécutez la commande SQL suivante pour installer l’extension `vector`. Pour obtenir des instructions détaillées, consultez [Comment activer et utiliser `pgvector` sur Azure Database pour PostgreSQL - Serveur flexible](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-use-pgvector#enable-extension).

    ```sql
    CREATE EXTENSION vector;
    ```

3. Pour installer l’extension `azure_ai`, exécutez la commande SQL suivante : Vous aurez besoin du point de terminaison et de la clé d’API pour la ressource Azure OpenAI. Pour obtenir des instructions détaillées, consultez [Activer l’extension `azure_ai`](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/generative-ai-azure-overview#enable-the-azure_ai-extension).

    ```sql
    CREATE EXTENSION azure_ai;
    SELECT azure_ai.set_setting('azure_openai.endpoint', 'https://<endpoint>.openai.azure.com');
    SELECT azure_ai.set_setting('azure_openai.subscription_key', '<API Key>');
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

## Créer et stocker des vecteurs d’incorporation

Maintenant que nous avons des exemples de données, il est temps de générer et de stocker les vecteurs d’incorporation. L’extension `azure_ai` facilite l’appel de l’API d’incorporation Azure OpenAI.

1. Ajoutez la colonne de vecteur d’incorporation.

    Le modèle `text-embedding-ada-002` est configuré pour retourner 1 536 dimensions. Utilisez-le donc pour la taille de colonne vectorielle.

    ```sql
    ALTER TABLE listings ADD COLUMN listing_vector vector(1536);
    ```

1. Générez un vecteur d’incorporation pour la description de chaque annonce en appelant Azure OpenAI via la fonction create_embeddings définie par l’utilisateur, qui est implémentée par l’extension azure_ai :

    ```sql
    UPDATE listings
    SET listing_vector = azure_openai.create_embeddings('embedding', description, max_attempts => 5, retry_delay_ms => 500)
    WHERE listing_vector IS NULL;
    ```

    Notez que cela peut prendre plusieurs minutes, en fonction du quota disponible.

1. Consultez un exemple de vecteur en exécutant cette requête :

    ```sql
    SELECT listing_vector FROM listings LIMIT 1;
    ```

    Vous obtiendrez un résultat similaire à celui-ci, mais avec 1 536 colonnes vectorielles :

    ```sql
    postgres=> SELECT listing_vector FROM listings LIMIT 1;
    -[ RECORD 1 ]--+------ ...
    listing_vector | [-0.0018742813,-0.04530062,0.055145424, ... ]
    ```

## Effectuer une requête de recherche sémantique

Maintenant que vous avez répertorié les données augmentées avec des vecteurs d’incorporation, il est temps d’exécuter une requête de recherche sémantique. Pour ce faire, obtenez le vecteur d’incorporation de chaîne de requête, puis effectuez une recherche cosinus pour rechercher les annonces dont les descriptions sont les plus sémantiquement similaires à la requête.

1. Générez l’incorporation pour la chaîne de requête.

    ```sql
    SELECT azure_openai.create_embeddings('embedding', 'bright natural light');
    ```

    Vous obtiendrez un résultat similaire au suivant :

    ```sql
    -[ RECORD 1 ]-----+-- ...
    create_embeddings | {-0.0020871465,-0.002830255,0.030923981, ...}
    ```

1. Utilisez l’incorporation dans une recherche cosinus (`<=>` représente l’opération de distance de cosinus), en récupérant les 10 premières annonces les plus similaires à la requête.

    ```sql
    SELECT id, name FROM listings ORDER BY listing_vector <=> azure_openai.create_embeddings('embedding', 'bright natural light')::vector LIMIT 10;
    ```

    Vous obtiendrez un résultat similaire à celui-ci : Les résultats peuvent varier, car les vecteurs d’incorporation ne sont pas garantis comme déterministes :

    ```sql
        id    |                name                
    ----------+-------------------------------------
     6796336  | A duplex near U district!
     7635966  | Modern Capitol Hill Apartment
     7011200  | Bright 1 bd w deck. Great location
     8099917  | The Ravenna Apartment
     10211928 | Charming Ravenna Bungalow
     692671   | Sun Drenched Ballard Apartment
     7574864  | Modern Greenlake Getaway
     7807658  | Top Floor Corner Apt-Downtown View
     10265391 | Art filled, quiet, walkable Seattle
     5578943  | Madrona Studio w/Private Entrance
    ```

1. Vous pouvez également projeter la colonne `description` pour pouvoir lire le texte des lignes correspondantes dont les descriptions étaient sémantiquement similaires. Par exemple, cette requête retourne la meilleure correspondance :

    ```sql
    SELECT id, description FROM listings ORDER BY listing_vector <=> azure_openai.create_embeddings('embedding', 'bright natural light')::vector LIMIT 1;
    ```

    Le résultat ressemble à celui-ci :

    ```sql
       id    | description
    ---------+------------
     6796336 | This is a great place to live for summer because you get a lot of sunlight at the living room. A huge living room space with comfy couch and one ceiling window and glass windows around the living room.
    ```

Pour comprendre intuitivement la recherche sémantique, observez que la description ne contient pas réellement les termes « lumineux » ou « naturel ». Mais il met en surbrillance « été » et « lumière du soleil », « fenêtres », et « fenêtre de toit ».

## Vérifier votre travail

Après avoir effectué les étapes ci-dessus, la table `listings` contient des exemples de données de [Seattle Airbnb Open Data](https://www.kaggle.com/datasets/airbnb/seattle/data?select=listings.csv) sur Kaggle. Les annonces ont été augmentées avec des vecteurs d’incorporation pour exécuter des recherches sémantiques.

1. Vérifiez que la table d’annonces comporte quatre colonnes : `id`, `name`, `description` et `listing_vector`.

    ```sql
    \d listings
    ```

    Cela doit ressembler à ce qui suit :

    ```sql
                            Table "public.listings"
          Column    |         Type           | Collation | Nullable | Default 
    ----------------+------------------------+-----------+----------+---------
      id            | integer                |           | not null | 
      name          | character varying(255) |           | not null | 
      description   | text                   |           | not null | 
     listing_vector | vector(1536)           |           |          | 
     Indexes:
        "listings_pkey" PRIMARY KEY, btree (id)
    ```

1. Vérifiez qu’au moins une ligne a une colonne listing_vector remplie.

    ```sql
    SELECT COUNT(*) > 0 FROM listings WHERE listing_vector IS NOT NULL;
    ```

    Le résultat doit afficher `t`, pour true. Cela indique qu’au moins une ligne comporte des incorporations de la colonne de description correspondante :

    ```sql
    ?column? 
    ----------
    t
    (1 row)
    ```

    Vérifiez que le vecteur d’incorporation a 1 536 dimensions :

    ```sql
    SELECT vector_dims(listing_vector) FROM listings WHERE listing_vector IS NOT NULL LIMIT 1;
    ```

    Rendement :

    ```sql
    vector_dims 
    -------------
            1536
    (1 row)
    ```

1. Vérifiez que les recherches sémantiques retournent les résultats.

    Utilisez l’incorporation dans une recherche cosinus, en récupérant les 10 premières annonces les plus similaires à la requête.

    ```sql
    SELECT id, name FROM listings ORDER BY listing_vector <=> azure_openai.create_embeddings('embedding', 'bright natural light')::vector LIMIT 10;
    ```

    Vous obtiendrez un résultat semblable à celui-ci, en fonction des lignes affectées aux vecteurs d’incorporation :

    ```sql
     id |                name                
    --------+-------------------------------------
     315120 | Large, comfy, light, garden studio
     429453 | Sunny Bedroom #2 w/View: Wallingfrd
     17951  | West Seattle, The Starlight Studio
     48848  | green suite seattle - dog friendly
     116221 | Modern, Light-Filled Fremont Flat
     206781 | Bright & Spacious Studio
     356566 | Sunny Bedroom w/View: Wallingford
     9419   | Golden Sun vintage warm/sunny
     136480 | Bright Cheery Room in Seattle House
     180939 | Central District Green GardenStudio
    ```

## Nettoyage

Une fois cet exercice terminé, supprimez les ressources Azure que vous avez créées. La capacité configurée vous est facturée, et non la quantité de base de données utilisée. Suivez ces instructions pour supprimer votre groupe de ressources et toutes les ressources que vous avez créées pour ce labo.

1. Ouvrez un navigateur web et accédez au [portail Azure](https://portal.azure.com/), puis, dans la page d’accueil, dans Services Azure, sélectionnez **Groupes de ressources**.

    ![Capture d’écran du service Groupes de ressources encadré en rouge dans la section Services Azure du portail Azure.](media/13-azure-portal-home-azure-services-resource-groups.png)

2. Dans le filtre de n’importe quelle zone de recherche de champ, entrez le nom du groupe de ressources que vous avez créé pour ce labo, puis sélectionnez votre groupe de ressources dans la liste.

3. Dans la page **Vue d’ensemble** de votre groupe de ressources, sélectionnez **Supprimer le groupe de ressources**.

    ![Capture d’écran du panneau Vue d’ensemble du groupe de ressources avec le bouton Supprimer le groupe de ressources encadré en rouge.](media/13-resource-group-delete.png)

4. Dans la boîte de dialogue de confirmation, entrez le nom de votre groupe de ressources pour confirmer, puis sélectionnez **Supprimer**.
