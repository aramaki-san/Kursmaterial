# Skript zur Übung / Einführung in neo4j (26. Mai 2021)

Zum Überprüfen welche Procedures und Functions zur Verfügung stehen, bzw. testen ob APOC oder GDS-Plugins richtig installiert sind. In diesem Fall sollten Procedures mit dem Präfix apoc. oder gds. in der Liste zur Verfügung stehen
```
CALL dbms.procedures();
```
Weitere Hilfe zur einzelnen Procedures können über apoc.help abgerufen werden. In Klammern kann dabei das gewünschte Anwendungsgebiet angegeben werden.
```
CALL apoc.help('text');
CALL apoc.help('relationship');
```
Weitere Hilfestellungen und Dokumentationen können über ```:help``` abgerufen werden.

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
UNWIND value.collection.metadata AS metadata 
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

Für einen Export und/oder zur Visualisierung wäre es aber vielleicht schön ein strukturiertes JSON-Dokument nach dem Schema 

``` 
{"nodes": 
    [hier kommen alle Knoten rein], 
 "links":
    [und hier alle relationships mit Start- und Endpunkt]
}
``` 
zu haben. 

Hierfür können wir mit ```UNWIND``` und ```WITH``` Knoten des Pfades p sozusagen "einsammeln". Mit ```collect(distinct n)``` wird sichergestellt, dass alle Knoten und Relationships nur einmal mit aufgeführt werden.

```
MATCH p=(c1:OBJECT)-[]-(r1:RECORD)-[]-(g:GEOGNAME)-[]-(r2:RECORD)-[]-(c2:OBJECT) WHERE id(c1)=$ AND id(c2)=$ 
UNWIND nodes(p) as n
WITH p, collect(distinct n) as nodes
UNWIND relationships(p) AS r
WITH nodes, collect(distinct r) AS rel
RETURN {nodes:nodes, links:rel};
```

## Beispiel Inselverlagsarchiv Marbach - Kippenbergarchiv

Im Repositorium findet sich auch ein "längeres" Skript zum Import der EAD-Datei über das Inselverlagsarchiv-Kippenberg aus dem DLA Marbach. EAD ist ein Export-Format, welches vor allem in Archiven Anwendung findet und sich u.a. durch eine extrem verschachtelte Struktur auszeichnet. Das Vorgehen zum Import richtet sich im Prinzip nach den gleichen Richtlinien wie der Libreto-Import und wurde wo nötig an die Anforderungen von EAD angepasst. Zur leichteren Fehlersuche wurde der Import in viele kleine Einzelschritte heruntergebrochen.

Nach dem Import können folgende Abfragen einmal testweise ausprobiert werden.
```
MATCH path=(p)-[r1:Adressat]->()<-[r2:Verfasser]-(p2)
RETURN p, type(r1) , count(type(r1)), p2, type(r2), count (type(r2));
```
Sucht nach Personen die in einem Adressat-Verfasser-Verhältnis über einen Zwischenknoten miteinander verbunden sind. () in der Mitte ist die Möglichkeit einfach irgendeinen Knoten unabhängig von Label oder Properties für die Abfrage zu definieren. Mit ```type(r)``` wird das Label der Relationship ausgegeben ```count(type(r))``` wird gezählt wie oft die angegebenen Personen miteinander verbunden sind (mehrere Verbindungen sind möglich, da beide Personen in verschiedenen im Bestand erschlossenen Briefkonvoluten auftreten können)


