Zum Überprüfen welche Procedures und Functions zur Verfügung stehen, bzw. testen ob APOC oder GDS-Plugins richtig installiert sind. In diesem Fall sollten Procedures mit dem Präfix apoc. oder gds. in der Liste zur Verfügung stehen
```
CALL dbms.procedures();
```
Weitere Hilfe zur einzelnen Procedures können über apoc.help abgerufen werden. In Klammern kann dabei das gewünschte Anwendungsgebiet angegeben werden.
```
CALL apoc.help('text');
CALL apoc.help('relationship');
```
Weitere Hilfestellungen und Dokumentationen können über :help abgerufen werden.

Die Zentralen CYPHER Befehle zum Erzeugen, Bearbeiten und Löschen der Nodes und Relationships sind:
```
CREATE
MERGE
SET
DELETE
MATCH
```
Zum Importieren sind zudem noch die Befehle ``` UNWIND ``` und ``` WITH ``` wichtig zu unterscheiden.

CALL apoc.load.json ("https://raw.githubusercontent.com/aramaki-san/import/main/bahnsen.json") 
YIELD value 
RETURN value;

WITH value.collection.metadata AS metadata 
RETURN metadata;

WITH value.collection.item AS item
RETURN item;

MERGE (c:COLLECTION {title: metadata.title})
MERGE (cat:OBJECT {title: metadata.title, date:metadata.year, digitalRepresentation: metadata.base})
SET cat.type=["Buch", "Katalog"] 
MERGE (p:PERSON {name: metadata.owner, gnd:metadata.ownerGND})
MERGE (i:INSTITUTION {name: metadata.institution})
MERGE (g:GEOGNAME {name: metadata.placeCat})
MERGE (c)-[:isReferencedBy]->(cat)
MERGE (p)-[:bestandsbildner]->(c)
MERGE (i)-[:bestandshaltendeInstitution {signatur: metadata.shelfmark}]->(cat)
MERGE (g)-[:erscheinungsort]->(cat)
RETURN c, cat, p, i, g; 

CALL apoc.load.json ("https://raw.githubusercontent.com/aramaki-san/import/main/bahnsen.json") 
YIELD value 
UNWIND value.collection.metadata AS metadata
UNWIND value.collection.item AS item
UNWIND item.persons.person AS person
UNWIND item.places.place AS place
MERGE (r:RECORD {title:item.titleCat, project_id:item.id})
SET r.titleBib=item.titleBib, r.titleNormalized=item.titleNormalized,r.date=item.year
MERGE (p:PERSON {name: person.persName, gnd: coalesce (person.gnd, "")})
MERGE (g:GEOGNAME {name: place.placeName, getty: coalesce(place.getty, "")})
SET g.lat=place.geoData.lat, g.long=place.geoData.long
MERGE (m:MANIFESTATION {manifestid: coalesce(item.manifestation.idManifestation, "Ausgabe nicht bestimmbar"), system:coalesce(item.manifestation.systemManifestation, "Ausgabe nicht bestimmbar")})
MERGE (g)-[:bezugsort]->(r)
MERGE (m)-[:manifestation]->(r)
WITH person, r, p, g, metadata, item, m
MATCH (cat:OBJECT {title: metadata.title})
MERGE (r)-[:isReferencedBy {page: item.pageCat, number:item.numberCat}]->(cat)
WITH g, r, p, person,m 
CALL apoc.create.relationship (p, person.role, {}, r) YIELD rel
RETURN r, p, g, m LIMIT 100;

CALL apoc.load.json ("https://raw.githubusercontent.com/aramaki-san/import/main/sturm.json") 
YIELD value 
UNWIND value.collection.metadata AS metadata 
MERGE (c:COLLECTION {title: metadata.heading})
MERGE (cat:OBJECT {title: metadata.title, date:metadata.year, digitalRepresentation: metadata.base})
SET cat.type=["Buch", "Katalog"] 
MERGE (p:PERSON {name: metadata.owner, gnd:metadata.ownerGND})
MERGE (i:INSTITUTION {name: metadata.institution})
MERGE (c)-[:isReferencedBy]->(cat)
MERGE (p)-[:bestandsbildner]->(c)
MERGE (i)-[:bestandshaltendeInstitution {signatur: metadata.shelfmark}]->(cat)
RETURN c, cat, p, i; 

