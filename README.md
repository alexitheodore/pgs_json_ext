# Overview:
The purpose of this library/extension is to make it easier to interact with JSON(B) data in Postgres. To that end there are a number of custom operators that reference native functions but with much more friendly syntax. For example, perhaps the most commonly used would be the JSON constructor. In native Postgres, the only way to do this is with the "jsonb\_build_object()" function:

`select json_build_object('foo',1);`

but in short hand form this would be:

`select 'foo' +> 1;`

The choice of operator characters is the best attempt at a balance of existing convention, available/allowed characters, common sense and managing conflicts. There is a fair argument that using these operator symbols makes the code opinonated or difficult for standard eyes to read. However if you ever want to use JSONB extensively, its a messy and cumbersome experience without them. Its recommended that you make a clear annotation in your code if you plan to use them so that other developers know they aren't crazy.

# Instalation:

The current version supports up to Postgres 9.6.

It is recommended to run the entire SQL build file as some functions may contain dependancies. Its also recommended to consider the user that is executing the file/code for the appropriate permissions. Ideally they should be fun under an admnistrative user account.

# List of operators:

| Operator | Description                                                                                                                                                                                       | Inputs (left, right) | Outputs |                                                                       Example                                                                       |                            Example Result                            |
|:--------:|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:--------------------:|:-------:|:---------------------------------------------------------------------------------------------------------------------------------------------------:|:--------------------------------------------------------------------:|
|    +>    | Builds a JSONB object from the left key and right value.                                                                                                                                          |   TEXT, ANYELEMENT   |  JSONB  |                                                                       'poo'+>1                                                                      |                               {"poo":1}                              |
|   ->##   | Returns the numeric value from the left object per the key name given by right parameter.                                                                                                         |      JSONB, TEXT     | NUMERIC |                                                          ('{"poo":3.14}'::JSONB) ->## 'poo'                                                         |                                 3.14                                 |
|    ->#   | Returns the integer value from the left object per the key name given by right parameter.                                                                                                         |      JSONB, TEXT     |   INT   |                                                           ('{"poo":7}'::JSONB) ->## 'poo'                                                           |                                   7                                  |
|   ->>&   | Returns the text array from the left object per the key name given by right parameter.                                                                                                            |      JSONB, TEXT     |  TEXT[] |                                                      '{"poo":["a","b","c"]}'::JSONB ->>& 'poo'                                                      |                                {a,b,c}                               |
| ->?      | Returns the boolean value from the left object per the key name given by right parameter.                                                                                                         | JSONB, TEXT          | BOOLEAN | select '{"poo":true}'::JSONB ->? 'poo'                                                                                                              | TRUE                                                                 |
| ->@      | Returns the date from the left object per the key name given by right parameter.                                                                                                                  | JSONB, TEXT          | DATE    | select '{"poo":"2018-01-01"}'::JSONB ->@ 'poo'                                                                                                      | 2018-01-01                                                           |
| #&       | Returns the top-level objects in the left object which are listed as key names in the right text array parameter.                                                                                 | JSONB, TEXT[]        | JSONB   | select '{"poo1":1,"poo2":2,"poo3":3}'::JSONB #& '{poo1, poo3}'::text[]                                                                              | {"poo1": 1, "poo3": 3}                                               |
| #        | Returns the top-level object from the left argument which matches the key name as specified by the right argument.                                                                                | JSONB, TEXT          | JSONB   | select '{"poo1":1,"poo2":2,"poo3":3}'::JSONB # 'poo2'                                                                                               | {"poo2": 2}                                                          |
| &        | Combines the left and right objects using the conventional concatenate mechanism except with the added benefit of ignoring nulls.                                                                 | JSONB, JSONB         | JSONB   | select '{"poo1":1}'::JSONB & '{"poo2":2,"poo3":3}'::JSONB;  select NULL & '{"poo2":2,"poo3":3}'::JSONB;  select '{"poo1":1}'::JSONB & NULL;         | {"poo1": 1, "poo2": 2, "poo3": 3} {"poo2": 2, "poo3": 3} {"poo1": 1} |
| ->#&     | Returns the integer array from the left argument for the key name of the right argument.                                                                                                          | JSONB, TEXT          | INT[]   | select '{"poo":[1,2,3]}'::JSONB ->#& 'poo';                                                                                                         | {1,2,3}                                                              |
| **       | Returns the given JSONB object formatted "pretty".                                                                                                                                                | JSONB                | JSONB   |                                                                                                                                                     |                                                                      |
| ?&!      | Returns FALSE if any of the keys provided in the left object are not specified in the right text array, otherwise it returns TRUE. In other words, if the key names exceed the scope of the list. | JSONB, TEXT[]        | BOOLEAN | select '{"poo1":1, "poo2":2, "poo3":3}'::JSONB ?&! '{poo1}'::TEXT[]; select '{"poo1":1, "poo2":2, "poo3":3}'::JSONB ?&! '{poo1,poo2,poo3}'::TEXT[]; | FALSE  TRUE                                                          |
| @        | Casts the right text into a JSONB object.                                                                                                                                                         | TEXT                 | JSONB   | select @'{"poo":1}'                                                                                                                                 | {"poo": 1}                                                           |	

# JSONB Functions:

There are only a handful of functions included here which are not related to the operators above. Those are:

### JSONB table insert

Emulates a single-row table insert query using json where the supplied object keys correspond to the destination table column names. The first argument is the destination schema, the second argument is the destination table and the third argument is the object with values for columns (invalid column names are ignored). The response is the newly added row.

Example:

> create table poo (
> 	col1	text
> ,	col2	int
> ,	col3	date
> )
> ;

> select jsonb\_table_insert(
> 	'public'
> ,	'poo'
> ,	'{"col1":"abc"}'::JSONB
> )
> ;

> select jsonb\_table_insert(
> 	'public'
> ,	'poo'
> ,	'{"col1":"abc", "col2":2, "col3":"2018-01-01"}'::JSONB
> )
> ;


### JSONB table update

Emulates a single-row table update query using json where the supplied object keys correspond to the destination table column names. The first argument is the destination schema, the second argument is the destination table, the third argument is the json object to specify the where constraints and the fourth argument is the json object with values for updated columns (invalid column names are ignored). The response is the newly added row.

Example:

> select jsonb\_table_update(
> 	'public'
> ,	'poo'
> ,	'{"col1":"abc"}'::JSONB
> ,	'{"col1":"XYZ"}'::JSONB
> )
> ;


### JSONB Delta Comparison

Computes a diff between the first and second JSONB arguments (top-level keys only).

> select jsonb_delta('{"poo":1}', '{"poo":2}') 
>> {"poo": {"left": 1, "right": 2}}

