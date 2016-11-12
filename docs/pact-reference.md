![](img/kadena-logo-210px.png)

This document is a reference for the Pact smart-contract language, designed for
correct, transactional execution on a [high-performance blockchain](http://kadena.io). For more
background, please see the [white paper](http://kadena.io/docs/Kadena-PactWhitepaper-Oct2016.pdf).

Copyright (c) 2016, Stuart Popejoy. All Rights Reserved.

Concepts {#concepts}
========

Execution Modes {#execmodes}
---

Pact is designed to be used in distinct *execution modes* to address
the performance requirements of rapid linear execution on a
blockchain. These are:

1. Contract definition.
2. Transaction execution.
3. Queries and local execution.

### Contract Definition {#definitionmode}

In this mode, a large amount of code is sent into the blockchain to
establish the smart contract, as comprised of code (modules), tables
(data), and keysets (authorization). This can also include "transactional"
(database-modifying) code, for instance to initialize data.

For a given smart contract, these should all be sent as a single message
into the blockchain, so that any error will rollback the entire smart contract
as a unit.

#### Keyset definition {#keysetdefinition}

[Keysets](#keysets) are customarily defined first, as they are used to specify
admin authorization schemes for modules and tables.

#### Module declaration {#moduledeclaration}

[Modules](#module) define the API for smart contracts, comprised of [function](#defun)
and ["pact"](#defpact) definitions. When a module is declared, all references to native functions
or functions from other modules are resolved. Resolution failure results in transaction rollback.

Modules can be re-defined as controlled by their admin keyset. Module versioning is not supported,
except by including a version sigil in the module name (e.g., "accounts-v1").

#### Table Creation {#tablecreation}

Tables are [created](#create-table) at the same time as modules, due to the tight relationship
of database tables to code modules, as described in [Table Guards](#tableguards).

There is no restriction on how many tables may be created. However, table names are not
namespaced, so care must be taken to ensure unique names.

Tables are schema-less, so there is no need to drop or redefine tables. Table data can
be migrated for new module code by simply executing the necessary data modification.

### Transaction Execution {#transactionexec}

"Transactions" refer to business events enacted on the blockchain, like a payment, a sale, or
a workflow step of a complex contractual agreement. A transaction is generally a single call to
a module function. However there is no limit on how many statements can be executed. Indeed,
the difference between "transactions" and "smart contract definition" is simply the *kind* of code
executed, not any actual difference in the code evaluation.

### Queries and Local Execution {#queries}

Querying data is generally not a business event, and can involve data payloads that could impact
performance, so querying is carried out as a *local execution* on the node receiving the message.
Historical queries use a *transaction ID* as a point of reference, to avoid any race
conditions and allow asynchronous query execution.

Transactional vs local execution is accomplished by targeting different API endpoints; pact code
has no ability to distinguish between transactional and local execution.

Database Interaction {#dbinteraction}
---

Pact presents a database metaphor reflecting the unique requirements of blockchain execution,
which can be adapted to run on different back-ends.

### Atomic execution {#atomicexecution}

A single message sent into the blockchain to be evaluated by Pact is *atomic*: the transaction
succeeds as a unit, or does not succeed at all, known as "transactions" in database literature.
There is no explicit support for rollback handling, except in [multi-step transactions](#pacts).

### Key-Row Model {#dbkeyrow}

Blockchain execution can be likened to OLTP (online transaction processing) database workloads,
which favor denormalized data written to a single table. Pact's data-access API reflects this
by presenting a *key-row* model, where a row of column values is accessed by a single key.

As a result, Pact does not support *joining* tables, which is more suited for an OLAP (online
analytical processing) database, populated from exports from the Pact database. This does not
mean Pact cannot *record* transactions using relational techniques -- for example, a Customer table whose keys
are used in a Sales table would involve the code looking up the Customer record before writing
to the Sales table.

### Schema-less {#schemaless}

Pact tables do not declare a schema. Modules are expected to enforce data access to ensure
type safety (through the use of functions like [is-string](#is-string), [is-decimal](#is-decimal) etc).

### No Nulls {#nonulls}

Pact has no concept of a NULL value in its database metaphor. The main function for computing
on database results, [with-read](#with-read), will error if any column value is not found.
Authors must ensure that values are present for any transactional read. This is a safety feature
to ensure *totality* and avoid needless, unsafe control-flow surrounding null values.

### Versioned History {#dbversions}

The key-row model is augmented by every change to column values being versioned by transaction ID.
For example, a table with three columns "name", "age", and "role" might update "name" in transaction #1,
and "age" and "role" in transaction #2. Retreiving historical data will return just the change to "name"
under transaction 1, and the change to "age" and "role" in transaction #2.

### Back-ends {#backends}

Pact guarantees identical, correct execution at the smart-contract layer within the blockchain. As
a result, the backing store need not be identical on different consensus nodes.
Pact's implementation allows for integration of industrial RDBMSs, to assist large migrations onto
a blockchain-based system, by facilitating bulk replication of data to downstream systems.

Keysets and Authorization {#keysets}
---

Pact is inspired by Bitcoin scripts to incorporate public-key authorization directly into smart
contract execution and administration.

### Keyset definition {#keysetdefinition}

Keysets are [defined](#define-keyset) by [reading](#read-keyset) definitions from the message
payload. Keysets consist of a list of public keys and a *keyset predicate*.

### Keyset Predicates {#keysetpredicates}

A keyset predicate references a function by name which will compare the public keys in the keyset
to the key or keys used to sign the blockchain message. The function accepts two arguments,
"count" and "matched", where "count" is the number of keys in the keyset and "matched" is how many
keys on the message signature matched a keyset key.

Support for multiple signatures is the responsibility of the blockchain layer, and is a powerful
feature for Bitcoin-style "multisig" contracts (ie requiring at least two signatures to release funds).

Pact comes with built-in keyset predicates: [keys-all](#keys-all), [keys-any](#keys-any), [keys-2](#keys-2).
Module authors are free to define additional predicates.

### Key rotation {#keyrotation}

Keysets can be rotated, but only by messages authorized against the current keyset definition
and predicate. Once authorized, the keyset can be easily [redefined](#define-keyset).

### Module Table Guards {#tableguards}

When [creating](#create-table) a table, a module name must also be specified. By this mechanism,
tables are "guarded" or "encapsulated" by the module, such that direct access to the table
via [data-access functions](#Database) is authorized by the module's admin keyset. However,
*within module functions*, table access is unconstrained. This gives contract authors great
flexibility in designing data access, and is intended to enshrine the module as the main
"user" data access API.

### Row-level keysets {#rowlevelkeysets}

Keysets can be stored as a column value in a row, allowing for *row-level* authorization.
The following code indicates how this might be achieved:

```lisp
(defun create-account (id)
  (insert 'accounts id { "balance": 0.0, "keyset": (read-keyset "owner-keyset") }))

(defun read-balance (id)
  (with-read { "balance":= bal, "keyset":= ks }
    (with-keyset ks
      (format "Your balance is {}" bal))))
```

In the example, `create-account` reads a keyset definition from the message payload using [read-keyset](#read-keyset)
to store as "keyset" in the table. `read-balance` only allows that owner's keyset to read the balance,
by first enforcing the keyset using [with-keyset](#with-keyset).

Computational Model {#computation}
---

Here we cover various aspects of Pact's approach to computation.

### Turing-Incomplete {#turingincomplete}
Pact is turing-incomplete, in that there is no recursion (recursion is detected before execution
and results in an error) and no ability to loop indefinitely. Pact does support operation on
list structures via [map](#map), [fold](#fold) and [filter](#filter), but since there is no ability
to define infinite lists, these are necessarily bounded.

Turing-incompleteness allows Pact module loading to resolve all references in advance, meaning that
instead of addressing functions in a lookup table, the function definition is directly injected (or "inlined")
into the callsite. This is an example of the performance advantages of a Turing-incomplete language.

### Variables {#variables}

Pact allows variable declarations in [let expressions](#let) and [bindings](#binding). Variables are
immutable: they cannot be re-assigned or modified in-place. The most common variable declaration
occurs in the [with-read](#with-read) function, assigning variables to column values by name.

Module-global constant values can be declared with [defconst](#defconst).

### Data Types {#datatypes}

Pact is not a explicitly-typed language, but does use fixed type representations "under the hood"
and does no coercion of types, so is strongly-typed nonetheless. Authors can employ functions
like [is-string](#is-string), [is-decimal](#is-decimal) to enforce type safety.

Pact's supported types are:

- [Strings](#string)
- [Integers](#integer)
- [Decimals](#decimal)
- [Booleans](#boolean)
- [Key sets](#keysets)
- [Lists](#list)
- [Objects](#object)
- [Function](#defun) and [pact](#defpact) definitions
- [JSON values](#json)

[Functions](#defun) may take no arguments and return bare values, which can function as *constants*.

### Performance {#performance}

Pact is designed to maximize the performance of [transaction execution](#transactionexec), penalizing
queries and module definition in favor of fast recording of business events on the blockchain.
Some tips for fast execution are:

#### Single-function transactions {#singlefunctiontx}
Design transactions so they can be executed with a single function call.

#### Call with references instead of `use` {#usereferences}
When calling module functions in transactions, use [reference syntax](#reference) instead of importing
the module with [use](#use). When defining modules that reference other module functions, `use` is
fine, as those references will be inlined at module definition time.

#### Hardcoded arguments vs. message values {#argsvmsgs}
A transaction can encode values directly into the transactional code:

```
(accounts.transfer "Acct1" "Acct2" 100.00)
```
or it can read values from the message JSON payload:

```
(defun transfer-msg ()
  (transfer (read-msg "from") (read-msg "to")
            (read-decimal "amount")))
...
(accounts.transfer-msg)
```
The latter will execute slightly faster, as there is less code to interpret at transaction time.

### Control Flow {#controlflow}
Pact supports conditionals via [if](#if), bounded looping, and of course function application.

#### "If" considered harmful {#evilif}
Consider avoiding `if` wherever possible: every branch makes code harder to understand and more
prone to bugs. The best practice is to put "what am I doing" code in the front-end, and "validate
this transaction which I intend to succeed" code in the smart contract.

Pact's original design left out `if` altogether (and looping), but it was decided that users should
be able to judiciously use these features as necessary.

#### Use enforce {#use-the-enforce-luke}
"If" should never be used to enforce business logic invariants: instead, [enforce](#enforce) is
the right choice, which will fail the transaction.

Indeed, failure is the only *non-local exit* allowed by Pact. This reflects Pact's emphasis on
*totality*.

### Functional Concepts {#fp}

Pact includes the functional-programming "greatest hits": [map](#map), [fold](#fold) and [filter](#filter).
These all employ [partial application](#partialapplication), where the list item is appended onto the application
arguments in order to serially execute the function.

```lisp
(map (+ 2) [1 2 3])
(fold (+) ["Concatenate" " " "me"]
```

Pact also has [compose](#compose), which allows "chaining" applications in a functional style.

### LISP {#lisp}

Pact's use of LISP syntax is intended to make the code reflect its
runtime representation directly, allowing contract authors focus
directly on program execution. Pact code is stored in human-readable
form on the ledger, such that the code can be directly verified, but
the use of LISP-style [s-expression syntax](#sexp) allows this
code to execute quickly.

### Message Data {#messagedata}

Pact expects code to arrive in a message with a JSON payload and signatures. Message data
is read using [read-msg](#read-msg) and related functions, while signatures are not directly
readable or writable -- they are evaluated as part of [keyset predicate](#keysetpredicates)
enforcement.

#### JSON support {#json}

Values returned from Pact transactions are expected to be directly represented as JSON
values.

When reading values from a message via [read-msg](#read-msg), Pact coerces JSON types
as follows:

- String -> String
- Number -> Integer (rounded)
- Boolean -> Boolean
- Object -> JSON Value
- Array -> JSON Value
- Null -> JSON Value

Decimal values are represented as Strings and read using [read-decimal](#read-decimal).

JSON Objects, Arrays, and Nulls are not coerced, intended for direct storage and retreival
as opaque payloads in the database.

Confidentiality {#confidentiality}
---

Pact is designed to be used in a *confidentiality-preserving* environment, where messages
are only visible to a subset of participants. This has significant implications for
smart contract execution.

### Entities
An *entity* is a business participant that is able or not able to see a confidential message.
An entity might be a company, a group within a company, or an individual.

### Disjoint Databases {#disjointdbs}
Pact smart contracts operate on messages organized by a blockchain, and serve to produce a
database of record, containing results of transactional executions. In a confidential environment,
different entities execute different transactions, meaning the resulting databases are now
*disjoint*.

This does not affect Pact execution; however, database data can no longer enact a "two-sided
transaction", meaning we need a new concept to handle enacting a single transaction over
multiple disjoint datasets.

### Pacts {#pacts}

Pacts are multi-step sequential transactions that are defined as a single body of code called
a [pact](#defpact). With a pact, participants ensure they are executing an identical code path,
even as they execute distinct "steps" in that path.

The concept of pacts reflect *coroutines* in software engineering: functions that can *yield* and
*resume* computation "in the middle of" their body. A [step](#step) in a pact designates a target
entity to execute it, after which the pact "yields" execution, completing the transaction and
initiating a signed "Resume" message into the blockchain.

The counterparty entity sees this "Resume" message and drops back into the pact body to find if
the next step is targetted for it, if so executing it.

Since any step can fail, steps can be designed with [rollbacks](#step-with-rollback) to undo changes
if a subsequent step fails.



Syntax
======

Literals {#literals}
--------

### Strings {#string}

String literals are created with double-ticks:

```
pact> "a string"
"a string"
```

Strings also support multiline by putting a backslash before and after whitespace (not interactively).

```lisp
(defun id (a)
  "Identity function. \
  \Argument is returned."
  a)
```

### Symbols {#symbol}

Symbols are string literals representing some unique item in the runtime, like a function or a table name.
Their representation internally is simply a string literal so their usage is idiomatic.

Symbols are created with a preceding tick, thus they do no support whitespace or multiline.

```
pact> 'a-symbol
"a-symbol"
```

### Integers {#integer}

Integer literals are unbounded positive naturals. For negative numbers use the unary [-](#n-) function.

```
pact> 12345
12345
```

### Decimals {#decimal}

Decimal literals are positive decimals to exact expressed precision.
```
pact> 100.25
100.25
pact> 356452.23451872
356452.23451872
```

### Booleans {#boolean}

Booleans are represented by `true` and `false` literals.

```
pact> (and true false)
false
```

### Lists {#list}

List literals are created with brackets.
This is actually a special form, which evaluates the [list](#listfun) function.
```
pact> [1 2 3]
[1 2 3]
pact> (= [1 2 3] (list 1 2 3))
true
```

### Objects {#object}

Objects are dictionaries, created with curly-braces specifying key-value pairs
using a colon `:`.
For certain applications (database updates), keys must be strings.

```
pact> { "foo": (+ 1 2), "bar": "baz" }
(TObject [("foo",3),("bar","baz")])
```

### Bindings {#binding}
Bindings are dictionary-like forms, also created with curly braces, to bind
database results to variables using the `:=` operator.
They are used in [with-read](#with-read), [with-default-read](#with-default-read), and
[bind](#bind) to assign variables to named columns in a row, or values in an object.

```lisp
(defun check-balance (id)
  (with-read 'accounts id { "balance" := bal }
    (enforce (> bal 0) (format "Account in overdraft: {}" bal))))
```

Special forms {#special}
---

### defun {#defun}

```(defun NAME ARGLIST [DOCSTRING] BODY...)```

Define NAME as a function, accepting ARGLIST arguments, with optional DOCSTRING.
Arguments are in scope for BODY, one or more expressions.

```lisp
(defun add3 (a b c) (+ a (+ b c)))

(defun scale3 (a b c s) "multiply sum of A B C times s"
  (* s (add3 a b c)))
```

### defconst {#defconst}
```(defun NAME VALUE [DOCSTRING])```

Define NAME as VALUE, with option DOCSTRING.

```lisp
(defconst COLOR_RED="#FF0000" "Red in hex")
(defconst COLOR_GRN="#00FF00" "Green in hex")
(defconst PI 3.14159265 "Pi to 8 decimals")
```

### defpact {#defpact}

```(defpact NAME ARGLIST [DOCSTRING] STEPS...)```

Define NAME as a _pact_, a multistep computation intended for private transactions.
Identical to [defun](#defun) except body must be comprised of [steps](#step).

```lisp
(defpact payment (payer payer-entity payee
                  payee-entity amount)
  (step-with-rollback payer-entity
    (debit payer amount)
    (credit payer amount))
  (step payee-entity
    (credit payee amount)))
```

### let {#let}

```(let (BINDPAIR [BINDPAIR [...]]) BODY)```

Bind variables in BINDPAIRs to be in scope over BODY. Variables
within BINDPAIRs cannot refer to previously-declared variables in
the same let binding; for this use (let\*){#letstar}.

```lisp
(let ((x 2)
      (y 5))
  (* x y))
> 10
```

### let\* {#letstar}

```(let\* (BINDPAIR [BINDPAIR [...]]) BODY)```

Bind variables in BINDPAIRs to be in scope over BODY. Variables
can reference previously declared BINDPAIRS in the same let.
`let\*` is expanded at compile-time to nested `let` calls for
each BINDPAIR; thus `let` is preferred where possible.

```lisp
(let* ((x 2)
       (y (* x 10)))
  (+ x y))
> 22
```

### step {#step}
```(step ENTITY EXPR)```

Define a step within a _pact_, which can only be executed by nodes representing ENTITY,
in order of execution specified in containing [defpact](#defpact).

### step-with-rollback {#step-with-rollback}
```(step-with-rollback ENTITY EXPR ROLLBACK-EXPR)```

Define a step within a _pact_, which can only be executed by nodes representing ENTITY,
in order of execution specified in containing [defpact](#defpact). If any subsequent
steps fail, ROLLBACK-EXPR will be executed.

### use {#use}
```(use MODULE-SYMBOL)```

Import an existing module into namespace.

```lisp
(use 'accounts)
(transfer "123" "456" 5 (time "2016-07-22T11:26:35Z"))
"Write succeeded"
```

### module {#module}
```(module NAME KEYSET [DOCSTRING] DEFS...)```

Define and install module NAME, guarded by keyset KEYSET, with optional DOCSTRING.
DEFS must be [defun](#defun) or [defpact](#defpact) expressions only.

```lisp
(module accounts 'accounts-admin
  "Module for interacting with accounts"

  (defun create-account (id bal)
   "Create account ID with initial balance BAL"
   (insert 'accounts id { "balance": bal }))

  (defun transfer (from to amount)
   "Transfer AMOUNT from FROM to TO"
   (with-read 'accounts from { "balance": fbal }
    (enforce (<= amount fbal) "Insufficient funds")
     (with-read 'accounts to { "balance": tbal }
      (update 'accounts from { "balance": (- fbal amount) })
      (update 'accounts to { "balance": (+ tbal amount) }))))
)
```

Expressions {#expression}
---

Expressions may be [literals](#literals), atoms, s-expressions, or references.


### Atoms {#atom}

Atoms are non-reserved barewords starting with a letter or allowed symbol, and containing letters,
digits and allowed symbols. Allowed symbols are `%#+-_&$@<>=?*!|/`. Atoms must
resolve to a variable bound by a [defun](#defun), [defpact](#defpact), [binding](#binding) form,
or to symbols imported into the namespace with [use](#use).

### S-expressions {#sexp}

S-expressions are formed with parentheses, with the first atom determining if
the expression is a [special form](#special) or a function application, in
which case the first atom must refer to a definition.

#### Partial application {#partialapplication}
An application with less than the required arguments is in some contexts a valid
*partial application* of the function. However, this is only supported in
Pact's [functional-style functions](#fp); anywhere else this will result in a
runtime error.

### References {#reference}

References are two atoms joined by a dot `.` to directly resolve to module
definitions.

```
pact> accounts.transfer
"(defun accounts.transfer (src,dest,amount,date) \"transfer AMOUNT from
SRC to DEST\")"
pact> transfer
Eval failure:
transfer<EOF>: Cannot resolve transfer
pact> (use 'accounts)
"Using \"accounts\""
pact> transfer
"(defun accounts.transfer (src,dest,amount,date) \"transfer AMOUNT from
SRC to DEST\")"
```

References are preferred to `use` for transactions, as references resolve faster.
However in module definition, `use` is preferred for legibility.

Built-in Functions {#builtins}
==================
