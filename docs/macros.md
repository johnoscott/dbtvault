## Table templates 
######(macros/tables)

These macros form the core of the package and can be called in your models to build the tables for your Data Vault.
___

### hub_template

Creates a hub with provided metadata.

```mysql 
dbtvault.hub_template(src_pk, src_nk, src_ldts, src_source,          
                      tgt_pk, tgt_nk, tgt_ldts, tgt_source,
                      source)                                        
```

#### Parameters

| Parameter     | Description                                         | Type (Single-Source) | Type (Union)         | Required?                                                          |
| ------------- | --------------------------------------------------- | -------------------- | -------------------- | ------------------------------------------------------------------ |
| src_pk        | Source primary key column                           | String               | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| src_nk        | Source natural key column                           | String               | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| src_ldts      | Source loaddate timestamp column                    | String               | String               | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| src_source    | Name of the column containing the source ID         | String               | String               | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| tgt_pk        | Target primary key column                           | List                 | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| tgt_nk        | Target natural key column                           | List                 | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| tgt_ldts      | Target loaddate timestamp column                    | List                 | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| tgt_source    | Name of the column which will contain the source ID | List                 | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| source        | Staging model reference or table name               | List                 | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |

#### Usage

``` yaml tab="Single-Source"

hub_customer.sql:

{{- config(...)                                                                         -}}
                                                                                           
{%- set src_pk = 'CUSTOMER_PK'                                                          -%}
{%- set src_nk = 'CUSTOMER_ID'                                                          -%}
{%- set src_ldts = 'LOADDATE'                                                           -%}
{%- set src_source = 'SOURCE'                                                           -%}                                                                                         
                                                                                           
{%- set tgt_pk = [src_pk, 'BINARY(16)', src_pk]                                         -%}
{%- set tgt_nk = [src_nk, 'VARCHAR(38)', src_nk]                                        -%}
{%- set tgt_ldts = [src_ldts, 'DATE', src_ldts]                                         -%}
{%- set tgt_source = [src_source, 'VARCHAR(15)', src_source]                            -%}
                                                                                           
{%- set source = [ref('stg_customer_hashed')]                                           -%}
                                                                                           
{{ dbtvault.hub_template(src_pk, src_nk, src_ldts, src_source,                             
                         tgt_pk, tgt_nk, tgt_ldts, tgt_source,                   
                         source)                                                         }}
```

``` yaml tab="Union"

hub_parts.sql:

{{- config(...)                                                                         -}}

{%- set src_pk = ['PART_PK', 'PART_PK', 'PART_PK']                                      -%}
{%- set src_nk = ['PART_ID', 'PART_ID', 'PART_ID']                                      -%}
{%- set src_ldts = 'LOADDATE'                                                           -%}
{%- set src_source = 'SOURCE'                                                           -%}

{%- set tgt_pk = [src_pk[0], 'BINARY(16)', src_pk[0]]                                   -%}
{%- set tgt_nk = [src_nk[0], 'NUMBER(38,0)', src_nk[0]]                                 -%}
{%- set tgt_ldts = [src_ldts, 'DATE', src_ldts]                                         -%}
{%- set tgt_source = [src_source, 'VARCHAR(15)', src_source]                            -%}

{%- set source = [ref('stg_parts_hashed'),
                  ref('stg_supplier_hashed'),
                  ref('stg_lineitem_hashed')]                                           -%}


{{ dbtvault.hub_template(src_pk, src_nk, src_ldts, src_source,
                         tgt_pk, tgt_nk, tgt_ldts, tgt_source,
                         source)                                                         }}
```


#### Output

```mysql tab="Single-Source"
SELECT DISTINCT 
                    CAST(stg.CUSTOMER_PK AS BINARY(16)) AS CUSTOMER_PK,
                    CAST(stg.CUSTOMER_ID AS VARCHAR(38)) AS CUSTOMER_ID,
                    CAST(stg.LOADDATE AS DATE) AS LOADDATE,
                    CAST(stg.SOURCE AS VARCHAR(15)) AS SOURCE
FROM (
    SELECT a.CUSTOMER_PK, a.CUSTOMER_ID, a.LOADDATE, a.SOURCE
      FROM MYDATABASE.MYSCHEMA.stg_customer_hashed AS a
) AS stg
LEFT JOIN MYDATABASE.MYSCHEMA.hub_customer AS tgt
ON stg.CUSTOMER_PK = tgt.CUSTOMER_PK
WHERE tgt.CUSTOMER_PK IS NULL
```

