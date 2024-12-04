---
lab:
  title: Effectuer un résumé extractif et abstrait
  module: Summarize data using Azure AI Services and Azure Database for PostgreSQL
---

# Effectuer un résumé extractif et abstrait

L’application de location gérée par Margie’s Travel permet aux gestionnaires immobiliers de décrire les annonces de location. La plupart des descriptions du système sont longues, donnant de nombreux détails sur la location, son quartier et ses attractions locales, magasins et autres commodités. Une fonctionnalité qui a été demandée lorsque vous implémentez de nouvelles fonctionnalités basés sur l’intelligence artificielle pour l’application utilise l’IA générative pour créer des résumés concis de ces descriptions, ce qui facilite la révision rapide des propriétés par vos utilisateurs. Dans cet exercice, vous utilisez l’extension `azure_ai` dans un serveur flexible Azure Database pour PostgreSQL afin d’effectuer des résumés abstraits et extractifs sur les descriptions des locations et comparer les résumés résultants.

## Avant de commencer

Vous avez besoin d’un [abonnement Azure](https://azure.microsoft.com/free) avec des droits d’administration.

### Déployer des ressources dans votre abonnement Azure

Cette étape vous guide tout au long de l’utilisation de commandes Azure CLI à partir d’Azure Cloud Shell pour créer un groupe de ressources et exécuter un script Bicep pour déployer les services Azure nécessaires pour effectuer cet exercice dans votre abonnement Azure.

1. Ouvrez un navigateur web et accédez au [portail Azure](https://portal.azure.com/).

2. Dans la barre d’outils du portail Azure, sélectionnez l’icône **Cloud Shell** pour ouvrir un nouveau volet [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) en bas de la fenêtre de votre navigateur.

    ![Capture d’écran de la barre d’outils du portail Azure, avec l’icône Cloud Shell encadrée en rouge.](media/15-portal-toolbar-cloud-shell.png)

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

1. Dans le [portail Azure](https://portal.azure.com/), accédez à votre instance de serveur flexible Azure Database pour PostgreSQL nouvellement créée.

2. Dans le menu de ressource, dans **Paramètres**, sélectionnez **Bases de données**, puis **Connecter** pour la base de données `rentals`.

    ![Capture d’écran de la page Bases de données d’Azure SQL Database pour PostgreSQL. Le paramètre Bases de données et l’élément Connecter de la base de données de location sont encadrés en rouge.](media/15-postgresql-rentals-database-connect.png)

3. À l’invite « Mot de passe pour l’utilisateur pgAdmin » dans Cloud Shell, entrez le mot de passe généré de manière aléatoire pour la connexion **pgAdmin**.

    Une fois connecté, l’invite `psql` de la base de données `rentals` s’affiche.

4. Tout au long de cet exercice, vous continuerez à travailler dans Cloud Shell. Il peut donc être utile d’étendre le volet dans la fenêtre de votre navigateur en sélectionnant le bouton **Agrandir** en haut à droite du volet.

    ![Capture d’écran du volet Azure Cloud Shell avec le bouton Agrandir encadré en rouge.](media/15-azure-cloud-shell-pane-maximize.png)

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

## Connecter votre compte Azure AI Services

Les intégrations Azure AI services incluses dans le schéma `azure_cognitive` de l’extension `azure_ai` fournissent un ensemble complet de fonctionnalités AI Language accessibles directement à partir de la base de données. Les fonctionnalités de résumé de texte sont activées via le [service Azure AI Language](https://learn.microsoft.com/azure/ai-services/language-service/overview).

1. Pour effectuer des appels sur vos services Azure AI Language à l’aide de l’extension `azure_ai`, vous devez fournir son point de terminaison et sa clé à l’extension. Depuis l’onglet du navigateur dans lequel Cloud Shell est ouvert, accédez à votre ressource de service de langage sur le [portail Azure](https://portal.azure.com/) et sélectionnez l’élément **Clés et point de terminaison** sous **Gestion des ressources** dans le menu de navigation de gauche.

    ![Capture d’écran de la page Clés et points de terminaison du service de langage Azure. Les boutons de copie de la clé 1 et du point de terminaison sont encadrés en rouge.](media/15-azure-language-service-keys-endpoints.png)

    > [!Note]
    >
    > Si vous avez reçu le message `NOTICE: extension "azure_ai" already exists, skipping CREATE EXTENSION` lors de l’installation de l’extension `azure_ai` ci-dessus et que vous avez précédemment configuré l’extension avec le point de terminaison et la clé de votre service de langage, vous pouvez utiliser la fonction `azure_ai.get_setting()` pour vérifier si ces paramètres sont corrects, puis ignorer l’étape 2 s’ils le sont.

2. Copiez les valeurs de point de terminaison et de clé d’accès, puis, dans les commandes ci-dessous, remplacez les jetons `{endpoint}` et `{api-key}` par les valeurs que vous avez copiées à partir du portail Azure. Exécutez les commandes à partir de l’invite de commandes `psql` dans Cloud Shell pour ajouter vos valeurs à la table `azure_ai.settings`.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '{api-key}');
    ```

## Examiner les fonctionnalités de résumé de l’extension

Dans cette tâche, vous allez examiner les deux fonctions de résumé dans le schéma `azure_cognitive`.

1. Pour le reste de cet exercice, vous allez travailler exclusivement dans Cloud Shell. Il peut donc être utile de développer le volet dans la fenêtre de votre navigateur en sélectionnant le bouton **Agrandir** en haut à droite du volet Cloud Shell.

    ![Capture d’écran du volet Azure Cloud Shell avec le bouton Agrandir encadré en rouge.](media/15-azure-cloud-shell-pane-maximize.png)

2. Lorsque vous utilisez `psql` dans Cloud Shell, activer l’affichage étendu pour les résultats de requête peut être utile, car cela améliore la lisibilité de la sortie pour les commandes suivantes. Exécutez la commande suivante pour que l’affichage étendu s’applique automatiquement.

    ```sql
    \x auto
    ```

3. Les fonctions de résumé de texte de l’extension `azure_ai` se trouvent dans le schéma `azure_cognitive`. Pour le résumé extractif, utilisez la fonction `summarize_extractive()`. Utilisez la [métacommande `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) pour examiner la fonction en exécutant :

    ```sql
    \df azure_cognitive.summarize_extractive
    ```

    La sortie de la métacommande affiche le schéma, le nom, le type de données de résultat et les arguments de la fonction. Ces informations vous aident à comprendre comment interagir avec la fonction à partir de vos requêtes.

    La sortie affiche trois surcharges de la fonction `summarize_extractive()`, ce qui vous permet de passer en revue leurs différences. La propriété `Argument data types` dans la sortie révèle la liste des arguments attendus par les trois surcharges de fonction :

    | Argument | Type | Default | Description |
    | -------- | ---- | ------- | ----------- |
    | texte | `text` ou `text[]` || Le ou les textes pour lesquels les résumés doivent être générés. |
    | language_text | `text` ou `text[]` || Le code de langue (ou tableau de codes de langues) représentant la langue du texte à résumer. Passez en revue la [liste des langues prises en charge](https://learn.microsoft.com/azure/ai-services/language-service/summarization/language-support) pour récupérer les codes de langue nécessaires. |
    | sentence_count | `integer` | 3 | Le nombre de phrases de résumé à générer. |
    | sort_by | `text` | 'offset' | L’ordre de tri des phrases de résumé générées. Les valeurs acceptables sont « offset » et « rank », où le décalage (« offset ») représente la position de départ de chaque phrase extraite dans le contenu d’origine et le classement (« rank ») est un indicateur généré par IA montrant la pertinence d’une phrase pour l’esprit du contenu. |
    | batch_size | `integer` | 25 | Uniquement pour les deux surcharges qui attendent une entrée de `text[]`. Spécifie le nombre d’enregistrements à traiter à la fois. |
    | disable_service_logs | `boolean` | false | Indicateur précisant s’il faut désactiver les journaux de service. |
    | timeout_ms | `integer` | NULL | Délai d’expiration en millisecondes après lequel l’opération est arrêtée. |
    | throw_on_error | `boolean` | true | Indicateur précisant si, en cas d’erreur, la fonction doit lever une exception entraînant une restauration des transactions d’enveloppement. |
    | max_attempts | `integer` | 1 | Nombre de nouvelles tentatives d’appel à Azure AI Services en cas d’échec. |
    | retry_delay_ms | `integer` | 1 000 | Durée d’attente, en millisecondes, avant une nouvelle tentative d’appel du point de terminaison Azure AI Services. |

4. Répétez l’étape ci-dessus, mais cette fois, exécutez la [métacommande `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) pour la fonction `azure_cognitive.summarize_abstractive()` et passez en revue la sortie.

    Les deux fonctions ont des signatures similaires, bien que `summarize_abstractive()` n’ait pas le paramètre `sort_by` et retourne un tableau de `text` contre un tableau de types composites `azure_cognitive.sentence` retournés par la fonction `summarize_extractive()`. Cette disparité est liée à la façon dont les deux méthodes différentes génèrent des résumés. Le résumé extractif identifie les phrases les plus critiques dans le texte qu’il résume, les classe, puis retourne ces phrases comme résumé. En revanche, le résumé abstrait utilise l’IA générative pour créer des phrases originales qui résument les points clés du texte.

5. Il est également impératif de comprendre la structure du type de données qu’une fonction retourne afin de pouvoir gérer correctement la sortie dans vos requêtes. Pour inspecter le type `azure_cognitive.sentence` retourné par la fonction `summarize_extractive()`, exécutez :

    ```sql
    \dT+ azure_cognitive.sentence
    ```

6. La sortie de la commande ci-dessus révèle que le type `sentence` est un `tuple`. Pour examiner la structure de ce `tuple` et les colonnes contenues dans le type composite `sentence`, exécutez :

    ```sql
    \d+ azure_cognitive.sentence
    ```

    La sortie de cette commande doit ressembler à ce qui suit :

    ```sql
                            Composite type "azure_cognitive.sentence"
        Column  |     Type         | Collation | Nullable | Default | Storage  | Description 
    ------------+------------------+-----------+----------+---------+----------+-------------
     text       | text             |           |           |        | extended | 
     rank_score | double precision |           |           |        | plain    |
    ```

    La `azure_cognitive.sentence` est un type composite contenant le texte d’une phrase extractive et un score de classement pour chaque phrase, indiquant la pertinence de la phrase par rapport au sujet principal du text. Le résumé des documents classe les phrases extraites. Vous pouvez déterminer si elles sont retournées dans l’ordre dans lequel elles apparaissent ou en fonction de leur classement.

## Créer des résumés pour les descriptions de propriétés

Dans cette tâche, vous utilisez les fonctions `summarize_extractive()` et `summarize_abstractive()` pour créer des résumés concis en deux phrases pour les descriptions de propriétés.

1. Maintenant que vous avez examiné la fonction `summarize_extractive()` et le `sentiment_analysis_result` qu’elle retourne, utilisons-la. Exécutez la requête simple suivante, qui effectue une analyse des sentiments sur une poignée de commentaires dans la table `reviews` :

    ```sql
    SELECT
        id,
        name,
        description,
        azure_cognitive.summarize_extractive(description, 'en', 2) AS extractive_summary
    FROM listings
    WHERE id IN (1, 2);
    ```

    Comparez les deux phrases du champ `extractive_summary` dans la sortie à la `description` originale, notant que les phrases ne sont pas originales, mais extraites de la `description`. Les valeurs numériques répertoriées après chaque phrase correspondent au score de classement attribué par le service de language.

2. Ensuite, effectuez un résumé abstrait sur les enregistrements identiques :

    ```sql
    SELECT
        id,
        name,
        description,
        azure_cognitive.summarize_abstractive(description, 'en', 2) AS abstractive_summary
    FROM listings
    WHERE id IN (1, 2);
    ```

    Les fonctionnalités de résumé abstrait de l’extension fournissent un résumé unique en langage naturel qui encapsule l’intention globale du texte d’origine.

    Si vous recevez une erreur similaire à la suivante, vous avez choisi une région qui ne prend pas en charge le résumé abstrait lors de la création de votre environnement Azure :

    ```bash
    ERROR: azure_cognitive.summarize_abstractive: InvalidRequest: Invalid Request.

    InvalidParameterValue: Job task: 'AbstractiveSummarization-task' failed with validation errors: ['Invalid Request.']

    InvalidRequest: Job task: 'AbstractiveSummarization-task' failed with validation error: Document abstractive summarization is not supported in the region Central US. The supported regions are North Europe, East US, West US, UK South, Southeast Asia.
    ```

    Pour pouvoir effectuer cette étape ainsi que les tâches restantes à l’aide d’un résumé abstrait, vous devez créer un service Azure AI Language dans l’une des régions prises en charge indiquées par le message d’erreur. Ce service peut être provisionné dans le même groupe de ressources que celui que vous avez utilisé pour d’autres ressources du labo. Vous pouvez également remplacer le résumé extractif pour les tâches restantes, mais vous ne bénéficierez pas de la possibilité de comparer la sortie des deux différentes techniques de résumé.

3. Exécutez une requête finale pour effectuer une comparaison côte à côte des deux techniques de résumé :

    ```sql
    SELECT
        id,
        azure_cognitive.summarize_extractive(description, 'en', 2) AS extractive_summary,
        azure_cognitive.summarize_abstractive(description, 'en', 2) AS abstractive_summary
    FROM listings
    WHERE id IN (1, 2);
    ```

    En plaçant les résumés générés côte à côte, il est facile de comparer la qualité des résumés générés par chaque méthode. Pour l’application Margie’s Travel, le résumé abstrait est la meilleure option, car elle fournit des résumés concis qui donnent des informations de haute qualité de manière naturelle et lisible. S’ils donnent des détails, les résumés extractifs sont cependant plus disjoints et offrent moins de valeur que le contenu d’origine créé par un résumé abstrait.

## Stocker des résumés des descriptions dans la base de données

1. Exécutez la requête suivante pour modifier la table `listings`, en ajoutant une nouvelle colonne `summary` :

    ```sql
    ALTER TABLE listings
    ADD COLUMN summary text;
    ```

2. Pour utiliser l’IA générative afin de créer des résumés pour toutes les propriétés existantes de la base de données, il est plus efficace d’envoyer les descriptions par lots, ce qui permet au service de langage de traiter plusieurs enregistrements simultanément.

    ```sql
    WITH batch_cte AS (
        SELECT azure_cognitive.summarize_abstractive(ARRAY(SELECT description FROM listings ORDER BY id), 'en', batch_size => 25) AS summary
    ),
    summary_cte AS (
        SELECT
            ROW_NUMBER() OVER () AS id,
            ARRAY_TO_STRING(summary, ',') AS summary
        FROM batch_cte
    )
    UPDATE listings AS l
    SET summary = s.summary
    FROM summary_cte AS s
    WHERE l.id = s.id;
    ```

    L’instruction update utilise deux expressions de table communes (CTE) pour travailler sur les données avant de mettre à jour la table `listings` avec des résumés. La première CTE (`batch_cte`) envoie toutes les valeurs `description` de la table `listings` au service de langage pour générer des résumés abstraits. Cela se fait par lots de 25 enregistrements à la fois. La deuxième CTE (`summary_cte`) utilise la position ordinale des résumés retournés par la fonction `summarize_abstractive()` pour affecter à chaque résumé `id` correspondant l’enregistrement depuis lequel la `description` provenait dans la table `listings`. Elle utilise également la fonction `ARRAY_TO_STRING` pour extraire les résumés générés à partir de la valeur de retour du tableau de texte (`text[]`) et la convertir en chaîne simple. Enfin, l’instruction `UPDATE` écrit le résumé dans la table `listings` de l’annonce associée.

3. À la dernière étape, exécutez une requête pour afficher les résumés écrits dans la table `listings` :

    ```sql
    SELECT
        id,
        name,
        description,
        summary
    FROM listings
    LIMIT 5;
    ```

## Générer un résumé de l’IA des avis pour une annonce

Pour l’application Margie’s Travel, l’affichage d’un résumé de tous les avis pour une propriété permet aux utilisateurs d’obtenir rapidement une idée générale des avis.

1. Exécutez la requête suivante, qui combine tous les avis d’une annonce dans une seule chaîne, puis génère un résumé abstrait sur cette chaîne :

    ```sql
    SELECT unnest(azure_cognitive.summarize_abstractive(reviews_combined, 'en')) AS review_summary
    FROM (
        -- Combine all reviews for a listing
        SELECT string_agg(comments, ' ') AS reviews_combined
        FROM reviews
        WHERE listing_id = 1
    );
    ```

## Nettoyage

Une fois cet exercice terminé, supprimez les ressources Azure que vous avez créées. La capacité configurée vous est facturée, et non la quantité de base de données utilisée. Suivez ces instructions pour supprimer votre groupe de ressources et toutes les ressources que vous avez créées pour ce labo.

1. Ouvrez un navigateur web et accédez au [portail Azure](https://portal.azure.com/), puis, dans la page d’accueil, dans Services Azure, sélectionnez **Groupes de ressources**.

    ![Capture d’écran du service Groupes de ressources encadré en rouge dans la section Services Azure du portail Azure.](media/15-azure-portal-home-azure-services-resource-groups.png)

2. Dans le filtre de n’importe quelle zone de recherche de champ, entrez le nom du groupe de ressources que vous avez créé pour ce labo, puis sélectionnez votre groupe de ressources dans la liste.

3. Dans la page **Vue d’ensemble** de votre groupe de ressources, sélectionnez **Supprimer le groupe de ressources**.

    ![Capture d’écran du panneau Vue d’ensemble du groupe de ressources avec le bouton Supprimer le groupe de ressources encadré en rouge.](media/15-resource-group-delete.png)

4. Dans la boîte de dialogue de confirmation, entrez le nom de votre groupe de ressources pour confirmer, puis sélectionnez **Supprimer**.
