# Databáza v Pikateru
Pikater v rámci svojej úlohy vykonáva rôzne operácie na vstupných datách a pritom vzniknú dáta rôzneho povahu: napr. pri klasifikácii vznikne nový, už klasifikovaný dataset a štatistika výpočtu. Tieto dáta musia byť niekam uložené a sprístupnené pre prípadné ďalšie experimenty. Pôvodná verzia Pikateru datasety mal pripravené vo tvare súborov na disku a ostatné dáta boli uložené v databázi.
##Datasety
Vstupné dáta pre experimenty nazývame datasety. Formát týchto datasetov je identický formátu ARFF používaný knižnicou Weka, čo mulitagentný systém Pikateru interne používa.


## Výber databázového systému
Pôvodná verzia Pikateru používala databázový systém MySQL ako úložisko vytvorených dát. Prístup k serveru prebiehol pomocou JDBC ovládača. Dotazovanie bolo vyriešené pomocou dynamckého vytvorenia SQL dotazov. Výsledkom týchto dotazov bol klasický javovský resultset.
Nevýhodou tohto princípu bola hlavne viazanosť k danému databázovému systému a neflexibilnosť pri zmenách v schémate.
Zmeny v schématu síce mali byť zriedkavé, ale keď sa k tomu dojde musí byť vyriěsená prípadná nekonzistentnosť medzi starou verziou dotazu a súčasnou verziou databáze.
### Výber PostgreSQL
Pri výbere nového databázového systému sme brali do úvahy tri systémy: MySQL, PostgreSQL a HyperSQL DB (HSQLDB). Všetky tri majú nejakú verziu dostupné bez poplatkov, čo je docela výhodné z finančného hladiska.
Pre projekt Pikater výsledná databáza musela splňovať kritérie multiplatformovosti, schopnosti tranzakčného spracovávania a - dôsledkem analýzy problému uloženia vstupných datasetov - schopnosti uloženia väčších (až niekolko 100 megabajtových) súborov.
Nasledujúca tabulka obsahuje dáta o tom, že aké vlastnosti majú jednotlivé kandidáty:


|               |   MySQL   |   HSQLDB  | PostgreSQL |
|:-------------:|:---------------:| :---------------:|:------------------:|
| tranzakcie | áno (InnoDB) | áno | áno |
| súbory (LOBy) | áno | áno (64TiB spolu) | áno (4TiB per súbor) |
| iné | povodne použitý, rozšírená | Java | rozšírená |


Síce MySQL podporuje tranzakčné spracovávanie, ale to je dostupné iba pre engine InnoDB, čo je použitelné iba za poplatok. Vzhladom na to, že ostatné dva to ponúkajú zadarmo MySQL už mal docela velkú nevýhodu v porovnaní s ostatnými dvomi.


Medzi HSQLDB a PostgreSQL sme sa rozhodli hlavne na základe prístupu k uloženým bytovým objektom. Súčasné databázové systémy už štandardne umožnia nám používať bytové pole v rámci databázového záznamu, kam môžeme uložit obsah súboru. Jediný problém znamená spôsob prístupu k takto uloženým dátam.
PostgreSQL používá na to vlastný prístup, čo umožní kopírovať dáta z/do databáze pomocou streamov. V HSQLDB k prístupu týmto záznamom máme používať setter a getter, čo pracuje s javovským java.lang.Object triedou. Vzhladom na to, že plánujeme pracovať s docela velkými súbormi, rozhodli sme sa pre PostgreSQL.
Trošku subjektívny pohlad, ale PostgreSQL je rozšírenejší ako HSQLDB, čo teoreticky môže znamenať lepšiu dostupnosť informácií a tým pádom aj jednoduchšie riešenie prípadných problémov.


