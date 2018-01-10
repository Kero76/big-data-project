# MongoDB - rapport d'installation et de test pour le projet de NoSQL

## Paquet téléchargé
[https://www.mongodb.com/dr/fastdl.mongodb.org/win32/mongodb-win32-x86_64-2008plus-ssl-3.6.1-signed.msi/download]

## Machine de test

PC HP ZBook15 G2, Intel Core I7 @2.7GHz, 8 GB RAM, Windows 7 Entreprise 64-bits.

## Installation

L'installation inclut :
- MongoDB server ;
- MongoDB client (interface en ligne de commandes) ;
- MongoDB Compass (GUI) ;
- des outils complets de monitoring.

## Premier lancement :

En lisant [https://docs.mongodb.com/manual/tutorial/install-mongodb-on-windows/], on crée notre [fichier de configuration][mongod.conf], avec deux directives de base : la spécification du répertoire de stockage de la base de données et celui de log.

## Mise en place d'un service Windows

Pour plus de comodité, il est intéressant de créer un service Windows MongoDB. Il suffit de lancer la commande :
'''
"C:\Program Files\MongoDB\Server\3.6\bin\mongod.exe" --config "C:\Program Files\MongoDB\Server\3.6\mongod.cfg" --install
'''
Ensuite, pour lancer le service :
'''
net start MongoDB
'''
