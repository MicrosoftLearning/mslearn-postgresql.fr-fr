---
lab:
  title: Configurer les paramètres système et explorer les métadonnées avec des catalogues et des vues système
  module: Configure and manage Azure Database for PostgreSQL
---

# Configurer les paramètres système et explorer les métadonnées avec des catalogues et des vues système

Dans cet exercice, vous examinez les paramètres système et les métadonnées dans PostgreSQL.

## Avant de commencer

> [!IMPORTANT]
> Vous devez disposer de votre propre abonnement Azure pour effectuer les exercices de ce module. Si vous n’avez pas d’abonnement Azure, vous pouvez configurer un compte d’essai gratuit en consultant [Créer dans le cloud avec un compte gratuit Azure](https://azure.microsoft.com/free/).

## Créer l’environnement de l’exercice

### Déployer des ressources dans votre abonnement Azure

Cette étape vous guide tout au long de l’utilisation de commandes Azure CLI à partir d’Azure Cloud Shell pour créer un groupe de ressources et exécuter un script Bicep pour déployer les services Azure nécessaires pour effectuer cet exercice dans votre abonnement Azure.

> Remarque
>
> Si vous effectuez plusieurs modules dans ce parcours d’apprentissage, vous pouvez partager l’environnement Azure entre eux. Dans ce cas, vous devez effectuer cette étape de déploiement de ressources une seule fois.

1. Ouvrez un navigateur web et accédez au [portail Azure](https://portal.azure.com/).

2. Dans la barre d’outils du portail Azure, sélectionnez l’icône **Cloud Shell** pour ouvrir un nouveau volet [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) en bas de la fenêtre de votre navigateur.

    ![Capture d’écran de la barre d’outils du portail Azure, avec l’icône Cloud Shell encadrée en rouge.](media/07-portal-toolbar-cloud-shell.png)

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

### Se connecter à la base de données avec Azure Data Studio

1. Si vous ne l’avez pas encore fait, clonez les scripts du labo localement à partir du référentiel GitHub [PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql.git) :
    1. Ouvrez une ligne de commande/un terminal.
    1. Exécutez la commande suivante :
       ```bash
       md .\DP3021Lab
       git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git .\DP3021Lab
       ```
       > REMARQUE
       > 
       > Si **Git** n’est pas installé, [téléchargez et installez l’application ***Git***](https://git-scm.com/download) et réessayez d’exécuter les commandes précédentes.
1. Si vous n’avez pas encore installé Azure Data Studio, [téléchargez et installez ***Azure Data Studio***](https://go.microsoft.com/fwlink/?linkid=2282284).
1. Si vous n’avez pas installé l’**extension PostgreSQL** dans Azure Data Studio, installez-la maintenant.
1. Ouvrez Azure Data Studio.
1. Sélectionnez **Connexions**.
1. Sélectionnez **Serveurs**, puis **Nouvelle connexion**.
1. Dans **Type de connexion**, sélectionnez **PostgreSQL**.
1. Dans **Nom d’u serveur**, tapez la valeur que vous avez spécifiée quand vous avez déployé le serveur.
1. Dans **Nom d’utilisateur**, tapez **pgAdmin**.
1. Dans **Mot de passe**, entrez le mot de passe généré de manière aléatoire pour la connexion **pgAdmin** que vous avez générée.
1. Sélectionnez **Mémoriser le mot de passe**.
1. Cliquez sur **Connexion**
1. Si vous n’avez pas encore créé la base de données zoodb, sélectionnez **Fichier**, **Ouvrir un fichier**, puis accédez au dossier dans lequel vous avez enregistré les scripts. Sélectionnez **../Allfiles/Labs/02/Lab2_ZooDb.sql** et **Ouvri**.
   1. Mettez en surbrillance les instructions **DROP** et **CREATE**, puis exécutez-les.
   1. En haut de l’écran, utilisez la flèche déroulante pour voir les bases de données sur le serveur, comme zoodb et les bases de données système. Sélectionnez la base de données **zoodb**.
   1. Mettez en surbrillance les sections **Créer des tables**, **Créer des clés étrangères** et **Remplir des tables** et exécutez-les.
   1. Mettez en surbrillance les 3 instructions **SELECT** à la fin du script et exécutez-les pour vérifier que les tables ont été créées et remplies.

## Tâche 1 : explorer le processus de nettoyage dans PostgreSQL

1. Ouvrez Azure Data Studio, si ce n’est pas déjà fait.
1. Dans Azure Data Studio, sélectionnez **Fichier**, **Ouvrir un fichier**, puis accédez aux scripts de labo. Sélectionnez **.. /Allfiles/Labs/07/Lab7_vacuum.sql**, puis sélectionnez **Ouvrir**. Reconnectez-vous au serveur si nécessaire.
1. Sélectionnez la base de données **zoodb** dans la liste déroulante de bases de données.
1. Surlignez et exécutez la section **Vérifier que la base de données zoodb est sélectionnée**. Si nécessaire, faites de zoodb la base de données active à l’aide de la liste déroulante.
1. Surlignez et exécutez la section **Afficher les tuples morts**. Cette requête affiche le nombre de tuples morts et vivants dans la base de données. Notez le nombre de tuples morts.
1. Surlignez et exécutez la section **Modifier le poids** dix fois de suite. Cette requête met à jour la colonne du poids pour tous les animaux.
1. Réexécutez la section sous **Afficher les tuples morts**. Notez le nombre de tuples morts une fois les mises à jour effectuées.
1. Exécutez la section sous **Exécuter manuellement VACUUM** pour exécuter le processus de nettoyage.
1. Réexécutez la section sous **Afficher les tuples morts**. Notez le nombre de tuples morts après l’exécution du processus de nettoyage.

## Tâche 2 : configurer les paramètres du serveur de nettoyage automatique

1. Dans le portail Azure, accédez à votre serveur flexible Azure Database pour PostgreSQL.
1. Sous **Paramètres**, sélectionnez **Paramètres du serveur**.
1. Dans la barre de recherche, tapez **`vacuum`**. Recherchez les paramètres suivants et modifiez les valeurs comme suit :
    1. autovacuum = ON (La valeur doit être ON par défaut.)
    1. autovacuum_vacuum_scale_factor = 0,1
    1. autovacuum_vacuum_threshold = 50

    Cela revient à exécuter le processus de nettoyage automatique quand 10 % d’une table comporte des lignes marquées pour suppression ou 50 lignes mises à jour ou supprimées dans une seule table.

1. Cliquez sur **Enregistrer**. Le serveur est redémarré.

## Tâche 3 : afficher les métadonnées PostgreSQL dans le portail Azure

1. Accédez au [portail Azure](https://portal.azure.com) et connectez-vous.
1. Recherchez et sélectionnez **Azure Database pour PostgreSQL**.
1. Sélectionnez le serveur flexible Azure Database pour PostgreSQL que vous avez créé pour cet exercice.
1. Dans **Supervision**, sélectionnez **Métriques**.
1. Sélectionnez **Métrique**, puis **Pourcentage d’UC**.
1. Notez que vous pouvez afficher différentes métriques sur vos bases de données.

## Tâche 4 : afficher des données dans les tables de catalogue système

1. Basculez sur Azure Data Studio.
1. Dans **SERVERS**, sélectionnez votre serveur PostgreSQL et attendez qu’une connexion soit établie et qu’un cercle vert s’affiche sur le serveur.
1. Cliquez avec le bouton droit sur le serveur et sélectionnez **Nouvelle requête**.
1. Entrez la requête SQL suivante, puis sélectionnez **Exécuter** :

    ```sql
    SELECT datname, xact_commit, xact_rollback FROM pg_stat_database;
    ```

1. Notez que vous pouvez afficher les validations et les restaurations pour chaque base de données.

## Afficher une requête de métadonnées complexe à l’aide d’une vue système

1. Cliquez avec le bouton droit sur le serveur et sélectionnez **Nouvelle requête**.
1. Entrez la requête SQL suivante, puis sélectionnez **Exécuter** :

    ```sql
    SELECT *
    FROM pg_catalog.pg_stats;
    ```

1. Notez que vous pouvez afficher une grande quantité d’informations sur les statistiques.
1. En utilisant des vues système, vous pouvez réduire la complexité des requêtes SQL que vous devez écrire. La requête précédente aurait besoin du code suivant si vous n’utilisiez pas la vue **pg_stats** :

    ```sql
    SELECT n.nspname AS schemaname,
    c.relname AS tablename,
    a.attname,
    s.stainherit AS inherited,
    s.stanullfrac AS null_frac,
    s.stawidth AS avg_width,
    s.stadistinct AS n_distinct,
        CASE
            WHEN s.stakind1 = 1 THEN s.stavalues1
            WHEN s.stakind2 = 1 THEN s.stavalues2
            WHEN s.stakind3 = 1 THEN s.stavalues3
            WHEN s.stakind4 = 1 THEN s.stavalues4
            WHEN s.stakind5 = 1 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS most_common_vals,
        CASE
            WHEN s.stakind1 = 1 THEN s.stanumbers1
            WHEN s.stakind2 = 1 THEN s.stanumbers2
            WHEN s.stakind3 = 1 THEN s.stanumbers3
            WHEN s.stakind4 = 1 THEN s.stanumbers4
            WHEN s.stakind5 = 1 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS most_common_freqs,
        CASE
            WHEN s.stakind1 = 2 THEN s.stavalues1
            WHEN s.stakind2 = 2 THEN s.stavalues2
            WHEN s.stakind3 = 2 THEN s.stavalues3
            WHEN s.stakind4 = 2 THEN s.stavalues4
            WHEN s.stakind5 = 2 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS histogram_bounds,
        CASE
            WHEN s.stakind1 = 3 THEN s.stanumbers1[1]
            WHEN s.stakind2 = 3 THEN s.stanumbers2[1]
            WHEN s.stakind3 = 3 THEN s.stanumbers3[1]
            WHEN s.stakind4 = 3 THEN s.stanumbers4[1]
            WHEN s.stakind5 = 3 THEN s.stanumbers5[1]
            ELSE NULL::real
        END AS correlation,
        CASE
            WHEN s.stakind1 = 4 THEN s.stavalues1
            WHEN s.stakind2 = 4 THEN s.stavalues2
            WHEN s.stakind3 = 4 THEN s.stavalues3
            WHEN s.stakind4 = 4 THEN s.stavalues4
            WHEN s.stakind5 = 4 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS most_common_elems,
        CASE
            WHEN s.stakind1 = 4 THEN s.stanumbers1
            WHEN s.stakind2 = 4 THEN s.stanumbers2
            WHEN s.stakind3 = 4 THEN s.stanumbers3
            WHEN s.stakind4 = 4 THEN s.stanumbers4
            WHEN s.stakind5 = 4 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS most_common_elem_freqs,
        CASE
            WHEN s.stakind1 = 5 THEN s.stanumbers1
            WHEN s.stakind2 = 5 THEN s.stanumbers2
            WHEN s.stakind3 = 5 THEN s.stanumbers3
            WHEN s.stakind4 = 5 THEN s.stanumbers4
            WHEN s.stakind5 = 5 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS elem_count_histogram
    FROM pg_statistic s
     JOIN pg_class c ON c.oid = s.starelid
     JOIN pg_attribute a ON c.oid = a.attrelid AND a.attnum = s.staattnum
     LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
    WHERE NOT a.attisdropped AND has_column_privilege(c.oid, a.attnum, 'select'::text) AND (c.relrowsecurity = false OR NOT row_security_active(c.oid));
    ```

## Nettoyage de l’exercice

1. Le serveur Azure Database pour PostgreSQL que nous avons déployé dans cet exercice entraîne des frais, vous pouvez supprimer le serveur après cet exercice. Vous pouvez également supprimer le groupe de ressources **rg-learn-work-with-postgresql-eastus** pour supprimer toutes les ressources que nous avons déployées dans le cadre de cet exercice.
1. Si nécessaire, supprimez le dossier .\DP3021Lab.