```mysql tab="Union"
SELECT DISTINCT 
                    CAST(stg.PART_PK AS BINARY(16)) AS PART_PK,
                    CAST(stg.PART_ID AS NUMBER(38,0)) AS PART_ID,
                    CAST(stg.LOADDATE AS DATE) AS LOADDATE,
                    CAST(stg.SOURCE AS VARCHAR(15)) AS SOURCE
FROM (
    SELECT src.PART_PK, src.PART_ID, src.LOADDATE, src.SOURCE,
    LAG(SOURCE, 1)
    OVER(PARTITION by PART_PK
    ORDER BY PART_PK) AS FIRST_SOURCE
    FROM (
      SELECT a.PART_PK, a.PART_ID, a.LOADDATE, a.SOURCE
      FROM MYDATABASE.MYSCHEMA.stg_parts_hashed AS a
      UNION
      SELECT b.PART_PK, b.PART_ID, b.LOADDATE, b.SOURCE
      FROM MYDATABASE.MYSCHEMA.stg_supplier_hashed AS b
      UNION
      SELECT c.PART_PK, c.PART_ID, c.LOADDATE, c.SOURCE
      FROM MYDATABASE.MYSCHEMA.stg_lineitem_hashed AS c
      ) as src
) AS stg
LEFT JOIN MYDATABASE.MYSCHEMA.hub_parts AS tgt
ON stg.PART_PK = tgt.PART_PK
WHERE tgt.PART_PK IS NULL
AND stg.FIRST_SOURCE IS NULL
```

___

### link_template

Creates a link with provided metadata.

```mysql 
dbtvault.link_template(src_pk, src_fk, src_ldts, src_source,          
                       tgt_pk, tgt_fk, tgt_ldts, tgt_source,
                       source)                                        
```

#### Parameters

| Parameter     | Description                                         | Type (Single-Source) | Type (Union)         | Required?                                                          |
| ------------- | --------------------------------------------------- | ---------------------| ---------------------| ------------------------------------------------------------------ |
| src_pk        | Source primary key column                           | String               | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| src_fk        | Source foreign key column                           | List                 | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| src_ldts      | Source loaddate timestamp column                    | String               | String               | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| src_source    | Name of the column containing the source ID         | String               | String               | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| tgt_pk        | Target primary key column                           | List                 | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| tgt_fk        | Target foreign key column                           | List                 | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| tgt_ldts      | Target loaddate timestamp column                    | List                 | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| tgt_source    | Name of the column which will contain the source ID | List                 | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| source        | Staging model reference or table name               | List                 | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |

#### Usage

``` yaml tab="Single-Source"

link_customer_nation.sql:

{{- config(...)                                                                             -}}

{%- set src_pk = 'CUSTOMER_NATION_PK'                                                       -%}
{%- set src_fk = ['CUSTOMER_PK', 'NATION_PK']                                               -%}
{%- set src_ldts = 'LOADDATE'                                                               -%}
{%- set src_source = 'SOURCE'                                                               -%}

{%- set tgt_pk = [src_pk, 'BINARY(16)', src_pk]                                             -%}
{%- set tgt_fk = [['CUSTOMER_PK', 'BINARY(16)', 'CUSTOMER_FK'],
                  ['NATION_PK', 'BINARY(16)', 'NATION_FK']]                                 -%}

{%- set tgt_ldts = [src_ldts, 'DATE', src_ldts]                                             -%}
{%- set tgt_source = [src_source, 'VARCHAR(15)', src_source]                                -%}

{%- set source = [ref('stg_crm_customer_hashed')]                                           -%}

{{ dbtvault.link_template(src_pk, src_fk, src_ldts, src_source,
                          tgt_pk, tgt_fk, tgt_ldts, tgt_source,
                          source)                                                            }}
```

