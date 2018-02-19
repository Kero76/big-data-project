# MongoDB - rapport d'installation et de test pour le projet de NoSQL

Arthur BREUNEVAL, Grégoire POMMIER, Nicolas GILLE

## Paquet téléchargé
[mongodb-win32-x86_64-2008plus-ssl-3.6.1-signed.msi](https://www.mongodb.com/dr/fastdl.mongodb.org/win32/mongodb-win32-x86_64-2008plus-ssl-3.6.1-signed.msi/download)

## Machine de test

PC HP ZBook15 G2, Intel Core I7 @2.7GHz, 8 GB RAM, Windows 7 Entreprise 64-bits.

## Installation

L'installation inclut :
- MongoDB server ;
- MongoDB client (interface en ligne de commandes) ;
- MongoDB Compass (GUI) ;
- des outils complets de monitoring.

## Premier lancement :

En lisant [installer MongoDB sous Windows](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-windows/), on crée notre [fichier de configuration](./mongod.conf), avec deux directives de base : la spécification du répertoire de stockage de la base de données et celui de log.

## Mise en place d'un service Windows

Pour plus de comodité, il est intéressant de créer un service Windows MongoDB. Il suffit de lancer la commande :
`"C:\Program Files\MongoDB\Server\3.6\bin\mongod.exe" --config "C:\Program Files\MongoDB\Server\3.6\mongod.cfg" --install`
Ensuite, pour lancer le service :
`net start MongoDB`

## Insertion de données

### Dans le Shell Mongo

- L'insertion de plusieurs documents se fait avec la commande `db.<collection>.insertMany()`, où `<collection>` est le nom de la collection dans laquelle insérer les documents.
`insertMany()` prend en paramètre au minimum un tableau de documents, entre `[]`. 
Si la collection n'existe pas, elle est créée si l'opération réussit.
`insertMany()` insère par défaut les documents dans l'ordre spécifié, sauf si "ordered" (3ème paramètre, optionnel) est positionné sur false. Dans ce cas, le serveur Mongo s'autorise à insérer les documents de façon concurrente si cela augmente les performances.
La documentation explique clairement qu'il est bon de positionner "ordered" sur false tant que faire se peut, car les applications ne devraient de toute façon pas dépendre de l'ordre d'insertion pour leur fonctionnement.
Le 2ème paramètre, "Write Concern", également optionnel, est un document décrivant les acquittements requis pour l'opération :
  - w : Le nombre de noeuds devant acquitter l'écriture du document. Très utile pour savoir si la donnée a bien été répliquée.
  - j : Demande un acquittement pour l'écriture de l'opération dans le journal du serveur Mongo : dans la RAM (false) ; sur le disque (true).
  - wtimeout : limite de temps, en millisecondes, dans laquelle l'opération doit être effectuée. Utile pour éviter les blocages.

Lorsque chaque document est inséré, le serveur Mongo lui associe un ObjectId (identifiant unique), si celui-ci n'est pas spécifié dans la requête (champ _id que tout document possède). L'ObjectId est un hexadécimal sur 12 octets, basé sur :
- 4 octets pour le nombre de secondes écoulées depuis l'Unix Epoch ;
- 3 octets d'identifiant unique pour la machine physique qui procède à l'insertion ;
- 2 octets d'identifiant de processus ;
- 3 octets pour un compteur dont le départ est initialisé avec un nombre aléatoire.

- L'insertion d'un seul document se fait avec la commande `db.<collection>.insertOne()`. Son comportement est assez similaire à `insertMany()`, excepté qu'elle prend un seul document en paramètre, et qu'on ne peut spécifier si l'insertion doit être ordonnée ou non, ce qui est évidemment tout-à-fait logique...

`insertMany()` et `insertOne()` renvoient un document contenant :
Si l'opération s'est déroulée avec succès :
- acknowledged : toujours à true quand ce document est retournée, confirme que l'opération s'est déroulée dans les conditions spécifiées par writeConcern.
- insertedId (insertOne()) / insertedIds (insertMany()) : précise les identifiants du(des) nouveau(x) document(s) inséré(s).

Si une erreur s'est produite :
Pour `insertOne()` :
- writeError, document contenant les erreurs d'écriture et les messages d'erreurs associés.
- writeConcernError si l'opération n'a pu satisfaire les conditions précisées par writeConcern.

Pour `insertMany()` :
- Un document contenant writeErrors et writeConcernErrors.
- À noter que, mise à part les writeConcernError, toutes les erreurs induisent l'arrêt des opérations d'insertion restantes, si "ordered" est à true. Si "ordered" est à false, aucune erreur ne stoppe la suite des opérations.


### Insertion via mongoimport

La commande mongoimport permet d'insérer des données directement en ligne de commandes depuis le Shell du système plutôt que le Shell mongo. Très utile quand on a des fichiers tout prêts.
Nous avons testé avec des restaurants :
`mongoimport --db test --collection restaurants --drop --file ./Bureau/restaurants.json`
> connected to: 127.0.0.1
> Fri Jan 12 15:41:30.935 dropping: test.restaurants
> Fri Jan 12 15:41:31.548 check 9 25359
> Fri Jan 12 15:41:31.599 imported 25359 objects

L'option --collection permet de préciser la collection dans laquelle importer les données qui vont suivre avec l'option --file.


## Mise à jour

On utilise la commande `db.<collection>.update(filter, newDoc, updateParams)`. :
- filter : le motif de recherche pour trouver le(s) document(s) à mettre à jour.
- newDoc : si c'est un document complet, remplace l'existant par newDoc ; sinon, remplace les attributs du document existants par ceux du nouveau.
- updateParams : document acceptant les paramètres de mise à jour : upsert (bool, optionnel, défaut à false) pour spécifier si newDoc doit être inséré si filter ne filtre aucun document ; multi (booléen, optionnel, défaut à false) précise si un ou plusieurs documents doivent être mis à jour si filter filtre plusieurs documents ; writeConcern (document, optionnel) voir l'insertion pour plus d'informations ; collation (document, optionnel) spécifie des conditions sur la capitalisation des lettres dans les chaînes de caractère, la locale, etc. ; arrayFilters (tableau, optionnel) spécifie des filtres pour la mise à jour des tableaux.

Exemple :
```
db.restaurants.update(
    { "name" : "Juni" },
    {
     $set: { "cuisine": "American (New)" },
     $currentDate: { "lastModified": true }
    }
)
```
Cette commande met à jour un document dont le nom du restaurant est "Juni", et met à jour le paramètre "cuisine" avec la valeur "American (new)". 


## Suppression

Il n'y a pas grand-chose à dire sur la suppression. La commande `db.<collection>.delete()` prend obligatoirement en paramètre le filtre spécifiant le(s) document(s) à supprimer, et éventuellement un document en 2ème paramètre, avec notamment le champ justOne qui, s'il est à true, indique que seul le premier document trouvé par le filtre est supprimé, sinon tous les documents qui passent le filtre le sont.

Exemple :
`db.restaurants.remove( { "borough": "Manhattan" } )`
`db.restaurants.remove( { "borough": "Queens" }, { justOne: true } )`


## Recherche

La recherche est très simple : il suffit d'utiliser la méthode `db.<collection>.find()` et de passer en paramètre un morceau de document JSON pour filtrer les résultats.
Par exemple, la commande :
`db.restaurants.find({ "address.street": "Madison Avenue" })`
Renvoie tous les restaurants dont la rue est "Madison Avenue".


## Aggrégation

La commande `db.<collection>.aggregate()` prend en paramètre un tableau conditions pouvant combiner filtres et opérations typiques (min, max, égalité, ...), et éventuellement un document décrivant les options. On peut aussi spécifier une liste ee documents pour les conditions d'aggrégation, mais dans ce cas on ne peut pas passer d'options.
Par exemple :
```
db.restaurants.aggregate(
  [
    { $match: { "borough": "Manhattan"} },
    { $group: { "_id": "$borough", "count": { $sum: 1 } } }
  ]
);
```
prend tous les restaurants dont le quartier est Manhattan, les groupe par identifiant et finalement, limite le résultat à 1 (renvoie le restaurant qui a le plus petit id, donc).


# Avec les corpus de textes de PubMed

Maintenant que nous avons vu quelques exemples de requêtages de MongoDB, langage assez intuitif par ailleurs, nous allons effectuer quelques tests de performances et de comportements sur des grosses données. Pour cela, nous utiliserons une grande quantité de données récupérées depuis PubMed, la base de nonnées mondiale des articles scientifiques autour du domaine médical.


## Insertion de 880033 articles de pubmed au format JSON.

Le dossier courant contient tous les fichiers JSON. On exécute :
```
for f in *;do
  mongoimport --db test --collection pubmed --file $f
done
```
Car en effet, mongoimport ne permet malheureusement pas d'importer une liste de fichiers, seulement des fichiers un par un...
Les données représentent au total 7.601 Go.
Nous n'avons réussit qu'à insérer à peu près 85% des données. En effet, après 51h d'insertion, la fenêtre du terminal a mystérieusement disparu. Étant donné le temps démentiel que cela a pris, nous n'avons pas souhaité réitérer l'opération. Mais au moins, nous nous sommes rendu compte que les PC classiques ne sont pas **du tout** adaptés à ce genre d'opération.
Pour la même raison, nous sommes un peu frileux quant à exécuter d'autres requêtes (type recherche / aggrégation) sur les données insérées. C'est pourquoi nous nous sommes plutôt penchés sur le système de réplication afin d'essayer de comprendre plus profondément le fonctionnement de MongoDB, au lieu d'exécuter des requêtes interminables.


# Réplication

Un ensemble de réplication est un ensemble de noeuds maintenant une version du même ensemble de données.

Dans MongoDB, le système de réplication s'apparente au système de load-balancing. En effet, un noeud, dit noeud primaire, reçoit les requêtes clientes. Ce noeud distribue les opérations sur les noeuds dits secondaires pour distribuer la charge de travail. Si plusieurs opérations concurrentes ont lieu, il y a un système d'arbitre. Cet arbitre est toujours le même est il élit un quorum afin de décider quelle version de la donnée est la bonne. Dans le cas où la requête cliente a un writeConcernavec un w à "majority", chaque membre du quorum vote pour la version de la donnée dont il dispose, et la majorité remporte le vote pour définir la nouvelle version de la donnée. Si w va-t 1, c'est le vote du noeud primaire qui prime, la décision du chef...

Un arbitre n'a pas besoin d'un matériel spécifique, puisqu'il ne fait qu'appliquer un algorithme tous les X temps pour déterminer un nouveau quorum. Ainsi, il est très facile d'ajouter un arbitre au système de réplication.

En revanche, un noeud primaire peut perdre son pouvoir et passer à un noeud secondaire, et vice-versa. Ainsi, si un noeud primaire est surchargé, un noeud secondaire peut être élu pour encaisser la charge, alors qu'il était noeud secondaire à la base, puis retourner à son rôle de noeud secondaire ensuite.

Pour ce qui est des opérations asynchrones, les noeuds primaires distribuent simplement les opérations aux noeuds secondaires.

Lorsque le noeud primaire ne répond plus, un noeud secondaire déclenche automatiquement l'élection d'un nouveau noeud primaire. La détection du fait que le noeud primaire ne répond plus peut aller de 10 à 30 semondes, et l'élection elle-même peut également prendre de 10 à 30 secondes.
Depuis la version 3.2 de MongoDB, ce temps est réduit.

Par défaut, les opérations de lecture s'exécutent sur le noeud primaire, mais les clients peuvent spécifier un noeud secondaire. Cependant, il y a un risque que les données lues ne reflètent pas les données du noeud primaire.
