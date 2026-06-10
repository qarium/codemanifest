# DSL Specification

A DSL document (manifest) resides in the directory whose interface it describes.
The directory is called a **cell**. The manifest filename is fixed as `CODEMANIFEST`.

YAML key casing is **case-sensitive**: keys must appear in the exact casing shown in this specification. Any deviation constitutes a structural error.

A cell may contain **practices** (usages) — documentation files that describe how to work with the cell and consume its API. Practices reside in the `.usages` directory within the cell.

## Cell Structure

```
cell/
├── CODEMANIFEST
└── .usages/*.md
```

* cell — directory named after the cell
* CODEMANIFEST — YAML DSL describing the API contract
* .usages — directory containing practices for working with the cell

**IMPORTANT**: each cell stores its practices in `.usages`. Practices explain how to consume the cell; they do not define requirements for the cell or its contract.

## CODEMANIFEST File Example

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

"ParseInput(input: string) -> data:List<byte>":
  location: parser.<ext>
  annotations: |
    Description of routine.

    `input`: description of input

    Use `pattern` for implementation
    Next requirements to routine ...

"HTTPServer(name: String)":
  location: server.<ext>
  annotations: |
    Description of entity.

    `name`: description of name

    Use `pattern` for implementation
    Use `AnotherCellType` from Imports for data types
    Next requirements to entity ...
  properties:
    "host -> String": |
      Description of property
  methods:
    "handleRequest(req: Request) -> resp:Response": |
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

The DSL defines a **cell contract** — a language-agnostic set of types and their expected API, expressed in YAML.

The document consists of three logical sections:

1. **Header** (meta-level) — establishes context:
   - Type sources (`Imports`)
   - Practices (`Usages`)
   - Global directives (`Annotations`)

2. **Body** (contract definition) — type declarations and their expected behavior

3. **Footer** (meta-level) — metadata that does not affect the contract architecture:
   - Author name (`Author`)
   - Creation date (`CreatedAt`)
   - Manifest description (`Description`)

Sections are separated using the YAML document marker:

```yaml
---
```

The section order is **mandatory**:
1. Header
2. Body
3. Footer

**IMPORTANT**: the document does not prescribe *how* to implement the code. It specifies **the API and behavioral expectations** that the implementation must satisfy.

---

### Header

The Header defines the interpretation context for the entire file.

#### Importing Types and Practices

Types from other cells are brought in via `Imports` and referenced throughout the body.

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

Connects types and practices available within the project.

- `Types` — list of type names to import
- `Usages` — list of practice names to import
- `From` — source path (directory relative to the working directory containing the `CODEMANIFEST` file)
- The `ObjectTwo AS Object` syntax imports `ObjectTwo` under the alias `Object`

**Casing is significant.**

Imported types may:
- Appear in declared interfaces
- Undergo mutation and extension
- Be embedded into the current contract

Imported practices:
- Reside in the source cell at `{From}/.usages/`
- Are referenced by filename without the `.md` extension; the full path resolves to `{From}/.usages/{name}.md`
- Establish a **tracked dependency** — when a practice changes, consumers are identified through the import graph
- **Do not** create contractual obligations — they serve as documentation for the consumer
- Must not conflict with local `Usages` keys in the document header (resolve conflicts via the `AS` alias in `Imports`)

Restrictions:
- Cross-imports between cells are prohibited: cell `A` cannot import from cell `B` if cell `B` imports from cell `A`

#### Practices (Usages)

`Usages` is a header directive that declares a named set of practices for reference in the document's annotations.

A practice is documentation covering one of the following:
- A library
- A design pattern
- A convention
- Any specification

Values in the `Usages` section accept:
- A path to an md file, relative to the project root (must reside in `.goga/usages/`)
- A URL
- An inline description

```yaml
Usages:
  library: .goga/usages/encoding.md
  structures: http://goga.example/structures.md
  pattern: |
    Usage description here...
```

The **key** serves as the practice identifier for backtick references in annotations, e.g., `pattern`.

Practices are connected through two mechanisms:

1. **Declaration** in the `Usages` section — the practice is defined by its value
2. **Import** via the `Imports` section — the practice is imported from another cell's `.usages/` directory by filename without the `.md` extension. Import establishes a tracked dependency without creating a contractual obligation.