``` yaml tab="Union"

link_customer_nation_union.sql:  

{{- config(...)                                                                             -}}

{%- set src_pk = ['CUSTOMER_NATION_PK', 'CUSTOMER_NATION_PK', 'CUSTOMER_NATION_PK']         -%}

{%- set src_fk = [['CUSTOMER_PK', 'NATION_PK'], ['CUSTOMER_PK', 'NATION_PK'],
                  ['CUSTOMER_PK', 'NATION_PK']]                                             -%}

{%- set src_ldts = 'LOADDATE'                                                               -%}
{%- set src_source = 'SOURCE'                                                               -%}

{%- set tgt_pk = [src_pk[0], 'BINARY(16)', src_pk[0]]                                       -%}
{%- set tgt_fk = [['CUSTOMER_PK', 'BINARY(16)', 'CUSTOMER_FK'],
                  ['NATION_PK', 'BINARY(16)', 'NATION_FK']]                                 -%}

{%- set tgt_ldts = [src_ldts, 'DATE', src_ldts]                                             -%}
{%- set tgt_source = [src_source, 'VARCHAR(15)', src_source]                                -%}

{%- set source = [ref('stg_sap_customer_hashed'),
                  ref('stg_crm_customer_hashed'),
                  ref('stg_web_customer_hashed')]                                           -%}

{{ dbtvault.link_template(src_pk, src_fk, src_ldts, src_source,
                          tgt_pk, tgt_fk, tgt_ldts, tgt_source,
                          source)                                                            }}
```


#### Output

```mysql tab="Single-Source"
SELECT DISTINCT 
                    CAST(stg.CUSTOMER_NATION_PK AS BINARY(16)) AS CUSTOMER_NATION_PK,
                    CAST(stg.CUSTOMER_PK AS BINARY(16)) AS CUSTOMER_FK,
                    CAST(stg.NATION_PK AS BINARY(16)) AS NATION_FK,
                    CAST(stg.LOADDATE AS DATE) AS LOADDATE,
                    CAST(stg.SOURCE AS VARCHAR(15)) AS SOURCE
FROM (
    SELECT a.CUSTOMER_NATION_PK, a.CUSTOMER_PK, a.NATION_PK, a.LOADDATE, a.SOURCE
      FROM MYDATABASE.MYSCHEMA.stg_crm_customer_hashed AS a
) AS stg
LEFT JOIN MYDATABASE.MYSCHEMA.link_customer_nation AS tgt
ON stg.CUSTOMER_NATION_PK = tgt.CUSTOMER_NATION_PK
WHERE tgt.CUSTOMER_NATION_PK IS NULL
```

```mysql tab="Union"
SELECT DISTINCT 
                    CAST(stg.CUSTOMER_NATION_PK AS BINARY(16)) AS CUSTOMER_NATION_PK,
                    CAST(stg.CUSTOMER_PK AS BINARY(16)) AS CUSTOMER_FK,
                    CAST(stg.NATION_PK AS BINARY(16)) AS NATION_FK,
                    CAST(stg.LOADDATE AS DATE) AS LOADDATE,
                    CAST(stg.SOURCE AS VARCHAR(15)) AS SOURCE
FROM (
    SELECT src.CUSTOMER_NATION_PK, src.CUSTOMER_PK, src.NATION_PK, src.LOADDATE, src.SOURCE,
    LAG(SOURCE, 1)
    OVER(PARTITION by CUSTOMER_NATION_PK
    ORDER BY CUSTOMER_NATION_PK) AS FIRST_SOURCE
    FROM (
      SELECT a.CUSTOMER_NATION_PK, a.CUSTOMER_PK, a.NATION_PK, a.LOADDATE, a.SOURCE
      FROM MYDATABASE.MYSCHEMA.stg_sap_customer_hashed AS a
      UNION
      SELECT b.CUSTOMER_NATION_PK, b.CUSTOMER_PK, b.NATION_PK, b.LOADDATE, b.SOURCE
      FROM MYDATABASE.MYSCHEMA.stg_crm_customer_hashed AS b
      UNION
      SELECT c.CUSTOMER_NATION_PK, c.CUSTOMER_PK, c.NATION_PK, c.LOADDATE, c.SOURCE
      FROM MYDATABASE.MYSCHEMA.stg_web_customer_hashed AS c
      ) AS src
) AS stg
LEFT JOIN MYDATABASE.MYSCHEMA.link_customer_nation_union AS tgt
ON stg.CUSTOMER_NATION_PK = tgt.CUSTOMER_NATION_PK
WHERE tgt.CUSTOMER_NATION_PK IS NULL
AND stg.FIRST_SOURCE IS NULL
```

