---
lab:
  title: Explorer Azure Database pour PostgreSQL
  module: Explore PostgreSQL architecture
---

# Explorer Azure Database pour PostgreSQL

Dans cet exercice, vous allez créer un serveur flexible Azure Database pour PostgreSQL et configurer la période de rétention de sauvegarde.

## Avant de commencer

Pour effectuer cet exercice, vous avez besoin de votre propre abonnement Azure. Si vous n’avez pas d’abonnement Azure, vous pouvez demander un [essai gratuit d’Azure](https://azure.microsoft.com/free).

### Création d’un groupe de ressources

1. Dans un navigateur web, accédez au [portail Azure](https://portal.azure.com). Connectez-vous à l’aide d’un compte propriétaire ou contributeur.
2. Sous Services Azure, sélectionnez **Groupes de ressources**, puis **+ Créer**.
3. Vérifiez que l’abonnement approprié s’affiche, puis entrez le nom de groupe de ressources **rg-PostgreSQL_Flexi**. Sélectionnez une **Région**.
4. Sélectionnez **Revoir + créer**. Sélectionnez ensuite **Créer**.

### Créer un serveur flexible Azure Database pour PostgreSQL

1. Sous Services Azure, sélectionnez **Créer une ressource**.
    1. Dans **Recherche sur le Marketplace**, tapez **`azure database for postgresql flexible server`**, choisissez **Serveur flexible Azure Database pour PostgreSQL**, puis cliquez sur **Créer**.
1. Sous l’onglet **Général** du serveur flexible, renseignez chaque champ comme suit :
    1. Abonnement : votre abonnement.
    1. Groupe de ressources : **rg-PostgreSQL_Flexi**.
    1. Nom du serveur : **psql-postgresql-fx9999** (le nom du serveur doit être globalement unique, remplacez donc 9999 par quatre chiffres aléatoires).
    1. Région - sélectionnez la même région que le groupe de ressources.
    1. Version de PostgreSQL : sélectionnez 16.
    1. Type de charge de travail - **Développement**.
    1. Calcul + stockage - **Burstable, B1ms**. Sélectionnez **Configurer le serveur** et examinez les options de configuration. N’apportez aucune modification et fermez le volet dans le coin supérieur droit.
    1. Zone de disponibilité - Aucune préférence.
    1. Haute disponibilité : activée.
    1. Méthode d’authentification : **authentification PostgreSQL uniquement**
    1. Dans **Nom d’utilisateur administrateur**, entrez **`demo`**.
    1. Dans **Mot de passe**, entrez un mot de passe complexe approprié.
    1. Sélectionnez **Suivant : Réseau >**.
1. Sous l’onglet **Mise en réseau** du serveur flexible, renseignez chaque champ comme suit :
    1. Méthode de connectivité : (o) Accès public (adresses IP autorisées) et point de terminaison privé.
    1. Accès public : sélectionnez **Autoriser l’accès public à cette ressource via Internet à l’aide d’une adresse IP publique**.
    1. Sous Règles de pare-feu, sélectionnez **+ Ajouter l’adresse IP cliente actuelle** pour ajouter votre adresse IP actuelle en tant que règle de pare-feu. Vous pouvez éventuellement nommer cette règle de pare-feu en quelque chose de significatif.
1. Sélectionnez **Revoir + créer**. Passez en revue vos paramètres, puis sélectionnez **Créer** pour créer votre serveur flexible Azure Database pour PostgreSQL. Une fois le déploiement terminé, sélectionnez **Accéder à la ressource** prête pour l’étape suivante.

## Examiner les paramètres du serveur

1. Sous **Paramètres**, sélectionnez **Paramètres du serveur**.
1. Dans la zone **Rechercher pour filtrer les éléments...**, entrez **`connections`**. Les paramètres de serveur liés aux connexions sont affichés. Notez la valeur de **max_connections**. N’apportez aucune modification.
1. Dans le menu de gauche, sélectionnez **Vue d’ensemble** pour quitter les **Paramètres du serveur**.

## Changer de période de rétention de la sauvegarde

1. Accédez au panneau **Vue d’ensemble**, sous **Paramètres**, et sélectionnez **Calcul + stockage**. Cette section affiche votre niveau de calcul actuel et l’option de mise à niveau. Elle affiche également la quantité de stockage que vous avez provisionnée et l’option d’augmentation du stockage.
1. Sous **Sauvegardes**, la période de rétention de sauvegarde en jours s’affiche. À l’aide de la barre du curseur, remplacez la période de rétention de sauvegarde par 14 jours. Sélectionnez **Enregistrer** pour conserver vos modifications.
1. Quand vous avez terminé cet exercice, accédez au panneau **Vue d’ensemble**, puis sélectionnez **ARRÊTER** pour arrêter le serveur.
    1. Vous ne serez pas facturé pendant que le serveur est arrêté, mais sachez que le serveur sera redémarré sous sept jours si vous ne l’avez pas supprimé.

## Exercice facultatif : Configurer un serveur à haute disponibilité

1. Sous Services Azure, sélectionnez **Créer une ressource**.
    1. Dans **Rechercher dans la Place de marché**, tapez**`azure database for postgresql flexible server`**, choisissez **Serveur flexible Azure Database pour PostgreSQL**, puis cliquez sur **Créer.**
1. Sous l’onglet **Général** du serveur flexible, renseignez chaque champ comme suit :
    1. Abonnement : votre abonnement.
    1. Groupe de ressources : **rg-PostgreSQL_Flexi**.
    1. Nom du serveur : **psql-postgresql-fx8888** (le nom du serveur doit être globalement unique, remplacez donc 8888 par quatre chiffres aléatoires).
    1. Région - sélectionnez la même région que le groupe de ressources.
    1. Version de PostgreSQL : sélectionnez 16.
    1. Type de charge de travail - **Production (petite/moyenne)**
    1. Calcul + stockage : laissez sur **Usage général**.
    1. Zone de disponibilité : vous pouvez laisser ce paramètre sur « Aucune préférence » et Azure choisira automatiquement les zones de disponibilité pour vos serveurs principaux et secondaires. Vous pouvez également spécifier une zone de disponibilité pour colocaliser votre application.
    1. Activer la haute disponibilité - Cochez. Notez les coûts estimés lorsque cette option est sélectionnée.
    1. Mode haute disponibilité : choisissez **Redondance interzone : un serveur de secours est toujours disponible dans une autre zone de la même région que le serveur principal**
    1. Méthode d’authentification : **authentification PostgreSQL uniquement**
    1. Dans **Nom d’utilisateur de l’administrateur**, entrez **demo**.
    1. Dans **Mot de passe**, entrez un mot de passe complexe approprié.
    1. Sélectionnez **Suivant : Réseau >**.
1. Sous l’onglet **Mise en réseau** du serveur flexible, renseignez chaque champ comme suit :
    1. Méthode de connectivité : (o) Accès public (adresses IP autorisées) et point de terminaison privé.
    1. Accès public : sélectionnez **Autoriser l’accès public à cette ressource via Internet à l’aide d’une adresse IP publique**.
    1. Sous Règles de pare-feu, sélectionnez **+ Ajouter l’adresse IP cliente actuelle** pour ajouter votre adresse IP actuelle en tant que règle de pare-feu. Vous pouvez éventuellement nommer cette règle de pare-feu en quelque chose de significatif.
1. Sélectionnez **Revoir + créer**. Passez en revue vos paramètres, puis sélectionnez **Créer** pour créer votre serveur flexible Azure Database pour PostgreSQL. Une fois le déploiement terminé, sélectionnez **Accéder à la ressource** prête pour l’étape suivante.

### Inspecter le nouveau serveur

1. Sous **Paramètres**, sélectionnez **Haute disponibilité**. La haute disponibilité est activée et la zone de disponibilité principale devrait être 1.
    1. La zone de disponibilité de secours a été allouée automatiquement et sera différente de la zone de disponibilité principale ; elle est généralement appelée zone 2.

### Forcer un basculement

1. Dans le panneau **Haute disponibilité**, dans le menu supérieur, sélectionnez **Basculement forcé**. Un message s’affiche ; sélectionnez **OK**.
1. Le processus de basculement démarre. Un message s’affiche lorsque l’opération de basculement s’est terminée avec succès.
1. Dans le panneau **Haute disponibilité**, vous pouvez voir que la zone principale est maintenant 2 et que la zone de disponibilité de secours est 1. Vous devrez peut-être actualiser votre navigateur pour voir les dernières informations.
1. Lorsque vous avez terminé cet exercice, supprimez le serveur.

## Nettoyage

Le serveur de cet exercice de labo entraîne des frais. Supprimez le groupe de ressources **rg-PostgreSQL_Flexi** une fois l’exercice terminé. Cela supprime le serveur et toutes les autres ressources que vous avez déployées dans cet exercice de labo.
