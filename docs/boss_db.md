# BossDB

BossDB is a database abstraction layer used for querying and updating the database. Currently Tokyo Tyrant, Mnesia, MySQL, and PostgreSQL are supported.
This documentation is taken from the main repository of ChicagoBoss and can be seen in original here: (http://chicagoboss.org/doc/api-db.html)[http://chicagoboss.org/doc/api-db.html]
It is also worth noting that this documentation is not complete and is a work in progress.

## Functions

Functions in the boss_db module include:

### migrate
Apply migrations from list `[{Tag, Fun}]` currently runs all migrations 'up'

### migrate
Run database migration `{Tag, Fun}` in Direction

### find(Id::string()) -> Value | {error, Reason}
Find a BossRecord with the specified Id (e.g. "employee-42") or a value described by a dot-separated path (e.g. "employee-42.manager.name").

### find(Type::atom(), Conditions, Options::proplist()) -> [BossRecord]
Query for BossRecords. Returns BossRecords of type Type matching all of the given Conditions. Options may include limit (maximum number of records to return), offset (number of records to skip), order_by
(attribute to sort on), descending (whether to sort the values from highest to lowest), and include (list of belongs_to associations to pre-cache)

### find_first(Type::atom()) -> Record | undefined
Query for the first BossRecord of type Type.

### find_first(Type::atom(), Conditions) -> Record | undefined
Query for the first BossRecord of type Type matching all of the given Conditions

### find_first(Type::atom(), Conditions, Sort::atom()) -> Record | undefined
Query for the first BossRecord of type Type matching all of the given Conditions, sorted on the attribute Sort.

### find_last(Type::atom()) -> Record | undefined
Query for the last BossRecord of type Type.

### find_last(Type::atom(), Conditions) -> Record | undefined
Query for the last BossRecord of type Type matching all of the given Conditions

### find_last(Type::atom(), Conditions, TimeoutSort) -> Record | undefined
Query for the last BossRecord of type Type matching all of the given Conditions

### count(Type::atom()) -> ::integer()
Count the number of BossRecords of type Type in the database.

### count(Type::atom(), TimeoutConditions) -> ::integer()
Count the number of BossRecords of type Type in the database matching all of the given Conditions.

### counter(Id::string()) -> ::integer()
Treat the record associated with Id as a counter and return its value. Returns 0 if the record does not exist, so to reset a counter just use "delete".

### incr(Id::string()) -> ::integer()
Treat the record associated with Id as a counter and atomically increment its value by 1.

### incr(Id::string(), Increment::integer()) -> ::integer()
Treat the record associated with Id as a counter and atomically increment its value by Increment.

### delete(Id::string()) -> ok | {error, Reason}
Delete the BossRecord with the given Id.

### create_table(TableName::string(), TableDefinition) -> ok | {error, Reason}
Create a table based on TableDefinition

### execute(Commands::iolist()) -> RetVal
Execute raw database commands on SQL databases

### execute(Commands::iolist(), Params::list()) -> RetVal
Execute database commands with interpolated parameters on SQL databases

### transaction(TransactionFun::function()) -> {atomic, Result} | {aborted, Reason}
Execute a fun inside a transaction.

### save_record(RecordBossRecord) -> {ok, SavedBossRecord} | {error, [ErrorMessages]}
Save (that is, create or update) the given BossRecord in the database. Performs validation first; see `validate_record/1`.

### validate_record(RecordBossRecord) -> ok | {error, [ErrorMessages]}
Validate the given BossRecord without saving it in the database. ErrorMessages are generated from the list of tests returned by the BossRecord's `validation_tests/0` function (if defined).
The returned list should consist of `{TestFunction, ErrorMessage}` tuples, where TestFunction is a fun of arity 0 that returns true if the record is valid or false if it is invalid.
ErrorMessage should be a (constant) string which will be included in ErrorMessages if the TestFunction returns false on this particular BossRecord.

### validate_record(RecordBossRecord, IsNew) -> ok | {error, [ErrorMessages]}
Validate the given BossRecord without saving it in the database. ErrorMessages are generated from the list of tests returned by the BossRecord's `validation_tests/1` function (if defined),
where parameter is `on_create | on_update`. The returned list should consist of `{TestFunction, ErrorMessage}` tuples, where TestFunction is a `fun/0` that returns `true` if the record is
valid or `false` if it is invalid. ErrorMessage should be a string which will be included in ErrorMessages if the TestFunction returns `false` on this particular BossRecord.

### validate_record_types(RecordBossRecord) -> ok | {error, [ErrorMessages]}
Validate the parameter types of the given BossRecord without saving it to the database.

### type(Id::string()) -> ::atom()
Returns the type of the BossRecord with Id, or undefined if the record does not exist.

## Creating a Model from a JSON record

If your project is anything like mine then you will probably have a case where you get a JSON in from the user or an api and you want to convert it to a model. we have a new function `boss_record:new_from_json/2` which lets you do that automaticly. To run this call `new_from_json/2`
with your model (atom) and your json record. Note that if you have a field called `id` you will have to rename or remove it

## Conditions and Comparison Operators

The `find` and `count` functions each take a set of Conditions, which specify search criteria. Similar to Microsoft's LINQ, the Conditions can use a special non-Erlang syntax for conciseness.
This special syntax can't be compiled with Erlang's default compiler, so you'll have to let Boss compile your controllers which use it.

Conditions looks like a list, but each element in the list uses a notation very similar to abstract mathematical notation with a left-hand side (an atom corresponding to a record attribute),
a single-character operator, and a right-hand side (values to match to the attribute). The mathematical operators are not all ASCII! You may want to copy-paste the UTF-8
operators below. As an alternative, you can also specify each condition with a 3-tuple with easier-to-type operator names.

As a quick example, to count the number of people younger than 25 with occupation listed as "student" or "unemployed", you would use:

```erlang
boss_db:count(person,
    [age < 25, occupation ∈ ["student", "unemployed"]]).
```
This could also be written:

```erlang
boss_db:count(person,
    [{age, 'lt', 25},
    {occupation, 'in', ["student", "unemployed"]}]).
```
## Valid Conditions

```erlang
key = Value
{key, 'equals', Value}
```
The `key` attribute is exactly equal to Value.

```erlang
key ≠ Value
{key, 'not_equals', Value}
```
The `key` attribute is not exactly equal to Value. This is also used to attain a query like foo IS NOT NULL: `{foo, 'not_equals', undefined}`.

```erlang
key ∈ [Value1, Value2, ...]
{key, 'in', [Value1, Value2, ...]}
```
The `key` attribute is equal to at least one element on the right-hand side.

```erlang
key ∉ [Value1, Value2, ...]
{key, 'not_in', [Value1, Value2, ...]}
```
The `key` attribute is not equal to any element on the right-hand side.

```erlang
key ∈ {Min, Max}
{key, 'in', {Min, Max}}
```
The `key` attribute is numerically between Min and Max.

```erlang
key ∉ {Min, Max}
{key, 'not_in', {Min, Max}}
```
The `key` attribute is not between Min and Max.

```
key ∼ RegularExpression
{key, 'matches', RegularExpression}
```
The `key` attribute matches the RegularExpression. To perform a case-insensitive match, the expression should start with an asterisk (e.g. *erlang).

```erlang
key ≁ RegularExpression
{key, 'not_matches', RegularExpression}
```
The `key` attribute does not match the RegularExpression..

```
key ∋ Token
{key, 'contains', Token}
```
The `key` attribute contains Token.

```erlang
key ∌ Token
{key, 'not_contains', Token}
```
The `key` attribute does not contain Token.

```erlang
key ⊇ [Token1, Token2, ...]
{key, 'contains_all', [Token1, Token2, ...]}
```
The `key` attribute contains all tokens on the right-hand side.

```erlang
key ⊉ [Token1, Token2, ...]
{key, 'not_contains_all', [Token1, Token2, ...]}
```
The `key` attribute does not contain all tokens on the right-hand side.

```erlang
key ∩ [Token1, Token2, ...]
{key, 'contains_any', [Token1, Token2, ...]}
```
The `key` attribute contains at least one of the tokens on the right-hand side.

```erlang
key ⊥ [Token1, Token2, ...]
{key, 'contains_none', [Token1, Token2, ...]}
```
The `key` attribute contains none of the tokens on the right-hand side.

```erlang
key > Value
{key, 'gt', Value}
```
The `key` attribute is greater than Value.

```erlang
key < Value
{key, 'lt', Value}
```
The `key` attribute is less than Value.

```erlang
key ≥ Value
{key, 'ge', Value}
```
The `key` attribute is greater than or equal to Value.

```erlang
key ≤ Value
{key, 'le', Value}
```
The `key` attribute is less than or equal to Value.
