# DSL Specification

A DSL document (manifest) must be located in the folder whose interface it describes.
The folder is called a cell. The manifest file name is strictly fixed — `CODEMANIFEST`.

Case sensitivity of keys in the `yaml` document is **IMPORTANT**: if the specification provides examples with a key in uppercase
or lowercase, the key must be named exactly as shown; any other spelling must result in a document structure error.

Inside the folder (hereinafter — cell), usage practices (usages) may be stored, describing approaches to working with the cell
and using its API. Usages are stored inside the cell in the `.usages` folder, but are not required.

## Cell Structure

```
cell/
├── CODEMANIFEST
└── .usages/*.md
```

* cell — folder whose name is the cell name
* CODEMANIFEST — yaml DSL describing the API contract
* .usages — folder containing usages that describe how to work with the cell

**IMPORTANT**: each cell stores its usage descriptions in `.usages`, which explain how to use the cell,
but they do not store and are not the source of requirements for the cell and its contract.

## CODEMANIFEST File Example

_Example in Go style._

```yaml
Imports:
  - Types:
      - AnotherCellType
    Usages:
      - another_cell_usage
    From: path/to/another_cell

Usages:
  conventions: .goga/usages/conventions.md
  pattern: |
    Some pattern here
  testing: |
    Requirements to tests

Annotations: |
  Use `conventions` for write code.
  Use `testing` for write tests.
  Use `another_cell_usage` from Imports for additional context.

---

"ParseInput(input string) -> data:[]byte":
  location: parser.go
  annotations: |
    Description of routine.

    `input`: description of input

    Use `pattern` for implementation
    Next requirements to routine ...

"HTTPServer(name string)":
  location: server.go
  annotations: |
    Description of entity.

    `name`: description of name

    Use `pattern` for implementation
    Use `AnotherCellType` from Imports for data types
    Next requirements to entity ...
  properties:
    "Host -> string": |
      Description of property
  methods:
    "HandleRequest(req Request) -> resp:Response": |
      Description of method.

      `req`: description of req

      Use `pattern` for implementation
      Next requirements to method ...

---

Author: FirstName LastName
CreatedAt: 01/01/26
Description: |
  Description of CODEMANIFEST file
```

## CODEMANIFEST Document Structure

The DSL describes the **cell contract** — a set of types and their expected API, independent of a specific
programming language or implementation method, based on `yaml`.

The document is divided into three logical parts:

1. **Header (meta-level)** — sets the context:
   - type sources (`Imports`)
   - used usages (`Usages`)
   - global directives (`Annotations`)

2. **Body (contract description)** — type declarations and their expected behavior

3. **Footer (meta-level)** — defines additional meta-information that does not affect the contract architecture:
   - author name (`Author`)
   - document creation date (`CreatedAt`)
   - manifest description (`Description`)

The separation is done according to the `yaml` standard using:

```yaml
---
```

The order of parts is **IMPORTANT**:
1. Header
2. Body
3. Footer

**IMPORTANT**: the document does not describe *how exactly to implement the code*, but fixes the **expectations from the API and behavior** that need to be implemented.

---

### Header

The header sets the context in which the entire file should be interpreted.

#### Importing Types and Usages

Types from other files are connected via `Imports` and then used in the document body.

```yaml
Imports:
  - Types:
      - ObjectOne
      - ObjectTwo AS Object
    Usages:
      - example_one # path/to/cell/.usages/example_one.md
      - example_two AS example # path/to/cell/.usages/example_two.md
    From: path/to/cell
```

This connects available types and usages within the project.

- `Types` — list of names of imported types
- `Usages` — list of names of imported usages
- `From` — source (folder relative to the working directory where the `CODEMANIFEST` file is located)
- The syntax `ObjectTwo AS Object` means that `ObjectTwo` is imported with the alias `Object`

**Case is important**.

Imported types can:
- be used in declared interfaces
- be mutated and extended
- be embedded into the current contract

