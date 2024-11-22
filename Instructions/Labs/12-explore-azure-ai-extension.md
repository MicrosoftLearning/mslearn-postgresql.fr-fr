---
lab:
  title: "Explorer l’extension Azure\_AI"
  module: Explore Generative AI with Azure Database for PostgreSQL
---

# Explorer l’extension Azure AI

En tant que développeur principal pour Margie’s Travel, vous avez été chargé de créer une application basée sur l’IA pour fournir à vos clients des recommandations intelligentes sur les locations. Vous souhaitez en savoir plus sur l’extension `azure_ai` pour Azure Database pour PostgreSQL et sur la façon dont elle peut vous aider à intégrer la puissance de l’IA générative (GenAI) dans votre application. Dans cet exercice, vous installez l’extension `azure_ai` dans un serveur flexible Azure Database pour PostgreSQL et explorez ses fonctionnalités pour intégrer les services Azure AI et ML.

## Avant de commencer

Vous avez besoin d’un [abonnement Azure](https://azure.microsoft.com/free) avec des droits d’administration.

### Déployer des ressources dans votre abonnement Azure

Cette étape vous guide tout au long de l’utilisation de commandes Azure CLI à partir d’Azure Cloud Shell pour créer un groupe de ressources et exécuter un script Bicep pour déployer les services Azure nécessaires pour effectuer cet exercice dans votre abonnement Azure.

1. Ouvrez un navigateur web et accédez au [portail Azure](https://portal.azure.com/).

2. Dans la barre d’outils du portail Azure, sélectionnez l’icône **Cloud Shell** pour ouvrir un nouveau volet [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) en bas de la fenêtre de votre navigateur.

    ![Capture d’écran de la barre d’outils du portail Azure, avec l’icône Cloud Shell encadrée en rouge.](media/12-portal-toolbar-cloud-shell.png)

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
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy.bicep" --parameters restore=false adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Le script de déploiement Bicep approvisionne les services Azure requis pour effectuer cet exercice dans votre groupe de ressources. Les ressources déployées incluent un serveur flexible Azure Database pour PostgreSQ, Azure OpenAI et un service Azure AI Language. Le script Bicep effectue également certaines étapes de configuration, telles que l’ajout des extensions `azure_ai` et `vector` à la _liste d’autorisation_ du serveur PostgreSQL (via le paramètre de serveur azure.extensions), la création d’une base de données nommée `rentals` sur le serveur et l’ajout d’un déploiement nommé `embedding` à l’aide du modèle `text-embedding-ada-002` à votre service Azure OpenAI. Notez que le fichier Bicep est partagé par tous les modules de ce parcours d’apprentissage. Vous pouvez donc utiliser uniquement certaines des ressources déployées dans certains exercices.

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

Dans cette tâche, vous allez vous connecter à la base de données `rentals` sur votre serveur flexible Azure Database pour PostgreSQL à l’aide de l’[utilitaire de ligne de commande psql](https://www.postgresql.org/docs/current/app-psql.html) à partir d’[Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview).

1. Dans le [portail Azure](https://portal.azure.com/), accédez à votre serveur flexible Azure Database pour PostgreSQL nouvellement créé.

2. Dans le menu de ressource, dans **Paramètres**, sélectionnez **Bases de données**, puis **Connecter** pour la base de données `rentals`.

    ![Capture d’écran de la page Bases de données d’Azure SQL Database pour PostgreSQL. Le paramètre Bases de données et l’élément Connecter de la base de données de location sont encadrés en rouge.](media/12-postgresql-rentals-database-connect.png)

3. À l’invite « Mot de passe pour l’utilisateur pgAdmin » dans Cloud Shell, entrez le mot de passe généré de manière aléatoire pour la connexion **pgAdmin**.

    Une fois connecté, l’invite `psql` de la base de données `rentals` s’affiche.

4. Tout au long de cet exercice, vous continuerez à travailler dans Cloud Shell. Il peut donc être utile d’étendre le volet dans la fenêtre de votre navigateur en sélectionnant le bouton **Agrandir** en haut à droite du volet.

    ![Capture d’écran du volet Azure Cloud Shell avec le bouton Agrandir encadré en rouge.](media/12-azure-cloud-shell-pane-maximize.png)

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

## Examiner les objets contenus dans l’extension `azure_ai`

L’examen des objets contenus dans l’extension `azure_ai` peut vous aider à mieux comprendre ses fonctionnalités. Dans cette tâche, vous allez inspecter les différents schémas, fonctions définies par l’utilisateur (UDF) et les types composites ajoutés à la base de données par l’extension.

1. Lorsque vous utilisez `psql` dans Cloud Shell, activer l’affichage étendu pour les résultats de requête peut être utile, car cela améliore la lisibilité de la sortie pour les commandes suivantes. Exécutez la commande suivante pour que l’affichage étendu s’applique automatiquement.

    ```sql
    \x auto
    ```

2. La [métacommande `\dx`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DX-LC) est utilisée pour répertorier les objets contenus dans une extension. Exécutez ce qui suit à partir de l’invite de commandes `psql` pour afficher les objets de l’extension `azure_ai`. Vous devrez peut-être appuyer sur la barre d’espace pour afficher la liste complète des objets.

    ```psql
    \dx+ azure_ai
    ```

    La sortie de la métacommande indique que l’extension `azure_ai` crée quatre schémas, plusieurs fonctions définies par l’utilisateur (UDF) et plusieurs types composites dans la base de données et la table `azure_ai.settings`. En dehors des schémas, tous les noms d’objets sont précédés du schéma auquel ils appartiennent. Les schémas sont utilisés pour regrouper les fonctions et les types associés que l’extension ajoute dans des compartiments. Le tableau ci-dessous répertorie les schémas ajoutés par l’extension et fournit une brève description de chacun d’entre eux :

    | schéma      | Description                                              |
    | ----------------- | ------------------------------------------------------------------------------------------------------ |
    | `azure_ai`    | Schéma principal dans lequel résident la table de configuration et les fonctions définies par l’utilisateur permettant d’interagir avec l’extension. |
    | `azure_openai`  | Contient les fonctions définies par l’utilisateur qui activent l’appel d’un point de terminaison Azure OpenAI.                    |
    | `azure_cognitive` | Fournit des fonctions définies par l’utilisateur et des types composites liés à l’intégration de la base de données à Azure AI Services.     |
    | `azure_ml`    | Inclut les fonctions définies par l’utilisateur pour l’intégration des services Azure Machine Learning (ML).                |

### Explorer le schéma Azure AI

Le schéma `azure_ai` fournit l’infrastructure permettant d’interagir directement avec les services Azure AI et ML à partir de votre base de données. Il contient des fonctions permettant de configurer des connexions à ces services et de les récupérer à partir de la table `settings`, qui est également hébergée dans le même schéma. La table `settings` fournit un stockage sécurisé dans la base de données pour les points de terminaison et les clés associés à vos services Azure AI et ML.

1. Pour passer en revue les fonctions définies dans un schéma, utilisez la [métacommande `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC), en spécifiant le schéma dont les fonctions doivent être affichées. Exécutez ce qui suit pour afficher les fonctions dans le schéma `azure_ai` :

    ```sql
    \df azure_ai.*
    ```

    La sortie de la commande doit ressembler à la table suivante :

    ```sql
                  List of functions
     Schema |  Name  | Result data type | Argument data types | Type 
    ----------+-------------+------------------+----------------------+------
     azure_ai | get_setting | text      | key text      | func
     azure_ai | set_setting | void      | key text, value text | func
     azure_ai | version  | text      |           | func
    ```

    La fonction `set_setting()` vous permet de définir le point de terminaison et la clé de vos services Azure AI et ML afin que l’extension puisse s’y connecter. Elle accepte une **clé** et la **valeur** à lui affecter. La fonction `azure_ai.get_setting()` permet de récupérer les valeurs que vous définissez avec la fonction `set_setting()`. Elle accepte la **clé** du paramètre que vous souhaitez afficher et retourne la valeur qui lui est affectée. Pour les deux méthodes, la clé doit être l’une des suivantes :

    | Clé | Description |
    | --- | ----------- |
    | `azure_openai.endpoint` | Un point de terminaison OpenAI pris en charge (par exemple, <https://example.openai.azure.com>). |
    | `azure_openai.subscription_key` | Une clé d’abonnement pour une ressource Azure OpenAI. |
    | `azure_cognitive.endpoint` | Un point de terminaison Azure AI Services pris en charge (par exemple, <https://example.cognitiveservices.azure.com>). |
    | `azure_cognitive.subscription_key` | Une clé d’abonnement pour une ressource Azure AI Services. |
    | `azure_ml.scoring_endpoint` | Un point de terminaison de scoring Azure ML pris en charge (par exemple, <https://example.eastus2.inference.ml.azure.com/score>). |
    | `azure_ml.endpoint_key` | Une clé de point de terminaison pour un déploiement Azure ML. |

    > Important
    >
    > Étant donné que les informations de connexion pour Azure AI services, y compris les clés API, sont stockées dans une table de configuration dans la base de données, l’extension `azure_ai` définit un rôle appelé `azure_ai_settings_manager` pour garantir que ces informations sont protégées et accessibles uniquement aux utilisateurs affectés à ce rôle. Ce rôle permet la lecture et l’écriture des paramètres liés à l’extension. Seuls les membres du rôle `azure_ai_settings_manager` peuvent appeler les fonctions `azure_ai.get_setting()` et `azure_ai.set_setting()`. Dans le serveur flexible Azure Database pour PostgreSQL, tous les utilisateurs administrateurs (dotés du rôle `azure_pg_admin`) se voient également affecter le rôle `azure_ai_settings_manager`.

2. Pour mettre en pratique l’utilisation des fonctions `azure_ai.set_setting()` et `azure_ai.get_setting()`, configurez la connexion à votre compte Azure OpenAI. À l’aide de l’onglet du navigateur dans lequel votre Cloud Shell est ouvert, réduisez ou restaurez le volet Cloud Shell, puis accédez à votre ressource Azure OpenAI dans le [portail Azure](https://portal.azure.com/). Une fois que vous êtes sur la page de ressources Azure OpenAI, dans le menu des ressources, dans la section **Gestion des ressources**, sélectionnez **Clés et point de terminaison**, puis copiez votre point de terminaison et l’une des clés disponibles.

    ![Capture d’écran de la page Clés et points de terminaison du service Azure OpenAI. Les boutons de copie de la clé 1 et du point de terminaison sont encadrés en rouge.](media/12-azure-openai-keys-and-endpoints.png)

    Vous pouvez utiliser soit `KEY 1`, soit `KEY 2`. Avoir toujours deux clés vous permet de faire pivoter et de régénérer en toute sécurité les clés sans provoquer d’interruption de service.

3. Une fois en possession de votre point de terminaison et de votre clé, agrandissez à nouveau le volet Cloud Shell, puis utilisez les commandes ci-dessous pour ajouter vos valeurs à la table de configuration. Veillez à remplacer les jetons `{endpoint}` et `{api-key}` par les valeurs copiées à partir du portail Azure.

    ```sql
    SELECT azure_ai.set_setting('azure_openai.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_openai.subscription_key', '{api-key}');
    ```

4. Vous pouvez vérifier les paramètres écrits dans la table `azure_ai.settings` à l’aide de la fonction `azure_ai.get_setting()` et des requêtes suivantes :

    ```sql
    SELECT azure_ai.get_setting('azure_openai.endpoint');
    SELECT azure_ai.get_setting('azure_openai.subscription_key');
    ```

    L’extension `azure_ai` est désormais connectée à votre compte Azure OpenAI.

### Passer en revue le schéma Azure OpenAI

Le schéma `azure_openai` permet d’intégrer l’incorporation vectorielle de valeurs de texte dans votre base de données à l’aide d’Azure OpenAI. À l’aide de ce schéma, vous pouvez [générer des incorporations avec Azure OpenAI](https://learn.microsoft.com/azure/ai-services/openai/how-to/embeddings) directement à partir de la base de données pour créer des représentations vectorielles du texte d’entrée, qui peuvent ensuite être utilisées dans des recherches de similarité vectorielle et consommées par des modèles Machine Learning. Le schéma contient une fonction unique, `create_embeddings()`, avec deux surcharges. L’une des surcharges accepte une seule chaîne d’entrée, et l’autre attend un tableau de chaînes d’entrée.

1. Comme vous l’avez fait ci-dessus, vous pouvez utiliser la [méta-commande `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) pour afficher les détails des fonctions dans le schéma `azure_openai` :

    ```sql
    \df azure_openai.*
    ```

    La sortie affiche les deux surcharges de la fonction `azure_openai.create_embeddings()`, ce qui vous permet d'étudier les différences entre les deux versions de la fonction et les types qu’elles retournent. La propriété `Argument data types` dans la sortie révèle la liste des arguments attendus par les deux surcharges de fonction :

    | Argument    | Type       | Default | Description                                                          |
    | --------------- | ------------------ | ------- | ------------------------------------------------------------------------------------------------------------------------------ |
    | deployment_name | `text`      |    | Nom du déploiement dans Azure OpenAI Studio qui contient le modèle `text-embedding-ada-002`.               |
    | input     | `text` ou `text[]` |    | Texte d’entrée (ou tableau de texte) pour lequel des incorporations sont créées.                                |
    | batch_size   | `integer`     | 100  | Uniquement pour la surcharge qui attend une entrée de `text[]`. Spécifie le nombre d’enregistrements à traiter simultanément.          |
    | timeout_ms   | `integer`     | 3600000 | Délai d’expiration en millisecondes après lequel l’opération est arrêtée.                                 |
    | throw_on_error | `boolean`     | true  | Indicateur précisant si, en cas d’erreur, la fonction doit lever une exception entraînant une restauration des transactions d’enveloppement. |
    | max_attempts  | `integer`     | 1   | Nombre de nouvelles tentatives d’appel à Azure OpenAI Service en cas d’échec.                     |
    | retry_delay_ms | `integer`     | 1 000  | Durée d’attente, en millisecondes, avant une nouvelle tentative d’appel du point de terminaison de service Azure OpenAI.        |

2. Pour fournir un exemple simplifié d’utilisation de la fonction, exécutez la requête suivante, qui crée une incorporation vectorielle pour le champ `description` dans la table `listings`. Le paramètre `deployment_name` de la fonction est défini sur `embedding`, qui est le nom du déploiement du modèle `text-embedding-ada-002` dans votre service Azure OpenAI (il a été créé avec ce nom par le script de déploiement Bicep) :

    ```sql
    SELECT
        id,
        name,
        azure_openai.create_embeddings('embedding', description) AS vector
    FROM listings
    LIMIT 1;
    ```

    Le résultat ressemble à ce qui suit :

    ```sql
     id |      name       |              vector
    ----+-------------------------------+------------------------------------------------------------
      1 | Stylish One-Bedroom Apartment | {0.020068742,0.00022734122,0.0018286322,-0.0064167166,...}
    ```

    À des fins de concision, les incorporations vectorielles sont abrégées dans la sortie ci-dessus.

    Les [incorporations](https://learn.microsoft.com/azure/postgresql/flexible-server/generative-ai-overview#embeddings) sont un concept d’apprentissage automatique et de traitement du langage naturel (NLP) qui implique la représentation d’objets, tels que des mots, des documents ou des entités, en tant que [vecteurs](https://learn.microsoft.com/azure/postgresql/flexible-server/generative-ai-overview#vectors) dans un espace multidimensionnel. Les incorporations permettent aux modèles de Machine Learning d’évaluer le niveau de similitude entre deux informations. Cette technique identifie efficacement les relations et les similitudes entre les données, ce qui facilite l’identification des modèles par les algorithmes et améliore la précision des prédictions.

    L’extension `azure_ai` vous permet de générer des incorporations pour le texte d’entrée. Pour permettre aux vecteurs générés d’être stockés en même temps que le reste de vos données dans la base de données, vous devez installer l’extension `vector` en suivant les instructions de la documentation [activer la prise en charge des vecteurs dans votre base de données](https://learn.microsoft.com/azure/postgresql/flexible-server/how-to-use-pgvector#enable-extension). Cependant, cela sort du cadre de cet exercice.

### Examiner le schéma azure_cognitive

Le schéma `azure_cognitive` fournit l’infrastructure pour interagir directement avec Azure AI Services à partir de votre base de données. Les intégrations Azure AI services incluses dans le schéma fournissent un ensemble complet de fonctionnalités AI Language accessibles directement à partir de la base de données. Les fonctionnalités incluent l’analyse des sentiments, la détection de la langue, l’extraction de phrases clés, la reconnaissance d’entité, la synthèse de textes et la traduction. Ces fonctionnalités sont activées via le [service Azure AI Language](https://learn.microsoft.com/azure/ai-services/language-service/overview).

1. Pour passer en revue toutes les fonctions définies dans un schéma, vous pouvez utiliser la [méta-commande `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) comme vous l’avez fait précédemment. Pour afficher les fonctions dans le schéma `azure_cognitive`, exécutez :

    ```sql
    \df azure_cognitive.*
    ```

2. Il existe de nombreuses fonctions définies dans ce schéma, de sorte que la sortie de la [méta-commande `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) peut être difficile à lire. Il est donc préférable de la diviser en blocs plus petits. Exécutez ce qui suit pour examiner uniquement la fonction `analyze_sentiment()` :

    ```sql
    \df azure_cognitive.analyze_sentiment
    ```

    Dans la sortie, observez que la fonction a trois surcharges, l’une acceptant une seule chaîne d’entrée et les deux autres attendant des tableaux de texte. La sortie affiche le schéma, le nom, le type de données de résultat et les types de données d’argument de la fonction. Ces informations peuvent vous aider à comprendre comment utiliser la fonction.

3. Exécutez à nouveau la commande ci-dessus, en remplaçant le nom de la fonction, `analyze_sentiment`, par chacun des noms de fonction suivants, pour inspecter toutes les fonctions disponibles dans le schéma :

   - `detect_language`
   - `extract_key_phrases`
   - `linked_entities`
   - `recognize_entities`
   - `recognize_pii_entities`
   - `summarize_abstractive`
   - `summarize_extractive`
   - `translate`

    Pour chaque fonction, inspectez les différentes formes de la fonction et leurs entrées attendues ainsi que les types de données résultants.

4. Outre des fonctions, le schéma `azure_cognitive` contient également plusieurs types composites utilisés comme types de données de retour à partir des différentes fonctions. Il est impératif de comprendre la structure du type de données qu’une fonction retourne afin de pouvoir gérer correctement la sortie dans vos requêtes. Par exemple, exécutez la commande suivante pour inspecter le type `sentiment_analysis_result` :

    ```sql
    \dT+ azure_cognitive.sentiment_analysis_result
    ```

5. La sortie de la commande ci-dessus révèle que le type `sentiment_analysis_result` est un `tuple`. Vous pouvez étudier plus en détail la structure de ce `tuple` en exécutant la commande suivante pour examiner les colonnes contenues dans le type `sentiment_analysis_result` :

    ```sql
    \d+ azure_cognitive.sentiment_analysis_result
    ```

    La sortie de cette commande doit ressembler à ce qui suit :

    ```sql
             Composite type "azure_cognitive.sentiment_analysis_result"
       Column  |   Type   | Collation | Nullable | Default | Storage | Description 
    ----------------+------------------+-----------+----------+---------+----------+-------------
     sentiment   | text      |     |     |    | extended | 
     positive_score | double precision |     |     |    | plain  | 
     neutral_score | double precision |     |     |    | plain  | 
     negative_score | double precision |     |     |    | plain  |
    ```

    Le `azure_cognitive.sentiment_analysis_result` est un type composite qui contient les prédictions de sentiment du texte d’entrée. Il comprend le sentiment, qui peut être positif, négatif, neutre ou mixte, et les scores pour les aspects positifs, neutres et négatifs trouvés dans le texte. Les scores sont représentés sous forme de nombres réels compris entre 0 et 1. Par exemple, dans (neutre, 0,26, 0,64, 0,09), le sentiment est neutre, avec un score positif de 0,26, neutre de 0,64 et négatif à 0,09.

6. Comme avec les fonctions `azure_openai`, pour effectuer des appels auprès d’Azure AI Services à l’aide de l’extension `azure_ai`, vous devez fournir le point de terminaison et une clé pour votre service Azure AI Language. À l’aide de l’onglet du navigateur dans lequel Cloud Shell est ouvert, réduisez ou restaurez le volet Cloud Shell, puis accédez à votre ressource de service de langage dans le [portail Azure](https://portal.azure.com/). Dans le menu de la ressource, sous **Gestion des ressources**, sélectionnez **Clés et points de terminaison**.

    ![Capture d’écran de la page Clés et points de terminaison du service de langage Azure. Les boutons de copie de la clé 1 et du point de terminaison sont encadrés en rouge.](media/12-azure-language-service-keys-and-endpoints.png)

7. Copiez les valeurs de point de terminaison et de clé d’accès, puis remplacez les jetons `{endpoint}` et `{api-key}` par les valeurs que vous avez copiées à partir du portail Azure. Agrandissez à nouveau Cloud Shell et exécutez les commandes à partir de l’invite de commandes `psql` dans Cloud Shell pour ajouter vos valeurs à la table de configuration.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '{api-key}');
    ```

8. Exécutez maintenant la requête suivante pour analyser le sentiment de quelques avis :

    ```sql
    SELECT
        id,
        comments,
        azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment
    FROM reviews
    WHERE id IN (1, 3);
    ```

    Observez les valeurs `sentiment` dans la sortie, `(mixed,0.71,0.09,0.2)` et `(positive,0.99,0.01,0)`. Celles-ci représentent le `sentiment_analysis_result` retourné par la fonction `analyze_sentiment()` dans la requête ci-dessus. L’analyse a été effectuée sur le champ `comments` de la table `reviews`.

## Inspecter le schéma Azure ML

Le schéma `azure_ml` permet aux fonctions de se connecter aux services Azure ML directement à partir de votre base de données.

1. Pour passer en revue les fonctions définies dans un schéma, vous pouvez utiliser la [méta-commande `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC). Pour afficher les fonctions dans le schéma `azure_ml`, exécutez :

    ```sql
    \df azure_ml.*
    ```

    Dans la sortie, remarquez qu’il existe deux fonctions définies dans ce schéma, `azure_ml.inference()` et `azure_ml.invoke()`, dont les détails sont affichés ci-dessous :

    ```sql
                  List of functions
    -----------------------------------------------------------------------------------------------------------
    Schema       | azure_ml
    Name        | inference
    Result data type  | jsonb
    Argument data types | input_data jsonb, deployment_name text DEFAULT NULL::text, timeout_ms integer DEFAULT NULL::integer, throw_on_error boolean DEFAULT true, max_attempts integer DEFAULT 1, retry_delay_ms integer DEFAULT 1000
    Type        | func
    ```

    La fonction `inference()` utilise un modèle Machine Learning entraîné pour effectuer des prédictions ou générer des sorties basées sur des données nouvelles et encore jamais vues.

    En fournissant un point de terminaison et une clé, vous pouvez vous connecter à un point de terminaison déployé Azure ML comme vous vous êtes connecté à vos points de terminaison Azure OpenAI et Azure AI Services. Interagir avec Azure ML nécessite un modèle entraîné et déployé, ce qui n’entre pas dans le cadre de cet exercice et ne sera pas abordé ici.

## Nettoyage

Une fois cet exercice terminé, supprimez les ressources Azure que vous avez créées. La capacité configurée vous est facturée, et non la quantité de base de données utilisée. Suivez ces instructions pour supprimer votre groupe de ressources et toutes les ressources que vous avez créées pour ce labo.

1. Ouvrez un navigateur web et accédez au [portail Azure](https://portal.azure.com/), puis, dans la page d’accueil, dans Services Azure, sélectionnez **Groupes de ressources**.

    ![Capture d’écran du service Groupes de ressources encadré en rouge dans la section Services Azure du portail Azure.](media/12-azure-portal-home-azure-services-resource-groups.png)

2. Dans le filtre de n’importe quelle zone de recherche de champ, entrez le nom du groupe de ressources que vous avez créé pour ce labo, puis sélectionnez votre groupe de ressources dans la liste.

3. Dans la page **Vue d’ensemble** de votre groupe de ressources, sélectionnez **Supprimer le groupe de ressources**.

    ![Capture d’écran du panneau Vue d’ensemble du groupe de ressources avec le bouton Supprimer le groupe de ressources encadré en rouge.](media/12-resource-group-delete.png)

4. Dans la boîte de dialogue de confirmation, entrez le nom de votre groupe de ressources pour confirmer, puis sélectionnez **Supprimer**.