CALL apoc.load.json ("https://raw.githubusercontent.com/aramaki-san/import/main/sturm.json") 
YIELD value 
UNWIND value.collection.metadata AS metadata
UNWIND value.collection.item AS item
UNWIND item.persons.person AS person
UNWIND item.places.place AS place
MERGE (r:RECORD {title:coalesce (item.titleCat, "kein Titel?"), project_id:item.id})
SET r.titleBib=item.titleBib, r.titleNormalized=item.titleNormalized,r.date=item.year
MERGE (p:PERSON {name: person.persName, gnd: coalesce (person.gnd, "")})
MERGE (g:GEOGNAME {name: place.placeName, getty: coalesce(place.getty, "")})
SET g.lat=place.geoData.lat, g.long=place.geoData.long
MERGE (m:MANIFESTATION {manifestid: coalesce(item.manifestation.idManifestation, "Ausgabe nicht bestimmbar"), system:coalesce(item.manifestation.systemManifestation, "Ausgabe nicht bestimmbar")})
MERGE (g)-[:bezugsort]->(r)
MERGE (m)-[:manifestation]->(r)
WITH person, r, p, g, metadata, item, m
MATCH (cat:OBJECT {title: metadata.title})
MERGE (r)-[:isReferencedBy {page: item.pageCat, number:item.numberCat}]->(cat)
WITH g, r, p, person,m 
CALL apoc.create.relationship (p, person.role, {}, r) YIELD rel
RETURN r, p, g, m LIMIT 100;

MATCH p=(c1:OBJECT)-[]-(r1:RECORD)-[]-(g:GEOGNAME)-[]-(r2:RECORD)-[]-(c2:OBJECT) WHERE id(c1)=4841 AND id(c2)=4636 RETURN p;

MATCH p=(c1:OBJECT)-[]-(r1:RECORD)-[]-(g:GEOGNAME)-[]-(r2:RECORD)-[]-(c2:OBJECT) WHERE id(c1)=4841 AND id(c2)=4636 UNWIND nodes(p) as n
WITH p, collect(distinct n) as nodes
UNWIND relationships(p) AS r
WITH nodes, collect(distinct r) AS rel
RETURN {nodes:nodes, links:rel};

MATCH path=(p)-[r1:Adressat]->()<-[r2:Verfasser]-(p2) WHERE p.name <> "Kippenberg, Anton" AND p2.name <>  "Kippenberg, Anton" 
RETURN p, type(r1) , count(type(r1)), p2, type(r2), count (type(r2));

MATCH (p)-[r1:Adressat]->()<-[r2:Verfasser]-(p2) WITH DISTINCT p AS pers, p2 CALL apoc.create.vRelationship (pers, "Absender / Adressat", {}, p2) YIELD rel RETURN pers, p2, rel LIMIT 100;

MATCH path=(p)-[r1:Adressat]->()<-[r2:Verfasser]-(p2) WHERE p.name <> "Kippenberg, Anton" AND p2.name <>  "Kippenberg, Anton" 
RETURN p, type(r1) , count(type(r1)), p2, type(r2), count (type(r2));
MATCH (p)-[r1:Adressat]->()<-[r2:Verfasser]-(p2) 
WITH DISTINCT p AS pers, p2 
CALL apoc.create.vRelationship (pers, "Absender / Adressat", {}, p2) YIELD rel MATCH path=(pers)-[rel]-(p2)
UNWIND nodes(path) as n
UNWIND relationships(path) AS r
WITH Collect (distinct n) AS nl, Collect (distinct r) AS rl RETURN {nodes:nl, links:rl};

