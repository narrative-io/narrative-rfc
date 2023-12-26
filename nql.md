# NQL: Narrative SQL Interface

## Abstract

The Narrative SQL Interface ("NQL") provides a syntax for querying one or more datasets via a Narrative-specific implementation of Structured Query Language. NQL acts on one or more **Datasets.** NQL is interpreted by a query engine whose purpose is to parse the query and validate syntax, validate the permissions (**[Access Rules]**) associated to the actor submitting the query, execute the query in a supported query engine and, when appropriate, write the output as a Dataset.

## 1. Introduction 

Narrative SQL (NQL) commands are statements that execute queries on the Narrative Data Collaboration Platform.  NQL input consists of a sequence of commands. A command is composed of a sequence of tokens, terminated by a semicolon (“;”). The end of the input stream also terminates a command. Which tokens are valid depends on the syntax of the particular command.

### 1.1 Requirements Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [[RFC2119]{.underline}](https://datatracker.ietf.org/doc/html/rfc2119).

### 1.2 Definitions

- **Commands**: A command is composed of a sequence of tokens, terminated by a semicolon (";"). The end of the input stream also terminates a command. Which tokens are valid depends on the syntax of the particular command.

- **Tokens**: A token can be a key word, an identifier, a quoted identifier, a literal (or constant), or a special character symbol. Tokens are normally separated by whitespace (space, tab, newline)

- **Keywords**: The set of words that NQL is aware of and knows how to parse. 

- **Identifiers**: Names of tables, columns, or other database objects, depending on the command they are used in.

- **Operators**: Keywords that reveal a relationship between two expressions and result in a true or false value.

- **Value expressions**: Used in a variety of contexts, such as in the target list of the SELECT command, or in search conditions in a number of commands. The result of a value expression is sometimes called a scalar, to distinguish it from the result of a table expression (which is a table). Value expressions are therefore also called scalar expressions (or even simply expressions). The expression syntax allows the calculation of values from primitive parts using arithmetic, logical, set, and other operations.

  A value expression is one of the following:

  - A constant or literal value
  - A column reference

## 2. Scope
NQL is differentiated from other SQL engines due to its native integration with the Narrative platform. Specifically, NQL interoperates with the following components of the Narrative platform. 

### 2.1 Datasets
 A dataset is a system of record for any row of data that is ingressed, egressed, or registered on a supported storage engine in the Narrative platform. Datasets are identifiable by their unique identifiers and a unique name or "slug" that is unique within the Company's namespace. Datasets define the schema of the underlying data contained within the dataset. Schemas are immutable. 

- NQL ***SHALL*** allow querying of data that resides in a Dataset, via an Access Rule.

### 2.2 Rosetta Stone Attribute Catalog

The Narrative Data Collaboration Platform provides a standardized global attribute catalog known as the "Rosetta Stone" for NQL-based queries. This catalog comprises Rosetta Stone Attributes—data points normalized and aggregated from multiple sources across participating companies. Queries against these attributes are permissible only if the dataset containing them is governed by an appropriate [Access Rule](RFC_Link_Here) (also see [Knowledge Base Article](KB_Article_Link_Here) for more details).

Users MAY utilize Rosetta Stone Attributes to query datasets within their own company account or any other account that has granted access permissions.

Attributes SHALL be of the following types: string, long, double, timestamptz, or object. Object types MUST contain nested properties. To reference nested properties within an object attribute, dot-notation SHALL be used.

### 2.2. Permissions (Access Rules)

Permissions within the NQL environment are regulated through Access Rules. Each Access Rule ***MUST*** delineate:

- The specific dataset it applies to.
- The authorized principals, which ***MAY*** be companies or individual users.
- The fields within the dataset that are eligible for querying.
- Any applicable pricing model for executing a query.
- Additional constraints for query analysis, if any.

By default, users ***SHALL*** be able to execute queries on all tables for which they possess the requisite permissions, given that the query adheres to established rules and syntax.


## 3. Supported Keywords

NQL ***SHALL*** support the following keywords.

