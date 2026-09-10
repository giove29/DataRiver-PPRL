# PRIMAT — Documentazione sintetica dei moduli Data Owner e Linkage Unit

> Basata sull'analisi diretta del codice sorgente in `src/main` (i pacchetti `src/test` non sono stati considerati). Package base:
> - Data Owner: `de.uni_leipzig.dbs.pprl.primat.dataowner`
> - Linkage Unit: `de.uni_leipzig.dbs.pprl.primat.lu`

---

# 1. MODULO DATA OWNER (`primat-data-owner`)

Responsabile di due fasi: **pre-processing** dei dati in chiaro e **codifica privacy-preserving** (encoding) dei record prima dell'invio alla Linkage Unit.

## 1.1 Preprocessing (`preprocessing`)

Interfaccia contrattuale:
- **`Preprocessor`** — `preprocess(Record)`, `preprocess(List<Record>)`, `updateSchema()`. Ogni fase di pulizia dati implementa questo contratto.

Implementazioni concrete:
| Classe | Funzione |
|---|---|
| `FieldSplitter` | Divide un attributo in più campi secondo una `SplitDefinition` (es. nome completo → nome + cognome) |
| `FieldMerger` | Unisce più campi in un unico attributo secondo una `MergeDefinition` |
| `FieldNormalizer` | Applica una catena di normalizzazione a un campo secondo una `NormalizeDefinition` |
| `FieldPruner` | Rimuove attributi non necessari secondo una `PruneDefinition` |
| `PartySupplier` | Assegna/etichetta i record con la parte (party/data owner) di appartenenza |

Classi di configurazione (definiscono *come* applicare le trasformazioni sopra): `SplitDefinition`, `MergeDefinition`, `NormalizeDefinition`, `PruneDefinition`, `SimpleNormalizeDefinition`.

### 1.1.1 Splitting (`preprocessing/splitting`)
- **`Splitter`** (interfaccia): `split(String)`, `parts()`.
- Implementazioni: `RegexSplitter` (base per pattern regex custom), `BlankSplitter` (spazi), `CommaSplitter`, `DotSplitter`, `HyphenSplitter`, `PunctuationSplitter`, `PositionSplitter` (split per posizione/indice fisso).

### 1.1.2 Merging (`preprocessing/merging`)
- **`Merger`** (interfaccia): `merge(String[])`.
- Implementazioni: `SimpleMerger` (base), `BlankMerger` (concatena con spazio), `DotMerger` (con punto), `SlashMerger` (con slash).

### 1.1.3 Normalizing (`preprocessing/normalizing`) — 26 classi
- **`Normalizer`** (interfaccia): `normalize(String)`.
- `NormalizerChain`: compone più normalizzatori in sequenza ordinata.
- `StandardStringNormalizer` / `StandardNumberNormalizer`: catene predefinite rispettivamente per stringhe generiche e per campi numerici.
- `RegexReplacer` (base per sostituzioni regex) → `AccentRemover` (rimuove diacritici), `DigitRemover`, `NonDigitRemover`, `PunctuationRemover`, `SpecialCharacterRemover`, `WhitespaceRemover`.
- `CharacterReplacer` (base sostituzione carattere↔numero) → `LetterLowerCaseToNumberNormalizer`, `LetterUpperCaseToNumberNormalizer`, `NumberToLetterLowerCaseNormalizer`, `NumberToLetterUpperCaseNormalizer`, `UmlautNormalizer` (sostituisce le Umlaut tedesche con equivalenti).
- `StringMapper` (base mapping stringa→stringa) → `GenderNormalizer` (uniforma le rappresentazioni di genere), `NullRemover` (rimuove valori nulli/vuoti).
- Altri: `LowerCaseNormalizer`, `UpperCaseNormalizer`, `TrimNormalizer` (rimuove spazi iniziali/finali), `SubstringNormalizer` (estrae una sottostringa), `StringReplacer` (sostituzione diretta stringa→stringa).

## 1.2 Encoding (`encoding`)

Interfaccia contrattuale:
- **`Encoder`** — `encode(Record)`, `encode(List<Record>)` (default, itera sul singolo), `getSchema()`.

Due tecniche di encoding implementate:

### 1.2.1 Bloom Filter (`encoding/bloomfilter`)
- `BloomFilterEncoder` (implementa `Encoder`): codifica gli attributi sensibili in Bloom filter, mappando le feature estratte (es. n-grammi) tramite hashing su una struttura a bit.
- `BloomFilter`: struttura dati bit-array a dimensione fissa.
- `CountingBloomFilter`: variante con contatori per posizione anziché singolo bit (permette rimozione/conteggio occorrenze).
- `BloomFilterDefinition` / `BloomFilterExtractorDefinition`: parametri di configurazione (dimensione, numero funzioni hash, schema di estrazione feature).

**Hashing (`encoding/bloomfilter/hashing`)** — determina come le feature vengono mappate in posizioni del Bloom filter:
- `HashingMethod` (classe base astratta): `hash(...)`, `hashToInts(...)`, gestione salt e mapping feature→posizioni.
- `StandardHashing`, `DoubleHashing`, `TripleHashing`, `EnhancedDoubleHashing`, `RandomHashing`.

**Hardening (`encoding/bloomfilter/hardening`)** — tecniche applicate dopo la creazione del Bloom filter per aumentarne la protezione da attacchi crittoanalitici:
- `BloomFilterHardener` (interfaccia): `hardenBloomFilter(BloomFilter)`.
- `NoHardener` (nessun hardening, baseline), `Balancer` (bilanciamento bit 0/1, tecnica di Schnell et al.), `BitFlipper`, `RandomNoise`, `RandomizedResponse` (basata su RAPPOR di Google), `ReHashing` (finestra scorrevole sul filtro), `ReSamplingXor`, `WindowXor` (entrambe da Ranbaduge & Schnell, CIKM 2020), `Rule30`, `Rule90` (automi cellulari), `XorFolder` (XOR-folding, tecnica mutuata dalla chemo-informatica per comprimere/proteggere il filtro).

### 1.2.2 Two-Step Hash (`encoding/two_step_hash`)
- `TwoStepHashEncoder` (implementa `Encoder`) con `TwoStepHashDefinition`: schema di encoding alternativo al Bloom filter, basato su doppio hashing.

---

# 2. MODULO LINKAGE UNIT (`primat-linkage-unit`)

Riceve i dati codificati da più data owner ed esegue l'intero processo di linkage: blocking, calcolo similarità, classificazione, post-processing e valutazione qualità.

## 2.1 Blocking (`blocking`)

- **`Blocker`** (interfaccia): `getBlocks(Map<Party, Collection<Record>>)` → riduce lo spazio di confronto raggruppando i record in blocchi.
- `Block` / `SubBlock`: un blocco è l'insieme di record che condividono una chiave di blocking; `SubBlock` isola il sottoinsieme relativo a una singola parte.
- `BlockSpace`: collezione di tutti i blocchi generati.
- `NoBlocker`: nessun blocking (confronto completo, baseline).
- `TimedBlocker`: wrapper che misura il tempo di esecuzione di un blocker.
- **`standard/StandardBlocker`**: blocking classico basato su lista di chiavi di blocking esatte.
- **`lsh/LshBlocker`** (estende `StandardBlocker`): blocking basato su Locality-Sensitive Hashing per ridurre il numero di confronti su grandi volumi.

## 2.2 Similarity Function (`similarity_function`)

Funzioni di similarità elementari tra singoli valori:
- **`SimilarityFunction<T>`** (interfaccia): `calculateSimilarity(T, T)`.
- `ExactSimilarity`: uguaglianza esatta (0/1).
- **`distance_function`**: `DistanceFunction<T>` (interfaccia, `computeDistance`) con `BinaryDistanceFunction` per `BitSet`.

**Stringhe (`similarity_function/string`)** — `StringSimilarityFunction` (base) →
`LevenshteinSimilarity`, `JaroWinklerSimilarity`, `QGramSimilarity`, `StringJaccardSimilarity`, `ContainmentStringSimilarity`, `PrefixStringSimilarity`, `SuffixStringSimilarity`.

**Insiemi (`similarity_function/set`)** — `SetSimilarityFunction<T>` (base) →
`JaccardSimilarity`, `DiceSimilarity`, `OverlapSimilarity`, `BraunBlanquetSimilarity`.

**Binarie/BitSet (`similarity_function/binary`)** — `BinarySimilarity` (enum) e `BitMaskSimilarityFunction` per confronti tra Bloom filter/BitSet mascherati.