## Práca so súbormi v DB
Po výbere PostgreSQL ako databázový systém pre Pikater sme museli naimplementovať prístup k súborum uložené v databáze. Súbory sú uložene ako Postgre Large Objekty (PgLOBy, alebo jednoducho LOBy), čo znamená, že je uložené v systémovej tabulke pg_largeobject danej databáze. Pre každý súbor je vytvorený množstvo záznamov, ktoré obsahujú súbor rozsekaný na niekolko byteových polí.
Pri získaní dát alebo zapisovaní do databáze ale vystačíme s object IDm (oid) daného PgLOBu. Objekt JDBC pripojenia do databáze vytvorí odpovedajúci stream na čítanie/zápis.
V projektu Pikater toto mechanizmus je sprístupnená pomocou tried uložené v balíčku [org.pikater.shared.database.postgre.largeobject](http://github.com/krajj7/pikater/tree/master/src/org/pikater/shared/database/postgre/largeobject).
Trieda `PGLargeObjectReader` odvodená od `java.io.Reader` slúži načítanie dát z PgLOBu, keď známe jeho identifikačné číslo. Táto trieda je využitá v `PGLargeObjectAction`, čo sa nachádza v rovnakom balíčku, ale nie je obmedzené jej samostatné používanie.
Trieda `PGLargeObjectAction` podporuje kopírovanie z/do súboru do/z PgLOBu. Táto trieda je použitá pri uploadovaní nových datasetov, alebo pri cachovaní pri vizualizácii.
## Prechod na JPA
V kapitole o výbere databázového systému sme zmienili o tom, že pôvodná verzia Pikateru používala prístup viazaný na databázový systém MySQL. Rozhodli sme sa to zmeniť na použitie technológie JPA, čo nám umožnila mapovania záznamov z databáze na javovské objekty a späť.
Obrovskou výhodou JPA je, že objekty pripravené na prácu s JPA nemusia byť zmenené pri zmene databázového systému. Takto pripravené objekty nazývame entitami a najjednoduchšie sú rozpoznatelné na základe značky `@Entity` pred definíciou javovskej triedy.
Pri vytvorení novej štruktúry entít - a tým pádom aj súčasnú schématu databáze - sme vela zachovali z pôvodnej schémy. Napr.: entita na uloženie štatistiky experimentu obsahuje prevaže premenná obsahujúce rovnaké údaje, ako sloupce pôvodnej tabulky. Na druhej strane bolo potrebné vytvoriť chýbajúce vzťahy medzi entitami (napr.: experiment má viac podexperimentov) a podklady pre nové funkcie (napr.: uloženie datasetov).
Práve u uložení datasetov (vstupných súborov) sme museli trošku opustiť princíp entít využívaný v JPA. PostgreSQL podporuje streamový prístup k PgLOBom. V JPA entite `JPADatasetLO` máme jednu premennú typu `long`, čo obsahuje OID a na základe tohto OID môžeme pomocou `PGLargeObjectReader` čítať stream uloženého súboru.
Síce môžeme používať anotáciu `@Lob` u premenných, čo pre JPA naznačí, aby danú premennú uložil do bytoveho pola, ale nemáme istotu, že velké súbory budú optimalne vyriešené. V najhoršom prípade môže sa nám stať, že celý obsah súboru je kopírovaný do pamäte a potom je prenesený do databáze.
### Výber JPA connectoru
Rozhodnutie pre JPA znamená, že, okrem databáze a technológie pre prístup k nemu, potrebujeme niečo, čo spojuje svet Javy so svetom databáze. V praxi to znamená prevod operácií v JPA na rôzne príkazy v SQL a naopak pre výsledok dotazu vytvoriť odpovedajúce entity. Obvykle výber tohto connectoru neovplyvní vývoj aplikácie, okrem použitia nejakých špeciálnych funkcí. Dva z najrozšírenejších connectorov sú Hibernate a Eclipselink. My sme sa rozhodli pre Ecipselink z dôvodu, že u Eclipselink je možné zmeniť vlastnosti databázového pripojenia zo zdrojového kódu, čo môžeme v budúcnosti využívať.
## JPA entity Pikateru
V súčasnej verzii pre Pikater máme 16 vytvorených entit, ktoré sa nachádzajú v balíčku `org.pikater.shared.database.jpa` . Každá entita je odvodená od abstraktnej entity [JPAAbstractEntity](http://github.com/krajj7/pikater/tree/master/src/org/pikater/shared/database/jpa/JPAAbstractEntity.java), čo sa nachádza v rovnakom balíčku. Tento spoločný predok obsahuje iba ID slúžiace, ako primárny klúč v databáze.
Zoznam entít:


1. [JPAAgentInfo](http://github.com/krajj7/pikater/tree/master/src/org/pikater/shared/database/jpa/JPAAgentInfo.java) - obsahuje záznamy o agentoch, ktoré sú k dispozicii v systému Pikater. Hlavným účelom týchto entit, je prevod informácii z jadra Pikateru na webové rozhranie, čo získa dáta z databáze.
2. [JPADataSetLO](http://github.com/krajj7/pikater/tree/master/src/org/pikater/shared/database/jpa/JPADataSetLO.java) - obsahuje záznamy o datasetoch uložené v databáze. Nejdôležitejší je položka OID, čo obsahuje ID PgLOBu, kde je daný dataset uložený. Položka hash slúži na identifikáciu datasetu pomocou MD5 hashe, čo je vypočítaný pri nahrávaní datasetu. Síce môže existovať viac JPADataSetLO entít pre rovnaký hash, ale datasety sú považované za identické pri uploadovaní, takže v databáze sú uložené iba raz a všetky entity obsahujú rovnaký OID .
Z pohladu rôznych metod strojového učenia metadata datasetov hrajú významnú rolu, pretože na základe nich môžeme odhadnúť podobnosť dvoch datasetov bez jejich načítaní. Z tohto dôvodu tieto metadata sú vypočítané už pri uploadovaní datasetov a sú uložené v odpovedajúcich entitách, ktoré sú spojené s entitou datasetu.
3. [JPAGlobalMetaData](http://github.com/krajj7/pikater/tree/master/src/org/pikater/shared/database/jpa/JPAGlobalMetaData.java) - entita pre metadata, ktoré je možné zistiť u každého platného datasetu. Obsahuje dáta o tom, že dataset kolko má dátových riadkov (inštancií) a aké atributy obsahuje
4. [JPAAttributeMetadata](http://github.com/krajj7/pikater/tree/master/src/org/pikater/shared/database/jpa/JPAAttributeMetadata.java) - entita pre metadata jednotlivých relací (stĺpcov) v rámci datasetu. Tieto metadata je možné zistiť u každého typu atributu a možné z nich zistiť pomer chýbajúcich hodnot (napr. počet riadkov, bez klasifikácie), triedy entropie, či atribut je cielový a aj pôvodne poradie v datasetu.
5. [JPAAttributeCategoricalMetadata](http://github.com/krajj7/pikater/tree/master/src/org/pikater/shared/database/jpa/JPAAttributeCategoricalMetadata.java) - entita, čo obsahuje dáta špecifické pre kategorické atributy - ktoré obsahujú hodnoty z nejakej množiny, čo je definovaná v hlavičke datasetu - a je odvodená od JPAAttributeMatadata. V súčasnej dobe dodefinuje premennú pre počet kategorií daného datasetu.
6. [JPAAttributeNumericalMetadata](http://github.com/krajj7/pikater/tree/master/src/org/pikater/shared/database/jpa/JPAAttributeNumericalMetadata.java) - táto entita je, podobne ako JPAAttributeCategoricalMetadata, odvodená od entity JPAAttributeMetadata. Obsahuje metadata pre numerické relácie datasetu. Numerické relácie môžu obsahovať hodnoty s pohyblivou desatinnou čiarkou. Pri vytvorení týchto metadát Weka vypočíta interval týchto hodnot, priemer a rozptyl.
7. [JPAFilemapping](http://github.com/krajj7/pikater/tree/master/src/org/pikater/shared/database/jpa/JPAFileMapping.java) - jednotlivé záznamy predstavujú, pre každý dataset párovanie jeho povodného názvu na MD5 hash súboru. Tento prístup možno nie je najlepší, preto vytvári akoby duplicitný záznam pre datasetov, ale zachová prevodnú tabulku zo starého Pikateru. Hlavným dovodom možnosti použitia povodných názvov v rámci jednotlivých experimentov je zaistenia povodného prístupu, čo zaistil pridanie experimentu z príkazového riadka z XML súboru, čo obsahuje iba klasické názvy datasetov.
8. [JPATaskType](http://github.com/krajj7/pikater/tree/master/src/org/pikater/shared/database/jpa/JPATaskType.java) - každý dataset má definovaný typ prednastaveného experimentu. Počítali sme s tým, že množina týchto úloh môže byť v budúcnosti rozšírený, preto sme sa rozhodli pre zavedenie entity, čo môžeme dynamicky vytvárať pre nové neznáme úlohy.


Hlavnou úlohou Pikateru je spustenie exprimentov strojového učenia. Tieto experimenty sú vytvorené na webovom rozhraní a potom sa dastanú do jadra systému. Na uloženie experimentov sme chceli využívať databázový systém. Aby sme to mohlo vyriešiť potrebovali sme ďalšie entity spojené s experimentami.


9. [JPABatch](http://github.com/krajj7/pikater/tree/master/src/org/pikater/shared/database/jpa/JPABatch.java) - celý experiment vytvorené na webovom rozhraní je zakódovaný do XML reťazce, čo je potom spracovaný jadrom. Tento reťazec tvorí obsah jednej premennej v tejto entite. Ďalej entita obsahuje premenná na monitorovanie stavu experiment, dáta o užívatelovi, kto vytvoril daný experiment, časové záznamy o vytvorení, spustení a ukončení experimentu.
 
10. [JPAExperiment](http://github.com/krajj7/pikater/tree/master/src/org/pikater/shared/database/jpa/JPAExperiment.java) - experiment pri spracovávaní rozpadne na niekolko menších časti (podexperimenty). Tieto menšie časti sú spracované nezávisle a na monitorovanie časových údajov štartu a ukončenia týchto častí táto entita obsahuje premenná, podobne ako `JPABatch`. Entita má pripojené zoznam štatistik [JPAResult](http://github.com/krajj7/pikater/src/org/pikater/shared/database/jpa/JPAResult.java), ako prebiehal experiment. U jednoduchších podexperimentov tento zoznam obsahuje obvykle iba jednu položku.


11. [JPAModel](http://github.com/krajj7/pikater/tree/master/src/org/pikater/shared/database/jpa/JPAModel.java) - táto entita slúži na uloženie natrénovaných modelov experimentu. Na jednoznačnú identifikáciu modelu slúži meno agenta, čo je skrývané v modeli. Staršie modely sú vymazané zo systému. Aby užívatel mohol vybrať, že ktoré modely chce zachovať natrvalo, entita obsahuje premennú na označenie.


12. [JPAResult](http://github.com/krajj7/pikater/tree/master/src/org/pikater/shared/database/jpa/JPAResult.java) - táto je entita štatistiky menších částí experimentov (podexperimentov). Obsahuje premenná, kde sú uložené hodnoty, ako názov vytvárajúceho agent, rôzne štatistiky (kappa, rôzne chyby, pomer chýb), nastavenia experimentu, časové údaje štartu a ukončenia (kvôli tomu, že podexperimenty môžu mať pripojené viac výsledkových štatistík), vytvorený model (nemusí každý podexperiment vytvoriť nový model)


13. [JPAExternalAgent](http://github.com/krajj7/pikater/tree/master/src/org/pikater/shared/database/jpa/JPAExternalAgent.java) - entita popíše agent, ktorý bol pridaný do Pikateru užívatelom. Agent je uložený ako bytové pole v tejto entite. Spolu s nim sú uložené aj dáta o vlastníkovi, názvu, názvu triedy agentu a o povolení agentu v systému.


Naša nadstavba Pikateru umožní prácu s viacerými užívatelmi. Aby sme to mohli umožniť, potrebovali sme zaviesť entity, ktoré slúžia na správu užívatelov:


14. [JPAUser](http://github.com/krajj7/pikater/tree/master/src/org/pikater/shared/database/jpa/JPAUser.java) - táto je entita užívatela. V entitách, kde sme potrebovali uložiť informáciu o vlastníkovani, je referencovaná inštancia tejto entity. Entita obsahuje sám v sebe obsahuje login, zahashované heslo, e-mailovú adresu, maximálnu prioritu, čas vytvorenia a čas posledného prihlásenia. Status užívatela môže byť aktivný a suspendovaný.


15. [JPAUserPriviledge](http://github.com/krajj7/pikater/tree/master/src/org/pikater/shared/database/jpa/JPAUserPriviledge.java) - táto entita slúži na uloženie rôznych privilégií v systému. Síce privilégia sú pevne dané v enumu [org.pikater.shared.database.jpa.security.PikaterPriviledge](http://github.com/krajj7/pikater/tree/master/src/org/pikater/shared/database/jpa/security/PikaterPriviledge.java), ale v budúcnosti môže sa nám chodiť, že privilégie sú uložené aj v databázi.


16. [JPARole](http://github.com/krajj7/pikater/tree/master/src/org/pikater/shared/database/jpa/JPARole.java) - rôzne podmnožiny privilégií sú spojené do rôznych rolí,ktoré sú potom priradené k jednotlivým užívatelomi.


## JPA-špecifická anotácia
Pomocou anotovania môžeme zmeniť spôsob, ako JPA spravuje jednotlivé objekty.
Každá entita má zadefinovaná, že do ktorej tabulky bude uložený jej obsah. K tomu slúži značka `@Table(name=”<názov tabulky>”)` . Táto značka nemusí byť uvedená, v tom prípade názov tabulky je rovnaký ako meno triedy. Tieto tabulky sú vytvorené automaticky, na základe konfigurácie JPA.


Okrem toho vela dotazov je vyriešený pomocou pomenovaných dotazov, ktoré sú zapísané v entitách pomocou značiek `@NameQueries` a `@NamedQuery`.


U premenných vo väčšine prípadov nemusíme nič riešiť, lebo vyhovuje nám štandardný prístup JPA. Napríklad primitívne typy sa mapujú bez problémov na primitívne typy databáze.
Na druhej strane občas potrebujeme používať nasledujúce značky:


* `@Id` a `@GeneratedValue(strategy = GenerationType.AUTO)` - používame u premennej, čo chceme používať ako primárny klúč danej entity. Súčasne používame v entite `JPAAbstractEntity` a pomocou dedičnosti máme vyriešený primárny klúč pre všetky entity.
* `@Transient` - takto označené premenné nie sú uložené do databáze. V súčasnej verzii používame pre premennú obsahujúce meno entity.
* `@Inheritance(strategy=InheritanceType.TABLE_PER_CLASS` alebo `InheritanceType.JOINED`) - značka slúží na zadefinovanie, že ako chceme vytvoriť tabulky relácií pre jednotlivé entity. Nastavenie `TABLE_PER_CLASS` vytvorí vlastnú tabulku so všetkými hodnotami, ktoré v danej entite vyskytujú (vlastné aj zdedené). Nastavenie JOINED vytvorí iba jednu tabulu, pre danú entitu a všetky z nej odvodené. Znamená to, že niektoré položky sú prázdne, ale v niektorých prípadoch je to viac prehladný.
* `@Lob` - síce uloženie velkých súborov máme vyriešené pomocou PgLOBov, ale menšie môžeme uložiť aj pomocou JPA anotácie. V tomto prípade premenná sa serializuje a uloží sa do bytového pola - alebo iného kompatibilného typu - v databázovom záznamu. Tento prístup využívame napr. u uložení modelov výpočtov, alebo u dlhých reťazcov XMLov.
* `@Temporal(TemporalType.TIMESTAMP)` - časové údaje označime pomocou tejto značky a definujeme, že v akom formátu chcem ich uložiť
* `@Nullable` - daná premenná može byť null v databáze
* `@Enumerated(EnumType.STRING)` - u premenných odvodených od java.lang.Enum môžeme zadefinovať, či chceme aby sa do databáze uložil názov hodnoty, alebo jeho poradové číslo (`EnumType.ORDINAL`). Uložiť Enum ako String je výhodnejšie z tohto dôvodu, že pri pridaní novej položky nemusíme brať do úvahy poradie hodnôt. Vymazanie nejakej položky zo zdrojového kódu môže spôsobiť výnimku pri získaní hodnôt z databáze. Na druhej strane pri uložení ako `EnumType.ORDINAL` nemusíme získať výnimku, čo môže sposobiť zákerné chyby, pretože hodnoty enumov sú zle namapované.
* `@OneToMany(cascade=CascadeType.PERSIST)` - anotácia zoznamov, nastavenie `CascadeType.PERSIST` umožní, aby prvky zoznamu nemuseli byť zvlášť persistované v databáze. Pri uložení entity automaticky sa uložia všetky prvky v zoznamu.


# Konfigurácia JPA


# Práca JPA entitami
## DAO objekty
## Uloženie entity
## Dotazovanie entit
### JPQL dotazy
### Pomenované dotazy
### Criteria dotazy
## Vymazanie entít




# Použité skratky


* _ARFF_ - Attribute-Relation File Format
* _CSV_ - Comma-Separated Values
* _JDBC_ - Java Database Connectivity
* _SQL_ - Strucutered Query Language
* _HSQLDB_ - Hyper SQL Database
* _PgLOB_ - Postgre Large Object
* _JPA_ - Java Persistence API