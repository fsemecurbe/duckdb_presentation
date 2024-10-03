# duckdb_presentation

L'objectif de l'atelier est de présenter DuckDB et son utilisation en fonction de nos besoins.
Dans  un premier temps, il n'est pas nécessaire d'installer ou d'ouvrir une application. Il vous suffit de vous rendre sur le site duckdb.org et d'ouvrir la démonstration en direct (live-demo) ou d'accéder directement à https://shell.duckdb.org/.

La base de données que nous allons utiliser se trouve à cette adresse : https://www.data.gouv.fr/fr/datasets/revenus-pauvrete-et-niveau-de-vie-en-2019-donnees-carroyees-dispositif-fichier-localise-social-et-fiscal-filosofi/. Elle est directement accessible via le lien suivant : https://www.data.gouv.fr/fr/datasets/r/b480cead-3f46-4b1b-a943-62a009b83f7a. Ce sont les données carroyées de l'INSEE pour 2019.

DuckDB est un outil de gestion de bases de données contenu dans un seul fichier. Il permet de remplacer, dans certaines situations, PostgreSQL, PostGIS et SQLite, mais aussi R et Python

L'adresse du fichier à requêter n'est pads commode, on va commencer par créer une vue (un pointeur vers notre fichier).
```
CREATE OR REPLACE VIEW filosofi AS from  read_parquet('https://www.data.gouv.fr/fr/datasets/r/b480cead-3f46-4b1b-a943-62a009b83f7a');
```
Pour connaitre le contenu de la table :
```
SUMMARIZE filosofi;
```

Pour obtenir le nombre de lignes dans notre table : 

```
select count(*) 
from filosofi;
```

Pour obtenir la population dans la commune d'Auzeville (code insee 31035) :

```
select lcog_geo,  sum(ind) as ind 
from filosofi
where lcog_geo = '31035'
group by lcog_geo;
```

Il manque de la population car lcog_geo n'est pas le code commune mais la concaténation des codes communes des communes intersectants le carreau. Pour obtenir un majorant :
```
select substr(lcog_geo,1,5),  sum(ind) as ind 
from filosofi
where substr(lcog_geo,1,5) = '31035'
group by substr(lcog_geo,1,5);
```

A présent, nous allons calculer la population communale de toutes les communes :
```
select substr(lcog_geo,1,5),  sum(ind) as ind 
from filosofi
group by substr(lcog_geo,1,5);
```
Que constatez vous ? Comment l'expliquez vous ? 

Maintenant à vous de calculer la population par département :

```
select substr(lcog_geo,1,2) as dep,  sum(ind) as ind 
from filosofi
group by substr(lcog_geo,1,2);
```

Maintenant, on va faire un peu de géographie, on commence par charger le module spatial :

```
install spatial;
load spatial;
```

On va essayer de calculer la population à moins d'un kilomètre de la Tour Eiffel. On commence par créer le point de la Tour Eiffel puis le disque d'un kilomètre. 
Attention le système de projection des carreaux est le 3035 LAEA europe. La commande pour le point :
```
select ST_Point (3756295, 2889313);
```
La commande pour le disque :
```
select st_buffer(ST_Point (3756295, 2889313),1000);
```
Maintenant, on va faire l'intersection spatiale en créant une variable **hubblo**:
```
set variable hubblo = st_buffer(ST_Point (3756295, 2889313),1000);

select *
from filosofi
where ST_Intersects(filosofi.geometry, getvariable('hubblo'));
```
On préocéde à l'agrégation des valeurs des carreaux :
```
select sum(ind) as ind
from filosofi
where ST_Intersects(filosofi.geometry, getvariable('hubblo'));
```
Cela prend 5 secondes, on est d'accord, ce n'est pas énorme, mais peut on faire mieux en créant une table et en utilisant une indexation spatiale :
```
create or replace table carreaux as select *  from filosofi;
create index sindex on carreaux USING RTREE(geometry);
```

Bon, c'est diabolique :
```
select sum(ind) as ind
from carreaux
where ST_Intersects(carreaux.geometry, getvariable('hubblo'));
```

Un peu plus compliqué, on fait une interpolation spatiale par la surface :

```
select 'hubblo' as  unit, sum(ind*weight) as ind, 
from 
(SELECT *, st_area(st_intersection(carreaux.geometry, getvariable('hubblo'))) / st_area(carreaux.geometry) as weight 
from carreaux
where ST_Intersects(carreaux.geometry, getvariable('hubblo')) ;
```
Avec l'ensemble des colonnes :

```
select 'hubblo' as  unit, sum(ind*weight) as ind, sum(men*weight) as men,
       sum(men_pauv*weight) as men_pauv, sum(men_1ind*weight) as men1_ind,
       sum(men_5ind*weight) as men_5ind , sum(men_prop*weight) as men_prop,
       sum(men_fmp*weight) as men_fmp, sum(ind_snv*weight) as ind_snv, 
       sum(men_surf*weight) as men_surf, sum(men_coll*weight) as men_coll,
       sum(men_mais*weight) as men_mais, sum(log_av45*weight) as log_av45,
       sum(log_70_90*weight) as log_70_90, sum(log_ap90*weight) as log_ap90,
       sum(log_inc*weight) as log_inc, sum(log_soc*weight) as log_soc,
       sum(ind_0_3*weight) as ind_0_3, sum(ind_4_5*weight) as ind_4_5,
       sum(ind_6_10*weight) as ind_6_10, sum(ind_11_17*weight) as ind_11_17,
       sum(ind_18_24*weight) as ind_18_24, sum(ind_25_39*weight) as ind_25_39,
       sum(ind_40_54*weight) as ind_40_54, sum(ind_55_64*weight) as ind_55_64,
       sum(ind_65_79*weight) as ind_65_79, sum(ind_80p*weight) as ind_80p ,
       sum(ind_inc*weight) as ind_inc
from 
(SELECT *, st_area(st_intersection(carreaux.geometry, getvariable('hubblo')) / st_area(carreaux.geometry) as weight 
from carreaux
where ST_Intersects(carreaux.geometry, getvariable('hubblo')) ;
```









