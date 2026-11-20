# Encoding and Evolution

Application inevitably change over time. Features are added or modified which also require a change to data that it stores. We should aim to build systems that make it easy to adapt (evolvability).

The data models have different ways of coping with this:

- Relational dbs assume all data in the database conforms to one schema although it can be changed (through migrations), there is exactly one schema in force at any point of time. 

- Schema-on-read dbs don't enforce a schema.

When a data format or schema changes, a corresponding change to application code often needs to happen and they cannot happen instantaneously

- In server-side application we can do **Rolling upgrade**: Deploying the new version to a few nodes at a time, checking whether it is running smoothly and gradually working through all the nodes. Deployment without service downtime, more frequent releases and better evolvability.

- With client-side applications it all depends on users.

Old and new versions can co-exist at the same time, we need to maintain compatibility in both directions:

- **Backward compatibility**: Newer code can read data written by older code. Easy to achieve as we know the structure of the old data.

- **Forward compatibility**: Older code can read data that was written by newer code. Trickier because it requires older to ignore additions made by the new version.

## Formats for Encoding Data

Programs usually work with data in two different representations:

1. In memory, data is kept in objects, structs, lists, hash tables, trees, etc. These data structures are optimized for efficient access and manipulation by the CPU (using pointers)

2. To write a data to a file or send it over a network, it must be encoded in some kind of self-contained sequence of bytes (JSON).

We need translation between the two representations. The translation from the in-memory repesentation to a byte sequence is called **encoding** (also known as serialization and marshalling) and the reverse is called **decoding** (parsing, deserialization, unmarshalling).

### Languagae-Specific Formats

Many have built-in support for encoding in-memory objects into byte sequences. These are convenient as they allow in-memory objects to be saved and restored with minimal additional code. However, the disadvantages are:

- The encoding is often tied to a particular programming language.

- In order to restore data in the same object types, the decoding process needs to be able to instantiate arbitrary classes. This leads to security issues like remote code execution.

- Versioning the data is often an afterthought in these libraries. Forward and backward compatibility is often neglected.

- Efficiency is also an afterthought.