___

### sat_template

Creates a satellite with provided metadata.

```mysql 
dbtvault.sat_template(src_pk, src_hashdiff, src_payload,          
                      src_eff, src_ldts, src_source,              
                      tgt_pk, tgt_hashdiff, tgt_payload,
                      tgt_eff, tgt_ldts, tgt_source,              
                      src_table, source)                          
```

#### Parameters

| Parameter     | Description                                         | Type                 | Required?                                                          |
| ------------- | --------------------------------------------------- | ---------------------| ------------------------------------------------------------------ |
| src_pk        | Source primary key column                           | String               | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| src_hashdiff  | Source hashdiff column                              | String               | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| src_payload   | Source payload column(s)                            | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| src_eff       | Source effective from column                        | String               | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| src_ldts      | Source loaddate timestamp column                    | String               | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| src_source    | Name of the column containing the source ID         | String               | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| tgt_pk        | Target primary key column                           | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| tgt_hashdiff  | Target hashdiff column                              | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| tgt_payload   | Target payload column                               | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| tgt_eff       | Target effective from column                        | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| tgt_ldts      | Target loaddate timestamp column                    | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| tgt_source    | Name of the column which will contain the source ID | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |
| source        | Staging model reference or table name               | List                 | <i class="md-icon" alt="Yes" style="color: green">check_circle</i> |

#### Usage


``` yaml

sat_customer_details.sql:  

{{- config(...)                                                                                         -}}

{%- set src_pk = 'CUSTOMER_PK'                                                                          -%}
{%- set src_hashdiff = 'CUSTOMER_HASHDIFF'                                                              -%}
{%- set src_payload = ['CUSTOMER_NAME', 'CUSTOMER_DOB', 'CUSTOMER_PHONE']                               -%}
                                                                                                           
{%- set src_eff = 'EFFECTIVE_FROM'                                                                      -%}
{%- set src_ldts = 'LOADDATE'                                                                           -%}
{%- set src_source = 'SOURCE'                                                                           -%}
                                                                                                           
{%- set tgt_pk = [src_pk , 'BINARY(16)', src_pk]                                                        -%} 

{%- set tgt_hashdiff = [ src_hashdiff , 'BINARY(16)', 'HASHDIFF']                                       -%} 

{%- set tgt_payload = [[ src_payload[0], 'VARCHAR(60)', 'NAME'],                                           
                       [ src_payload[1], 'DATE', 'DOB'],                                                   
                       [ src_payload[2], 'VARCHAR(15)', 'PHONE']]                                       -%}
                                                                                                           
{%- set tgt_eff = ['EFFECTIVE_FROM', 'DATE', 'EFFECTIVE_FROM']                                          -%}
{%- set tgt_ldts = ['LOADDATE', 'DATE', 'LOADDATE']                                                     -%}
{%- set tgt_source = ['SOURCE', 'VARCHAR(15)', 'SOURCE']                                                -%}
                                                                                                           
{%- set source = [ref('stg_customer_details_hashed')]                                                   -%}
                                                                                                           
{{  dbtvault.sat_template(src_pk, src_hashdiff, src_payload,                                               
                          src_eff, src_ldts, src_source,                                                   
                          tgt_pk, tgt_hashdiff, tgt_payload,                                     
                          tgt_eff, tgt_ldts, tgt_source,                                                   
                          source)                                                                        }}
```


#### Output