Imported usages:
- are located in the source cell's folder at the path `{From}/.usages/`
- are imported by file name without the `.md` extension; the full file path is `{From}/.usages/{name}.md`
- create a **tracked dependency** — when a usage changes, consumers are found through the import graph
- do **not** create contractual obligations — they remain at the documentation level for the consumer
- must not have names conflicting with current `Usages` in the document header (conflicts are resolved by creating an alias in `Imports`)

Restrictions:
- Imports cannot be cross-referenced between cells, meaning cell `A` cannot import a type/usage from cell `B` if cell `B` imports a type from cell `A`

#### Usages

`Usages` — a directive in the CODEMANIFEST header that defines a named set of usages for use
in the annotations of the current document.

A usage is documentation that can be about:
- a library
- a pattern
- a convention
- or any specification

Value formats in the `Usages` section:
- path to an md-file, relative to the project root (the file must be located in `.goga/usages/`)
- URL
- inline description

```yaml
Usages:
  library: .goga/usages/encoding.md
  structures: http://goga.example/structures.md
  pattern: |
    Usage description here...
```

The **key** is the usage name for references in annotations using backticks, for example `pattern`.

Usages are connected in two ways:

1. **Declaration** in the `Usages` section — the usage is described by its value
2. **Import** via the `Imports` section — the usage is imported from the `.usages/` directory of another cell
   by file name without the `.md` extension. Import creates a tracked dependency but not a contractual obligation.

**IMPORTANT**: usages cannot directly bind to contract interfaces. They provide only context — informing the executing agent about external resources and how to work with them.

#### Annotations

Global directives for the agent.

These are:
- implementation requirements
- constraints
- architectural expectations
- hints on using usages

They apply to the entire document.

```yaml
Annotations: |
  Logic of contract here
```

---

### Body

The body describes **contract types** — what API elements should exist and how they should behave.

Three main constructs are used:

1. Type declaration
2. Type embedding
3. Type mutation

#### Types

A type in the DSL is an abstract unit of API.

It can be:
- a class
- a structure
- an object
- a function
- a service
- any other entity

The DSL does not fix the implementation form — only the expected contract.

#### Type Declaration

A type is defined by its signature:

```yaml
"<Name><Signature>":
  location: <file.ext>
  annotations: |
    ...
  methods:
    ...
  properties:
    ...
```

##### Signature

```yaml
"TypeName<Signature>":
  location: <file.ext>
  annotations: |
    ...
  methods:
    "<signature>": |
       ...
```

The signature is written in free form, close to programming languages, which an LLM can easily associate with code.

It:
- describes the API shape
- helps the agent understand the expected model
- does not require strict formal grammar

Basic requirements:
* The signature describes the input and output of the contract
* Input and output must have the specified data type
* The output type is associated with a variable/label to convey the semantic meaning of the return value with the specified data type

##### location

```yaml
Type():
  location: file.ext
```

Specifies the logical placement of the type relative to the root of the current directory in file name format.

Restrictions:
* The file must be at the same level as `CODEMANIFEST`
* The file must include an extension
* The path cannot go up a level or descend into subdirectories

This defines the **expected file system structure**, not the implementation method.

#### Entity Type

Must have `methods` and/or `properties`.

_Example in Swift style._

```yaml
"User(login: String)":
  location: User.swift
  annotations: |
    ...
  methods:
    "greet() -> message:String": |
      ...
  properties:
    "identifier -> Int": |
      ...
```

##### Methods

Define available operations.

_Example in JavaScript style._

```yaml
ApiClient():
  location: api.js
  annotations: |
    HTTP client for external APIs.
  methods:
    "fetchData(endpoint: string) -> response:Promise<string>": |
      this is annotation of method

      `endpoint`: url to fetch
```

Each method:
- has a unique name within the entity
- is defined by its signature
- is accompanied by an annotation

##### Properties

Define type properties.

_Example in Go style._