**Array di interi (`similarity_function/integer_array`)** — `IntegerArraySimilarityFunction` (base) con due varianti: `IntegerArraySimilarityVariant1`, `IntegerArraySimilarityVariant2`.

**XOR-BitSet (`similarity_function/xor_bitset`)** — `XorBitSetSimilarityFunction` (base) → `XorFoldingSimilarity`, `TightXorFoldingSimilarity` (usate per confrontare Bloom filter con hardening XOR-fold applicato lato Data Owner).

## 2.3 Similarity Calculation (`similarity_calculation`)

Livello che applica le similarity function sopra a livello di **attributo**, **record** e **cluster**:

- **`attribute_similarity`**: `AttributeSimilarityCalculator<T,V>` (calcola similarità su un attributo tipizzato) con implementazioni `StringAttributeSimilarityCalculator`, `BitSetAttributeSimilarityCalculator`, `IntegerSetAttributeSimilarityCalculator`, `XorBitSetAttributeSimilarityCalculator`; pattern Visitor via `AttributeSimilarityCalculatorVisitor` / `BaseAttributeSimilarityCalculatorVisitor`.
- **`record_similarity`**: `RecordSimilarityCalculator` (interfaccia, `calculateSimilarity(Record, Record)` → `SimilarityVector`) con `BaseRecordSimilarityCalculator` come implementazione che aggrega le similarità di tutti gli attributi di un record.
- **`record_cluster_similarity`**: `RecordClusterSimilarityCalculator` (interfaccia) per confrontare un record con un intero cluster di record già collegati; due strategie: `PhysicalClusterRepresentantSimilarityCalculator` (usa un record fisico rappresentante) e `VirtualClusterRepresentantSimilarityCalculator` (usa un rappresentante virtuale/aggregato) — utili per il matching incrementale.

## 2.4 Similarity Vector (`similarity_vector`)

- `SimilarityVector`: struttura che raccoglie i punteggi di similarità per ciascun attributo confrontato tra due record.
- `SimilarityVectorFlattener` (interfaccia, `flatten(...)`) con `BaseSimilarityVectorFlattener`: riduce il vettore a una rappresentazione piatta (`FlatSimilarityVector`).
- `FlatSimilarityVectorAggregator` (interfaccia, `aggregate(...) → Double`) con `BaseSimilarityVectorAggregator`: aggrega il vettore piatto in un unico punteggio scalare complessivo (es. media pesata).

## 2.5 Similarity Classification (`similarity_classification`)

- **`SimilarityClassification`** (interfaccia): `classifyRecords(Set<Party>, Collection<Block>) → LinkageResult<Record>` — orchestra blocchi → calcolo similarità → classificazione per l'intero dataset.
- `BatchSimilarityClassification`: implementazione batch (non incrementale).
- `ComparisonStrategy` (enum): strategie di confronto tra parti (es. tutte le coppie, solo tra parti diverse).
- `RedundancyCheckStrategy` (enum): come evitare confronti ridondanti tra blocchi/parti.

## 2.6 Classification (`classification`)

- **`Classificator`** (interfaccia): `classify(SimilarityVector) → MatchStatus`.
- `ThresholdClassificator`: classificazione a soglia (match/non-match/possibile-match in base a un valore soglia sul punteggio aggregato).
- `MatchStatus` (enum): esiti possibili della classificazione.

## 2.7 Matching (`matching`)

- **`Matcher<T>`** (interfaccia): `match(Map<Party, Collection<Record>>) → LinkageResult<T>`.
- **`batch/BatchMatcher`**: esecuzione completa su tutto il dataset in un'unica passata.
- **`incremental`**: `IncrementalMatcher<T>` (astratta, aggiunge `getMatches(Map<String, Record> newRecords)`) con due varianti — `RecordBasedIncrementalMatcher` (confronta nuovi record con record esistenti) e `ClusterBasedIncrementalMatcher` (confronta nuovi record con cluster già consolidati).

## 2.8 Linkage Result (`linkage_result`)

- `LinkageResult<T>`: contenitore del risultato di linkage (coppie collegate/non collegate).
- `LinkedPair<T>`: coppia di record collegati.
- `LinkageResultPartition` / `LinkageResultPartitionFactory`: partizionamento del risultato (utile per elaborazione parallela/distribuita).
- `IncrementalLinkageResult`: variante del risultato per il matching incrementale.