```mysql 
SELECT DISTINCT 
                    CAST(e.CUSTOMER_HASHDIFF AS BINARY(16)) AS HASHDIFF,
                    CAST(e.CUSTOMER_PK AS BINARY(16)) AS CUSTOMER_PK,
                    CAST(e.CUSTOMER_NAME AS VARCHAR(60)) AS NAME,
                    CAST(e.CUSTOMER_DOB AS DATE) AS DOB,
                    CAST(e.CUSTOMER_PHONE AS VARCHAR(15)) AS PHONE,
                    CAST(e.LOADDATE AS DATE) AS LOADDATE,
                    CAST(e.EFFECTIVE_FROM AS DATE) AS EFFECTIVE_FROM,
                    CAST(e.SOURCE AS VARCHAR(15)) AS SOURCE
FROM MYDATABASE.MYSCHEMA.stg_customer_details_hashed AS e
LEFT JOIN (
    SELECT d.CUSTOMER_PK, d.HASHDIFF, d.NAME, d.DOB, d.PHONE, d.EFFECTIVE_FROM, d.LOADDATE, d.SOURCE
    FROM (
          SELECT c.CUSTOMER_PK, c.HASHDIFF, c.NAME, c.DOB, c.PHONE, c.EFFECTIVE_FROM, c.LOADDATE, c.SOURCE,
          CASE WHEN RANK()
          OVER (PARTITION BY c.CUSTOMER_PK
          ORDER BY c.LOADDATE DESC) = 1
          THEN 'Y' ELSE 'N' END CURR_FLG
          FROM (
            SELECT a.CUSTOMER_PK, a.HASHDIFF, a.NAME, a.DOB, a.PHONE, a.EFFECTIVE_FROM, a.LOADDATE, a.SOURCE
            FROM MYDATABASE.MYSCHEMA.sat_customer_details as a
            JOIN MYDATABASE.MYSCHEMA.stg_customer_details_hashed as b
            ON a.CUSTOMER_PK = b.CUSTOMER_PK
          ) as c
    ) AS d
WHERE d.CURR_FLG = 'Y') AS src
ON src.HASHDIFF = e.CUSTOMER_HASHDIFF
WHERE src.HASHDIFF IS NULL
```

___

## Staging Macros
######(macros/staging)

These macros are intended for use in the staging layer.
___

### multi_hash