```yaml
Config():
  location: config.go
  annotations: |
    Server configuration.
  properties:
    "host -> string": |
      Server hostname
    "port -> int64": |
      Server port
```

Each property:
- has a unique name within the entity
- specifies the data type of the return value
- is accompanied by an annotation

#### Routine Type

A Routine does NOT have `methods` and `properties`; it has a contract of the form input -> output (optional if nothing is returned).

_Example in Python style._

```yaml
"calculate_total(a: int, b: int) -> total:int":
  location: calculator.py
  annotations: |
    this is annotation of routine

    `a`: first operand
    `b`: second operand
```

In this example, **total** is a semantic label associated with the `int` type for better understanding of the return value's meaning.
**a** and **b** are input parameters of type `int` for a class constructor, structure, or function call.

#### Minimal Declaration

If `methods` and `properties` are not specified, the type is treated as a callable unit — a procedure (function, functor — depending on the capabilities of the programming language).

_Example in C++ style._

```yaml
"lookup_entry(std::string key) -> value:int":
  location: engine.hpp
  annotations: |
    ...
```

The DSL does not fix the implementation form — only the expected contract.

#### Type Mutation

The following form is used for type mutation:

```yaml
"Object::SomeClass()":
  ...
```

This means:

- source type `Object`
- target form `SomeClass`

Important:
- the DSL does not define the mutation mechanism
- it can be:
  - inheritance
  - composition
  - adapter
  - interface implementation
  - decoration
  - or any other strategy

Only the fact is fixed:
**there exists a type that represents a concretization of the base type and its extension**

For procedures, mutation can mean that the user wants to achieve the same result,
but with a different signature and modified logic; this notation can require:
- extension through decoration
- complete replacement of the original logic
- or any other strategy

The number of types for mutation is not limited.

```yaml
"ObjectOne::ObjectTwo::SomeClass()":
  ...
```

Semantically, this means that `SomeClass` must be a mutation of both `ObjectOne` and `ObjectTwo`.

---

### Footer

The footer is optional and describes the manifest metadata.

```yaml
Author: FirstName SecondName
CreatedAt: day/month/year

Description: |
  Manifest description
```

Fields:
- `Author`: first and last name of the manifest author
- `CreatedAt`: manifest creation date
- `Description`: manifest description

---

## usages/ Directory

Usages are stored at two levels:

**Project level** — the `.goga/usages/` directory in the project root. Common usages not tied to a specific cell:
libraries, tools, conventions. Connected via a path in the `Usages` directive in CODEMANIFEST.
Project-level usages **can only be located** in `.goga/usages/`.

**Cell level** — the `.usages/` directory inside a cell. Usages for consumers of a specific cell's API:
how to work with the cell facade, which patterns to apply. Consumers connect them via `Imports`,
referencing the cell provider.

---

## Type Embedding

Embedding means including a type in the current contract.

Restrictions:
- the type must be available via `Imports`
- embedding from `Usages` is not recommended

```yaml
Imports:
  - Types:
      - Entity
    From: path/to/cell

---

->Entity: {}
```

Semantically, this means including the imported type (`Entity`) in the current contract.

---

## Annotations

Annotations are the key mechanism for controlling generation.

These are not descriptions of "what something is", but directives about:

- what is expected as output
- how the API should behave
- which usages to apply
- which constraints to follow

Annotations can reference usages from `Usages` in the header and from `Imports`.

_Example in Kotlin style._

```yaml
Imports:
  - Usages:
      # path/to/cell/.usages/example.md
      - example
    From: path/to/cell

Usages:
  pattern: |
    Pattern example

Annotations: |
  Use `example` from Imports
  Use `pattern` from Usages

---

"UserRepository()":
  location: repository.kt
  annotations: |
    Use `example` from Imports
    Use `pattern` from Usages
  methods:
    "findUserById(id: Int) -> user:List<String>": |
      Use `example` from Imports
      Use `pattern` from Usages
  properties:
    "tableName -> String": |
      Use `example` from Imports
      Use `pattern` from Usages
```

