# Data Models and Query Languages 

Most applications are built by layering one data model on top of another. Each asking the same question: *How is it represented in terms of the next lower layer?*

App developer(real world data) -> Objects or data structures and APIs -> Stored in JSON or XML -> Bytes in memory -> Electric currents

Each layer hides the complexity of layers below it providing a clean data model **(Abstraction)**.

## Relational Model vs Document Model

### 1. Relational Model

Data is organized in relations(tables) where each relation is an unordered collection of tuples(rows).

The goal was to hide the implementation detail of internal representation of the data in the database behind a cleaner interface.

Advantages over network model:

- Insertion can be done without worrying about to and fro paths from other data
- Can read any or all rows in a table.
- Query optimizer automatically decides which parts of the query to execute in order and how to use indexes.
- Took out load from application developer and delegated to query optimizer

### 2. Document Model (NoSQL)

Open source, distributed, non-relational databases.

Driving forces behind the adoption:

- Need for greater scalability and very high write throughput
- Specialized query operations
- Restrictiveness of relational schemas

### Object relational mismatch

Most of the development is done in object-oriented programming. 

If the data is stored in relational tables, a translational layer is required between objects and database tables **(impedance mismatch)**.

ORMs helps in this but they can't completely hide the differences.

**Example**: A LinkedIn profile

User can have multiple jobs, educations and contact info.

Ways in which we can store in relational db:

- Seperate tables
- Structured datatypes and XML/JSON data
- Encode the JSON/XML as text file and let application intepret

In NoSQL databases like MongoDB it can be stored as a self-contained document. This has better locality

- In relational case we have to do multiple queries to get the data, which can be done in one query in NoSQL case.

This is a good case of **One-to-many Relationships**

### Many-to-one and Many-to-many relationships

For regions and industry like paramenters it is better to store them as an ID and reference it using the stored information for human use **(Normalization: Removing duplication)**.

- Has consistent style and spelling
- Avoids ambiguity
- Ease of updating
- Localization support and better search

*Normalization of data requires many-to-one relationship.*

Document databases doesn't support this out of the box like relational databases support in the form of joins.

Document databases generally use hierarchichal model

**Network Model**:

- Same as hierarchical(tree) model but a record can have multiple parents
- The links between records in the network model are like pointers
- Access path is like traversal of linked list
- The code for querying and updating the database is very complicated and inflexible

### Tradeoffs between relational model and document model 

1. **Which model leads to simpler application code?**

    - If the application has document like structure (tree of one-to-many relationships and entire tree is loaded once) use document model.

    - If many-to-many relationships are present, then its difficult to denormalize the data and keep it consistent in document model. Use relational model.

2. **Schema Flexibility**

    - Document databases have *schema-on-read* (the structure of data is implicit and only intepreted when data is read) like dynamic(runtime) type checking. Better if data is heterogeneous (different types of objects are there or structure is determined by external systems)

    - Relational databases have *schema-on-write* like static (compile-time) type checking.

3. **Data locality for queries**

    - In document database, data is stored as a single continuous string and accessing it becomes very easy. But if very small portion of data is useful fetching whole document can be wasteful. Updates also need the whole document to be rewritten.

    - Relational database requires mulitple index lookups, more disk seeks as data is split across tables. There are use cases where locality can also be brought to these like Google's spanner db, oracle's multi-table index cluster tables.

### Convergence of document and relational databases

- Relational databases have support for XML/JSON and it is possible to do local modifications, query inside the data, add indexes to it. 

- Document databases like RethinkDB support relational-like joins. MongoDB drivers automatically resolve database references (by doing client-side join, but it is slower due to additional network round trips and is less optimized)

## Query Languages for Data

SQL is a *declarative* query language.

- Declarative language requires us to just to specify what conditions the results must meet and how we want it to be tranformed.

- It is very easy to execute them in parallel.

### MapReduce Querying

- Neither a declarative query language nor a fully imperative query.

- Map(collect) and reduce(fold or inject) are declared as an imperative function but in reduce the query can be written in declarative way.

- These functions must be pure which means they only use the data that is passed to them as input, cannot perform additional database queries and not have any side effects.

## Graph like Data Models

Relational models can handle simple cases of many-to-many relationships.

Graph databases examples:

- Social graphs: People and their connections
- Web graphs: Webpages and links between them
- Road or rail networks

Algorithms like shortest path and PageRank can be easily used on these.

Graphs are not limited to homogeneous data. Facebook maintains single graph with vertices representing people, locations, events, comments, etc.

### Property graphs

Each vertex consists of:

1. A unique identifier
2. A set of outgoing edges
3. A set of incoming edges
4. A collection of properties (key-value pairs).

Each edge consists of:

1. A unique identifier
2. Tail vertex
3. Head vertex
4. A label to describe the kind of relationship between two vertices
5. A collection of properties

### The Cypher Query Language

```sql
CREATE
    (NAmerica:Location {name:'North America', type:'continent'}),
    (USA:Location {name:'United States', type:'country' }),
    (Idaho:Location {name:'Idaho', type:'state' }),
    (Lucy:Person {name:'Lucy' }),
    (Idaho) -[:WITHIN]-> (USA) -[:WITHIN]-> (NAmerica),
    (Lucy) -[:BORN_IN]-> (Idaho)
```

```sql
MATCH
    (person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (us:Location {name:'United States'}),
    (person) -[:LIVES_IN]-> () -[:WITHIN*0..]-> (eu:Location {name:'Europe'})
RETURN person.name
```

### Triple-Stores and SPARQL

Information is stored in the form of three part statements (subject, predicate, object).

Subject -> Vertex
Predicated -> Key or Edge
Object -> Value or Vertex

**The Semantic Web**: Websites publish information as machine-readable data for computers to read. This becomes *web of data*.

**RDF Data Model**: The Resource Description Framework was intended as a mechanism for different websites to publish data in a consistent format.

**SPARQL Query Language**:

```sql
PREFIX : <urn:example:>
SELECT ?personName WHERE {
    ?person :name ?personName.
    ?person :bornIn / :within* / :name "United States".
    ?person :livesIn / :within* / :name "Europe".
}
```
### The foundation: Datalog

Data model: Predicate(subject, object)

Example data:

```sql
name(namerica, 'North America').
type(namerica, continent).
name(usa, 'United States').
type(usa, country).
within(usa, namerica).
name(idaho, 'Idaho').
type(idaho, state).
within(idaho, usa).
name(lucy, 'Lucy').
born_in(lucy, idaho).
```

Example query:

```sql
/* Rule 1 */
within_recursive(Location, Name) :- name(Location, Name). 

/* Rule 2 */
within_recursive(Location, Name) :- 
    within(Location, Via), 
    within_recursive(Via, Name).

/* Rule 3 */
migrated(Name, BornIn, LivingIn) :-
    name(Person, Name),
    born_in(Person, BornLoc),
    within_recursive(BornLoc, BornIn),
    lives_in(Person, LivingLoc),
    within_recursive(LivingLoc, LivingIn).


?- migrated(Who, 'United States', 'Europe').
```