**IMPORTANT**: practices cannot bind directly to contract interfaces. They provide context only — informing the implementing agent about external resources and how to work with them.

#### Annotations

Global directives addressed to the implementing agent.

They convey:
- Implementation requirements
- Constraints
- Architectural expectations
- Practice application hints

They apply to the entire document.

```yaml
Annotations: |
  Logic of contract here
```

---

### Body

The body defines **contract types** — which API elements must exist and how they must behave.

Three constructs are available:

1. Type declaration
2. Type embedding
3. Type mutation

#### Types

A type in the DSL is an abstract API unit.

It may represent:
- A class
- A structure
- An object
- A function
- A service
- Any other entity

The DSL does not prescribe an implementation form — only the expected contract.

#### Type Declaration

A type is declared by its signature:

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

The signature uses free-form notation close to programming language syntax, enabling an LLM to map it directly to code.

It:
- Describes the API shape
- Helps the agent understand the expected model
- Does not require a strict formal grammar

Basic requirements:
* The signature describes the contract's input and output
* Input and output must specify a data type
* The output type is paired with a variable/label that conveys the semantic meaning of the returned value

##### location

```yaml
Type():
  location: file.ext
```

Specifies the logical file placement relative to the current directory root, in filename format.

Restrictions:
* The file must reside at the same directory level as `CODEMANIFEST`
* The file must include an extension
* The path must not traverse parent directories or descend into subdirectories

This defines the **expected filesystem structure**, not the implementation method.

#### Entity Type

An Entity must have `methods` and/or `properties`.

```yaml
"User(login: String)":
  location: User.<ext>
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

Define the available operations.

```yaml
ApiClient():
  location: api.<ext>
  annotations: |
    HTTP client for external APIs.
  methods:
    "fetchData(endpoint: string) -> response:Promise<string>": |
      this is annotation of method

      `endpoint`: url to fetch
```

Each method:
- Has a unique name within the entity
- Is defined by its signature
- Is accompanied by an annotation

##### Properties

Define the type's data fields.

```yaml
Config():
  location: config.<ext>
  annotations: |
    Server configuration.
  properties:
    "host -> string": |
      Server hostname
    "port -> int64": |
      Server port
```

Each property:
- Has a unique name within the entity
- Specifies the data type of the return value
- Is accompanied by an annotation

#### Routine Type

A Routine has no `methods` or `properties`. Its contract follows an input → output form (output is optional when nothing is returned).

```yaml
"calculate_total(a: int, b: int) -> total:int":
  location: calculator.<ext>
  annotations: |
    this is annotation of routine

    `a`: first operand
    `b`: second operand
```

In this example, **total** is a semantic label paired with the `int` type to clarify the meaning of the return value.
**a** and **b** are input parameters of type `int` for a class constructor, struct, or function call.

#### Minimal Declaration

When `methods` and `properties` are absent, the type is treated as a callable unit — a procedure (function, functor, etc., depending on language capabilities).

```yaml
"lookup_entry(std::string key) -> value:int":
  location: engine.<ext>
  annotations: |
    ...
```

The DSL does not prescribe the implementation form — only the expected contract.

#### Type Mutation

Type mutation uses the following syntax:

```yaml
"Object::SomeClass()":
  ...
```

This denotes:
- Source type: `Object`
- Target form: `SomeClass`

Key points:
- The DSL does not define the mutation mechanism
- Mutation may be realized through:
  - Inheritance
  - Composition
  - Adapter pattern
  - Interface implementation
  - Decorator pattern
  - Any other strategy

The specification fixes only the fact:
**a type exists that represents a concretization of the base type and its extension**

For routines, mutation may indicate that the user wants the same outcome with a different signature and modified logic. This notation may require:
- Extension via decoration
- Complete replacement of the original logic
- Any other strategy

The number of types involved in mutation is unbounded.

```yaml
"ObjectOne::ObjectTwo::SomeClass()":
  ...
```

Semantically, `SomeClass` must be a mutation of both `ObjectOne` and `ObjectTwo`.

---

### Footer

The footer is optional and specifies manifest metadata.

```yaml
Author: FirstName SecondName
CreatedAt: day/month/year

Description: |
  Manifest description
