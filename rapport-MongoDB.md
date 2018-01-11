# MongoDB - rapport d'installation et de test pour le projet de NoSQL

## Paquet téléchargé
https://www.mongodb.com/dr/fastdl.mongodb.org/win32/mongodb-win32-x86_64-2008plus-ssl-3.6.1-signed.msi/download

## Machine de test

PC HP ZBook15 G2, Intel Core I7 @2.7GHz, 8 GB RAM, Windows 7 Entreprise 64-bits.

## Installation

L'installation inclut :
- MongoDB server ;
- MongoDB client (interface en ligne de commandes) ;
- MongoDB Compass (GUI) ;
- des outils complets de monitoring.

## Premier lancement :

En lisant https://docs.mongodb.com/manual/tutorial/install-mongodb-on-windows/, on crée notre fichier de configuration mongod.conf, avec deux directives de base : la spécification du répertoire de stockage de la base de données et celui de log.

## Mise en place d'un service Windows

Pour plus de comodité, il est intéressant de créer un service Windows MongoDB. Il suffit de lancer la commande :
"C:\Program Files\MongoDB\Server\3.6\bin\mongod.exe" --config "C:\Program Files\MongoDB\Server\3.6\mongod.cfg" --install
Ensuite, pour lancer le service :
net start MongoDB

## Insertion de données

- L'insertion de plusieurs documents se fait avec la commande db.<collection>.insertMany(), où <collection> est le nom de la collection dans laquelle insérer les documents.
insertMany() prend en paramètre au minimum un tableau de documents, entre []. 
Si la collection n'existe pas, elle est créée si l'opération réussit.
insertMany() insère par défaut les documents dans l'ordre spécifié, sauf si "ordered" (3ème paramètre, optionnel) est positionné sur false. Dans ce cas, le serveur Mongo s'autorise à insérer les documents de façon concurrente si cela augmente les performances.
La documentation explique clairement qu'il est bon de positionner ordered sur false tant que faire se peut, car les applications ne devraient de toute façon pas dépendre de l'ordre d'insertion pour leur fonctionnement.
Le 2ème paramètre, Write Concern, également optionnel, est un document décrivant les acquittements requis pour l'opération :
- w : Le nombre de noeuds devant acquitter l'écriture du document. Très utile pour savoir si la donnée a bien été répliquée.
- j : Demande un acquittement pour l'écriture de l'opération dans le journal du serveur Mongo : dans la RAM (false) ; sur le disque (true).
- wtimeout : limite de temps, en millisecondes, dans laquelle l'opération doit être effectuée. Utile pour éviter les blocages.

Lorsque chaque document est inséré, le serveur Mongo lui associe un ObjectId (identifiant unique), si celui-ci n'est pas spécifié dans la requête (champ _id que tout document possède). L'ObjectId est un hexadécimal sur 12 octets, basé sur :
- 4 octets pour le nombre de secondes écoulées depuis l'Unix Epoch ;
- 3 octets d'identifiant unique pour la machine physique qui procède à l'insertion ;
- 2 octets d'identifiant de processus ;
- 3 octets pour un compteur dont le départ est initialisé avec un nombre aléatoire.

- L'insertion d'un seul document se fait avec la commande db.<collection>.insertOne(). Son comportement est assez similaire à insertMany(), excepté qu'elle prend un seul document en paramètre, et qu'on ne peut spécifier si l'insertion doit être ordonnée ou non, ce qui est évidemment tout-à-fait logique...

insertMany() et insertOne() renvoient un document contenant :
Si l'opération s'est déroulée avec succès :
- acknowledged : toujours à true quand ce document est retournée, confirme que l'opération s'est déroulée dans les conditions spécifiées par writeConcern.
- insertedId (insertOne()) / insertedIds (insertMany()) : précise les identifiants du(des) nouveau(x) document(s) inséré(s).

Si une erreur s'est produite :
Pour insertOne :
- writeError, document contenant les erreurs d'écriture et les messages d'erreurs associés.
- writeConcernError si l'opération n'a pu satisfaire les conditions précisées par writeConcern.

Pour insertMany() :
- Un document contenant writeErrors et writeConcernErrors.
- À noter que, mise à part les writeConcernError, toutes les erreurs induisent l'arrêt des opérations d'insertion restantes, si "ordered" est à true. Si "ordered" est à false, aucune erreur ne stoppe la suite des opérations.