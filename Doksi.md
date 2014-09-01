Why PostgreSQL? (Choosing DB)
  - considered MySQL, HSQLDB, Postgre (table of features)
   MySQL: free, no transaction, blob yes 4GiB, but problems: http://stackoverflow.com/questions/5775571/what-is-the-maximum-length-of-data-i-can-put-in-a-blob-column-in-mysql
  Postgre: free, transaction, blob ( own format, 4TiB from 9.3 http://www.postgresql.org/docs/9.3/static/lo-intro.html , streamable) 
 PG LOB handling - opening, where is it stored (table)
  HSQLDB: free, lobs (also streamable),rather private opinion
  LOB problems discussed later (ugly JPA interface for large LOBs) http://stackoverflow.com/questions/9898739/save-blob-using-a-stream-from-ejb-to-database-in-a-memory-efficient-way
  - Conclusion: free, multiplatform, transactions, well documented, widespread, nice LOB features
Why JPA?
  - original version
      - SQL queries, Database dependent
  - maybe ported for other DB
  - to be less painful, than it was now
  - common usage of JPA
  - which connector - eclipselink vs hibernate
    eclipselink - nice feature of db configuration
JPA configuration
 - database connection injection
 - entity manager injection
     - DAO API interfaces
     - proxying
     - Abstract, inheritance, less code
     - using Persistence - more useful configurations
 - persistence.xml location
 - Beans.xml location
 - using classes - base project
 - Database configurator
     -change config files in jars
 - in the future:
     configuration out of source
Initial checks - DB and JPA is alive (raw connection, eclipselink JAR)
JPA persistence API
 - entity manager-persistence context
 - transaction context
 - local copy
 - object relational model
Entities in Pikater
    - some special features of entities
    - special fields: temporal (id vs named), dates
    - multiple relations (Many to one)
 - storing
 - retrieval
      query types: pure JPQL, Named Queries, Criteria API
      pros, cons
         JPQL , NQ, Crit - can be external, performance (criteria cache), modifying queries (params, String join, API calls)
      example queries (named and criteria)
 - usage in Pikater
      core vs web (ordering, visibility)
      security is only matter to web
 - entity removal
Dataset storage
  - in old Pikater (maybe reference to other parts of documentation)
  - considerations: centralized file based, FTP distribution 
  - current version: Postgre LOB + stream distribution, uniqeness of datasets
        -streaming of datasets
  - dataset removal (just visibility)
Storing models
  - byte array - standard way (@Lob - property)
  - each model is unique - no modifying
  - removal
 Storing experiments
   - XML + String and Lob
Dataset file conversion
  - libs decision, ARFF description, header (w/wo)
Agent_DataManager
   - what the functions do?
Agent_MetaDataQueen
   - metadata
Visualisation
   - maybe its connected to the metadataqueen
Why are we just adding datas? (experiment dependency)


 
________________


# Databáza v Pikateru
Pikater v rámci svojej úlohy vykonáva rozne operácie na vstupných datách a pritom vzniknú dáta rozneho povahu: napr. pri klasifikácii vznikne nový, už klasifikovaný dataset a štatistika výpočtu. Tieto dáta musia byť niekam uložené a sprístupnené pre prípadné ďalšie experimenty. Povodná verzia Pikateru datasety mal pripravené vo tvare súborov na disku a ostatné dáta boli uložené v databázi.
##Datasety
Vstupné dáta pre experimenty nazývame datasety. Formát týchto datasetov je identický formátu ARFF používaný knižnicou Weka, čo mulitagentný systém Pikateru interne používa.


## Výber databázového systému
Povodná verzia Pikateru používala databázový systém MySQL ako úložisko vytvorených dát. Prístup k serveru prebiehol pomocou JDBC ovládača. Dotazovanie bolo vyriešené pomocou dynamckého vytvorenia SQL dotazov. Výsledkom týchto dotazov bol klasický javovský resultset.
Nevýhodou tohto princípu bola hlavne viazanosť k danému databázovému systému a neflexibilnosť pri zmenách v schémate.
Zmeny v schématu síce mali byť zriedkavé, ale keď sa k tomu dojde musí byť vyriěsená prípadná nekonzistentnosť medzi starou verziou dotazu a súčasnou verziou databáze.
### Výber PostgreSQL
Pri výbere nového databázového systému sme brali do úvahy tri systémy: MySQL, PostgreSQL a HyperSQL DB (HSQLDB). Všetky tri majú nejakú verziu dostupné bez poplatkov, čo je docela výhodné z finančného hladiska.
Pre projekt Pikater výsledná databáza musela splňovať kritérie multiplatformovosti, schopnosti tranzakčného spracovávania a - dosledkem analýzy problému uloženia vstupných údajov - schopnosti uloženia vačších (až niekolko 100 megabajtových) súborov.
Nasledujúca tabulka obsahuje dáta o tom,že aké vlastnosti majú jednotlivé kandidáty:


|               |   MySQL   |   HSQLDB  | PostgreSQL |
|:-------------:|:---------------:| :---------------:|:------------------:|
| tranzakcie | áno (InnoDB) | áno | áno |
| súbory (LOBy) | áno | áno (64TiB spolu) | áno (4TiB per súbor) |
| iné | povodne použitý, rozšírená | Java | rozšírená |


Síce MySQL podporuje tranzakčné spracovávanie, ale to je dostupné iba pre engine InnoDB, čo je použitelné iba za poplatok. Vzhladom na to, že ostatné dva to ponúkajú zadarmo MySQL už začal s hendikepom.
Medzi HSQLDB a PostgreSQL sme rozhodli hlavne na základe prístupu k uloženým bytovým objektom. Súčasné databázové systémy už štandardne umožnia nám používať bytové pole v rámci databázového záznamu, kam možeme uložit obsah súboru. Jediný problém znamená sposob prístupu k takto uloženým datám.
PostgreSQL používá na to vlastný prístup, čo umožní kopírovať dáta z/do databáze pomocou streamov. V HSQLDB k prístupu týmto záznamom máme používať setter a getter, čo pracuje s javovským java.lang.Object triedou. Vzhladom na to, že plánujeme pracovať s docela velkými súbormi, rozhodli sme sa pre PostgreSQL.
Trošku subjektívny pohlad, ale PostgreSQL je rozšírenejší ako HSQLDB, čo teoreticky može znamenať lepšiu dostupnosť informácií a tým pádom aj jednoduchšie riešenie prípadných problémov.
## Práca so súbormi v DB
Po výbere Postgre ako databázový systém pre Pikater sme museli naimplementovať prístup k súborum uložené v databáze. Súbory sú uložene ako Postgre Large Objekty (PgLOBy, alebo jednoducho LOBy), čo znamená, že je uložené v systémovej tabulke pg_largeobject danej databáze. Pre každý súbor je vytvorený množstvo záznamov, ktoré obsahujú súbor rozsekaný na niekolko byteových polí.
Pri získaní dát alebo zapisovaní do databáze ale vystačíme s object IDm (oid) daného PgLOBu JDBC objekt pripojenia do databáze vytvorí odpovedajúci stream na čítanie/zápis.
V projektu Pikater toto mechanizmus je sprístupnená pomocou tried uložené v balíčku `org.pikater.shared.database.postgre.largeobject`.
Trieda PGLargeObjectReader odvodená od java.io.Reader slúži načítanie dát z PgLOBu, keď známe jeho identifikačné číslo. Táto trieda je využitá v `PGLargeObjectAction`, čo sa nachádza v rovnakom balíčku, ale nie je obmedzené jej samostatné používanie.
Trieda `PGLargeObjectAction` podporuje kopírovanie z/do súboru do/z PgLOBu. Táto trieda je použitá pri uploadovaní nových datasetov, alebo pri cachovaní pri vizualizácií.
## Prechod na JPA
V kapitole o výbere sme zmienili o tom, že povodná verzia Pikateru používala prístup viazaný na databázový systém MySQL. Rozhodli sme sa to zmeniť na použitie technológie JPA, čo nám umožnila mapovania záznamov z DB na javovské objekty a spať.
Obrovskou výhodou JPA je, že objekty pripravené na prácu s JPA nemusia byť zmenené pri zmene databázového systému. Takto pripravené objekty nazývame entitami a najjednoduchšie sú rozpoznatelné na základe značky `@Entity` pre definíciou javovskej tridy.
Pri vytvorení novej štruktúry entít - a tým pádom aj súčasnú schématu databáze - sme vela zachovali z povodnej schematy. Napr.: entita na uloženie štatistiky experimentu obsahuje prevaže premenná obsahujúce rovnaké údaje, ako sloupce povodnej tabulky. Na druhej strane bolo potrebné vytvoriť chýbajúce vzťahy medzi entitami (napr.: batch má viac experimentov) a podklady pre nové funkcie (napr.: uloženie datasetov).
Práve u uložení datasetov (vstupných súborov) sme museli trošku opustiť princip entit využívaný v JPA. PostgreSQL podporuje streamový prístup k PgLOBom. V JPA entite JPADatasetLO máme jednu premennú typu `long`, čo obsahuje OID a na základe tohto OID možeme pomocou `PGLargeObjectReader` čítať stream uloženého súboru.
Síce možeme používať anotáciu `@Lob` u premenných, čo pre JPA naznačí, aby danú premennú uložil do bytoveho pola, ale nemáme istotu, že velké súbory budú optimalne vyriešené. V najhoršom prípade može sa nám stať, že obsah súboru je celý kopírovaný do pamate a potom je prenesený do databáze.
### Výber JPA connectoru
Rozhodnutie pre JPA znamená, že okrem databáze a technológie pre prístup k nemu potrebujeme, čo spojuje svet Javy so svetom databáze. V praxi to znamenű prevod operácii v JPA na rozne príkazy v SQL a naopak vytvorenie pre výsledok dotazu vytvoriť odpovedajúce entity. Obvykle výber tohto connectoru neovplyvní vývoj aplikácie, okrem použitia nejakých špeciálnych funkcí. Dva z najrozšírenejších connectorov sú Hibernate a Eclipselink. My sme sa rozhodli pre Ecipselink z dovodu, že u Eclipselink je možne zmeniť vlastnosti databázového pripojenia zo zdrojového kódu, čo možeme v budúcnosti využívať.
## JPA entity Pikateru
V súčasnej verzii pre Pikater je vytvorené 16 entit, ktoré sa nachádzajú v balíčku `org.pikater.shared.database.jpa` . Každá entita je odvodená od abstraktnej entity JPAAbstractEntity, čo sa nachádza v rovnakom balíčku. Tento spoločný predek obsahuje iba ID slúžiace, ako primárny klúč v databáze.
Zoznam entit:
1. `JPAAgentInfo` - obsahuje záznamy o agentoch, ktoré sú k dispozicii v systému Pikater. Hlavným účelom týchto entit, je prevod informácii z jadra Pikateru na webové rozhranie, čo získa dáta z databáze
2. `JPADataSetLO` - obsahuje záznamy o datasetoch uložené v databáze. Nejdolezitejsi je polozka OID, čo obsahuje ID PgLOBu, kde je daný dataset uložený. Položka hash slúži na identifikáciu datasetu pomocou MD5 hashe, čo je vypočítaný pri nahrávaní datasetu. Síce može existovať viac JPADataSetLO entit pre rovnaký hash, ale datasety sú považované za identické pri uploadovaní, takže v databáze sú uložené iba raz a všetky entity obsahujú rovnaký OID .
Z pohladu roznych metod strojového učenia metadata datasetov hrajú významnú rolu, pretože na základe nich možeme odhadnúť podobnosť dvoch datasetov bez jejich načítaní. Z tohto dovodu tieto metadata sú vypočítané už pri uploadovaní datasetov a sú uložené v odpovedajúcich entitách, ktoré sú spojené s entitou datasetu.
1. `JPAGlobalMetaData` - entita pre metadata, ktoré je možné zistiť u každého platného datasetu. Obsahuje dáta o tom, že dataset kolko má dátových riadkov (instancov) a aké atributy obsahuje
2. JPAAttributeMetadata - entita pre metadata jednotlivých relací (stĺpcov) v rámci datasetu. Tieto metadata je možné zistiť u každého typu atributu a možné z nich zistiť pomer chýbajúcich hodnot (napr. počet riadkov, bez klasifikácie), triedy entropie, či atribut je cielový a aj povodne poradie v datasetu.
3. JPAAttributeCategoricalMetadata - entita, čo obsahuje dáta špecifické pre kategorické atributy - ktoré obsahujú hodnoty z nejakej množiny, čo je definovaná v hlavičke datasetu - a je odvodená od JPAAttributeMatadata. V súčasnej dobe dodefinuje premennú pre počet kategorií daného datasetu.
4. JPAAttributeNumericalMetadata - táto entita je podobne ako JPAAttributeCategoricalMetadata je odvodená od entity JPAAttributeMetadata. Obsahuje metadata pre numerické relácie datasetu. Numerické relácie možu obsahovať hodnoty s pohyblivou desatinnou čiarkou. Pri vytvorení týchto metadat Weka vypočíta interval týchto hodnot, priemer a rozptyl.
U datasetov požívame dve dalšie entity, ktoré majú 
1. JPAFilemapping - jednotlivé záznamy predstavujú, pre každý dataset párovanie jeho povodného názvu na MD5 hash súboru. Tento prístup možno nie je najlepší, preto vytvári akoby duplicitný záznam pre datasetov, ale zachová prevodnú tabulku zo starého Pikateru. Hlavným dovodom možnosti použitia povodných názvov v rámci jednotlivých experimentov je zaistenia povodného prístupu, čo zaistil pridanie experimentu z príkazového riadka z XML súboru, čo obsahuje iba klasické názvy datasetov.
2. JPATaskType - každý dataset má definovaný typ prednastaveného experimentu. Počítali sme s tým, že množina týchto úloh može byť rozšírený, preto sme rozhodli pre zavedenie entity, čo možeme dynamicky vytvárať pre nové neznáme úlohy.
3. [JPABatch](http://github.com/krajj7/pikater/src/org/pikater/shared/database/jpa/JPABatch.java)
4. JPAExperiment
5. JPAExternalAgent
6. JPAModel
7. JPAResult
8. JPAUser
9. JPAUserPriviledge
10. JPARole


## JPA-špecifická anotácia
Pomocou anotovania možeme zmeniť sposob, ako JPA spravuje jednotlivé objekty.
Každá entita má zadefinovaná, že do ktorej tabulky bude uložený jej obsah. K tomu slúži značka `@Table(name=”<názov tabulky>”)` . Táto značka nemusí byť uvedená, v tom prípade názov tabulky, je rovnaký ako meno triedy. Tieto tabulky sú vytvorené automaticky, na základe konfigurácie JPA.
Okrem toho vela dotazov je vyriešený pomocou pomenovaných dotazov, ktoré sú zapísané v entitách pomocou značiek `@NameQueries` a `@NamedQuery`.
U premenných vo vačšine prípadov nemusíme nič riešiť, lebo vyhovuje nám štandardný prístup JPA. Napríklad primitívne typy sa mapujú bez problémov na primitívne typy databáze.
Na druhej strane občas potrebujeme používať nasledujúce značky:
`@Id` a `@GeneratedValue(strategy = GenerationType.AUTO)` - používame u premennej, čo chceme používať ako primárny klúč danej entity. Súčasne používame v entite JPAAbstractEntity a pomocou dedičnosti máme vyriešený primárny klúč pre všetky entity.
`@Transient` - takto označené premenné nie sú uložené do databáze. V súčasnej verzii používame pre premennú obsahujúce meno entity.
`@Inheritance(strategy=InheritanceType.TABLE_PER_CLASS` alebo `InheritanceType.JOINED`) - značka slúží na zadefinovanie, že ako chceme vytvoriť tabulky relácií pre jednotlivé entity. Nastavenie `TABLE_PER_CLASS` vytvorí vlastnú tabulku so všetkými hodnotami, ktoré v danej entite vyskytujú (vlastné aj dedené). Nastavenie JOINED vytvorí iba jednu tabulu, pre danú entitu a všetky z nej odvodené. Znamená to, že niektoré položky sú prázdne, ale v niektorých prípadoch je to viac prehladný.
`@Lob` - síce uloženie velkých súborov máme vyriešené pomocou PgLOBov, ale menšie možeme uložiť aj pomocou JPA anotácie. V tomto prípade premenná sa serializuje a uloží sa do bytového pola - alebo iného kompatibilného typu - v databázovom záznamu. Tento prístup využívame napr. u uložení modelov výpočtov, alebo u dlhých reťazcov XMLov.
`@Temporal(TemporalType.TIMESTAMP)` - časové údaje označime pomocou tejto značky a definujeme, že v akom formátu chcem ich uložiť
`@Nullable` - daná premenná može byť null v databáze
`@Enumerated(EnumType.STRING)` - u premenných odvodených od java.lang.Enum možeme zadefinovať, či chcem aby sa do databáze uložil název hodnoty, alebo jeho poradové číslo (`EnumType.ORDINAL`). Uložiť Enum ako String je výhodnejšie z tohto dovodu, že pri pridaní novej položky nemusím brať do úvahy poradie hodnot. Vymazávanie nejakej položky zo zdrojového kódu može sposobiť výnimku pri získaní hodnot z databáze. Na druhej strane pri uložení ako `EnumType.ORDINAL` nemusíme získať výnimku, čo može sposobiť zákerné chyby, pretože hodnoty enumov sú zle namapované.
`@OneToMany(cascade=CascadeType.PERSIST)` - anotácia zoznamov, nastavenie `CascadeType.PERSIST` umožní, aby prvky zoznamu nemuseli byť zvlášť persistované v databáze. Pri uložení entity automaticky sa uložia všetky prvky v zoznamu.
________________


# Použité skratky
ARFF - Attribute-Relation File Format
CSV - Comma-Separated Values
JDBC - Java Database Connectivity
SQL - Strucutered Query Language
HSQLDB - Hyper SQL Database
PgLOB - Postgre Large Object
JPA - Java Persistence API