---
lab:
  title: Migration en ligne de base de données PostgreSQL
  module: Migrate to Azure Database for PostgreSQL Flexible Server
---

## Migration en ligne de base de données PostgreSQL

Dans cet exercice, vous allez configurer la réplication logique entre un serveur PostgreSQL source et un serveur flexible Azure Database pour PostgreSQL afin de permettre une activité de migration en ligne.

## Avant de commencer

Pour effectuer cet exercice, vous avez besoin de votre propre abonnement Azure. Si vous n’avez pas d’abonnement Azure, vous pouvez demander un [essai gratuit d’Azure](https://azure.microsoft.com/free).

> **Remarque :** cet exercice nécessite que le serveur que vous utilisez comme source de la migration soit accessible au serveur flexible Azure Database pour PostgreSQL afin qu’il puisse connecter et migrer les bases de données. Cela nécessite que le serveur source soit accessible via une adresse IP et un port publics. > Une liste d’adresses IP de région Azure peut être téléchargée à partir de [Plages d’adresses IP Azure et étiquettes de service : cloud public](https://www.microsoft.com/en-gb/download/details.aspx?id=56519) pour réduire les plages autorisées d’adresses IP dans vos règles de pare-feu en fonction de la région Azure utilisée.

Ouvrez le pare-feu de votre serveur pour permettre à la fonctionnalité Migration dans le serveur flexible Azure Database pour PostgreSQL d’accéder au serveur PostgreSQL source, par défaut le port TCP 5432.
>
Lorsque vous utilisez une appliance de pare-feu devant vos bases de données sources, vous devrez peut-être ajouter des règles de pare-feu pour permettre à la fonctionnalité Migration dans le serveur flexible Azure Database pour PostgreSQL d’accéder aux bases de données sources pour la migration.
>
> La version maximale prise en charge de PostgreSQL pour la migration est la version 16.

### Prérequis

> **Remarque** : avant de commencer cet exercice, vous devrez avoir effectué l’exercice précédent afin que les bases de données source et cible soient prêtes pour la configuration de la réplication logique, car cet exercice s’appuie sur l’activité précédente.

## Créer une publication - Serveur source

1. Ouvrez PGAdmin et connectez-vous au serveur source contenant la base de données qui va agir comme source pour la synchronisation des données vers le serveur flexible Azure Database pour PostgreSQL.
1. Ouvrez une nouvelle fenêtre de requête connectée à la base de données source contenant les données que nous voulons synchroniser.
1. Configurez le paramètre wal_level du serveur source sur **logical** pour permettre la publication des données.
    1. Recherchez et ouvrez le fichier **postgresql.conf** dans le répertoire bin du répertoire d’installation PostgreSQL.
    1. Recherchez la ligne contenant le paramètre de configuration **wal_level**.
    1. Vérifiez que la ligne n’est pas commentée et définissez la valeur sur **logical**.
    1. Enregistrez et fermez le fichier.
    1. Redémarrez le service PostgreSQL.
1. À présent, configurez une publication qui contiendra toutes les tables de la base de données.

    ```SQL
    CREATE PUBLICATION migration1 FOR ALL TABLES;
    ```

## Créer un abonnement - Serveur cible

1. Ouvrez PGAdmin et connectez-vous au serveur flexible Azure Database pour PostgreSQL contenant la base de données qui va agir comme cible pour la synchronisation de données à partir du serveur source.
1. Ouvrez une nouvelle fenêtre de requête connectée à la base de données source contenant les données que nous voulons synchroniser.
1. Créez l’abonnement au serveur source.

    ```sql
    CREATE SUBSCRIPTION migration1
    CONNECTION 'host=<source server name> port=<server port> dbname=adventureworks application_name=migration1 user=<username> password=<password>'
    PUBLICATION migration1
    WITH(copy_data = false)
    ;    
    ```

1. Vérifiez l’état de réplication des tables.

    ```SQL
    SELECT PT.schemaname, PT.tablename,
        CASE PS.srsubstate
            WHEN 'i' THEN 'initialize'
            WHEN 'd' THEN 'data is being copied'
            WHEN 'f' THEN 'finished table copy'
            WHEN 's' THEN 'synchronized'
            WHEN 'r' THEN ' ready (normal replication)'
            ELSE 'unknown'
        END AS replicationState
    FROM pg_publication_tables PT,
            pg_subscription_rel PS
            JOIN pg_class C ON (C.oid = PS.srrelid)
            JOIN pg_namespace N ON (N.oid = C.relnamespace)
    ;
    ```

## Tester la réplication des données

1. Sur le serveur source, vérifiez le nombre de lignes de la table workorder.

    ```SQL
    SELECT COUNT(*) FROM production.workorder;
    ```

1. Sur le serveur cible, vérifiez le nombre de lignes de la table workorder.

    ```SQL
    SELECT COUNT(*) FROM production.workorder;
    ```

1. Vérifiez que les nombres de lignes correspondent.
1. Téléchargez maintenant le fichier Lab11_workorder.csv à partir du référentiel qui se trouve [ici](https://github.com/MicrosoftLearning/mslearn-postgresql/tree/main/Allfiles/Labs/11) vers C:\
1. Chargez de nouvelles données dans la table workorder du serveur source à partir du fichier CSV à l’aide de la commande suivante.

    ```Bash
    psql --host=localhost --port=5432 --username=postgres --dbname=adventureworks --command="\COPY production.workorder FROM 'C:\Lab11_workorder.csv' CSV HEADER"
    ```

La sortie de la commande doit être `COPY 490`, indiquant que 490 lignes supplémentaires ont été écrites dans la table à partir du fichier CSV.

1. Vérifiez que le nombre de lignes de la table workorder de la source (72591 lignes) est le même dans la destination pour vous assurer que la réplication des données fonctionne.

## Nettoyage de l’exercice

Le serveur Azure Database pour PostgreSQL que nous avons déployé dans cet exercice entraîne des frais, vous pouvez supprimer le serveur après cet exercice. Vous pouvez également supprimer le groupe de ressources **rg-learn-work-with-postgresql-eastus** pour supprimer toutes les ressources que nous avons déployées dans le cadre de cet exercice.
