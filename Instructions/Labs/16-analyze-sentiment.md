---
lab:
  title: Analyser des sentiments
  module: Perform Sentiment Analysis and Opinion Mining using Azure Database for PostgreSQL
---

# Analyser des sentiments

Dans le cadre de l’application basée sur l’IA que vous créez pour Margie’s Travel, vous souhaitez fournir aux utilisateurs des informations sur le sentiment des avis individuels et le sentiment global de tous les avis pour une location donnée. Pour ce faire, vous utilisez l’extension `azure_ai` dans un serveur flexible Azure Database pour PostgreSQL afin d’intégrer la fonctionnalité d’analyse des sentiments dans votre base de données.

## Avant de commencer

Vous avez besoin d’un [abonnement Azure](https://azure.microsoft.com/free) avec des droits d’administration.

### Déployer des ressources dans votre abonnement Azure

Cette étape vous guide tout au long de l’utilisation de commandes Azure CLI à partir d’Azure Cloud Shell pour créer un groupe de ressources et exécuter un script Bicep pour déployer les services Azure nécessaires pour effectuer cet exercice dans votre abonnement Azure.

1. Ouvrez un navigateur web et accédez au [portail Azure](https://portal.azure.com/).

2. Dans la barre d’outils du portail Azure, sélectionnez l’icône **Cloud Shell** pour ouvrir un nouveau volet [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) en bas de la fenêtre de votre navigateur.

    ![Capture d’écran de la barre d’outils du portail Azure, avec l’icône Cloud Shell encadrée en rouge.](media/11-portal-toolbar-cloud-shell.png)

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

Vous pouvez rencontrer quelques erreurs lors de l’exécution du script de déploiement Bicep. Les messages les plus courants et les étapes à suivre pour les résoudre sont les suivants :

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

1. Dans le [portail Azure](https://portal.azure.com/), accédez à votre instance de serveur flexible Azure Database pour PostgreSQL nouvellement créée.

2. Dans le menu de ressource, dans **Paramètres**, sélectionnez **Bases de données**, puis **Connecter** pour la base de données `rentals`. Notez que la sélection de **Se connecter** ne vous connecte pas réellement à la base de données ; elle fournit simplement des instructions pour se connecter à la base de données en utilisant différentes méthodes. Consultez les instructions pour **Se connecter depuis le navigateur ou localement** et utilisez-les pour vous connecter en utilisant Azure Cloud Shell.

    ![Capture d’écran de la page Bases de données d’Azure SQL Database pour PostgreSQL. Le paramètre Bases de données et l’élément Connecter de la base de données de location sont encadrés en rouge.](media/17-postgresql-rentals-database-connect.png)

3. À l’invite « Mot de passe pour l’utilisateur pgAdmin » dans Cloud Shell, entrez le mot de passe généré de manière aléatoire pour la connexion **pgAdmin**.

    Une fois connecté, l’invite `psql` de la base de données `rentals` s’affiche.

## Remplir la base de données avec des exemples de données

Avant de pouvoir analyser le sentiment des avis sur la location à l’aide de l’extension `azure_ai`, vous devez ajouter des exemples de données à votre base de données. Ajoutez une table à la base de données `rentals` et remplissez-la avec les avis des clients afin de disposer de données sur lesquelles effectuer l’analyse des sentiments.

1. Exécutez la commande suivante pour créer une table nommée `reviews` afin de stocker les avis soumis par les clients sur la propriété :

    ```sql
    DROP TABLE IF EXISTS reviews;

    CREATE TABLE reviews (
        id int,
        listing_id int, 
        date date,
        comments text
    );
    ```

2. Ensuite, utilisez la commande `COPY` pour remplir la table avec des données provenant d’un fichier CSV. Exécutez la commande ci-dessous pour charger les avis des clients dans la table `reviews` :

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

Les intégrations Azure AI services incluses dans le schéma `azure_cognitive` de l’extension `azure_ai` fournissent un ensemble complet de fonctionnalités AI Language accessibles directement à partir de la base de données. Les fonctionnalités d’analyse des sentiments sont activées via le [service Azure AI Language](https://learn.microsoft.com/azure/ai-services/language-service/overview).

1. Pour effectuer des appels sur vos services Azure AI Language à l’aide de l’extension `azure_ai`, vous devez fournir son point de terminaison et sa clé à l’extension. Depuis l’onglet du navigateur dans lequel Cloud Shell est ouvert, accédez à votre ressource de service de langage sur le [portail Azure](https://portal.azure.com/) et sélectionnez l’élément **Clés et point de terminaison** sous **Gestion des ressources** dans le menu de navigation de gauche.

    ![Capture d’écran de la page Clés et points de terminaison du service de langage Azure. Les boutons de copie de la clé 1 et du point de terminaison sont encadrés en rouge.](media/16-azure-language-service-keys-endpoints.png)

> [!Note]
>
> Si vous avez reçu le message `NOTICE: extension "azure_ai" already exists, skipping CREATE EXTENSION` lors de l’installation de l’extension `azure_ai` ci-dessus et que vous avez précédemment configuré l’extension avec le point de terminaison et la clé de votre service de langage, vous pouvez utiliser la fonction `azure_ai.get_setting()` pour vérifier si ces paramètres sont corrects, puis ignorer l’étape 2 s’ils le sont.

2. Copiez les valeurs de point de terminaison et de clé d’accès, puis, dans les commandes ci-dessous, remplacez les jetons `{endpoint}` et `{api-key}` par les valeurs que vous avez copiées à partir du portail Azure. Exécutez les commandes à partir de l’invite de commandes `psql` dans Cloud Shell pour ajouter vos valeurs à la table `azure_ai.settings`.

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '{api-key}');
    ```

## Passer en revue les fonctionnalités d’analyse des sentiments de l’extension

Dans cette tâche, vous utilisez la fonction `azure_cognitive.analyze_sentiment()` pour évaluer les avis des annonces de location.

1. Pour le reste de cet exercice, vous allez travailler exclusivement dans Cloud Shell. Il peut donc être utile de développer le volet dans la fenêtre de votre navigateur en sélectionnant le bouton **Agrandir** en haut à droite du volet Cloud Shell.

    ![Capture d’écran du volet Azure Cloud Shell avec le bouton Agrandir encadré en rouge.](media/16-azure-cloud-shell-pane-maximize.png)

2. Lorsque vous utilisez `psql` dans Cloud Shell, activer l’affichage étendu pour les résultats de requête peut être utile, car cela améliore la lisibilité de la sortie pour les commandes suivantes. Exécutez la commande suivante pour que l’affichage étendu s’applique automatiquement.

    ```sql
    \x auto
    ```

3. Les fonctionnalités d’analyse des sentiments de l’extension `azure_ai` sont disponibles dans le schéma `azure_cognitive`. Utilisez la fonction `analyze_sentiment()`. Utilisez la [métacommande `\df`](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) pour examiner la fonction en exécutant :

    ```sql
    \df azure_cognitive.analyze_sentiment
    ```

    La sortie de la métacommande affiche le schéma, le nom, le type de données de résultat et les arguments de la fonction. Ces informations vous aident à comprendre comment interagir avec la fonction à partir de vos requêtes.

    La sortie affiche trois surcharges de la fonction `analyze_sentiment()`, ce qui vous permet de passer en revue leurs différences. La propriété `Argument data types` dans la sortie révèle la liste des arguments attendus par les trois surcharges de fonction :

    | Argument | Type | Default | Description |
    | -------- | ---- | ------- | ----------- |
    | texte | `text` ou `text[]` || Texte pour lequel le sentiment doit être analysé. |
    | language_text | `text` ou `text[]` || Code de langue (ou tableau de codes de langues) représentant la langue du texte à analyser pour le sentiment. Passez en revue la [liste des langues prises en charge](https://learn.microsoft.com/azure/ai-services/language-service/sentiment-opinion-mining/language-support) pour récupérer les codes de langue nécessaires. |
    | batch_size | `integer` | 10 | Uniquement pour les deux surcharges qui attendent une entrée de `text[]`. Spécifie le nombre d’enregistrements à traiter à la fois. |
    | disable_service_logs | `boolean` | false | Indicateur précisant s’il faut désactiver les journaux de service. |
    | timeout_ms | `integer` | NULL | Délai d’expiration en millisecondes après lequel l’opération est arrêtée. |
    | throw_on_error | `boolean` | true | Indicateur précisant si, en cas d’erreur, la fonction doit lever une exception entraînant une restauration des transactions d’enveloppement. |
    | max_attempts | `integer` | 1 | Nombre de nouvelles tentatives d’appel à Azure AI Services en cas d’échec. |
    | retry_delay_ms | `integer` | 1 000 | Durée d’attente, en millisecondes, avant une nouvelle tentative d’appel du point de terminaison Azure AI Services. |

4. Il est également impératif de comprendre la structure du type de données qu’une fonction retourne afin de pouvoir gérer correctement la sortie dans vos requêtes. Exécutez la commande suivante pour inspecter le type `sentiment_analysis_result` :

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
         Column     |     Type         | Collation | Nullable | Default | Storage  | Description 
    ----------------+------------------+-----------+----------+---------+----------+-------------
     sentiment      | text             |           |          |         | extended | 
     positive_score | double precision |           |          |         | plain    | 
     neutral_score  | double precision |           |          |         | plain    | 
     negative_score | double precision |           |          |         | plain    |
    ```

    Le `azure_cognitive.sentiment_analysis_result` est un type composite qui contient les prédictions de sentiment du texte d’entrée. Il comprend le sentiment, qui peut être positif, négatif, neutre ou mixte, et les scores pour les aspects positifs, neutres et négatifs trouvés dans le texte. Les scores sont représentés sous forme de nombres réels compris entre 0 et 1. Par exemple, dans (neutre, 0,26, 0,64, 0,09), le sentiment est neutre, avec un score positif de 0,26, neutre de 0,64 et négatif à 0,09.

## Analyser le sentiment des avis

1. Maintenant que vous avez examiné la fonction `analyze_sentiment()` et le `sentiment_analysis_result` qu’elle retourne, utilisons-la. Exécutez la requête simple suivante, qui effectue une analyse des sentiments sur une poignée de commentaires dans la table `reviews` :

    ```sql
    SELECT
        id,
        azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment
    FROM reviews
    WHERE id <= 10
    ORDER BY id;
    ```

    À partir des deux enregistrements analysés, notez les valeurs `sentiment` dans la sortie, `(mixed,0.71,0.09,0.2)` et `(positive,0.99,0.01,0)`. Celles-ci représentent le `sentiment_analysis_result` retourné par la fonction `analyze_sentiment()` dans la requête ci-dessus. L’analyse a été effectuée sur le champ `comments` de la table `reviews`.

    > [!Note]
    >
    > L’utilisation de la fonction `analyze_sentiment()` inline vous permet d’analyser rapidement le sentiment du texte dans vos requêtes. Bien que cela fonctionne bien pour un petit nombre d’enregistrements, cela peut ne pas être idéal pour analyser le sentiment d’un grand nombre d’enregistrements ou mettre à jour tous les enregistrements d’une table qui peut contenir des dizaines de milliers d’avis ou plus.

2. Une autre approche qui peut être utile pour les avis plus longs consiste à analyser le sentiment de chaque phrase qu’ils contiennent. Pour ce faire, utilisez la surcharge de la fonction `analyze_sentiment()`, qui accepte un tableau de texte.

    ```sql
    SELECT
        azure_cognitive.analyze_sentiment(ARRAY_REMOVE(STRING_TO_ARRAY(comments, '.'), ''), 'en') AS sentence_sentiments
    FROM reviews
    WHERE id = 1;
    ```

    Dans la requête ci-dessus, vous avez utilisé la fonction `STRING_TO_ARRAY` de PostgreSQL. En outre, la fonction `ARRAY_REMOVE` a été utilisée pour supprimer tous les éléments de tableau qui sont des chaînes vides, car ces éléments entraînent des erreurs avec la fonction `analyze_sentiment()`.

    La sortie de la requête vous permet de mieux comprendre le sentiment `mixed` affecté à l’avis global. Les phrases sont un mélange de sentiments positifs, neutres et négatifs.

3. Les deux requêtes précédentes ont retourné directement `sentiment_analysis_result` à partir de la requête. Toutefois, vous souhaiterez probablement récupérer les valeurs sous-jacentes dans le `sentiment_analysis_result``tuple`. Exécutez la requête suivante qui recherche des avis extrêmement positifs et extrait les composants de sentiment dans des champs individuels :

    ```sql
    WITH cte AS (
        SELECT id, comments, azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment FROM reviews
    )
    SELECT
        id,
        (sentiment).sentiment,
        (sentiment).positive_score,
        (sentiment).neutral_score,
        (sentiment).negative_score,
        comments
    FROM cte
    WHERE (sentiment).positive_score > 0.98
    LIMIT 5;
    ```

    La requête ci-dessus utilise une expression de table commune ou une expression CTE pour obtenir les scores de sentiments de tous les enregistrements de la table `reviews`. Elle sélectionne ensuite les colonnes de type composite `sentiment` à partir du `sentiment_analysis_result` retourné par l’expression CTE pour extraire les valeurs individuelles à partir de `tuple.`

## Stocker le sentiment dans la table Avis

Pour le système de recommandation de propriété de location que vous créez pour Margie’s Travel, vous souhaitez stocker les évaluations des sentiments dans la base de données afin d’éviter de devoir effectuer d’appels et d’engager des frais chaque fois que les évaluations des sentiments sont demandées. L’exécution d’une analyse des sentiments à la volée peut être idéale pour un petit nombre d’enregistrements ou pour l’analyse de données en quasi-temps réel. Néanmoins, il est judicieux d’ajouter des données de sentiment dans la base de données afin de les utiliser dans votre application pour vos avis stockés. Pour ce faire, vous devez modifier la table `reviews` afin d’ajouter des colonnes pour stocker l’évaluation des sentiments et les scores positifs, neutres et négatifs.

1. Exécutez la requête suivante pour mettre à jour la table `reviews` afin qu’elle puisse stocker les détails des sentiments :

    ```sql
    ALTER TABLE reviews
    ADD COLUMN sentiment varchar(10),
    ADD COLUMN positive_score numeric,
    ADD COLUMN neutral_score numeric,
    ADD COLUMN negative_score numeric;
    ```

2. Ensuite, vous devez mettre à jour les enregistrements existants dans la table `reviews` avec leur valeur de sentiment et leurs scores associés.

    ```sql
    WITH cte AS (
        SELECT id, azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment FROM reviews
    )
    UPDATE reviews AS r
    SET
        sentiment = (cte.sentiment).sentiment,
        positive_score = (cte.sentiment).positive_score,
        neutral_score = (cte.sentiment).neutral_score,
        negative_score = (cte.sentiment).negative_score
    FROM cte
    WHERE r.id = cte.id;
    ```

    L’exécution de cette requête prend beaucoup de temps, car les commentaires pour chaque avis de la table sont envoyés individuellement au point de terminaison du service de langage pour être analysés. L’envoi d’enregistrements par lots est plus efficace lorsque vous traitez de nombreux enregistrements.

3. Nous allons exécuter la requête ci-dessous pour effectuer la même action de mise à jour, mais cette fois-ci, envoyez des commentaires de la table `reviews` dans des lots de 10 (il s’agit de la taille maximale autorisée par lot) et évaluez la différence de performances.

    ```sql
    WITH cte AS (
        SELECT azure_cognitive.analyze_sentiment(ARRAY(SELECT comments FROM reviews ORDER BY id), 'en', batch_size => 10) as sentiments
    ),
    sentiment_cte AS (
        SELECT
            ROW_NUMBER() OVER () AS id,
            sentiments AS sentiment
        FROM cte
    )
    UPDATE reviews AS r
    SET
        sentiment = (sentiment_cte.sentiment).sentiment,
        positive_score = (sentiment_cte.sentiment).positive_score,
        neutral_score = (sentiment_cte.sentiment).neutral_score,
        negative_score = (sentiment_cte.sentiment).negative_score
    FROM sentiment_cte
    WHERE r.id = sentiment_cte.id;
    ```

    Bien que cette requête soit un peu plus complexe, l’utilisation de deux expressions CTE est beaucoup plus performante. Dans cette requête, la première expression CTE analyse le sentiment des lots de commentaires d’avis, et le second extrait la table résultante de `sentiment_analysis_results` dans une nouvelle table contenant un `id` basé sur une position ordinale et « sentiment_analysis_result » pour chaque ligne. La deuxième expression CTE peut ensuite être utilisée dans l’instruction de mise à jour pour écrire les valeurs dans la base de données.

4. Ensuite, exécutez une requête pour observer les résultats de la mise à jour, en recherchant des avis avec un sentiment **négatif**, en commençant par le plus négatif.

    ```sql
    SELECT
        id,
        negative_score,
        comments
    FROM reviews
    WHERE sentiment = 'negative'
    ORDER BY negative_score DESC;
    ```

## Nettoyage

Une fois cet exercice terminé, supprimez les ressources Azure que vous avez créées. La capacité configurée vous est facturée, et non la quantité de base de données utilisée. Suivez ces instructions pour supprimer votre groupe de ressources et toutes les ressources que vous avez créées pour ce labo.

1. Ouvrez un navigateur web et accédez au [portail Azure](https://portal.azure.com/), puis, dans la page d’accueil, dans Services Azure, sélectionnez **Groupes de ressources**.

    ![Capture d’écran du service Groupes de ressources encadré en rouge dans la section Services Azure du portail Azure.](media/16-azure-portal-home-azure-services-resource-groups.png)

2. Dans le filtre de n’importe quelle zone de recherche de champ, entrez le nom du groupe de ressources que vous avez créé pour ce labo, puis sélectionnez votre groupe de ressources dans la liste.

3. Dans la page **Vue d’ensemble** de votre groupe de ressources, sélectionnez **Supprimer le groupe de ressources**.

    ![Capture d’écran du panneau Vue d’ensemble du groupe de ressources avec le bouton Supprimer le groupe de ressources encadré en rouge.](media/16-resource-group-delete.png)

4. Dans la boîte de dialogue de confirmation, entrez le nom de votre groupe de ressources pour confirmer, puis sélectionnez **Supprimer**.