**Strategie di gestione dei match (`linkage_result/matches`)**:
- `MatchStrategy<T>` (interfaccia): `add(...)`, `isContained(...)`, `getMatches()`, `accept(visitor)`.
- `SimilarityGraphMatchStrategy` (+ relativa `Factory`): rappresenta i match come grafo di similarità (si integra con il modello a grafo in `model`); `SimilarityGraphVisitor` implementa il pattern Visitor su tale grafo.

**Strategie per i non-match (`linkage_result/non_matches`)**:
- `NonMatchStrategy<T>` (interfaccia, stessa forma di `MatchStrategy`).
- `CollectNonMatchStrategy` (+ `Factory`): colleziona esplicitamente i non-match (utile per valutazione qualità).
- `IgnoreNonMatchStrategy` (+ `Factory`): scarta i non-match (risparmio di memoria quando non servono).

## 2.9 Model (`model`) — rappresentazione a grafo

- `SimilarityGraph` (Serializable): grafo multipartito dei record con archi pesati dalla similarità (basato su JGraphT).
- `MultiPartiteSimilarityGraph`: variante esplicitamente multipartita (una partizione per data owner/parte).
- `RecordCluster` / `RecordClusterSpace`: rappresentazione dei cluster di record risultanti dal linkage (per il caso multi-party / deduplica).
- Scoring di nodi e archi (usati da algoritmi di clustering/post-processing basati su grafo):
  - `EdgeScoringAlgorithm` (interfaccia) → `EdgeGradeScoring`, `EdgeTypeScoring` (classifica gli archi per tipo, enum `EdgeType`).
  - `VertexScoringAlgorithm` → `VertexDegreeScoring` (grado del nodo), `VertexSelectivityScoring`, `VertexStrictSelectivityScoring` (selettività del nodo, utile a stimare l'affidabilità dei match).

## 2.10 Postprocessing (`postprocessing`)

Applica vincoli strutturali al risultato grezzo del matching (tipicamente 1:1) per aumentare la precisione:

- **`Postprocessor<T>`** (interfaccia): `clean(MatchStrategy<T>) → MatchStrategy<T>`.
- `NoPostprocessor`: nessun post-processing (baseline).
- `TimedPostprocessor`: wrapper per misurare il tempo di esecuzione.
- `PostprocessingStrategy` / `LinkStrength` (enum): parametri di configurazione e forza del vincolo di link.
- `JGraphTPostprocessor<T>` (base per gli algoritmi su grafo, sfrutta la libreria JGraphT) con tre varianti di matching su grafo:
  - `GreedyMaximumCardinalityPostprocessor` — matching greedy a cardinalità massimale.
  - `GreedyWeightedPostprocessor` — greedy pesato (privilegia le similarità più alte).
  - `HopcroftKarpMaximumCardinalityPostprocessor` — cardinalità massima esatta (algoritmo di Hopcroft–Karp).
- `CLIP`: algoritmo di post-processing dedicato (Cluster-based Link Improvement / CLIP, tipico della letteratura PPRL).

**Best match (`postprocessing/best_match`)** — vincolo "miglior match per record":
`MaxOneSidePostprocessor` (base) → `MaxLeftPostprocessor`, `MaxRightPostprocessor`; `MaxBothPostprocessor` (miglior match su entrambi i lati contemporaneamente).

**Hungarian (`postprocessing/hungarian`)** — assegnamento ottimo bipartito:
`HungarianAlgorithm` (implementazione dell'algoritmo ungherese per il problema di assegnamento), `HungarianPostprocessor`, `MaximumWeightMatching`.

**Stable marriage (`postprocessing/stable_marriage`)**:
`GaleShapleyPostprocessor` (algoritmo di Gale-Shapley per matching stabile), `GreedyPostprocessor` (variante greedy più semplice).

**Clustering (`postprocessing/clustering`)** — per il caso multi-party (più di due data owner):
- `ClusteringStrategy` (interfaccia): `cluster(LinkageResult<Record>) → Set<Cluster>`.
- `ClusterBuilder` (interfaccia): `getCluster(...)`, con `SimpleClusterBuilder`.
- `LocalViewClustering` / `DefaultLocalViewClustering`: clustering basato su vista locale del grafo.
- `GlobalViewClustering`: clustering basato su vista globale del grafo.

**Affinity Propagation (`postprocessing/affinity_propagation`)** — clustering avanzato multi-source:
- `AbstractMscdAffinityPropagationSeq` (base), `SparseMscdAffinityPropagationSeq` (implementazione sequenziale sparsa dell'algoritmo Multi-Source Clean-Dirty Affinity Propagation), `AffinityPropagationPostprocessor` (wrapper come `Postprocessor`).
- Strutture dati di supporto (`data_structures`): `ApConfig`, `PreferenceConfig`, `ApEvaluation`, `ApEvaluationStatistics`, `EvaluationStatistics`, `ApType` (AP diretto vs. gerarchico divide-and-conquer), `ApExemplarAssignmentType`.
- `exception/ConvergenceException`: eccezione per mancata convergenza dell'algoritmo.
- Utility (`utils`): `MatrixUtils`, `VectorUtils` (operazioni matriciali/vettoriali), `NoiseUtils` (gestione rumore sui valori di similarità), `ParameterAdaptionUtils` (adattamento parametri durante l'esecuzione), `PreferenceUtils` (gestione del parametro "preference" di AP), `CleanSourceInformationUtils` (gestione informazione "clean source" nella variante MSCD).

## 2.11 Quality Estimation (`quality_estimation`)

Stima (senza ground truth) della qualità del linkage:
- **`QualityEstimate`** (interfaccia base).
- **`precision`**: `PrecisionEstimation` (base) → `PrecisionStructureOneToOne`, `PrecisionStructureOneToN`, `PrecisionOneToOneBase`, `PrecisionProbabilityDeduplicated`, `PrecisionPPBasedTPEstimation` (stima privacy-preserving dei veri positivi); `PersonalizedPageRank` come supporto algoritmico (PageRank personalizzato sul grafo di similarità).
- **`recall`**: `RecallEstimation` (base) → `RecallStructureOneToOne`, `RecallStructureOneToN`, `RecallStructureNToM`, `RecallCryptoSetBased` (base per stime basate su insiemi crittografici) → `RecallCryptoSetOneToOne`, `RecallCryptoSetOneToN`, `RecallCryptoSetProb`; `OverlapEstimation` come supporto per stimare la sovrapposizione tra dataset.

## 2.12 Evaluation (`evaluation`)

Valutazione con ground truth disponibile (usata per benchmark/confronto workflow):
- `QualityMetrics`: calcolo di recall, precision, f-measure.
- `PerformanceMetrics`: metriche di scalabilità (runtime, reduction ratio).
- `QualityEvaluator`: orchestratore che confronta il risultato di linkage con la verità nota.
- `MetricCollector` (enum), `MetricFormat`, `ResultType` (enum): supporto a raccolta/formattazione delle metriche.
- **`true_match_checker`**: `TrueMatchChecker` (interfaccia, `isTrueMatch(Record, Linkable)`) con quattro strategie per riconoscere il vero match dagli identificativi dei dataset di test — `IdEqualityTrueMatchChecker`, `IdMappingTrueMatchChecker`, `IdFirstPartUnderscoreTrueMatchChecker`, `IdMiddlePartHyphenTrueMatchChecker`.

## 2.13 Utils (`utils`)

- `Timed` (interfaccia): contratto per componenti che misurano il proprio tempo di esecuzione (implementato da `TimedBlocker`, `TimedPostprocessor`).
- `GraphSerializer`: serializzazione/deserializzazione dei grafi di similarità.
- `ThresholdClassificationRefinement` (interfaccia, `refine(LinkageResult<Record>)`) con `NoThresholdRefinement` (nessuna rifinitura) e `StandardThresholdClassificationRefinement` (rifinitura post-classificazione basata su soglia).

## 2.14 Altri componenti
- `database/DbConnection` (enum): configurazioni di connessione al database usato per persistere dati/risultati.

---

## Nota metodologica
Questa documentazione è stata generata analizzando direttamente le classi in `src/main/java` dei due moduli (non `src/test`), raggruppandole per package funzionale. Le descrizioni derivano dal Javadoc presente nel codice dove disponibile, altrimenti dal nome e dalla posizione della classe nella gerarchia (interfaccia implementata). Non include quindi campi interni, metodi privati o dettagli implementativi minori.
