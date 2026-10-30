Incident PostgreSQL -- Saturation du nombre de connexions

1. Contexte

L'application Satisfaction a rencontré un problème de démarrage sur
l'environnement de production.

Les pods applicatifs ne parvenaient plus à démarrer correctement.
L'analyse des logs Kubernetes a mis en évidence une erreur lors de
l'initialisation Spring Boot / Flyway.

Erreur observée

SQL State : 53300
Error Code : 0

FATAL: remaining connection slots are reserved for
non-replication superuser connections

L'exception remontait notamment depuis Flyway :

org.flywaydb.core.internal.exception.FlywaySqlUnableToConnectToDbException:
Unable to obtain connection from database

La chaîne d'erreur montrait ensuite le passage par PostgreSQL et
HikariCP.

2. Impact

La saturation des connexions PostgreSQL empêchait l'ouverture de
nouvelles connexions.

Lors du démarrage d'un pod, Flyway doit obtenir une connexion afin de
vérifier/appliquer les migrations de la base. PostgreSQL refusant toute
nouvelle connexion applicative, l'initialisation Spring Boot échouait et
le pod ne pouvait pas démarrer correctement.

Saturation des connexions PostgreSQL
              |
              v
Plus de slot disponible
              |
              v
Flyway ne peut pas obtenir de connexion
              |
              v
Échec de l'initialisation Spring Boot
              |
              v
Pod applicatif indisponible

3. Configuration applicative constatée

L'application utilise HikariCP comme pool de connexions JDBC.

hikari:
  connection-timeout: 10000
  maximum-pool-size: 20

maximum-pool-size: 20 signifie qu'une instance de l'application peut
disposer d'un pool allant jusqu'à 20 connexions.

Il faut donc prendre en compte le nombre de replicas de l'application
lors du dimensionnement global des connexions PostgreSQL.

5 replicas × 20 connexions maximum
= jusqu'à 100 connexions potentielles

4. Diagnostic PostgreSQL

Compter les connexions

SELECT count(*) AS connexions_actuelles
FROM pg_stat_activity;

Lors de l'incident, environ 343 entrées étaient présentes.

Répartition par utilisateur

SELECT
    usename,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY usename
ORDER BY connections DESC;

Cette vérification a permis de constater une concentration importante
des connexions sur un compte applicatif.

Analyse détaillée

SELECT
    usename,
    application_name,
    client_addr,
    state,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY
    usename,
    application_name,
    client_addr,
    state
ORDER BY connections DESC;

Attention : sur IBM Cloud Databases for PostgreSQL, certaines
informations de pg_stat_activity peuvent être masquées ou
restreintes selon les privilèges du compte utilisé.

5. Particularité rencontrée sur IBM Cloud PostgreSQL

La commande PostgreSQL classique :

SHOW max_connections;

n'était pas disponible avec le compte utilisé :

ERROR: unrecognized configuration parameter "max_connections"

Il ne faut donc pas considérer que l'absence de résultat signifie que
PostgreSQL n'impose aucune limite.

Le message suivant confirmait que la capacité disponible pour les
connexions applicatives avait été atteinte :

FATAL: remaining connection slots are reserved for
non-replication superuser connections

6. Procédure appliquée

Étape 1 -- Arrêt temporaire des pods

Les replicas applicatifs ont d'abord été positionnés à 0 afin
d'empêcher les pods de recréer immédiatement des connexions pendant le
nettoyage.

kubectl scale deployment <deployment> \
  -n <namespace> \
  --replicas=0

Étape 2 -- Identification des connexions

SELECT
    usename,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY usename
ORDER BY connections DESC;

Cette étape est importante avant toute suppression afin d'éviter
d'impacter les connexions appartenant à d'autres applications.

Étape 3 -- Libération des connexions applicatives

SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE usename = '<USER_APPLICATION>'
  AND pid <> pg_backend_pid();

Le retour t indique que PostgreSQL a accepté la terminaison de la
session concernée.

Important : ne pas exécuter une terminaison globale de toutes les
sessions de pg_stat_activity. La base étant managée et
potentiellement utilisée par plusieurs composants, cela pourrait
provoquer une interruption plus large.

Certaines tentatives sur des processus protégés peuvent retourner :

ERROR: must be a superuser to terminate superuser process

Ces processus ne doivent pas être terminés par le compte applicatif.

Étape 4 -- Vérification après nettoyage

SELECT
    usename,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY usename
ORDER BY connections DESC;

Le nombre de connexions applicatives a fortement diminué, libérant
suffisamment de slots pour permettre le redémarrage de l'application.

7. Redémarrage contrôlé

L'application a ensuite été redémarrée progressivement. Dans un premier
temps, un seul replica a été lancé :

kubectl scale deployment <deployment> \
  -n <namespace> \
  --replicas=1

Surveillance :

kubectl get pods -n <namespace> -w

Contrôle des logs :

kubectl logs -n <namespace> -f deployment/<deployment>

Le pod a démarré correctement et Flyway a de nouveau pu établir une
connexion PostgreSQL. Le service est ensuite redevenu opérationnel.

8. Résolution

Statut : Résolu

La cause immédiate de l'indisponibilité était la saturation des
connexions PostgreSQL, empêchant l'application et Flyway d'obtenir une
nouvelle connexion.