```

Fields:
- `Author`: first and last name of the manifest author
- `CreatedAt`: manifest creation date
- `Description`: description of the manifest

---

## usages/ Directory

Practices reside at two levels:

**Project level** — the `.goga/usages/` directory at the project root. Contains shared practices not bound to a specific cell: libraries, tools, conventions. Referenced by path in the `Usages` directive of CODEMANIFEST. Project-level practices **must reside exclusively** in `.goga/usages/`.

**Cell level** — the `.usages/` directory inside a cell. Contains practices for consumers of a specific cell's API: how to work with the cell facade, which patterns to apply. Consumers connect them through `Imports`, referencing the provider cell.

---

## Type Embedding

Embedding includes a type in the current contract.

Restrictions:
- The type must be available via `Imports`
- Embedding from `Usages` is not recommended

```yaml
Imports:
  - Types:
      - Entity
    From: path/to/cell

---

->Entity: {}
```

Semantically, this includes the imported type (`Entity`) into the current contract.

---

## Annotations

Annotations are the primary mechanism for controlling code generation.

They are not descriptions of "what something is" — they are directives specifying:
- Expected output
- API behavior requirements
- Which practices to apply
- Which constraints to enforce

Annotations may reference practices from both the header `Usages` and `Imports`.

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
  location: repository.<ext>
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

A reference is a backtick-enclosed identifier within an annotation (e.g., `param`) that points to a named document element: a signature variable, a type, or a practice. A reference binds the annotation text to a specific entity within the current `CODEMANIFEST` context.

References may target:
- Signature variables
- Any type in the current `CODEMANIFEST` file context, including those from `Imports`
- Practices in `Usages` and `Imports`

Restrictions:
- References must be enclosed in backticks, e.g., `link_name`
- Annotations must not reference entities outside the current `CODEMANIFEST` file context

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

  Use `usage_link` in this annotations

  Use `ObjectTwo` link from imports
  Use `Object` link from imports with alias

---

Repository():
  location: repository.<ext>
  annotations: |
    Use `usage_from_imports` from Imports

    Use `usage_link` in this annotations

    Use `Object` link from imports with alias
    Use `ObjectTwo` link from imports
  methods:
    "findById(userId: Int) -> user:List<String>": |
      Use `usage_from_imports` from Imports

      Use `usage_link` in this annotations

      Use `Object` link from imports with alias
      Use `ObjectTwo` link from imports

      Use `userId` in this annotations
      Use `user` in this annotations
  properties:
    "tableName -> String": |
      Use `usage_from_imports` from Imports

      Use `usage_link` in this annotations

      Use `Object` link from imports with alias
      Use `ObjectTwo` link from imports
```

### Global Annotations

```yaml
Annotations: |
  Global annotations in document header
```

Establish shared context:

- Libraries in use
- Implementation principles
- Execution specifics
- And so forth

Applied to the entire document.

### Practice Annotations

```yaml
Usages:
  usage_file: .goga/usages/usage.md
  usage_url: http://usage.url/usage.md
  usage_text: |
    Inline text of usage in document header
```

Practices may be specified as:

- A path to an md file
- A URL
- Inline text

They define:
- How to implement
- How to consume
- Which approaches to apply
- Which constraints to account for
- And so forth

---

### Type Annotations

```yaml
ExampleType():
  location: types.<ext>
  annotations: |
    Type annotations here
```

Specify expectations for the type:

- Behavior
- Purpose
- Operational rules
- Interaction with other entities

They may:
- Clarify the signature
- Introduce requirements
- Reference practices

---

### Property and Method Annotations

```yaml
NetworkManager():
  location: network.<ext>
  properties:
    "baseURL -> String": |
      Property annotations here
  methods:
    "fetchProfile(userId: Int) -> profile:UserProfile": |
      Method annotations here
```

Provide specifics on:

- Operation logic
- Data structure
- Result format
- Processing rules

These are not descriptions — they constitute a **behavioral contract** that must be implemented.

---

## Practices

Practices form the documentation layer for consumers of a cell's API.

They do not define entities. They describe how to work with the cell facade.

---

### Connecting and Using a Practice

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

**IMPORTANT**: a practice receives a local reference name in the document that can be used in annotations, e.g., `pattern`.

Practices are referenced within annotations to convey:

- Instructions to use a specific library
- References to a design pattern
- Requirements to follow a specific structure
- And so forth

In summary:

- The DSL describes **what must exist**
- Practices define **how to use the cell's API**
- Annotations bind these two layers together
