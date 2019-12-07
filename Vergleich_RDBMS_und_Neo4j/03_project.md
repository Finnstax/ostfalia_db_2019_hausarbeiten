# 3. Projektvorstellung / Bibliothek mittelalterliche Handschriften
Die Staatsbibliothek zu Berlin ermöglicht den ortsunabhängigen und kostenfreien Zugang zu den Ergebnissen der Handschriftenkatalogisierung im deutschen Sprachraum. Alle Metadaten der abendländischen mittelalterlichen Handschriften
sollen in diesem Projekt als Datengrundlage verwendet werden. Die Metadaten beschreiben den Handschriftenbestand und liegen in Form von XML Dateien vor. Diese Daten sollen in ein sehr vereinfachtes Datenmodell überführt und 
in den entsprechenden Datenbanksystemen persitiert werden. 

# 3.2 Technologie Stack

Um die theoretischen Fragestellungen praktisch evaluieren zu können wurde ein Java Enterprise Projekt mit Hilfe des [Spring Frameworks](https://spring.io/) aufgesetzt. 
Um in kurzer Zeit eine lauffähige Serverumgebung zu bekommen wurde [Spring Boot](https://docs.spring.io/spring-boot/docs/current/reference/html/) mit einem integriertem Tomcat Servlet Container als Laufzeitumgebung eingesetzt. Als Build Werkzeug 
kommt [Gradle](https://gradle.org/) zum Einsatz. Der gesamte Quellcode wird mit Hilfe des Versionierungssystem [Git](https://git-scm.com/) auf der Github Platform verwaltet.  

Um die relevanten Datenbanksystem zur Verfügung zu stellen haben wir uns entschieden [Docker](https://www.docker.com/) als Container Technologie zu verwenden. So konnten wir sehr leicht eine relationale Postgres Datenbank
und eine NEO4J Graphendatenbank anbinden. Um die Daten in diesen System zu persistieren wurden die ORM Bibliotheken [Spring NEO4J](https://spring.io/guides/gs/accessing-data-neo4j/) und [Spring Data](https://spring.io/projects/spring-data) verwendet. 

# 3.3 Domainmodell 

![Johannes Versor Handschrift 15 Jahrhundert](johannesversorbuch.jpeg)

Das Datenmodell dieses Projektes besteht aus folgenden Komponenten: 

  *  **Beschreibungsdokument**: Enthält alle Metadaten zu einer abendländischen Handschriftenbeschreibung. Es stellt im Context von Domain Driven Design das Root Aggregate dar. 
  *  **Dokumentenelement**: Ist ein inhaltlicher Bestandteil einer Beschreibung. Das kann Textabschnitt zur Beschreibung eines Einbandes oder eine Sammlung von Texten sein. 
  *  **Beteiligte**:  Ist ein abstraktes Element um gemeinsame Normdatenattribute für Personen und Koerperschaften zu verwalten. 
  *  **Ort**: Informationen zu einem Ort. 
  *  **Person**: Informationen zu einer Person, was in diesem Kontext ein Autor oder ein Besitzer einer Handschrift sein kann. 
  *  **Koerperschaft**: Informationen zu einer Institution. 
  *  **Provenienz**: Zeitlich begrenztes Besitzverhältnis. 
  *  **Digitalisat**: Ergebnis eines Digitalisierungsprozesses. In der Regel eine Bilddatei für eine Seite bzw. eines Handschriftenbestandteiles.
  
Das Beschreibungsdokument führt mehrere Objekte zusammen und verwaltet dieses. Ein Beschreibungsdokument besteht aus einer Hierarchie von Dokumentenelementen und den wichtigsten Metadaten. Zu jeder Handschrift werden Daten 
zu folgenden wichtigen Informationen erfasst: 

  * Titel: [Johannes Versor, Johannes Tinctoris](http://www.manuscripta-mediaevalia.de/#|5)
  * Signatur: S 67
  * Entstehungsdatum: 1460 
  * Bestandhaltende Einrichtung: Bonn, Universitäts- und Landesbibliothek Bonn
  * Dokumentenbestandteile
  * Provenienz
  * Buchbinder 

Nachfolgende Grafiken zeigen die fachliche Struktur einer Handschriftenbeschreibung. 

![Beispiel Handschrift aus dem 15 Jahrhundert](johannesversor.png)

![Beispiel Handschrift aus dem 15 Jahrhundert Inhalt](johannesversor_inhalt.png)

Beispiel: Dokumentenelement Text

![Beispiel Handschrift aus dem 15 Jahrhundert Inhalt](dokumentenelement.png) 

Anhand dieses Beispiel wurde kurz die fachliche Struktur einer Handschriftenbeschreibung dargestellt. Zu erkennen ist, das ein Dokument viele Beziehungen zu Orten, Personen, Instiutionen mit zeitlicher Varianz haben kann. 
  
# 3.4 Unterschiede in der Implementierung 

# 3.4.1 Objekt Mapping für RDBMS und Graphdatenbank
Alle Bestandteile des Datenmodells wurden mit den entsprechenden Annotation der der ORM Frameworks ausgezeichnet, sodass der OR - Mapper diese persistieren kann. Nachfolgende soll die Beschreibungsentität kurz erläutert werden: 

    @Entity
    @Table(name = "beschreibungen")
    @NodeEntity
    public class Beschreibungsdokument {
    
      protected Beschreibungsdokument() {
      }
    
      public Beschreibungsdokument(String id, String titel, String signatur) {
        this.id = id;
        this.titel = titel;
        this.signatur = signatur;
        this.bestandteile = new HashSet<>();
        this.orte = new HashSet<>();
      }
    
      @javax.persistence.Id
      @Id
      private String id;
    
      @Column(length = 4096)
      private String titel;
    
      private String signatur;
    
      @ManyToMany(fetch = FetchType.EAGER)
      private Set<Ort> orte;
    
      @ManyToOne(fetch = FetchType.EAGER)
      @Relationship(type = "BUCHBINDER")
      private Beteiligte buchbinder;
    
      @OneToMany(fetch = FetchType.EAGER, cascade = {CascadeType.ALL})
      @Relationship(type = "ENTHAELT")
      private Set<DokumentElement> bestandteile;
      

**Annotation für GRAPH Datenbank** 

Ein Neo4J Knoten wird durch die Annotation **@NodeEntity** gekennzeichnet. Zusätzlich benötigt ein Knoten eine eindeutige Identifikationsnummer. Eine Beziehung zu anderen Knoten wird 
mit Hilfe der Annotation **@Relationship** gekennzeichnet. Mit Hilfe dieser Annotationen kann ein Objekt Graph Mapper, wie er im Spring Data Neo4J verwendet wird dieses Objekt in einer Graphendatenbank persistieren. 

# 3.4.1.1 Objekt Mapping, Besonderheit RelationshipEntity

Eine Besonderheit im Bereich der Graphendatenbank stellt hierbei das Objekt Provienz dar. Dieses Objekt ist ein Relationsobjekt **@RelationshipEntity()** welches zwei Entitäten miteinander verwendet. Ein solches Objekt gibt es im Kontext des RDBMS nicht. 

    @Entity
    @RelationshipEntity()
    public class Provenienz {
  
    protected Provenienz() {
    }
  
    public Provenienz(String id,ProvenienzTyp typ,
        Beteiligte beteiligte, String vonJahr, String bisJahr, Beschreibungsdokument beschreibungsdokument) {
      this.id = id;
      this.typ = typ;
      this.beteiligte = beteiligte;
      this.vonJahr = vonJahr;
      this.bisJahr = bisJahr;
      this.beschreibungsdokument = beschreibungsdokument;
    }
  
    @javax.persistence.Id
    @Id
    private String id;
  
    @Enumerated
    private ProvenienzTyp typ;
  
    @ManyToOne(fetch = FetchType.EAGER)
    @EndNode
    private Beteiligte beteiligte;
  
    private String vonJahr;
  
    private String bisJahr;
  
    @OneToOne(fetch = FetchType.EAGER)
    @StartNode
    private Beschreibungsdokument beschreibungsdokument;
    
Die Provienz verbindet hierbei die Entitäten Beschreibungsdokument und Betetiligte in einer gerichtet Form. Die Richtung wird durch die Annotationen **@StartNode** und **@EndNode** gekennzeichnet. 

# 3.4.1.1 Objekt Mapping, Besonderheit Vererbung 

In unserem Beispiel spielt für folgende Objekt die Vererbung im Sinne der Objektorientierten Programmierung eine Rolle. Die Objekte Ort, Koerperschaft und Person sind alle vom Typ Beteiligte. 

    public class Ort extends Beteiligte
    public class Koerperschaft extends Beteiligte
    public class Person extends Beteiligte
    
Für ein relationales Datenbanksystem muss eine Strategie festgelegt werden, in wieviel Tabellen die Daten vom Typ Beteiligte abgelegt werden. 

    Entity
    @Table(name = "beteiligte")
    @Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
    public abstract class Beteiligte {

In unserem Beispiel wird für jede Entität vom konkreten Typ eine Tabelle angelegt. So enthät die spätere Datenbank eine Tabelle Person, Ort und Körperschaft. Weitere Möglichkeiten die Daten 
im RDBMS abzulegen sind: 

  * SINGLE_TABLE, Eine Tabelle für alle Typen von Beteiligte 
  * JOINED, Eine Tabelle für gemeinsame Attribute und jeweils eine Tabelle für Typ spezifische Attribute. Diese werden mit einem Join verbunden. 
  
Für die Graphendatenbank werden hier keine Steuerungsangaben benötigt. Jeder Knoten ist lediglich von dem konkreten Typ und der abstrakte Typ Beteiligte findet keine Verwendung. 

**Annotationen für RDBMS** 

Ein Objekt welches in eine relationale Datenbank persistiert werden soll, muss mit der Annotation **@Entity** gekennzeichnet werden. Genau wie bei Neo4J muss dieses Objekt ein Attribute mit der **@Id** Annotation als primäry Key kennzeichnen. 
Für Beziehungen zu anderen Objekten stehen folgende Annotationen zur Verfügung: **@OneToOne**, **@OneToMany**, **@ManyToOne**, **@ManyToMany**. Für diese Annotationen müssen noch Werte für das Verhalten beim Laden angegeben werden. Zusätzlich können Daten zusammenhänged 
gespeichert, gelöscht oder geladen werden was die Arbeit mit Objekten des ORM in der Praxis oft komplex werden lässt. 

Bei der Konfiguration der Objekt Mapper scheint das relationale Datenbankmanagementsystem mehr Steuerung zu benötigen als die Graphendatenbank.



# 3.4.2 CREAD, READ, UPDATE and DELETE (CRUD) für RDBMS und Graphdatenbank

Mit Spring Data bekommt der Entwickler eine der einfachsten Möglichkeiten für die Umsetzung der CRUD Operation an die Hand die es meiner Meinung nach gibt. Für jede Entität die durch Spring Data verwaltet werden soll muss lediglich 
ein CRUDRepository erweitert werden. 

**Beispiel NEO4J**

    public interface BeschreibungsdokumentGraphRepository extends Neo4jRepository<Beschreibungsdokument,String> {
    
    }

**Beispiel RDBMS**
  
    public interface BeschreibungsdokumenteRDBMSRepository extends CrudRepository<Beschreibungsdokument,String> {
    
    }

Die Implementierung dieser Interfaces wird durch SpringData selbst vorgenommen. Mit Hilfe von Generics muss im Interface lediglich der Typ der Entität und der Typ des primäry Keys angebeben werden. In beiden Fällen ist dies jeweils
die Klasse Beschreibungsdokument und die Klasse String. Die Verwendung des Repositories ist in beiden Fällen sehr einfach: 

      @Autowired
      private BeschreibungsdokumentGraphRepository beschreibungsdokumentGraphRepository;
    
      @Autowired
      private BeschreibungsdokumenteRDBMSRepository beschreibungsdokumenteRDBMSRepository;
      
      beschreibungsdokumentGraphRepository.deleteAll();
      
      beschreibungsdokumenteRDBMSRepository.deleteAll();
      
      beschreibungsdokumentGraphRepository.saveAll(l);
      
      beschreibungsdokumenteRDBMSRepository.saveAll(l);
      
Im ersten Schritt muss die Implementierng via Dependency Injection einer Variable zugewiesen werden. Anschließend können die Entitäten mit save, delete, oder findBy Methoden in die entsprechende Datenbank 
persistiert, gelöscht oder aktualisiert werden. Für die Verwendung der Standard Operation macht es hierbei keinen Unterschied ob diese auf einer Graphendatenbank oder einem relationales DBMS durchgeführt werden.

      
# 3.4.3 Queries für RDBMS und Graphdatenbank