```
MATCH (p)-[r1:Adressat]->()<-[r2:Verfasser]-(p2) 
WITH DISTINCT p AS pers, p2 
CALL apoc.create.vRelationship (pers, "Absender / Adressat", {}, p2) YIELD rel 
RETURN pers, p2, rel LIMIT 100;
```
Ausgehen von der gleichen Abfrage wird mit ```apoc.creat.vRelationship``` eine virtuelle Relation zwischen den Personen angelegt mit dem Label "Absender / Adressat". Diese Relation existiert nur als Ergebnis dieser einen Abfrage und wird nicht permanent in der Datenbank gespeichert. So lässt sich Zwischenschritt über einen Knoten vermeiden und man erhält eine direkte Verbindung zwischen den Personen erzielt. ```WITH DISTINCT``` sorgt dafür, dass Duplikate ausgesondert werden, jede Person also genau einmal mit einer anderen Person verbunden sein sollte.
Virtuelle Relationen sowie virtuelle Knoten haben immer eine negative ID. Aktuell werden für solche Relationen und Knoten noch nicht alle Möglichkeiten von apoc und dem GDS-Plugin zur Netzwerkanalyse unterstützt, das heißt man kann sie leider noch nur begrenzt weiterverarbeiten. Es ist aber sehr wahrscheinlich, dass spätere Versionen von neo4j hier noch neue Funktionen anbieten werden.

```
MATCH (p)-[r1:Adressat]->()<-[r2:Verfasser]-(p2) 
WITH DISTINCT p AS pers, p2 
CALL apoc.create.vRelationship (pers, "Absender / Adressat", {}, p2) YIELD rel 
MATCH path=(pers)-[rel]-(p2)
UNWIND nodes(path) as n
UNWIND relationships(path) AS r
WITH Collect (distinct n) AS nl, Collect (distinct r) AS rl RETURN {nodes:nl, links:rl};
```
Hier wird wieder ein JSON-Objekt mit nodes- und links-array generiert indem die eben erschaffene virtuelle Relation jetzt als Bestandteil einer weiteren Abfrage genutzt wird. So ließe sich aus Bestandsdaten mit Briefhandschriften ein soziales Netzwerk aus Adressaten, Absendern oder in den Briefen erwähten Personen generieren und exportieren.

## Reader zum Seminarteil
### Einführung zu Graphen und Netzwerken

Jannidis et al. (Hrsg.): *Digital Humanities. Eine Einführung*, Stuttgart 2021. (Hier Kapitel 10 "Netzwerke" S. 141-161)
Needham, Mark / Hodler, Amy E.: *Graph algorithms - practical examples in Apache Spark and Neo4j*, Boston u.a. 2019. (Hier Kapitel 2 "Graph Theory and Concepts" S. 15-28)
Kuczera, Andreas: *Graphentechnologien in den Geisteswissenschaften*, https://kuczera.github.io/Graphentechnologien/contents/ (gleichzeitig auch eine schöne Einführung zur Anwendung von neo4j)

### RDF und Labeled Property Graph

Jannidis et al. (Hrsg.): *Digital Humanities. Eine Einführung*, Stuttgart 2021. (Hier Kapitel 11 "Ontologien" S. 162-176)
Sakr et al. (Hrsg.): *Linked Data. Storing, Querying, and Reasoning*, Stuttgart 2018.
https://neo4j.com/blog/rdf-triple-store-vs-labeled-property-graph-difference/
https://dvcs.w3.org/hg/rdf/raw-file/tip/rdf-primer/Overview.html

### Datenbanken

Meier, Andreas / Kaufmann, Michael: *SQL & NoSQL Databases*, Stuttgart 2019.

### Anwendungsbeispiele

Matschinegg et al.: *Daten neu verknoten - Die Verwendung einer Graphdatenbank für die Bilddatenbank REALonline*, http://webdoc.sub.gwdg.de/pub/mon/dariah-de/dwp-2019-31.pdf
Wübbena, Thorsten: "Von Warburg zu Wikidata – Vernetzung und Interoperabilität kunsthistorischer Datenbanksysteme am Beispiel von ConedaKOR", in: Canan Hastik und Philipp Hegel (Hrsg.), *Bilddaten in den Digitalen Geisteswissenschaften*, Wiesbaden 2020, S. 133-149, DOI: 10.13173/9783447114608.133
Oldman, Dominic / Tanase, Diana: "Reshaping the Knowledge Graph by Connecting Researchers, Data and Practices in ResearchSpace", in: Vrandečić D. et al. (Hrsg.) *The Semantic Web – ISWC 2018. ISWC 2018. Lecture Notes in Computer Science, Part 1*, Cham 2018. S. 325-340.

