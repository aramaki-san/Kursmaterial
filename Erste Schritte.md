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

## Beispiel-Import der mit Libreto erschlossenen Bibliothek Benedikt Bahnsen

Um ein JSON Dokument zu importieren wird apoc.load.json aufgerufen und der Link angegeben. Alternativ kann das Dokument auch in das Import-Verzeichnis von neo4j kopiert werden. Dann müssen jedoch noch die Einstellungen in der config-Datei angepasst werden, um auch aus dem lokalen Verzeichnis importieren zu können. Das Dokument wird dann der Variable value zugeordnet, die zugleich als root-Knoten fungiert. Über ``` RETURN value ``` kann das Dokument angezeigt werden.
```
CALL apoc.load.json ("https://raw.githubusercontent.com/aramaki-san/import/main/bahnsen.json") 
YIELD value 
RETURN value;
```
Von hier aus, kann der Pfad abwärts bis zu dem gewünschten JSON-Element angegeben werden. Hier wird das metadata-Element ausgewählt und in einer gleichnamigen (es können aber auch andere Namen vergeben werden) gespeichert. So können weitere Schritte direkt von dieser Variable aus geschehen. 

```
UNWIND value.collection.metadata AS metadata 
RETURN metadata;
```
Von hier aus können jetzt die unter metadata enthaltenen Elemente in Knoten und Kanten transformiert werden. Alle Schritte gehen jetzt von metadata als neuem Startpunkt aus. 

```
CALL apoc.load.json ("https://raw.githubusercontent.com/aramaki-san/import/main/bahnsen.json") 
YIELD value 
MERGE (c:COLLECTION {title: metadata.title})
MERGE (cat:OBJECT {title: metadata.title, date:metadata.year, digitalRepresentation: metadata.base})
MERGE (p:PERSON {name: metadata.owner, gnd:metadata.ownerGND})
MERGE (i:INSTITUTION {name: metadata.institution})
MERGE (g:GEOGNAME {name: metadata.placeCat})
MERGE (c)-[:isReferencedBy]->(cat)
MERGE (p)-[:bestandsbildner]->(c)
MERGE (i)-[:bestandshaltendeInstitution {signatur: metadata.shelfmark}]->(cat)
MERGE (g)-[:erscheinungsort]->(cat)
RETURN c, cat, p, i, g; 
```

Als nächster Schritt kann alle unter item erschlossenen Katalogeinträge der Bibliotheksrekonstruktion transformiert werden. Wichtig ist, dass in diesem Fall zwingend UNWIND und nicht WITH verwendet wird, da das item Array in einzelne Abschnitte zerlegt werden muss. Zum besseren Verständnis kann einmal das Ergebnis der Befehle 
```
CALL apoc.load.json ("https://raw.githubusercontent.com/aramaki-san/import/main/bahnsen.json") 
YIELD value 
WITH value.collection.item AS item
RETURN item;
```
mit 
```
CALL apoc.load.json ("https://raw.githubusercontent.com/aramaki-san/import/main/bahnsen.json") 
YIELD value 
UNWIND value.collection.item AS item
RETURN item;
```
verglichen werden.

An dieser Stelle nochmal einige Hinweise zu den Besonderheiten des ```Merge``` -Befehls. MERGE kombiniert im Grunde ```MATCH``` und ```CREATE```, d.h. es wird überprüft, ob ein entsprechender Knoten bereits existiert. Wenn nicht, wird ein neuer angelegt, wenn doch wird einfach auf den existierenden Knoten gematcht. Um Fehlermeldungen zu vermeiden, muss daher sichergestellt sein, dass alle angegebenen properties in jedem item vorhanden sind, z.B. jedes item einen titleCat hat und jede Person eine gnd. Wenn nicht dann führt dies zu einem Fehler, denn eine property in einem ```MERGE```-Befehl darf niemals null sein. Eine Möglichkeit damit umzugehen (bspw. bei Personen) ist ```coalesce ()```. In Klammern können verschiedene Variablen oder auch Strings angegeben werden und es wird so lange probiert, bis der erste Wert auftritt, welcher nicht null ist. Der Befehl ```apoc.create.relationship``` dient dem Zweck Relationship-types dynamisch zu vergeben. Zwischen ```MERGE``` und ```MATCH``` sowie ```MERGE``` und ```CALL``` muss in der Regel ```WITH``` kommen. Dabei ist wieder zu beachten, welche Variablen von oben weiter unten nochmals auftreten können oder sollen.

In dem Beispiel-Skript werden Knoten für Katalogeinträge (Record) Personen, Orte und Manifestationen angelegt. Theoretisch sind aber noch weitere Knoten denkbar (z.B. für languages, publishers oder genres). Diese dürfen gerne testweise versucht werden, sie selbst noch in das Skript zu implementieren. Direkt am Anfang werden gleich auch die Pfade für Personen und Orte mit ```UNWIND``` definiert. Dies ist kein Problem (da nicht ```WITH```) und macht das Ganze etwas übersichtlicher.

```
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
RETURN r, p, g, m;
```

Hier noch ein ähnliches Skript für das Libreto-Projekt zur Bibliothek von Johann Christoph Sturm. Die Libreto Projekte unterscheiden sich oft in kleinen Details was wie erschlossen wird, daher kann zwar in der Regel das gleich Skript verwendet werden, einige Anpassungen sind aber meist doch notwendig. 
```
CALL apoc.load.json ("https://raw.githubusercontent.com/aramaki-san/import/main/sturm.json") 
YIELD value 
UNWIND value.collection.metadata AS metadata 
MERGE (c:COLLECTION {title: metadata.heading})
MERGE (cat:OBJECT {title: metadata.title, date:metadata.year, digitalRepresentation: metadata.base})
MERGE (p:PERSON {name: metadata.owner, gnd:metadata.ownerGND})
MERGE (i:INSTITUTION {name: metadata.institution})
MERGE (c)-[:isReferencedBy]->(cat)
MERGE (p)-[:bestandsbildner]->(c)
MERGE (i)-[:bestandshaltendeInstitution {signatur: metadata.shelfmark}]->(cat)
RETURN c, cat, p, i; 
```
```
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
RETURN r, p, g, m;
```
## Abfragen und Datentransformation

Sind die beiden Libreto-JSONs importiert können die Bestände abgefragt werden, z.B. danach ob es Orte gibt, die in beiden Sammlungen / Bibliotheken vorkommen. Die Kette an Knoten und Relationships wird hier in der Variable p gespeichert, welche sozusagen den Pfad als Ergebnis ausliefert, welcher die abgefragte Bedingung erfüllt. Mit id(c1) und id(c2) muss die ID des Knotens, welcher während des metadata-Imports angelegt wurde, noch eingefügt werden. Diese lässt sich einfach (solange nur der Bahnsen- und Sturm-Katalog in der Datenbank sind) schnell über ```MATCH (c:OBJECT) RETURN c;``` abfragen. Dann auf die Knoten klicken und unten sollte die ID angezeigt werden.

```
MATCH p=(c1:OBJECT)-[]-(r1:RECORD)-[]-(g:GEOGNAME)-[]-(r2:RECORD)-[]-(c2:OBJECT) WHERE id(c1)=$ AND id(c2)=$ RETURN p;
```

MATCH p=(c1:OBJECT)-[]-(r1:RECORD)-[]-(g:GEOGNAME)-[]-(r2:RECORD)-[]-(c2:OBJECT) WHERE id(c1)=4841 AND id(c2)=4636 
UNWIND nodes(p) as n
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

