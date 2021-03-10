# PAM-Archiver

![Build](https://img.shields.io/badge/Build_with-Java8-orange.svg?style=for-the-badge&logo=java)
![EAP](https://img.shields.io/badge/-Jboss_EAP_7.3-orange.svg?style=for-the-badge)
![RHPAM](https://img.shields.io/badge/-RHPAM_7.8-orange.svg?style=for-the-badge)
![Fuse](https://img.shields.io/badge/-Fuse_7.8-orange.svg?style=for-the-badge)
![Postgres](https://img.shields.io/badge/-PostgreSQL-orange.svg?style=for-the-badge)
![License](https://img.shields.io/badge/License-Apache-green.svg?style=for-the-badge&logo=apache)

Ce dépôt est basé sur le projet suivant:  https://github.com/mgubaidullin/pam-archiver

Prototype RHPAM pour une solution d'archivage basée sur Red Hat Fuse 


## Architecture
![Architecture](architecture.png)

### Components
1. Main PAM - Process Automation Manager pour les instances de processus actives 
2. AMQ -  Plateforme de messagerie
3. Fuse - Plateforme d'intégration
4. Main Database - Base de données pour les instances de processus actives
5. Archive PAM - Process Automation Manager pour les instances de processus archivées
6. Archive Database - Base de données pour les instances de processus archivées


### Archiver process
![Process](process.png)

1. Démarrage d'une instance du processus 'Archiver' avec le nombre de jours minimum depuis la date de fin du processus ( 0 pour archiver tous les processus)
2. Le processus utilise la tâche de service JMSSend pour envoyer un message à l'utilitaire d'archivage Fuse
3. Le processus attend un signal de l'utilitaire d'archivage pour continuer. 
4. Lors de la réception du signal, la tâchee Verify Result est créée avec les résultats d'archivage.

### Archive routes
Les routes sont définies dans cette classe: [PamArchiver](src/main/java/one/entropy/archiver/PamArchiver.java) 

1. Reception du message d'archivage
2. Effectuer l'archivage
    - Sélection de tous les processus terminés plus anciens que la valeur de days_limit définie par l'utilisateur
    - Copie des processus sélectionnés vers la base de données de destination (avec l'approche UPSERT )
    - Suppression des processus sélectionnés de la base de données principale
    - Préparation des informations à retourner vers l'utilisateur
3. Archivage de la table NodeInstanceLog
    - Sélection de tous les noeuds correspondants aux processus sélectionnés
    - Copie des noeuds sélectionnés vers la base de données de destination (avec l'approche UPSERT )
    - Suppression des processus sélectionnés de la base de données principale
    - Préparation des informations à retourner vers l'utilisateur. 
4. Archivage de la table VariableInstanceLog
    - Sélection de toutes les variables correspondantes aux processus sélectionnés
    - Copie des variables sélectionnées vers la base de données de destination (avec l'approche UPSERT )
    - Suppression des variables sélectionnées de la base de données principale
    - Préparation des informations à retourner vers l'utilisateur. 
5. Aggregation et préparation des résultats
6. Envoie d'un signal au processus avec avec les résultats d'archivage en appelant l'API du Kie Server avec le composant Camel kie-server.

## Configuration additionnelle
1. Un usager et mot de passe avec les accès au Kie-Server est requis. Voir fichier archiver.properties
2. La tâche est présentement assignée à l'usager rhpamAdmin, modifier le processus au besoin.

## Préparation du démo
Pré-requis: Java 8 ou 11 et Maven 3.6+ 

1. Installation de Jboss EAP 7.3
2. Configuration de Jboss EAP 7.3
   - Datasource principale avec le nom JNDI: java:/MainDS  (ou modifier la classe [Datasources.java](/src/main/java/one/entropy/archiver/Datasources.java)
   - Datasource de la BD archive avec le nom JNDI: java:/ArchiveDS
   - Queue JMS Queue: archive-request avec le nom JNDI: java:/jms/queue/ArchiveRequest
3. Installation de RHPAM sur Jboss EAP
4. Installation de Fuse 7.8 sur Jboss EAP: [Fuse EAP](https://access.redhat.com/documentation/en-us/red_hat_fuse/7.8/html-single/installing_on_jboss_eap/index#installing-fuse-on-jboss-eap)
5. Démarrer Jboss EAP
   - ./bin/standalone.sh -c standalone-full.xml
6. Compiler et déployer PamArchiver
   -  mvn clean install -Pdeploy ou
   -  mvn compile package et copier target/pam-archiver.war dans le répertoire standalone/deployments/ de JBoss EAP
7. Importer le processus [ArchiverProcess](https://github.com/masauve/archiver-process) vers RHPAM
8. Dans le nouveau project RHPAM, configure la tâche de service JMSSendTask avec les paramêtres suivants:
   a. java:jboss/DefaultJMSConnectionFactory
   b. java:/jms/queue/ArchiveRequest
8. Compiler et déployer le processus 
9. Démarrer une instance au besoin.