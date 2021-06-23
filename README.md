# Guide Backend Datatlas
## Introduction
Datatlas est un projet piloté par Erasme dont l'objectif est de mettre en place un logitiel de  Webmapping permettant à cours termes  de cartographier les differentes industies de la métropole de Lyon et à moyen termes de définir la zone de plantabilité en se basant sur les données de la végétalisation.

Dans ce guide nous allons montrer tout d'abord  comment interconnecter notre base de données Notion à celle de Postgis à travers une intégration basé sur [N8N](https://n8n.io/).
Ensuite voir comment publier nos données en utilisant le serveur cartographique geoserveur.
Enfin nous allons montrer comment acceder à nos données en utilisant un client cartographique Openlayers.

---
## Architecture de la Solution
![Architecture](/images/archtecture.png "Architecture Title Text 1")

1. Intégration Notion-Postgre/PosGIS

Pour l’automatisation nous avons utilisé n8n qui  est un outil d'automatisation de flux de travail extensible,open source et disponible  pour l'auto-hébergement .
N8n utilise une approche basée sur les nœuds ce qui le rend très polyvalent, et permet de connecter des outils entre eux.



#### -Prérequis
Créez une base de données notion. Nous  cette [base de données](https://www.notion.so/erasme/a2d9c5c68f1e47ecbbd666edc3eb6047?v=2fb0f62f6d954f5fafe33ff3bd796ca1 )

Remarque: Les noms des champs de la base de données notion doivent correspondre à ceux de postgres (cf étape 3)
Par convention toutes les champs de la BD notion doivent être en minuscule

* Créer une nouvelle intégration notion à relier avec la BD notion 

Vous pouvez suivre la documentaion sur la création d'une [nouvel intégration ici ](https://docs.n8n.io/credentials/notion/#prerequisites)


Remarque: Seuls ceux qui ont un compte administrateur sur notion peuvent le faire.

* Créer une nouvelle base de données Postgres avec les mêmes champs que ceux de notion.

- Workflows

Puisque N8N n'offre pas pour le moment la possibilité de faire une mise à jour de la base de données nous avons configuré le workflows de maniére à vider la base donnée et de faire une nouvelle insertion

Pour la configuration du Workflows nous avions deux possibilité:
* La première consistait à faire une  mise à jour de la base de données à chaque interval de temps pour prendre en compte les nouvelles données insérées(**Pas possible pour le moment puisque N8N n'offre pas l'opération update pour faire une mise à jour de la base de données**)

* La deuxième option et c'est (**celle qui est choisie**) consiste à vider la base de donnée et de faire une nouvelle insertion à chaque intervalle de temps.

Le [![workflows](/images/archtecture.png "Wokflows Title Text 1")](https://n8n.datagora.erasme.org/workflow/3)] est constitué des noeuds suivant:

#### Cron node: ce nœud déclenche le workflow toutes les minutes
#### Notion node (Toutes les lignes): Ce nœud récupère toutes les lignes de la  base de données.
#### Postgres: Ce noeud inséré les données sur la base de données postgres spécifié
#### Set: permet de recréer la base de donnée

- Configuration des noeuds:

## Cron

* Cliquer sur le noeud
* Sur la fenêtre qui s’ouvre cliquer sur parameters
* Ajouter un cron times pour l'exécution du  workflow.
* Nous avons choisi  every minutes qui permet d'exécuter le workflow chaque minute.

![Configuration noeud Cron](/images/cron.png "Cron Title Text 1")

## Notion(Toutes les lignes):

En cliquant sur le noeud vous devez fournir les information suivant:

* Notion API: Vous devez ajouter la clé de l’API ![API KEY](/images/APIKEY.png "API KEY Title Text 1") fournie lors de la création de l’intégration et lui donner un nom.

* Resource:Vous choisissez la ressources à afficher.Ici on affiche la page contenant la BD notion Database Page.
* Opération:Choisissez l’opération à effectuer.Ici on affiche toutes les lignes de la base de données en choisissant l’opération GetAll
* Database ID:On sélectionne  la base de données notion pour l’intégration.

![Configuration noeud Notion](/images/notion.png "Notion Title Text 1")

## Set
Ce noeud permet de recréer la base de dooné supprimée en insérant les nouvelles lignes.
 
 cliquer sur le noeud pour le configurer
 * Name:ID permet de selectioner toutes la base de donnée
 * Value: 0 permet de filtrer les champs à inserer dans la base de données postgres(laisser ce champs à 0 si vous voullez selectionner toutes la bases).

 ![Configuration noeud Set](/images/set.png "Set Title Text 1")

 ## Postgres(Insert):
Avant de configurer ce noeud vous devez d'abord créer une base de données postgis avec les memes champs que ceux sur notion. 
La base de données est déjà déployé sur [Rancher](https://rancher.erasme.org)

### Configuration de la base de donnée Postgres (Insert)

#### ETAPE 1: modification du fichier de configuration
Vous pouvez visitez la [documentation postgres ](https://doc.ubuntu-fr.org/postgresql).

Il faut d'abord modifier le fichier de configuration pour autoriser les connexions via mot de passe chiffré  en remplaçant identsameuser par md5.Ensuite relancer postgres.

```
       nano  /usr/local/pgsql/data/pg_hba.conf
       sudo service postgresql restart
```
Voici la ![configuration](/images/fichierconf.png "conf Title Text 1")

### Etape 2: Créer un Super user(erasme)

```
psql -U postgres
CREATE USER erasme;
ALTER USER erasme SUPERUSER CREATEDB;
\du
```
Rq: \du permet de voir tous les utilisataeurs de la base de données.

### ETAPE 3:créer une base de donné pour utilisateur 
createdb -O <nom_utilisateur> -E UTF8 <ma_database>

```
createdb -O erasme -E UTF8 carte
\l
```

Rq: \l liste  toutes les base de données.

### Etape 4: Se connecter à la base de donné
psql -U <nom_utilisateur> <nom_database_de_l_utilisateur>
```
psql -U erasme carte

```

### Etape4 : créer la table carto
```
CREATE TABLE carto
(id VARCHAR(255), longitude numeric, latitude numeric, expertise VARCHAR(255), gouvernance VARCHAR(255), country VARCHAR(255), public VARCHAR(255), web VARCHAR(255), offre VARCHAR(255), title VARCHAR(255)
    );
```
### Etape 6: Ajouter une colone de géométrie

Postgres fournit 3 fonctions pour créer une géométrie sur les données:

-	ST_MakePoint : construit une géométrie ponctuelle à partir de coordonnées géographiques X (longitude) et Y (latitude);
-	ST_MakeLine : construit une ligne à partir de points;
-	ST_MakePolygon : construit un polygone à partir d’une ligne fermée.


```
  Alter table carto  add column geom Geometry(Point,2154);
  UPDATE carto SET geom=ST_SetSRID(ST_MakePoint(longitude,latitude),2154);

```


### Configuration du noeud

En cliquant sur le noeud vous devez fournir les information suivants

* Postgres: Donner les ![informations de connexion sur la base de données](/images/infoconnexion.png "connexion Title Text 1")
* Operation: Sélectionner l’opération à effectuer(Insert)
* Schema: Choisissez le schéma de la base(Public)
* Table:le nom de la table(carto)
* Columns:le nom du colonnes(lister les colones à afficher séparée par des virgules)
```
id,longitude,latitude,expertise,gouvernance,country,public,web,offre,title
```
* Return Fields:Choisir les champs de retour( l’option * affiche tous les champs)

![Configuration noeud Postgres (Insert)](/images/postgresinsert.png "insertion Title Text 1")

### Configuration de la base de donnée Postgres1 (Execute Query)

* Postgres: Donner le nom de la base de donnée.
* Operation: Sélectionner l’opération à effectuer(Execute Query)
  ```
      DELETE FROM carto *;

   ```


![Configuration noeud Postgres (Insert)](/images/postgres1.png "insertion Title Text 1")

---

2. Publication des données sur géoserver

Dans cette partie nous allons d'abord créer un espace de travail puis un entrepot de donnée et enfin une couche de donnée sur le [geoserver](http://geoserver.ud-reproducibility.datagora.erasme.org/geoserver/web/wicket/bookmarkable/org.geoserver.web.GeoServerLoginPage)  déjà déploiyé sur Rancher.

Géoserver permet de manipuler differents types de données comme le montre ![ce schema](/images/typesdonnéesgéoserver)

### Etape1: Création d'un espace de travail
Pour créer un espace de travail vous devez cliquer sur espace de travail pour ajouter un nouveau en  donnant:

* Son nom: erasme
* l'URIde l'espace de nommage :http://geoserver.ud-reproducibility.datagora.erasme.org/geoserver/web/ 

comme le montre la figure [espace-travail](/images/espacetravail)

### Etape2: Création d'un entrepot de donnée
Pour créer un entrepot de donnée vous devez:
 #### taper sur l'onglet entrepot  
 #### Ensuite sur ajouter un nouveau entrepot
 #### Ensuite PostGIS - PostGIS Database et remplir les informations suivantes:

* nom de l'espace de travail: erasme
* nom de la base de donnée: carto
* informations de connection de la base de donnée
 - host:bdd-pgsql.test.svc.cluster.local
    Remarque: la base de donné psql(Namespace:test) et le geoserver(Namespaces:ud-reproducibility) ne sont pas sur le namespace.
    Le host serait uniquement bdd-pgsql si c'était dans le contraire.
 - Port:5432
 - database:carte
 - shema:public
 - user:datatlas
 - passwd: ...

comme le montre la ![figure](/images/entrepot)suivante:

### Etape3 : Création d'une couche de données

- Pour créer une couche vous devez :

* cliquer sur l'onglet couches 
* Ensuite selectionner ajouter une nouvelle ressouches.
* Enfin choisissez votre entrepot de donnée

Une fenetre s'ouvre et vous montre toutes les  des bases données présentes.Vous pouvez maintenant choisir de publier une base de donnée  sur la ![listes](/images/listesbd).

- Publication de couches

Pour publier une couches de donnée vous deves
* Donnez les information sur la ![base de donnée(nom,Titre)](/images/infodonné)
* Donnez le système de [cordonnée(Lambert 93 pour la france)](/images/systéme de coordonné)
* Donnez l'![emprise de la donnée](/images/emprise) en choississant basées sur les données et calculées sur les emprises natives respectivement puis valider.

3. Client cartographique Openlayers

Le client carthographique se connecte en sur le geoserver pour télécharger les couches de donées en utilisant le protocole WMS/WFS

* connection WFS
```javascript
   var indusUrl = 'http://geoserver.ud-reproducibility.datagora.erasme.org/geoserver/erasme/ows?service=WFS&' +
    'version=1.0.0&request=GetFeature&typeName=erasme%3Acarto&' +
    'maxFeatures=50&outputFormat=application%2Fjson';

```

Connection WMS
  
   ```javascript
     		var wmsSource5 =new ol.source.ImageWMS({
			url:'http://geoserver.ud-reproducibility.datagora.erasme.org/geoserver/ows?',
			params: {'LAYERS':'erasme:carto'},
			serverType:'geoserver',
			crossOrigin: 'anonymous'
			});
  ```