La résolution a consisté à :

arrêter temporairement les replicas applicatifs ;

identifier la consommation des connexions via pg_stat_activity ;

terminer uniquement les connexions applicatives concernées ;

vérifier la libération des slots ;

redémarrer progressivement l'application ;

vérifier le bon démarrage de Spring Boot/Flyway.

Aucun redémarrage complet de l'instance PostgreSQL n'a finalement été
nécessaire.

9. Cause racine à confirmer

La saturation PostgreSQL est la cause technique immédiate de
l'indisponibilité, mais la raison pour laquelle autant de connexions
se sont accumulées reste à déterminer.

Les points suivants devront notamment être analysés :

nombre de replicas applicatifs au moment de l'incident ;

configuration maximum-pool-size de HikariCP ;

valeur de minimum-idle ;

idle-timeout ;

max-lifetime ;

éventuelle fuite de connexions ;

connexions non correctement fermées par certains traitements ;

nombre de pools Hikari réellement créés ;

dimensionnement de l'instance IBM Cloud PostgreSQL ;

éventuelle augmentation de la limite de connexions auprès de
l'équipe responsable de la base.

Il est préférable de ne pas conclure à ce stade que
maximum-pool-size: 20 est directement la cause de l'incident.

10. Surveillance recommandée

Connexions par utilisateur

SELECT
    usename,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY usename
ORDER BY connections DESC;

Nombre total de connexions

SELECT count(*) AS total_connections
FROM pg_stat_activity;

Répartition par état

SELECT
    usename,
    state,
    count(*) AS connections
FROM pg_stat_activity
GROUP BY usename, state
ORDER BY connections DESC;

Il serait également pertinent de mettre en place une alerte avant
saturation afin d'intervenir avant que PostgreSQL ne commence à
refuser les nouvelles connexions.

11. Runbook rapide en cas de récidive

Erreur SQLSTATE 53300
        |
        v
Vérifier pg_stat_activity
        |
        v
Identifier l'utilisateur consommant les connexions
        |
        v
Scaler l'application concernée à 0
        |
        v
Terminer uniquement ses connexions
        |
        v
Contrôler la baisse du nombre de sessions
        |
        v
Relancer 1 replica
        |
        v
Contrôler Flyway + Spring Boot
        |
        v
Surveiller la remontée des connexions
        |
        v
Remonter progressivement les autres replicas

12. Conclusion pour le post-mortem

Cause de l'incident constatée : épuisement des connexions PostgreSQL
disponibles.

Cause racine : à confirmer par une analyse complémentaire du
comportement du pool Hikari et du dimensionnement PostgreSQL.

Cette distinction permet de ne pas attribuer prématurément








Voici mon retour concernant la migration d'hier soir.

Globalement, la migration s'est bien déroulée, malgré plusieurs difficultés techniques qui ont rallongé la durée de l'intervention.

**Les principaux points rencontrés :**

- **Terraform :** les jobs ont été particulièrement lents, notamment lors de la création, de la suppression et de la promotion des bases PostgreSQL, ainsi que de la migration de la VIP. Tous les pipelines ont néanmoins été exécutés avec succès.

- **XL Release :** nous avons rencontré plusieurs blocages au niveau des étapes de vérification *Check QUAL* et *Check PROD*, nécessitant plusieurs relances manuelles pour poursuivre la migration.

- **Vault :** nous avons également rencontré un problème de droits d'accès (*Access Denied*) sur le chemin Vault contenant le certificat utilisé par l'application. J'ai corrigé les permissions nécessaires, puis procédé au redéploiement de l'application.

- **Durée de l'intervention :** initialement estimée à environ 2 heures, la migration s'est finalement prolongée jusqu'à 23h30 en raison de ces différents ralentissements et blocages.

**Vérifications post-MEP :**

Après le déploiement, plusieurs contrôles ont été effectués afin de s'assurer du bon fonctionnement de l'application sur le nouvel environnement OpenShift (ROKS) :

- **Logs applicatifs :** consultation et analyse des logs pour détecter d'éventuelles erreurs au démarrage ou pendant l'exécution.
- **Dynatrace :** vérification de la supervision applicative afin d'identifier d'éventuelles anomalies ou dégradations de performance.
- **Appels API :** attente et surveillance des premiers appels API pour vérifier que les requêtes sont correctement prises en charge par l'application.
- **Stabilité applicative :** surveillance du comportement de l'application après la migration afin de s'assurer de son bon fonctionnement.

**Axes d'amélioration pour les prochaines migrations :**

- Analyser et optimiser les temps d'exécution des jobs Terraform.
- Investiguer les blocages XL Release sur les étapes Check QUAL et Check PROD afin d'éviter les relances répétées.
- Vérifier en amont les droits d'accès Vault, notamment sur les chemins contenant les certificats nécessaires aux applications.
- Prévoir une marge supplémentaire dans l'estimation de la durée des migrations, en intégrant les vérifications post-MEP.

**Bilan :**

Malgré les difficultés rencontrées, la migration a été réalisée avec succès. Les premiers contrôles post-MEP sont rassurants et aucun incident particulier n'a été signalé lors des vérifications du lendemain matin.

Merci encore pour la coordination et le suivi !
