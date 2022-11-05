# Code plagiarism detector doc
<!-- Mermaid notes 
For nested generics use ‎ character
-->

[Link Github](https://github.com/tassiLuca/code-plagiarism-detector)

- [Code plagiarism detector doc](#code-plagiarism-detector-doc)
  - [Analisi dei requisiti](#analisi-dei-requisiti)
    - [Requisiti funzionali](#requisiti-funzionali)
    - [Requisiti non funzionali](#requisiti-non-funzionali)
    - [Modello del dominio](#modello-del-dominio)
  - [Architettura](#architettura)
  - [Design](#design)
    - [ProjectsProvider](#projectsprovider)
    - [Analyzer](#analyzer)
    - [Configurable Options + AntiPlagirismSession](#configurable-options--antiplagirismsession)

## Analisi dei requisiti
Si vuole realizzare un sistema software in grado di trovare eventuali porzioni di codice copiato nei progetti software del corso di OOP dell'Università di Bologna, sviluppati in linguaggio Java.

### Requisiti funzionali
- Il sistema riceve in input un progetto di cui si vuole verificare l'autenticità;
- Il sistema recupera i progetti su cui verificare eventuali plagi (i vecchi progetti sono mantenuti in repo pubbliche archiviate su GitHub e BitBucket);
- Il sistema deve fornire in output un elenco di sezioni di codice che, con un determinato livello di accuratezza, ha stabilito essere simili (laddove presenti);

### Requisiti non funzionali
- Le informazioni estrapolate dai sorgenti sono salvate in modo tale da essere riutilizzate nelle analisi successive di altri progetti;
- L'algoritmo per determinare le similarità deve essere interscambiabile e facilmente estendibile;
- è necessario che il sistema impieghi un tempo "ragionevole" per effettuare la computazione.

### Modello del dominio
Il sistema deve essere in grado, a partire da un insieme di progetti e sorgenti, di estrarne una rappresentazione confrontabile (_SourceRepresentation_), ad esempio i _token_, mediante opportuni algoritmi di analisi (_Analyzer_) e poter determinare eventuali parti di codice duplicato e/o somiglianze, generando dei report.

La principale difficoltà sarà individuare tecniche di analisi lessicali e di rilevamento delle somiglianze che siano robuste, ovvero permettano d'identificare casi di copiature anche se lo sviluppatore ha effettuato modifiche per nasconderle (come ad esempio cambiare identificatori, nomi, l'ordine dei parametri ecc...)
Inoltre, il requisito non funzionale sulle _performance_ richiederà un'analisi dei tempi di esecuzione quando il sistema sarà completato.

Schema :white_check_mark:: 
```mermaid
classDiagram 
direction TB
    class Report {
        <<interface>>
        + scoreOfSimilarity: Double
    } 
    Report *-- "*" ComparisonResult

    class Repository {
        <<interface>>
        +name: String
        +owner: String
    }
    Report *-- "1" Repository: submittedProject
    Report *-- "1" Repository: comparedProject

    class AntiPlagiarismSession {
        <<interface>>
        +invoke()
    }
    AntiPlagiarismSession "1" -- "*" Report: generates
    AntiPlagiarismSession *-- "1" RunConfiguration

    class RunConfiguration {
        <<interface>>
    }
    RunConfiguration *-- "1" PlagiarismDetector
    RunConfiguration *-- "1" Analyzer
    RunConfiguration *-- "1" Filter
    Repository "*" --* RunConfiguration

    class PlagiarismDetector {
        <<interface>>
    }
    PlagiarismDetector "*" -- "2" SourceRepresentation: input

    class ComparisonResult {
        <<interface>>
    }
    ComparisonResult *-- "2" SourceRepresentation: refers to
    ComparisonResult "*" -- "1" PlagiarismDetector: output

    class Analyzer {
        <<interface>>
    }
    Analyzer "1" -- "1" SourceRepresentation: creates

    class Filter {
        <<interface>>
    }
    Filter "*" -- "*" SourceRepresentation: filters

    class SourceRepresentation {
        <<interface>>
    }

    class SourceFile {
        <<interface>>
        +fileName: String
    }
    Repository *-- "*" SourceFile
    SourceFile "1" -- "1" SourceRepresentation: of
```

## Architettura
Schema :white_check_mark:: 
```mermaid
classDiagram
direction TB
    class AntiPlagiarismSession {
        <<interface>>
        +invoke()
    }
    AntiPlagiarismSession *--> RunConfiguration

    class RunConfigurator {
        <<interface>>
    }
    CLIConfigurator ..|> RunConfigurator

    class RunConfiguration {
        <<inteface>>
    }
    RunConfigurator ..> RunConfiguration: << creates >>

    class RepositoryProvider {
        <<interface>>
    }
    class Repository {
        <<interface>>
    }
    Repository "2..*" <--* RunConfiguration
    RepositoryProvider ..> Repository: << creates >>

    class PlagiarismDetector {
        <<interface>>
    }
    RunConfiguration *--> "1" PlagiarismDetector

    class Analyzer {
        <<interface>>
    }
    RunConfiguration *--> "1" Analyzer

    class Filter {
        <<interface>>
    }
    RunConfiguration *--> "1" Filter

    class KnoledgeBaseRepository {
        <<interface>>
        + save()
        + load()
    }
    KnoledgeBaseRepository "1" <--* AntiPlagiarismSession

    class Output {
        <<interface>>
    }
    class ReportsExporter {
        <<interface>>
        +export(reports: Set~Report~)
    }
    Output <|-- ReportsExporter
    ReportsExporter <|.. FileExporter
    RunConfiguration *--> "1" ReportsExporter
    Output <|.. CLIOutput
    AntiPlagiarismSession *--> "1" Output
```

## Design
### ProjectsProvider
  
Componenti:
- `RepositoryProvider`: un generico _provider_ di repository che consente al cliente di richiedere una repository in base al suo link diretto o attraverso un criterio.
- `AbstractRepositoryProvider` cattura l'implementazione comune dei due concreti `GitHub` e `BitbucketProvider`. 
- `TokenSupplierStrategy` è l'interfaccia a cui viene richiesto di recuperare il token di autenticazione ad un servizio. L'implementazione di default ricerca tra variabili d'ambiente.

Schema :white_check_mark:: 
```mermaid
classDiagram
direction BT
    class RepositoryProvider~in C : SearchCriteria~ {
        <<interface>>
        +byLink(url: URL) Repository
        +byCriteria(criteria: C) Sequence~Repository~
    }
    class AbstractRepositoryProvider~in C: SearchCriteria~ {
        <<abstract>>
        +byLink(url: URL) Repository
        #urlIsValid(urL: URL)* Boolean
        #getRepoByURL(url: URL)* Repository
    }
    AbstractRepositoryProvider ..|> RepositoryProvider

    class GitHubProvider~GitHubSearchCriteria~ {
        -GitHubProvider()
    }
    class BitbucketProvider~BitbucketSearchCriteria~ {
        -BitbucketProvider()
    }
    GitHubProvider --|> AbstractRepositoryProvider
    BitbucketProvider --|> AbstractRepositoryProvider

    class GitHubProviderFactory {
        <<companion object>>
        +connectAnonymously() GitHubProvider
        +connectWithToken(\nㅤㅤtokenSupplier: TokenSupplierStrategy\n) GitHubProvider
    }
    GitHubProviderFactory ..> GitHubProvider: creates

    class BitbucketProviderFactory {
        <<companion object>>
        +connectAnonymously() BitbucketProvider
        +connectWithToken(\nㅤㅤtokenSupplier: TokenSupplierStrategy\n) BitbucketProvider
    }
    BitbucketProviderFactory ..> BitbucketProvider: creates

    class TokenSupplierStrategy {
        <<interface>>
        +token: String
    }
    EnvironmentTokenSupplier ..|> TokenSupplierStrategy
    TokenSupplierStrategy <--o GitHubProviderFactory
    TokenSupplierStrategy <--o BitbucketProviderFactory
```

- L'interfaccia `Repository` espone proprietà e metodi per ottenere il suo nome, il suo _owner_ e i suoi sorgenti in base a un dato linguaggio di programmazione. Anche in questo caso `AbstractRepository` cattura l'implementazione comune di `GitHubRepository` e `BitBucketRepository`. 
- La logica di recupero dei repository remoti e dei relativi file è demandata a un'interfaccia via Strategy a `RepoContentSupplierStrategy`: l'implentazione di default (`RepoContentSupplierCloneStrategy`) clona localmente la repo (altri approcci potrebbero essere seguiti, motivo per il quale si è deciso di scorporare in un'interfaccia a sè stante lo specifico algoritmo di recupero del contenuto di una repo).

Schema :white_check_mark:: 
```mermaid
classDiagram
    direction BT
    class Repository {
        <<interface>>
        +name: String
        +owner: String
        +cloneUrl: URL
        +getSources(pattern: Regex) Sequence~File~
    }
    class AbstractRepository {
        <<abstract>>
        +getSources(pattern: Regex) Sequence~File~
    }
    AbstractRepository ..|> Repository
    class GitHubRepository {
        -repository: Repo
        +GitHubRepository(repository: GHRepository)
    }
    class BitBucketRepository {
        -repoInfos: JSONObject
        +BitBucketRepository(repoInfos: JSONObject)
    }
    GitHubRepository --|> AbstractRepository
    BitBucketRepository --|> AbstractRepository

    class RepoContentSupplierStrategy {
        <<interface>>
        +getFilesOf(extensions: Iterable~String~)
    }
    RepoContentSupplierStrategy <--* AbstractRepository
    CloneStrategy ..|> RepoContentSupplierStrategy
```

- `SearchCriteria` è un'interfaccia che rappresenta un criterio con cui filtrare le repo.
- Per fare in modo che i criteri siano componibili si è usato un _Decorator_: `GitHubCompoundCriteria`. In questo modo possono essere creati dinamicamente criteri compositi in base alle esigenze (ed è estendibile perché potrebbero essere aggiunti altri criteri di ricerca, come il numero di _stars_ o se archiviata o no...)
- [Link Bitbucket > Filter and sort API objects ](https://developer.atlassian.com/cloud/bitbucket/rest/intro/#filtering)
- [Link GitHub > Searching for repositories](https://docs.github.com/en/search-github/searching-on-github/searching-for-repositories)

Schema :white_check_mark:: 
```mermaid
classDiagram
    direction BT
    class SearchCriteria~in I, out O~ {
        <<interface>>
        +apply(subject: I) O
    }

    class GitHubSearchCriteria~GitHub, GHRepositorySearchBuilder~ {
        <<interface>>
        +apply(subject: GitHub) GHRepositorySearchBuilder
    }
    GitHubSearchCriteria --|> SearchCriteria
    class ByGitHubUser {
        +ByGitHubUser(username: String)
    }
    ByGitHubUser ..|> GitHubSearchCriteria
    class GitHubCompoundCriteria {
        <<abstract>>
        +GitHubCompoundCriteria(criteria: GitHubSearchCriteria)
    }
    GitHubCompoundCriteria ..|> GitHubSearchCriteria
    class ByGitHubName {
        +ByGitHubName(repoName: String, criteria: GitHubSearchCriteria)
    }
    ByGitHubName --|> GitHubCompoundCriteria
    class ByGitHubLanguage {
        +ByGitHubName(language: String, criteria: GitHubSearchCriteria)
    }
    ByGitHubLanguage --|> GitHubCompoundCriteria

    class BitbucketSearchCriteria~String, String~ {
        <<interface>>
        +apply(subject: String) String
    }
    BitbucketSearchCriteria --|> SearchCriteria
    class ByBitbucketUser {
        +ByBitbucketUser(username: String)
    }
    ByBitbucketUser ..|> BitbucketSearchCriteria
    class BitbucketCompoundCriteria {
        <<abstract>>
        +BitbucketCompoundCriteria(criteria: BitbucketSearchCriteria)
    }
    BitbucketCompoundCriteria ..|> BitbucketSearchCriteria
    class ByBitbucketName {
        +ByBitbucketName(repoName: String, criteria: BitbucketCompoundCriteria)
    }
    ByBitbucketName --|> BitbucketCompoundCriteria
    class ByBitbucketLanguage {
        +ByBitbucketLanguage(language: String, criteria: ByBitbucketLanguage)
    }
    ByBitbucketLanguage --|> BitbucketCompoundCriteria
```

### Analyzer

<!-- italian version:
Viene qui impiegato il pattern [Pipeline](https://java-design-patterns.com/patterns/pipeline/) per modellare la sequenza di trasformazioni che vengono eseguite per passare dal file sorgente alla sua rappresentazione confrontabile.

Il `TokenizationAnalyzerProxy` è l'oggetto intermediario che si occuperà, nel caso in cui quel file sia già stato tokenizzato e memorizzato in precedenza, di recuperarlo senza ri-effettuare la tokenizzazione.
-->

The [Pipeline](https://java-design-patterns.com/patterns/pipeline/) pattern is used to model the sequence of transformations performed to transform a source file into a comparable representation.

<!--
`TokenizationAnalyzerProxy` is an intermediary object that manages the recovery of previously analyzed and cached files without having to re-analyse them. 
-->

```mermaid 
classDiagram
    direction BT
    class Analyzer~out S: SourceRepresentation<T>~ {
        <<interface>>
        +invoke(input: File) S
    }

    class StepHandler~I, O~ {
        <<interface>>
        +process(input: I) O
    }

    class Parser~File, CompilationUnit~ {
        +process(input: File) CompilationUnit
    }
    Parser ..|> StepHandler

    class Preprocessor~CompilationUnit, CompilationUnit~ {
        +process(input: CompilationUnit) CompilationUnit
    }
    Preprocessor ..|> StepHandler

    class Tokenizer~CompilationUnit, Sequence< Token >~ {
        +process(input: CompilationUnit) Sequence~Token~
    }
    Tokenizer ..|> StepHandler

    class TokenizationAnalyzer~TokenizedSource, Sequence<Token>~ {
        <<abstract>>
        -pipeline: StepHandler
        +execute(input: File) TokenizedSource
    }
    TokenizationAnalyzer ..|> Analyzer

    JavaTokenizationAnalyzer --|> TokenizationAnalyzer
    StepHandler <--* JavaTokenizationAnalyzer
```

`SourceRepresentation` is the interface modeling the intermediate representation, which is generated from the source file, prior to comparison.
Among available representations, the source code token sequence is the most common one: `TokenizedSource` embodies a token-based representation of the source code, which is a sequence of structure-preserving terms found in the code files.

```mermaid
classDiagram
direction BT
    class SourceRepresentation~T~ {
        <<interface>>
        +sourceFile: File
        +representation: T
    }
    class TokenizedSource~Sequence<Token>~ {
        <<interface>>
        +splitInGramsOf(dimension: Int) Sequence~Gram<Token>~
    }
    TokenizedSource --|> SourceRepresentation

    class Token {
        <<interface>>
        +line: Int
        +column: Int
        +type: TokenType
    }
    Token <--* TokenizedSource

    class Gram~T~ {
        <<interface>>
        +items: List~T~
    }
    Token <--* Gram
    Gram -- TokenizedSource

    class TokenType {
        <<interface>>
        +name: String
        +languageConstructs: Set~String~
    }
    TokenType <--* Token

    class LanguageTokenTypes {
        <<interface>>
        +tokenFor(constructName: String) TokenType
        +isToken(constructName: String) Boolean
    }
    TokenType <--* LanguageTokenTypes

    class TokenTypesSupplier {
        <<interface>>
        +types: LanguageTokenTypes
    }
    LanguageTokenTypes <--* TokenTypesSupplier

    class FileTokenTypesSupplier {
        <<inteface>>
        +FileTokenTypesSuppier(configFileName: String)
    }
    TokenTypesSupplier <|.. FileTokenTypesSupplier
```

Filter:

```mermaid
classDiagram
direction BT
    class RepresentationFilter~S: SourceRepresentation<T>, T~ {
        <<interface>>
        +invoke(submission: S, corpus: Sequence~S~) Sequence~S~
    }
    class TokenizedSourceFilter~TokenizedSource, Sequence<Token>~
    RepresentationFilter <|.. TokenizedSourceFilter

    class Indexer~in S: SourceRepresentation<T>, T, I~ {
        +invoke(input: S) I
    }
    class TokenBasedIndexer~TokenizedSource, Sequence<Token>, Map<TokenType, Int>~
    TokenBasedIndexer ..|> Indexer

    TokenizedSourceFilter *--> TokenBasedIndexer
```

<!-- italian version:
Il `PlagiarismDetector` è la strategia (algoritmo) con cui viene calcolata la similarità tra una coppia di artefatti. 
-->
`PlagiarismDetector` is the component that detects similarities between two `SourceRepresentation`.
`ComparisonStrategy` encapsulates the specific algorithm used to detect the similarities.

`Match` represents two sections of `SourceRepresentation` that are similar.

```mermaid 
classDiagram
direction BT
    class PlagiarismDetector~in S: SourceRepresentation<T>, T, out M: Match~ {
        <<interface>>
        +invoke(first: S, second: S) ComparisonResult~M~
    }

    class ComparisonResult~out M: Match~ {
        <<interface>>
        +similarity: Double
        +matches: Sequence~M~
    }
    PlagiarismDetector -- ComparisonResult
    ComparisonResult <|.. TokenBasedComparisonResult~TokenMatch~

    class TokenBasedPlagiarismDetector~TokenizedSource, Sequence<Token, TokenMatch~ {
        -comparisonStrategy: ComparisonStrategy~TokenizedSource, Sequence<Token, TokenMatch~
        -similarityEstimationStrategy: TokenBasedSimilarityStrategy
        +invoke(Pair~TokenizedSource, TokenizedSource~) ComparisonResult~TokenMatch~
    }
    TokenBasedPlagiarismDetector ..|> PlagiarismDetector

    class SimilarityEstimationStrategy~in S: SourceRepresentation<T>, T, out M: Match~ {
        <<interface>>
        +similarityOf(representation: Pair~S, S~, matches: Set~M~) Double
    }
    class TokenBasedSimilarityStrategy~TokenizedSource, Sequence<Token>, TokenMatch~ {
        <<interface>>
    }
    SimilarityEstimationStrategy <|-- TokenBasedSimilarityStrategy
    NormalizedAverageSimilarityStrategy ..|> TokenBasedSimilarityStrategy
    NormalizedMaxSimilarityStrategy ..|> TokenBasedSimilarityStrategy
    TokenBasedSimilarityStrategy --* TokenBasedPlagiarismDetector

    class ComparisonStrategy~in S: SourceRepresentation<T>, T, out M: Match~ {
        <<interface>>
        +invoke(representations: Pair~S, S~) Set~Match~
    }
    TokenBasedPlagiarismDetector *--> ComparisonStrategy

    BaseGreedyStringTiling~TokenizedSource, Sequence<Token>, TokenMatch~ ..|> ComparisonStrategy
    GreedyStringTiling --|> BaseGreedyStringTiling
    RKRGreedyStringTiling --|> BaseGreedyStringTiling

    class Match {
        <<interface>>
    }
    
    class TokenMatch {
        <<interface>>
        +pattern: Pair~TokenizedSource, List<Token>~
        +text: Pair~TokenizedSource, List<Token>~
        +length: Int
    }
    TokenMatch --|> Match
    ComparisonStrategy -- TokenMatch
```

`KnoledgeBaseRepository` is the component that will take care of saving/loading the results of project sources processing so that they can be reused at a later time without having to be cloned again and re-analyzed

```mermaid
classDiagram
    direction BT
    class KnoledgeBaseRepository~S: SourceRepresentation<T>, T~ {
        <<interface>>
        +save(repository: Repository, representations: Sequence~S~)
        +loadIfExists(repository: Repository) Sequence~S~
    }
    class FileKnoledgeBaseRepository~S: SourceRepresentation<T>, T~ {
        <<interface>>
        -repositoryFolder: File
        -serializer: RepresentationSerializer~S, T~
        +FileKnoledgeBaseRepository(serializer: RepresentationSerializer~S, T~)
        +save(repository: Repository, representations: Sequence~S~)
        +loadIfExists(repository: Repository) Sequence~S~
    }
    FileKnoledgeBaseRepository ..|> KnoledgeBaseRepository

    class RepresentationSerializer~S: SourceRepresentation< T >, T~ {
        <<interface>>
        +serialize(representations: Set~S~, out: File)
        +deserialize(serializedContent: File) Set~S~
    }
    class TokenizedSourceSerializer~TokenizedSource, Sequence<Token>~ {
        +serialize(representations: Set~TokenizedSource~, out: File)
        +deserialize(serializedContent: File) Set~TokenizedSource~
    }
    TokenizedSourceSerializer ..|> RepresentationSerializer
    FileKnoledgeBaseRepository *--> RepresentationSerializer
```

### Configurable Options + AntiPlagirismSession

----
- **Positional arguments (A)**: 
  - **provides information to either the command or one of its options** (`-o file`)
  - advantage of being able to accept a variable number of values
  - commonly used for values like file paths or URLs or for required values
- **Options (O)**
  - follows Unix conventions: use options for most parameters
  - are limited to a fixed number of values
  - **modify the behavior of the command** (e.g. `-v`: verbose)
  - Can 
    - acts as flags (don't have to take values)
    - prompt for missing input
    - load values from environment variables
  - commonly used for "everything else"
- **Subcommands**: functions / (low-level) commands, which are used with "metacommands" that embed multiple separate commands, like `git status -s` - `status` is the subcommand and `-s` is an option of the subcommand

----

**Tokenization Technique**
- `--minimum-tokens`: The minimum token length which should be reported as a duplicate;

**Commons options**
- `--technique`: technique used to analyze and detect similarities.
- `--exporter`: the output type format.
- `--minimum-duplication-percentage`: the percentage of duplicated code in a source file under which are not reported;
- `--output-format`: report format - a set of possible choices;
- `--o`/`--output-dir`: the directory where to store the result
- `--language`: Sources code language;
- `--exclude`: File names to be excluded from checks (for example `Pair.java`).

**Provider Subcommands: `submission`, `corpus`**
**TODO**
(da aggiungere autenticazione ai provider?)

Example:

```bash
--output-dir /Users/lucatassi/Desktop/cpd-results
--minimum-tokens 15
submission
--url https://github.com/DanySK/Student-Project-OOP20-Chiarini-Pezzi-Quarneti-Tassinari-rogue
corpus
--repository-name Student-Project-OOP20
--user danysk
--service github
```

```mermaid
classDiagram
    direction BT
    class RunConfigurator {
        <<interface>>
        +sessionFrom(args: List~String~) AntiPlagiarismSession
    }
    RunConfigurator <|.. CLIConfigurator

    class RunConfiguration~M : Match~ {
        <<interface>>
        +technique: TechniqueFacade~M~
        +minDuplicatedPercentage: Double
        +submission: Set~Repository~
        +corpus: Set~Repository~
        +filesToExclude: Set~String~
        +exporter: ResultsExporter~M~
    }
    class RunConfigurationImpl~M : Match~
    RunConfiguration <.. RunConfigurator : creates

    class TechniqueFacade~out M : Match~ {
        <<interface>>
        +execute(\nㅤㅤsubmittedRepo: Repository, \n ㅤㅤcomparedRepo: Repository, \n ㅤㅤfilesToExclude: Set~String~, \n ㅤㅤminDuplicationPercentage: Double\n) Result~M~
    }
    class TokenizationFacade~TokenMatch~ {
        -analyzer: TokenizationAnalyzer
        -detector: TokenBasedPlagiarismDetector
    }
    TokenizationFacade ..|> TechniqueFacade

    class ResultExporter~in M : Match~ {
        <<interface>>
        +invoke(results: Set~Result<M>~)
    }
    class FileExporter~in M : Match~ {
        <<abstract>>
        +FileExporter(outputDirectory: Path)
    }
    FileExporter ..|> ResultExporter
    PlainFileExporter~in M : Match~ --|> FileExporter

    TechniqueFacade --* RunConfiguration
    ResultExporter --* RunConfiguration

    class AntiPlagiarismSession {
        <<interface>>
        +invoke()
    }
    class AntiPlagiarismSessionImpl~out C : Configuration<M>, M : Match~ {
        -configuration: C
        +AntiPlagiarismSessionImpl(configuration: C)
        +invoke()
    }
    AntiPlagiarismSessionImpl ..|> AntiPlagiarismSession
    AntiPlagiarismSessionImpl *--> RunConfiguration
```

```mermaid 
sequenceDiagram
    autonumber
    activate Main

    Main ->> CLIConfigurator: args
    activate CLIConfigurator
    CLIConfigurator ->> CLIConfigurator: create configuration
    CLIConfigurator ->> AntiPlagiarismSession: create
    CLIConfigurator -->> Main: session
    deactivate CLIConfigurator

    Main ->> AntiPlagiarismSession: run
    activate AntiPlagiarismSession
    AntiPlagiarismSession -->> Main: result
    deactivate AntiPlagiarismSession

    deactivate Main
```
