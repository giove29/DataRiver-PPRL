# PRIMAT — Documentazione sintetica dei moduli Common e Analysis

> Basata sull'analisi diretta del codice sorgente in `src/main` (i pacchetti `src/test` non sono stati considerati). Package base:
> - Common: `de.uni_leipzig.dbs.pprl.primat.common`
> - Analysis: `de.uni_leipzig.dbs.pprl.primat.analysis`

---

# 3. MODULO COMMON (`primat-common`)

È il modulo di base condiviso da tutti gli altri (Data Owner, Linkage Unit, Analysis): definisce il **modello dati** (record, attributi, party), le funzioni di **estrazione feature** (usate sia per l'encoding lato Data Owner sia per il blocking lato Linkage Unit), l'**I/O su CSV** e un ampio insieme di **utility** condivise (hashing, crittografia leggera, aggregazione, random).

## 3.1 Model (`model`)

Il cuore del modulo: rappresenta record, attributi e organizzazione in "party" (data owner).

- **`Record`** (implementa `CSVPrintable`, `Linkable`, `Serializable`): entità centrale che raccoglie un insieme di `Attribute`.
- **`Linkable`** (interfaccia, estende `Comparable<Linkable>`): `accept(LinkableVisitor)`, `getUniqueIdentifier()`, `getGlobalIdentifier()` — contratto comune tra `Record` e `Cluster`, che permette di trattarli in modo uniforme nel processo di linkage.
- **`LinkableVisitor`** (interfaccia, pattern Visitor): `visit(Record)`, `visit(Cluster)`.
- **`Party`** (Comparable): rappresenta un data owner/parte partecipante al linkage; gestisce l'insieme di record (`addRecord`, `removeRecord`, `getRecords`) e il flag `isDuplicateFree` (se il dataset della parte è già deduplicato).
- **`PartyPair`**: coppia di due `Party` da confrontare.
- **`IntegratedSource`** (estende `Party`): rappresenta una sorgente dati già integrata/consolidata.
- **`Cluster`** (implementa `Linkable`): gruppo di record che il processo di linkage ha identificato come riferiti alla stessa entità reale.
- **`ClusterFactory`**: crea istanze di `Cluster`.
- **`ClusterBlockingKeyStrategy`** (enum) / **`ClusterRepresentantStrategy`** (enum): strategie per assegnare chiavi di blocking e per scegliere il record rappresentante di un cluster.
- **`LinkageConstraint`** (enum): vincoli strutturali applicabili al risultato del linkage (es. 1:1, 1:N).
- **`RecordSchema`** (enum singleton) + **`RecordSchemaConfiguration`** (base) → `NamedRecordSchemaConfiguration`, `UnnamedRecordSchemaConfiguration`: definiscono lo schema dei record (nomi/posizioni colonna); `GenericRecordSchemaBuilder` costruisce lo schema a runtime; `RecordSchemaConfigurationVisitor` applica il pattern Visitor sulle due varianti di configurazione.
- **`CountingBloomFilter`**: variante del Bloom filter con contatori per posizione (duplicata rispetto a quella in Data Owner, qui usata come struttura dati condivisa).

### 3.1.1 Attributes (`model/attributes`) — 30 classi

Gerarchia tipizzata degli attributi di un record:
- **`Attribute<T>`** (base astratta): `getValue()`, `setValue(T)`, `getStringValue()`, `setValueFromString(String)`, `accept(AttributeVisitor)`, `isNull()`, `getType()`.
- **`AttributeVisitor`** (interfaccia): dispatcher per `IdAttribute`, `GlobalIdAttribute`, `PartyAttribute`, `QidAttribute<?>`, `BlockingKeyAttribute`.
- **`AttributeType`** (interfaccia) con enum concreti `PersonalAttributeType`, `QidAttributeType`, `NonQidAttributeType`: classificano il "tipo semantico" di un attributo.
- Attributi non-QID (identificativi, non usati per il matching): `IdAttribute`, `GlobalIdAttribute`, `PartyAttribute`, `BlockingKeyAttribute` (+ `BlockingKeyId`).
- **`QidAttribute<T>`** (Quasi-IDentifier, base per gli attributi usati nel confronto/matching): `getId()`, `getRecord()`, `accept(QidAttributeVisitor)`.
  - **`QidAttributeVisitor`** (interfaccia): dispatcher per tutte le varianti concrete.
  - Varianti concrete: `StringAttribute`, `NumericAttribute`, `DateAttribute`, `IntegerSetAttribute`, `BitSetAttribute` (per Bloom filter), `XorBitSetAttribute` (per Bloom filter con hardening XOR-folding, con relativa struttura dati `XorBitSet`).
  - Per ogni variante esiste un "Retriever"/"Collector" dedicato che implementa `QidAttributeVisitor` per estrarre o raccogliere il valore tipizzato: `StringAttributeRetriever`, `NumericAttributeRetriever`, `DateAttributeRetriever`, `IntegerSetAttributeRetriever` (+ `Collector`), `BitSetAttributeRetriever` (+ `Collector`), `XorBitSetAttributeRetriever`.
- `AttributeParseException` (RuntimeException): errore di parsing di un attributo da sorgente esterna.
- `ColumnNameHandler` (implementa `RecordSchemaConfigurationVisitor`): risolve i nomi di colonna in base alla configurazione di schema.

## 3.2 Extraction (`extraction`)

Definisce **come** vengono estratte le feature (token) dagli attributi, passo preliminare sia all'encoding (Bloom filter) sia al blocking basato su LSH.

- **`FeatureExtractor`** (interfaccia): `extract(QidAttribute<?>) → List<String>`.
- `ExtractorDefinition`: configurazione che specifica quale extractor applicare a un attributo.
- `FeatureExtraction`: esegue il processo di estrazione applicando le regole di `ExtractorDefinition`.
- `FeatureExtractorChain` (implementa `FeatureExtractor`): compone più extractor in sequenza.
- `IdentityExtractor`: funzione identità (nessuna estrazione, valore invariato).
- `IntegerRangeExtractor`: estrae feature come intervalli numerici.
- `MinHashExtractor`: estrae signature MinHash (per stima di similarità su insiemi di grandi dimensioni).
- `SubstringByPositionExtractor` / `SubstringByRegexExtractor`: estraggono sottostringhe per posizione fissa o tramite pattern regex.

### 3.2.1 Q-gram (`extraction/qgram`)
- `QGramExtractor` (base): estrae l'insieme di q-grammi di un valore.
- `UnigramExtractor`, `BigramExtractor`, `TrigramExtractor`: casi specifici per q=1,2,3.
- `SpecialTrigramExtractor`: variante che considera solo il primo trigramma estratto.

### 3.2.2 Phonetic (`extraction/phonetic`)
- `PhoneticCodeExtractor` (enum, implementa `FeatureExtractor`): algoritmi di codifica fonetica (trasformano le parole in rappresentazione basata sulla pronuncia) per estrarre feature robuste a errori di trascrizione.

### 3.2.3 LSH (`extraction/lsh`)
Locality-Sensitive Hashing, usato per generare chiavi di blocking privacy-preserving:
- **`LshKeyGenerator`** (base astratta): `generate() → List<LshBlockingFunction>`, con parametri `keySize`, `keys`, `valueRange`, `seed`.
- **`LshBlockingFunction`** (implementa `FeatureExtractor`): `apply(QidAttribute<?>)`, `apply(BitSet)` — funzione LSH applicata come chiave di blocking.
- Per la distanza di Hamming (su BitSet/Bloom filter): `HammingLshKeyGenerator` (base) → `RandomHammingLshKeyGenerator`, `RestrictedHammingLshKeyGenerator`; `HammingLshBlockingFunction`.
- Per la similarità di Jaccard (su insiemi): `JaccardLshKeyGenerator`, `JaccardLshBlockingFunction`.

## 3.3 CSV (`csv`)

I/O per l'importazione/esportazione dei dati in formato CSV:
- **`CSVPrintable`** (interfaccia): `getValuesToPrint()` — contratto per le classi esportabili come riga CSV (implementato da `Record`).
- `CSVReader`: legge e parsifica file CSV in `Record`.
- `CSVWriter`: scrive/esporta oggetti come righe CSV.
- `CSVRecordWrapper`: converte/incapsula un record CSV grezzo.
- `AttributeHandler` (implementa `AttributeVisitor`): gestisce la conversione dei diversi tipi di attributo durante il parsing da sorgente esterna.

## 3.4 Blocking (base condivisa) (`blocking`)

Versione "core" (riutilizzata dalla Linkage Unit) per l'assegnazione delle chiavi di blocking:
- `Blocker`: `addBlockingKeyDefinition(...)`, `addBlockingKeys(Record)`, `addBlockingKeys(List<Record>)` — assegna le chiavi di blocking ai record.
- `BlockingKeyDefinition`: definizione di una chiave di blocking (quale attributo/estrattore usare).

## 3.5 Utils (`utils`) — 36 classi

Libreria di utility trasversali usate da tutti i moduli:

**Hashing e crittografia**
- `HashUtils`: calcolo di valori hash.
- `HashFunctionGenerator`: generazione di famiglie di funzioni hash (usate nel Bloom filter encoding).
- `HMacAlgorithm` (enum): algoritmi HMAC supportati.
- `MessageDigestAlgorithm` (enum): algoritmi di digest sicuro (SHA, ecc.).
- `SecretDerivation`: derivazione di segreti/chiavi.
- `KeyManager`: gestione delle chiavi crittografiche/salt.

**Randomizzazione**
- `RNG` (enum) / `PRNG` (enum) / `RandomFactory` (enum): factory per generatori di numeri casuali (deterministici o crittograficamente sicuri).
- `RandomUtils`: funzioni di utilità basate su randomizzazione.

**Strutture dati bit-level**
- `BitSetUtils`: operazioni comuni su `BitSet`.
- `BitMatrix`: matrice di bit.
- `BitPairCounts` / `BigBitPairCounts`: conteggio delle coppie di bit (00/01/10/11) tra due BitSet, usato per calcolare similarità basate su Bloom filter.
- `ByteUtils`: conversioni/operazioni su byte.

**Aggregazione**
- `ListAggregator<T>` (interfaccia): `aggregate(List<T>) → T`.
  - `DoubleListAggregator` (enum), `StringListAggregator` (enum): implementazioni predefinite per liste di double/stringhe.
  - `BitSetListAggregator` (interfaccia specializzata) → `WeightedRandomSamplingBitSetAggregator`.
  - `WeightedSumAggregator`: somma pesata di valori double.
  - `WeightingFunction`: definisce funzioni di ponderazione da usare negli aggregatori.

**Collezioni e conversioni**
- `ArrayUtils`, `ListUtils`, `MapUtils`, `SetUtils`, `StringUtils`: utility generiche su array/liste/mappe/insiemi/stringhe.
- `CombinatoricsUtils`: funzioni combinatorie (es. calcolo coppie/combinazioni).
- `ClassNameObjectConverter`: crea dinamicamente oggetti a partire dal nome della classe e dai parametri (usato per istanziare a runtime le strategie configurate, es. normalizer/hardener/similarity function scelti da configurazione).
- `StringToObjectConverter`: converte stringhe nei tipi dato standard (int, float, double, ecc.).
- `Pair<A,B>` / `HomogenPair<T>` (Pair con stesso tipo su entrambi i lati): strutture dati coppia generiche.
- `RecordUtils`: funzioni di utilità specifiche sui `Record`.
- `DatasetReader`: lettura di dataset da sorgente.
- `ANSICode` (enum): codici ANSI per output colorato su console.
- `StreamingMode` (enum): modalità di elaborazione streaming vs batch.

---

# 4. MODULO ANALYSIS (`primat-analysis`)

Fornisce strumenti di **analisi esplorativa** di dataset, attributi, cluster/record e differenze testuali — utile in fase di preparazione/valutazione di un workflow PPRL (indipendente dal processo di linkage vero e proprio).

## 4.1 Core (`analysis` root)

- **`Analyzer`** (base): `getResultSet()`, `getName()` — contratto comune a tutti gli analizzatori, che produce un `ResultSet`.
- `DataSetAnalyzer`: analizzatore a livello di intero dataset.
- `DataSetAnalyzerCreator`: factory che istanzia gli analizzatori opportuni.
- `AnalysisResult`: contenitore del risultato prodotto da un `Analyzer`.

## 4.2 Attribute analysis (`attribute`)

Analisi statistica sui valori degli attributi, raggruppati per nome/tipo:
- `AttributeAnalyzer` (estende `Analyzer`): base per le analisi a livello di attributo.
- `AttributeAvailability`: misura la quota di valori validi/non validi (es. mancanti) per ciascun tipo di attributo.
- `AttributeLength`: misura la lunghezza dei valori per ciascun tipo di attributo.
- `AttributeBitPositionFrequency`: misura la frequenza delle posizioni di bit attive negli attributi di tipo BitSet (utile per analizzare la "densità" dei Bloom filter).
- `AttributeFrequencyAnalyzer` (base) → `AttributeMostFrequent` (trova e ordina i valori più frequenti per attributo), `AttributeMostFrequentNGrams` (trova gli n-grammi più frequenti per tipo di attributo), `AttributePatternFrequency` (frequenza dei valori che soddisfano determinati pattern regex).

## 4.3 Cluster analysis (`cluster`)

Analisi dei gruppi di record che si riferiscono alla stessa entità reale (ground truth o risultato di linkage):
- `ClusterAnalyzer` (estende `Analyzer`): base per l'analisi dei cluster.
- `ClusterSize`: analizza la distribuzione delle dimensioni dei cluster.
- `ClusterPairwiseEqual`: conta per quanti attributi le coppie di record in uno stesso cluster differiscono.
- `ClusterPairwiseDiff`: analizza nel dettaglio le differenze tra le coppie di record di uno stesso cluster (utile per capire i tipi di errori/varianti tra record dello stesso individuo).

## 4.4 Record analysis (`record`)

- `RecordAnalyzer` (estende `Analyzer`): base per l'analisi sull'insieme dei record.
- `RecordCounter`: conta il numero di record nel dataset e nei singoli gruppi.
- `RecordOverlap`: conta i record presenti in più gruppi/party (overlap reale).
- `RecordOverlapEstimate`: stima (senza accesso diretto ai dati in chiaro) il numero di record sovrapposti tra party.

## 4.5 Difference (`difference`)

Wrapper/adattamento della libreria "Diff Match and Patch" (Google) per confrontare stringhe a livello di carattere:
- `Diff`, `DiffCalculator`, `DiffMatchPatchUtils`, `Patch`: calcolo e rappresentazione delle differenze testuali tra due stringhe.
- `DiffOperation` (enum): tipo di operazione di diff (inserimento/cancellazione/uguaglianza).
- `CustomDiff` / `CustomDiffOperation`: variante personalizzata del calcolo diff per esigenze specifiche di PRIMAT.

## 4.6 Results (`results`)

- `Result`: singolo risultato prodotto da un analizzatore.
- `ResultSet`: collezione di `Result` restituita da un `Analyzer`.

## 4.7 Utils (`utils`)

- `CommandLineTable`: stampa formattata di tabelle su console.
- `TableSawUtils`: utility di supporto basate sulla libreria Tablesaw (manipolazione dati tabellari), usate presumibilmente per costruire/esportare i `ResultSet` come tabelle.

---

## Nota metodologica
Come per il documento su Data Owner e Linkage Unit, questa documentazione è stata generata analizzando direttamente le classi in `src/main/java` dei due moduli (non `src/test`), raggruppandole per package funzionale, con descrizioni derivate dal Javadoc presente nel codice dove disponibile e altrimenti dal nome/posizione della classe nella gerarchia.
