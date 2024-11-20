---
lab:
  title: Configurer les autorisations dans Azure Database pour PostgreSQL
  module: Secure Azure Database for PostgreSQL
---

# Configurer les autorisations dans Azure Database pour PostgreSQL

Dans ces exercices de labo, vous allez attribuer des rôles RBAC afin de contrôler l’accès aux ressources Azure Database pour PostgreSQL et PostgreSQL GRANTS pour contrôler l’accès aux opérations de base de données.

## Avant de commencer

Pour effectuer cet exercice, vous avez besoin de votre propre abonnement Azure. Si vous n’avez pas d’abonnement Azure, créez un [essai gratuit d’Azure](https://azure.microsoft.com/free).

Pour effectuer ces exercices, vous devez installer un serveur PostgreSQL connecté à Microsoft Entra ID (anciennement Azure Active Directory).

### Création d’un groupe de ressources

1. Dans un navigateur web, accédez au [portail Azure](https://portal.azure.com). Connectez-vous à l’aide d’un compte propriétaire ou contributeur.
2. Sous Services Azure, sélectionnez **Groupes de ressources**, puis **+ Créer**.
3. Vérifiez que l’abonnement approprié s’affiche, puis entrez le nom de groupe de ressources **rg-PostgreSQL_Entra**. Sélectionnez une **Région**.
4. Sélectionnez **Revoir + créer**. Sélectionnez ensuite **Créer**.

## Créer un serveur flexible Azure Database pour PostgreSQL

1. Sous Services Azure, sélectionnez **Créer une ressource**.
    1. Dans **Rechercher dans la Place de marché**, tapez**`azure database for postgresql flexible server`**, choisissez **Serveur flexible Azure Database pour PostgreSQL**, puis cliquez sur **Créer.**
1. Sous l’onglet **Général** du serveur flexible, renseignez chaque champ comme suit :
    1. Abonnement : votre abonnement.
    1. Groupe de ressources : **rg-PostgreSQL_Entra**.
    1. Nom du serveur : **psql-postgresql-fx7777** (le nom du serveur doit être globalement unique, remplacez donc 7777 par quatre chiffres aléatoires).
    1. Région - sélectionnez la même région que le groupe de ressources.
    1. Version de PostgreSQL : sélectionnez 16.
    1. Type de charge de travail - **Développement**.
    1. Calcul + stockage - **Burstable, B1ms**.
    1. Zone de disponibilité - Aucune préférence.
    1. Haute disponibilité - Laissez la case décochée.
    1. Méthode d’authentification : sélectionnez **Authentification PostgreSQL et Microsoft Entra**.
    1. Définir l’administrateur Microsoft Entra : sélectionnez **Définir l’administrateur**.
        1. Recherchez votre compte dans **Sélectionner les administrateurs Microsoft Entra** et (o) votre compte, puis sélectionnez **Sélectionner**.
    1. Dans **Nom d’utilisateur administrateur**, entrez **`demo`**.
    1. Dans **Mot de passe**, entrez un mot de passe complexe approprié.
    1. Sélectionnez **Suivant : Réseau >**.
1. Sous l’onglet **Mise en réseau** du serveur flexible, renseignez chaque champ comme suit :
    1. Méthode de connectivité : (o) Accès public (adresses IP autorisées) et point de terminaison privé.
    1. Accès public : sélectionnez **Autoriser l’accès public à cette ressource via Internet à l’aide d’une adresse IP publique**.
    1. Sous Règles de pare-feu, sélectionnez **+ Ajouter l’adresse IP cliente actuelle** pour ajouter votre adresse IP actuelle en tant que règle de pare-feu. Vous pouvez éventuellement nommer cette règle de pare-feu en quelque chose de significatif. Ajoutez également **Ajouter 0.0.0.0 - 255.255.255.255**, puis sélectionnez **Continuer**.
1. Sélectionnez **Revoir + créer**. Passez en revue vos paramètres, puis sélectionnez **Créer** pour créer votre serveur flexible Azure Database pour PostgreSQL. Une fois le déploiement terminé, sélectionnez **Accéder à la ressource** prête pour l’étape suivante.

## Installer Azure Data Studio

Pour installer Azure Data Studio afin de l’utiliser avec Azure Database pour PostgreSQL :

1. Dans un navigateur, accédez à [Télécharger et installer Azure Data Studio](/sql/azure-data-studio/download-azure-data-studio) et, sous la plateforme Windows, sélectionnez **Programme d’installation utilisateur (recommandé).** Le fichier exécutable est téléchargé dans votre dossier Téléchargements.
1. Sélectionnez **Ouvrir le fichier**.
1. Le contrat de licence s’affiche. Lisez et **acceptez le contrat**, puis sélectionnez **Suivant**.
1. Dans **Sélectionner des tâches supplémentaires**, sélectionnez **Ajouter à PATH** et tout autre ajout nécessaire. Cliquez sur **Suivant**.
1. La boîte de dialogue **Prêt à installer** s’affiche. Passez vos paramètres en revue. Sélectionnez **Précédent** pour apporter des modifications, ou sélectionnez **Installer**.
1. La boîte de dialogue **Fin de l’Assistant Installation d’Azure Data Studio** s’affiche. Sélectionnez **Terminer**. Azure Data Studio démarre.

### Installer l’extension PostgreSQL

1. Ouvrez Azure Data Studio s’il n’est pas déjà ouvert.
1. Dans le menu de gauche, sélectionnez **Extensions** pour afficher le panneau Extensions.
1. Dans la barre de recherche, entrez **PostgreSQL**. L’icône de l’extension PostgreSQL pour Azure Data Studio s’affiche.
1. Sélectionnez **Installer**. L’extension s’installe.

### Se connecter au serveur flexible Azure Database pour PostgreSQL

1. Ouvrez Azure Data Studio s’il n’est pas déjà ouvert.
1. Dans le menu de gauche, sélectionnez **Connexions**.
1. Sélectionnez **Nouvelle connexion**.
1. Sous **Détails de la connexion**, dans **Type de connexion**, sélectionnez **PostgreSQL** dans la liste déroulante.
1. Dans **Nom du serveur**, entrez le nom complet du serveur tel qu’il apparaît sur le portail Azure.
1. Dans **Type d’authentification**, laissez Mot de passe.
1. Dans Nom d’utilisateur et Mot de passe, entrez le nom d’utilisateur **demo** et le mot de passe complexe que vous avez créé ci-dessus.
1. Sélectionnez [ x ] Mémoriser le mot de passe.
1. Les champs restants sont facultatifs.
1. Sélectionnez **Connecter**. Vous êtes maintenant connecté au serveur Azure Database pour PostgreSQL.
1. Une liste des bases de données serveur s’affiche. Elle inclut les bases de données système et les bases de données utilisateur.

### Créer la base de données zoo

1. Accédez au dossier avec vos fichiers de script d’exercice ou téléchargez **Lab2_ZooDb.sql** à partir de [MSLearn PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql/Allfiles/Labs/02).
1. Ouvrez Azure Data Studio s’il n’est pas déjà ouvert.
1. Sélectionnez **Fichier**, **Ouvrir le fichier**, puis accédez au dossier dans lequel vous avez enregistré le script. Sélectionnez **../Allfiles/Labs/02/Lab2_ZooDb.sql** et **Ouvri**. Si un avertissement d’approbation s’affiche, sélectionnez **Ouvrir**.
1. Exécutez le script. La base de données zoodb est créée.

## Créer un compte d’utilisateur dans Microsoft Entra ID

> [!NOTE]
> Dans la plupart des environnements de production ou de développement, il est très possible que vous n’ayez pas les privilèges de compte d’abonnement pour créer des comptes sur votre service Microsoft Entra ID.  Dans ce cas, si votre organisation l’autorise, essayez de demander à votre administrateur Microsoft Entra ID de créer un compte de test pour vous. Si vous ne parvenez pas à obtenir le compte de test Entra, ignorez cette section et passez à la section **Accorder l’accès GRANT à Azure Database pour PostgreSQL**. 

1. Dans le [portail Azure](https://portal.azure.com), connectez-vous en utilisant un compte Propriétaire et accédez à Microsoft Entra ID.
1. Sous **Gérer**, sélectionnez **Utilisateurs**.
1. En haut à gauche, sélectionnez **Nouvel utilisateur**, puis sélectionnez **Créer un utilisateur**.
1. Dans la page **Nouvel utilisateur**, entrez ces détails, puis sélectionnez **Créer** :
    - **Nom d’utilisateur principal :** choisissez un nom de principe.
    - **Nom d’affichage :** choisissez un nom d’affichage.
    - **Mot de passe :** décochez **Générer automatiquement un mot de passe**, puis entrez un mot de passe fort. Prenez note du nom de principal et du mot de passe.
    - Cliquez sur **Vérifier + créer**.

    > [!TIP]
    > Quand l’utilisateur est créé, notez le **Nom d’utilisateur principal** complet afin de pouvoir l’utiliser ultérieurement pour vous connecter.

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
1. Choisissez **Me demander plus tard** si vous êtes invité à utiliser l’authentification multifacteur.
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

1. Ouvrez Azure Data Studio et connectez-vous à votre serveur Azure Database pour PostgreSQL à l’aide de l’utilisateur de **démonstration** que vous avez défini comme administrateur ci-dessus.
1. Dans le volet de requête, exécutez ce code sur la base de données postgres. Douze rôles d’utilisateur doivent être retournés, notamment le rôle de **démonstration** que vous utilisez pour vous connecter :

    ```SQL
    SELECT rolname FROM pg_catalog.pg_roles;
    ```

1. Pour créer un rôle, exécutez ce code

    ```SQL
    CREATE ROLE dbuser WITH LOGIN NOSUPERUSER INHERIT CREATEDB NOCREATEROLE NOREPLICATION PASSWORD 'R3placeWithAComplexPW!';
    GRANT CONNECT ON DATABASE zoodb TO dbuser;
    ```
    > [!NOTE]
    > Veillez à remplacer le mot de passe dans le script ci-dessus par un mot de passe complexe.

1. Pour répertorier le nouveau rôle, réexécutez la requête SELECT dans **pg_catalog.pg_roles**. Le rôle **dbuser** doit être répertorié.
1. Pour permettre au nouveau rôle d’interroger et de modifier les données contenues dans la table **animal** de la base de données **zoodb**, exécutez ce code sur la base de données zoodb :

    ```SQL
    GRANT SELECT, INSERT, UPDATE, DELETE ON animal TO dbuser;
    ```

## Tester le nouveau rôle

1. Dans Azure Data Studio, dans la liste des **CONNEXIONS**, sélectionnez le bouton Nouvelle connexion.
1. Dans la liste **Type de connexion**, sélectionnez **PostgreSQL**.
1. Dans la zone de texte **Nom du serveur**, tapez le nom complet du serveur pour votre ressource Azure Database pour PostgreSQL. Vous pouvez le copier à partir du portail Azure.
1. Dans la liste **Type d’authentification**, sélectionnez **Mot de passe**.
1. Dans la zone de texte **Nom d’utilisateur**, entrez **dbuser** et, dans la zone de texte **Mot de passe**, entrez le mot de passe complexe avec lequel vous avez créé le compte.
1. Cochez la case **Mémoriser le mot de passe**, puis sélectionnez **Se connecter**.
1. Sélectionnez **Nouvelle requête**, puis exécutez ce code :

    ```SQL
    SELECT * FROM animal;
    ```

1. Pour effectuer un test afin de déterminer si vous disposez du privilège UPDATE, exécutez ce code :

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

Ces tests montrent que le nouvel utilisateur peut exécuter des commandes DML (Data Manipulation Language) pour interroger et modifier les données, mais qu’il ne peut pas utiliser les commandes DDL (Data Definition Language) pour modifier le schéma. En outre, le nouvel utilisateur ne peut pas accorder de nouveaux privilèges (GRANT) pour contourner les autorisations.

## Nettoyage

Vous n’utiliserez plus ce serveur PostgreSQL. Supprimez donc le groupe de ressources que vous avez créé, ce qui supprimera également le serveur.