!!! warning
    This macro ***should not be*** used for cryptographic purposes.
    
    The intended use is for creating checksum-like fields only, so that a record change can be detected. 
    
    [Read More](https://www.md5online.org/blog/why-md5-is-not-safe/)
    
!!! seealso "See Also"
    [hash](#hash)
    
A macro for generating multiple lines of hashing SQL for columns:
```sql 
CAST(MD5_BINARY(UPPER(TRIM(CAST(column1 AS VARCHAR)))) AS BINARY(16)) AS alias1,
CAST(MD5_BINARY(UPPER(TRIM(CAST(column2 AS VARCHAR)))) AS BINARY(16)) AS alias2
```

#### Parameters

| Parameter        |  Description                                    | Type     | Required?                                                |
| ---------------- | ----------------------------------------------  | -------- | -------------------------------------------------------- |
| pairs            | (column, alias) pair                            | Tuple    | <i class="md-icon" style="color: green">check_circle</i> |
| pairs: columns   | Single column string or list of columns         | String   | <i class="md-icon" style="color: green">check_circle</i> |
| pairs: alias     | The alias for the column                        | String   | <i class="md-icon" style="color: green">check_circle</i> |
| pairs: sort      | Will alpha sort columns if true, default false. | Boolean  | <i class="md-icon" style="color: red">clear</i>          |


#### Usage

```yaml
{{ dbtvault.multi_hash([('CUSTOMERKEY', 'CUSTOMER_PK'),
                        (['CUSTOMERKEY', 'NAME', 'PHONE', 'DOB'], 
                         'HASHDIFF', true)])                        }}
```

#### Output

```mysql
CAST(MD5_BINARY(UPPER(TRIM(CAST(CUSTOMERKEY AS VARCHAR)))) AS BINARY(16)) AS CUSTOMER_PK,

CAST(MD5_BINARY(CONCAT(
    IFNULL(UPPER(TRIM(CAST(CUSTOMERKEY AS VARCHAR))), '^^'), '||',
    IFNULL(UPPER(TRIM(CAST(DOB AS VARCHAR))), '^^'), '||',
    IFNULL(UPPER(TRIM(CAST(NAME AS VARCHAR))), '^^'), '||',
    IFNULL(UPPER(TRIM(CAST(PHONE AS VARCHAR))), '^^') )) AS BINARY(16)) AS HASHDIFF
```

!!! success "Column sorting"
    You do not need to worry about providing the columns in any particular order as long as you set the 
    ```sort``` flag to true when creating hashdiffs.

___

### add_columns

A simple macro for generating sequences of the following SQL:
```mysql 
column AS alias
```

#### Parameters

| Parameter     | Description                         | Type           | Required?                                       |
| ------------- | ----------------------------------- | -------------- | ----------------------------------------------- |
| source_table  | A source reference                  | Source         | <i class="md-icon" style="color: red">clear</i> |
| pairs         | List of (column, alias) pairs       | List of tuples | <i class="md-icon" style="color: red">clear</i> |

!!! note
    At least one of the above parameters must be provided, both may be provided if required.  

#### Usage

```yaml
{{ dbtvault.add_columns(source('MYSOURCE', 'MYTABLE'),
                        [('CURRENT_DATE()', 'EFFECTIVE_FROM'),
                         ('!STG_CUSTOMER', 'SOURCE')])                }}
```

#### Output

```mysql 
<All columns from MYTABLE>,
CURRENT_DATE() AS EFFECTIVE_FROM,
'STG_CUSTOMER' AS SOURCE
```

#### Notes

##### Adding constants
With the ```add_columns``` macro, you may provide constants. 
These are additional 'calculated' columns created from hard-coded values.
To achieve this, simply provide the constant with a ```!``` in front of the desired constant,
and the macro will do the rest. See line 3 above, and the output it gives.


##### Getting columns from the source
The ```add_columns``` macro will automatically select all columns from the provided source reference.
If you need to override any of these columns and provide different values, for example a different ```SOURCE``` 
or ```LOADDATE``` column value, then you may. If you provide columns in the ```pairs``` parameter, then they will 
automatically take precedence over any columns coming from the source. 

Database functions may be used, for example ```CURRENT_DATE()```, to set the current date as the value of a column, as in the
example below:

```sql
{{ dbtvault.add_columns(source_table,
                        [('!TPCH', 'SOURCE'),
                         ('CURRENT_DATE()', 'EFFECTIVE_FROM')])                              }}
```

    
___

### from

Used in creating source/hashing models to complete a staging layer model.

```mysql 
FROM MYDATABASE.MYSCHEMA.MYTABLE
```

!!! info
    Sources need to be set up in dbt. [Read More](https://docs.getdbt.com/docs/using-sources)

#### Parameters

| Parameter     | Description                               | Type   | Required?                                                |
| ------------- | ----------------------------------------- | ------ | -------------------------------------------------------- |
| source_table  | A source reference                        | Source | <i class="md-icon" style="color: green">check_circle</i> | 

#### Usage

```yaml
{{ dbtvault.from( source('MYSOURCE', 'MYTABLE') ) }}
```

#### Output

```mysql 
FROM MYDATABASE.MYSCHEMA.MYTABLE
```

___

## Supporting Macros
######(macros/supporting)

Supporting macros are helper functions for use in models. It should not be necessary to call these macros directly, however they 
are used extensively in the [table templates](#table-templates). 

___

### cast

A macro for generating cast sequences:

```mysql
CAST(prefix.column AS type) AS alias
```

#### Parameters

| Parameter        |  Description                  | Required?                                                |
| ---------------- | ----------------------------- | -------------------------------------------------------- |
| columns          |  Triples or strings           | <i class="md-icon" style="color: green">check_circle</i> |
| prefix           |  A string                     | <i class="md-icon" style="color: red">clear</i>          |

#### Usage

!!! note
    As shown in the snippet below, columns must be provided as a list. 
    The collection of items in this list can be any combination of:

    - ```(column, type, alias) ``` 3-tuples 
    - ```[column, type, alias] ``` 3-item lists
    - ```'DOB'``` Single strings.

```yaml

{%- set tgt_pk = ['PART_PK', 'BINARY(16)', 'PART_PK']        -%}

{{ dbtvault.cast([tgt_pk,
                  'DOB',
                  ('PART_PK', 'NUMBER(38,0)', 'PART_ID'),
                  ('LOADDATE', 'DATE', 'LOADDATE'),
                  ('SOURCE', 'VARCHAR(15)', 'SOURCE')], 
                  'stg')                                      }}
```

#### Output

```mysql
CAST(stg.PART_PK AS BINARY(16)) AS PART_PK,
stg.DOB,
CAST(stg.PART_ID AS NUMBER(38,0)) AS PART_ID,
CAST(stg.LOADDATE AS DATE) AS LOADDATE,
CAST(stg.SOURCE AS VARCHAR(15)) AS SOURCE
```

___

### hash

!!! warning
    This macro ***should not be*** used for cryptographic purposes.
    
    The intended use is for creating checksum-like fields only, so that a record change can be detected. 
    
    [Read More](https://www.md5online.org/blog/why-md5-is-not-safe/)
    
A macro for generating hashing SQL for columns:
```sql 
CAST(MD5_BINARY(UPPER(TRIM(CAST(column AS VARCHAR)))) AS BINARY(16)) AS alias
```

- Can provide multiple columns as a list to create a concatenated hash
- Hashdiffs should be alpha sorted using the ```sort``` flag.
- Casts a column as ```VARCHAR```, transforms to ```UPPER``` case and trims whitespace
- ```'^^'``` Accounts for null values with a double caret
- ```'||'``` Concatenates with a double pipe 

#### Parameters

| Parameter        |  Description                                     | Type        | Required?                                                |
| ---------------- | -----------------------------------------------  | ----------- | -------------------------------------------------------- |
| columns          |  Columns to hash on                              | String/List | <i class="md-icon" style="color: green">check_circle</i> |
| alias            |  The name to give the hashed column              | String      | <i class="md-icon" style="color: green">check_circle</i> |
| sort             |  Will alpha sort columns if true, default false. | Boolean     | <i class="md-icon" style="color: red">clear</i>          |
                                

#### Usage

```yaml
{{ dbtvault.hash('CUSTOMERKEY', 'CUSTOMER_PK') }},
{{ dbtvault.hash(['CUSTOMERKEY', 'PHONE', 'DOB', 'NAME'], 'HASHDIFF', true) }}
```

!!! tip
    [multi_hash](#multi_hash) may be used to simplify the hashing process and generate multiple hashes with one macro.

#### Output

```mysql
CAST(MD5_BINARY(UPPER(TRIM(CAST(CUSTOMERKEY AS VARCHAR)))) AS BINARY(16)) AS CUSTOMER_PK,
CAST(MD5_BINARY(CONCAT(IFNULL(UPPER(TRIM(CAST(CUSTOMERKEY AS VARCHAR))), '^^'), '||',
                       IFNULL(UPPER(TRIM(CAST(DOB AS VARCHAR))), '^^'), '||',
                       IFNULL(UPPER(TRIM(CAST(NAME AS VARCHAR))), '^^'), '||',
                       IFNULL(UPPER(TRIM(CAST(PHONE AS VARCHAR))), '^^') )) 
                       AS BINARY(16)) AS HASHDIFF
```

___

### prefix

A macro for quickly prefixing a list of columns with a string:
```mysql
a.column1, a.column2, a.column3, a.column4
```

#### Parameters

| Parameter        |  Description                  | Type   | Required?                                                |
| ---------------- | ----------------------------- | ------ | -------------------------------------------------------- |
| columns          |  A list of column names       | List   | <i class="md-icon" style="color: green">check_circle</i> |
| prefix_str       |  The prefix for the columns   | String | <i class="md-icon" style="color: green">check_circle</i> |

#### Usage

```yaml
{{ dbtvault.prefix(['CUSTOMERKEY', 'DOB', 'NAME', 'PHONE'], 'a') }}
{{ dbtvault.prefix(['CUSTOMERKEY'], 'a') 
```

!!! Note
    Single columns must be provided as a 1-item list, as in the second example above.

#### Output

```mysql
a.CUSTOMERKEY, a.DOB, a.NAME, a.PHONE
a.CUSTOMERKEY
```

___

## Internal
######(macros/internal)

Internal macros support the other macros provided in this package. 
They are used to process provided metadata and should not be called directly. 