> **Note**: This is an incomplete list. For a complete list, refer to the [Calcite documentation on supported keywords](https://calcite.apache.org/docs/reference.html). Assume these keywords are supported **UNLESS OTHERWISE NOTED**.

- ***SELECT***: Executes a command with a "select list," which is a comma-separated list of value expressions.
  
- ***FROM***: Utilizes a comma-separated table reference list to derive a table.

- ***WHERE***: Adheres to the syntax `WHERE <search_condition>`, where `<search_condition>` is a boolean value expression.

- ***LIMIT***: Specifies a budget constraint in the format `LIMIT <amount> <unit>`, e.g., `LIMIT 100 USD`. This keyword restricts the number of rows returned, without exceeding the given limit. `LIMIT 0 USD PER CALENDAR_MONTH` implies purchasing only data with zero cost. If `LIMIT` is not included, a limit of 0 is assumed. To specify no budget constraint, `NO LIMIT` can be explicitly set.

- ***USD***: Denotes that the `LIMIT` is set in US Dollars. This is the only supported unit for `LIMIT` currently.

- ***PER***: Indicates the period over which the `LIMIT` applies.

- ***CALENDAR_MONTH***: Represents a calendar month as the time period for `LIMIT`.

- ***CALENDAR_DAY***: Represents a calendar day as the time period for `LIMIT`.

- ***AS***: Renames a column temporarily using an alias, effective only for the duration of the query.

## 4. Supported Operators

**Logical Operators (AND, OR, NOT)**: Logical negation, conjugation, and negation.

**Comparison Operators (<, >, <=, >=, =, <>, !=)**: These comparison operators are available for all built-in data types that have a natural ordering, including numeric, string, and date/time types. In addition, arrays, composite types, and ranges can be compared if their component data types are comparable. 

**Null Aware Operators (IS, IS NOT)**: Ordinary comparison operators yield null (signifying "unknown"), not true or false, when either input is null. For example, 7 = NULL yields null, as does 7 <> NULL. When this behavior is not suitable, use the IS [ NOT ] predicates
   
## 5. Identifiers 

### 5.1 Reserved Fields

#### 5.1.1 `_price_cpm_usd`
- **Definition**: Specifies the maximum cost-per-mille (CPM) in US Dollars that the querier is willing to pay for 1000 rows of data.
- **Data Type**: Numeric
- **Usage**: Can be used in the `WHERE` clause as a filtering criterion and in the `SELECT` clause as an output column.
- **Constraints**: Value must be a positive numeric value, up to two decimal places, or the value 0.
- - Setting a price (CPM) to 0 ***SHALL*** explicitly filter out access rules with a price, thereby only querying data that is freely available. 
- - Omitting a CPM filter ***SHALL*** apply no filter, allowing targeting of data at any price. This is particularly useful for `EXPLAIN` statements to understand available data, but not recommended for `CREATE MATERIALIZED VIEW` statements due to potential high costs.

#### 5.1.2 `_access_rule_id` (NOT YET IMPLEMENTED)
- **Definition**: Identifier for the Access Rule that governs the query's permissions.
- **Data Type**: String or Numeric ID
- **Usage**: Typically used in the `WHERE` clause to specify which Access Rule to apply for the query. Can also appear in the `SELECT` clause for debugging or auditing.
- **Constraints**: Must match an existing Access Rule ID.

#### 5.1.3 `_source_company_id` (NOT YET IMPLEMENTED)
- **Definition**: Identifier for the company or entity providing the Access Rule.
- **Data Type**: String or Numeric ID
- **Usage**: Used in the `WHERE` clause to specify data shared by a particular provider. Can also be used in the `SELECT` clause for output.
- **Constraints**: Must match an existing provider ID.

#### 5.1.4 `_source_company_slug` 
- **Definition**: A unique identifier for a company usually created off of a company's name.
- **Data Type**: String 
- **Usage**: Used in the `WHERE` clause to specify data shared by a particular provider. Can also be used in the `SELECT` clause for output.
- **Constraints**: Must be globally unique and only contain alphanumeric characters.

#### 5.1.5 `_access_rule_name` 
- **Definition**: a unique, human-readable name for a specific access rule in a company seat.
- **Data Type**: String 
- **Usage**:  Typically used in the `WHERE` clause to specify which Access Rule to apply for the query. Can also appear in the `SELECT` clause for debugging or auditing.
- **Constraints**: Must be unique within a company seat and only contain alphanumeric characters.

### 5.2 Identifier Quoting and Referencing 

- **Quoting Rule**: Identifiers not starting with a lowercase letter [a-z] ***MUST*** be enclosed in double quotation marks (`"`).
  
- **Strict Mode**: NQL operates in "strict mode," meaning most identifiers ***MUST*** be fully qualified. NQL ***SHALL NOT*** attempt to infer identifiers based on command context.

## 6. SELECT COMMAND

- A select_list **MUST** NOT use ``*`` notation. 

- A *FROM* clause **MUST** contain only a single table name.

- A *FROM* clause **MUST** contain the table name. Throws error if the table name is missing or if the buyer does not have access to the table. 

- select_list **MUST** only contain columns available in the underlying dataset. Throws an error if a column is requested that does not exist in the dataset.

## 7. FROM CLAUSE

- The *FROM* clause **MUST** contain both the schema name as well as the table name. Throws an error if wrong format.

- The *FROM* clause schema **MUST** be one of `company_data`, `narrative`. Error if not one of three valid values. 

- The user that is running the query **MUST** have access to the dataset being referenced in dataset_source. If the user does not have a valid access rule for the dataset a `Unknown table or insufficient table permissions` error should be thrown.

## 8. Search Conditions

- search_conditions **MUST** contain database and columns in reference

- search_conditions **MUST** only contain references to columns that are present in the underlying dataset OR Narrative reserved filter fields. If not, error is thrown with missing column or reserved field

- search_conditions **MUST** contain at least one filter related to price for each table referenced in the query. Currently only `_price_cpm_usd` is supported. Value must be >=0 and only the <= expression is supported. If not throw error with easy to understand error message

- *LIMIT* (Dollar Limit) expression **MUST** be present. Error if no LIMIT is present. To express no budget the proper syntax is `*LIMIT ALL*` 
   
## 9. Query Output

- Successful *SELECT* statements **MUST** return a message that the query has been accepted along with the dataset id that has been created and will be populated by the query

## 10. Query Features and Constraints

### 10.1 Cost Control in Queries

Queries in NQL can implement cost controls through two primary mechanisms: filtering based on the unit price per row and setting an overall budget for the query.

```sql
SELECT <select_list> FROM <dataset_source> WHERE <search_condition> [LIMIT <amount> <unit> PER <period>]
SELECT [...table.column] | [table.*] FROM [schema.table] WHERE [...table.column <filter_expression>] [_price_cpm_usd <= <price>] [LIMIT 100.50 USD PER CALENDAR_MONTH]
```

1. **Unit Price Filtering (`_price_cpm_usd`):**
   - This filter allows users to specify the maximum cost-per-mille (CPM) in US Dollars that they are willing to pay for 1000 rows of data.
   - Setting `_price_cpm_usd` to a specific value filters out data that costs more than the specified price, thereby querying data at or below the set price.
   - Setting `_price_cpm_usd` to 0 explicitly filters out any data that has a cost, thus only querying free data.
   - Omitting the `_price_cpm_usd` filter means there is no price filter applied, allowing the query to target data at any price.

2. **Budget Constraint (`LIMIT`):**
   - The `LIMIT` clause in a query sets a budget constraint, specified as `LIMIT <amount> <unit>`, for example, `LIMIT 100 USD`.
   - Using `LIMIT 0 USD PER CALENDAR_MONTH` indicates that only data with zero cost is to be purchased.
   - If the `LIMIT` clause is not included in a query, a default limit of 0 is assumed, implying no budget for purchasing data.
   - To specify that there is no budget constraint, the `NO LIMIT` clause can be used, allowing the purchase of data at any price.

These cost control mechanisms ensure that users can manage their data querying expenses effectively while accessing the necessary data.


### 10.2 Data and Cost Forecasting

Using `EXPLAIN <query>`, a forecast can be generated. This forecast estimates both the number of rows returned by the query and the associated cost to purchase the data, if any, according to the access rules that grant access. 

### 10.3 CREATING MATERIALIZED VIEWS

#### 10.3.1 Materialized Views
In databases that support them, a materialized view is a database object that stores the result of a query physically. It provides a way to cache expensive query results and improve query performance by reading from this pre-computed result set, which can be refreshed periodically or on-demand. Similarly, in NQL, creating a materialized view creates a new, unique dataset within the Narrative Data Collaboration Platform that can be further queried or actioned on downstream. 

Creating a materialized view effectively creates a new dataset with a unique name. Such datasets cannot ingest data from other sources. Data purchase costs may or may not be incurred when executing a query that materializes data as a new dataset, depending on the underlying query's access rules.  

#### 10.3.2 Materialized View Syntax

```sql
CREATE MATERIALIZED VIEW "<view_name>"
[ DISPLAY_NAME = '<display_name>' ] 	
[ DESCRIPTION = '<description_text>' ] 	
[ EXPIRE = { <ISO 8601 PERIOD> } // Supported syntax: "expire_when > P60", default retains all data. ]
[ STATUS = { 'active' | 'updating' | 'draft' } ] 
[ TAGS = ( '_nio_materialized_view', '<tag1>', ... ) ]
[ WRITE_MODE = { 'append' | 'overwrite' } ] 
[ EXTENDED_STATS = { 'all' | 'none' } ] 
[ PARTITIONED_BY <field> <transform>, <field2> <transform> ]
[ REFRESH_SCHEDULE = { '@hourly' | '@daily' | '@weekly' | '@monthly' | '@once' | cron expression} ]
AS SELECT <column_names> FROM <table_name>
```

Note: Materialized view names must be slugified and unique within a company's dataset space, using only alphanumeric and underscore characters. Typically, these names should be enclosed in double quotation marks.

#### 10.3.3 Materialized View Parameters
The following parameters apply to the dataset that is generated by the `CREATE MATERIALIZE VIEW` command: 

- `DISPLAY_NAME`: Human-readable name, usually describing the business context of the dataset.
    - Type: string
    - Default: Empty or generated by RosettaAI.
    
- `DESCRIPTION`: Human-readable description providing business context.
    - Type: string
    - Default: Empty or generated by RosettaAI.
  
- `EXPIRE`: Setting for the [retention policy](https://api.narrative.dev/#tag/Datasets/paths/~1datasets~1%7Bdataset_id%7D~1admin~1retention-policy~1preview/put).
    - Allowed Values:
        - `expireWhen > ISO PERIOD`
        - `retain_everything`
        - `expire_everything`
    - Type: enum
    - Default: `retain_everything`
    
- `STATUS`: Status of the materialized view.
    - Allowed Values:
        - `active`
        - `draft` (not yet implemented)
        - `updating`
    - Type: enum
    - Default: `active`

- `TAGS`: Default tag `_nio_materialized_view` is added. Users can add more.
    - Type: array
    - Default: Empty or generated by RosettaAI.
    
- `WRITE_MODE`: Write mode of the resulting dataset.
    - Allowed Values:
        - `append`
        - `overwrite`
    - Type: enum
    - Default: `append`
    
- `EXTENDED_STATS`: Advanced dataset statistics.
    - Allowed Values:
        - `all`
        - `none`
    - Type: enum
    - Default: `all`

- `PARTITIONED_BY`: Output dataset partitions defined by sets of `field` + `transformation`.
    - Type: expression
    - Default: Narrative sample partition is always present; users can add additional partitions. 

- `REFRESH_SCHEDULE`: Defines the frequency of updates for the materialized view. 
    - Allowed Values:
      - '@hourly'
      - '@daily'
      - '@weekly'
      - '@monthly'
      - '@once'
      - cron expressions
    - Type: enum | cron 
    - Default: '@once' 


### 10.5 Specialized Functions in NQL

NQL also supports a variety of specialized functions or User-Defined Functions (UDFs) to cater to specific use-cases.

#### 10.5.1 `ADDRESS_HASHES()`

The `ADDRESS_HASHES()` function generates libpostal address hashes from an unstructured address string. This is especially useful for conducting fuzzy address comparisons where exact string matching isn't sufficient. By hashing the addresses and then joining two input lists based on these hashes, users can find approximate address matches with high efficiency.

##### Example:

```sql
CREATE MATERIALIZED VIEW "address_hashes_sample_v2" AS
SELECT
  address_hashes(
    concat_ws(
      ' ',
      lower(UnitDesignator),
      lower(UnitDesignatorNumber),
      HouseNumber,
      lower(PrimaryAddress),
      lower(PreDirection),
      lower(StreetName),
      lower(StreetSuffix),
      lower(PostDirection),
      ZIPCode,
      ZIP4,
      lower(CityName),
      StateAbbreviation
    )
  ) as address_hashes,
...
```
#### 10.5.2 `country_code_3_to_2()`

The country_code_3_to_2('column') function takes in a column of ISO 3166-1 alpha-3 country code(s) and converts them to The ISO 3166-1 alpha-2 country code(s). The function is useful for matching the standard output for the `iso_3166_1_country` Rosetta Stone attribute, expressed as ISO 3166-1 alpha-2 country codes. 

##### Example:

```sql
CREATE MATERIALIZED VIEW "country_code_sample" AS
SELECT
  country_code_3_to_2(my_dataset.country_code) as two_letter_codes
FROM 
  company_date.my_dataset
...
```

### 10.6 Querying an Access Rule Directly 

An access rule has two identifiers: a `name` and an `id`. NQL supports querying internal or external access rules directly, in addition to the Rosetta Stone attribute catalog and dataset ids, through `_access_rule_name` and not `access_rule_id`. 

#### 10.6.1 Querying Internal Access Rules 

An access rule name, defined in section 5.1.5 as a unique, human-readable name for a specific access rule in a company seat, is added after the company identifier. When querying data in your own company seat, an access rule name always follows `company_data`.  

##### Example 

```sql
SELECT pd.hashed_emails
FROM company_data.access_rule_for_private_deal pd
```

#### 10.6.2 Querying External Access Rules 

An access rule name, defined in section 5.1.5 as a unique, human-readable name for a specific access rule in a company seat, is added after the company identifier. When querying data in an external company seat, an access rule name always follows `_source_company_slug`.  

##### Example 

```sql
SELECT teams.baseball_teams
FROM _source_company_id.access_rule_unique_name_1 teams
```

### 10.7 Embedded Namespaces in NQL 

#### 10.7.1 `_rosetta_stone`

 NQL supports Attribute querying via the Rosetta Embedded Namespace is facilitated by `_rosetta_stone`, a direct method to query Rosetta Stone attributes. `_rosetta_stone` acts as an attribute reference within the dataset or access rule. `_rosetta_stone` must follow either a dataset's `unique_name`, a dataset's `id`, or an access rule's `name`. In case of an absence of mappings or an incorrect attribute reference, the query will return an error.

 ##### Basic Usage
    
    ```sql
    SELECT ds_identifier._rosetta_stone."attribute_name" AS alias_name
    FROM dataset_source
    ```
    
    - **`ds_identifier`**: Alias or identifier for the dataset. A dataset can be referenced by its `id` or `unique_name`. 
    - **`attribute_name`**: The name of the Rosetta Stone attribute that is being selected.
    - **`alias_name`**: An optional alias for the selected attribute.

##### Example with Single Dataset
    
    ```sql
    SELECT ds_123._rosetta_stone."event_timestamp" AS event_time
    FROM company_data.ds_123 AS ds_123
    ```
    
##### Example Joining Multiple Datasets
    
    ```sql
    SELECT
        ds_123._rosetta_stone."attribute_1" AS attribute_from_a,
        ds_456._rosetta_stone."attribute_2" AS attribute_from_b,
        ds_123.email,
        ds_456.username
    FROM
        company_data.ds_123 AS ds_123
    JOIN
        company_data.ds_456 AS ds_456
    ON
        ds_123.user_id = ds_456.user_id
    ```
    
    In this example:
    
    - The first Rosetta Stone attribute (**`attribute_1`**) is being pulled from dataset **`ds_123`**.
    - The second Rosetta Stone attribute (**`attribute_2`**) is being pulled from dataset **`ds_456`**.

##### Example of Filtering with Rosetta Attributes
    
    ```sql
    SELECT
        ds_123._rosetta_stone."unique_id"."value" AS id
    FROM
        company_data.ds_123 AS ds_123
    WHERE
        ds_123.id = 123
    ```
    
    Here, the **`WHERE`** clause uses the Rosetta attribute **`unique_id.value`** from dataset **`ds_123`** for filtering.
    
##### Example of Nested Properties
  For nested properties, the same dot notation is used within the **`_rosetta_stone`** namespace.
    
    ```sql
    SELECT
        ds_123._rosetta_stone."nested"."attribute" AS nested_attribute
    FROM
        company_data.ds_123 AS ds_123
    ```

## 12. Example Queries

Querying Rosetta Stone Attributes:

```sql
EXPLAIN
SELECT
  narrative.rosetta_stone."unique_id"."value",
  narrative.rosetta_stone."hl7_gender"."gender"  
FROM
  narrative.rosetta_stone
WHERE 
  narrative.rosetta_stone."hl7_gender"."gender" = 'female'
  AND narrative.rosetta_stone."event_timestamp" > '2023-10-01' 
  AND narrative.rosetta_stone._price_cpm_usd <= 2.0
```

Single Dataset:

```sql
CREATE MATERIALIZED VIEW "running_times" AS

SELECT 
  company_data."20"."Workout_Timestamp",
  company_data."20"."Total_Output" AS output
FROM 
  company_data."20"
WHERE
  company_data."20"."Total_Output" > 200 
  AND company_data."20"."Fitness_Discipline" = 'Running'
  AND company_data."20"._price_cpm_usd <= 1.00
  LIMIT
  50 USD PER CALENDAR_MONTH
```

Joining datasets:

```sql
EXPLAIN
SELECT 
  rs."unique_id"."value",
  rs."hl7_gender"."gender",
  cd."Sha_256_Client_Email"
FROM
  "narrative"."rosetta_stone" AS rs
JOIN 
  "company_data"."833" AS cd
ON 
  rs."unique_id"."value" = cd."Sha_256_Client_Email"
WHERE 
  rs."hl7_gender"."gender" = 'female'
  AND rs."event_timestamp" > '2023-10-01'
  AND rs._price_cpm_usd <= 2.0
  ```

# APPENDIX

# CHANGE LOG

## Update 2023-11-05 
### Section 2 - Scope
- Revised to highlight NQL's integration with the Narrative platform.
- Expanded the definition of Datasets to include their role as a system of record for data managed within the Narrative platform, emphasizing their unique identifiers, immutable schemas, and namespace uniqueness.

### Section 3 - Supported Keywords
- Updated the LIMIT keyword description to clarify its role in budget constraints and the implications of using LIMIT 0 USD PER CALENDAR_MONTH and NO LIMIT.

### Section 5.1.1 - _price_cpm_usd
- Enhanced the definition to specify the behavior when setting the price (CPM) to 0, emphasizing its role in filtering out priced access rules and querying only free data.
- Clarified the implications of omitting a CPM filter, noting its utility in EXPLAIN statements and caution in CREATE MATERIALIZED VIEW statements due to potential cost implications.


## OUT OF SCOPE FOR THIS VERSION:

### DATA MANIPULATION

1. UPDATE 

2. DELETE

## POSSIBLE FUTURE SCOPE:

### EXPORT DATA / CONNECTORS  

*EXPORT DATA OPTIONS (**export_option_list**) FROM <table_name>*

### TABLE / DATASET STATISTICS

1. Show Table Schema
   
    *DESCRIBE SCHEMA <table_name>*

2. Show Table Statistics
   
    *DESCRIBE STATS <table_name>
    ANALYZE <table_name>;*

### DATASET DELETION
   
DROP TABLE <table_name>;