### Using References

A reference is an identifier in backticks inside an annotation (e.g., `param`) that points to
a named element of the document: a variable from the signature, a type, or a usage. A reference links the annotation text
to a specific entity from the context of the current `CODEMANIFEST`.

References can be to:
- variables in the signature
- any types present in the context of the current `CODEMANIFEST` file, including those in `Imports`
- usages in `Usages` and `Imports`

Restrictions:
- references must be enclosed in backticks, for example — \`link_name\`
- annotations must not reference anything that is not in the context of the current `CODEMANIFEST` file

_Example in Kotlin style._

```yaml
Imports:
  - Types:
      - ObjectOne AS Object
      - ObjectTwo
    Usages:
      - usage_from_imports
    From: path/to/cell

Usages:
  usage_link: |
    Pattern example

Annotations: |
  Use `usage_from_imports` from Imports

  Use `usage_link` in the annotations

  Use `ObjectTwo` link from imports
  Use `Object` link from imports with alias

---

Repository():
  location: repository.kt
  annotations: |
    Use `usage_from_imports` from Imports

    Use `usage_link` in the annotations

    Use `Object` link from imports with alias
    Use `ObjectTwo` link from imports
  methods:
    "findById(userId: Int) -> user:List<String>": |
      Use `usage_from_imports` from Imports

      Use `usage_link` in the annotations

      Use `Object` link from imports with alias
      Use `ObjectTwo` link from imports

      Use `userId` in the annotations
      Use `user` in the annotations
  properties:
    "tableName -> String": |
      Use `usage_from_imports` from Imports

      Use `usage_link` in the annotations

      Use `Object` link from imports with alias
      Use `ObjectTwo` link from imports
```

### Global Annotations

```yaml
Annotations: |
  Global annotations in document header
```

Define the general context:

- used libraries
- implementation principles
- execution features
- etc.

Apply to the entire document.

### Usage Annotations

```yaml
Usages:
  usage_file: .goga/usages/usage.md
  usage_url: http://usage.url/usage.md
  usage_text: |
    Inline text of usage in document header
```

Usages can be described as:

- path to an md-file
- URL
- inline text

They define:
- how to implement
- how to use
- which approaches to apply
- which constraints to consider
- etc.

---

### Type Annotations

```yaml
ExampleType():
  location: types.kt
  annotations: |
    Type annotations here
```

Define expectations from the type entity:

- behavior
- purpose
- rules of operation
- interaction with other entities

They can:
- clarify the signature
- introduce requirements
- reference usages

---

### Property and Method Annotations

_Example in Swift style._

```yaml
NetworkManager():
  location: network.swift
  properties:
    "baseURL -> String": |
      Property annotations here
  methods:
    "fetchProfile(userId: Int) -> profile:UserProfile": |
      Method annotations here
```

Used to clarify:

- operation logic
- data structure
- result format
- processing rules

This is not a description, but a **behavior contract** that must be implemented.

---

## Usages

Usages are a layer of documentation for consumers of a cell's API.

They do not create entities, but describe how to work with the cell facade.

---

### Connecting and Using a Usage

```yaml
Imports:
  - Usages:
      - usage_from_cell
    From: path/to/cell

Usages:
  usage_from_doc: .goga/usages/pattern.md
  usage_from_url: http://usage.example/usage.md

Annotations: |
  Use `usage_from_cell` for implementation
  Use `usage_from_doc` for implementation
  Use `usage_from_url` for implementation
```

**IMPORTANT**: a usage receives a local reference name in the document that can be used in annotations, for example \`pattern\`.

Usages are used inside annotations.

For example:

- an instruction to use a specific library
- a reference to a pattern
- a requirement to follow a specific structure
- etc.

Thus:

- the DSL describes **what should exist**
- usages define **how to use the cell's API**
- annotations link these two levels