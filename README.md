# kafka-gpss-replication
PoC implementing Oracle to Greenplum replication, using GoldenGate for CDC, Kafka for transport and GPSS for ingestion and merging

Source Oracle table:
```sql
CREATE TABLE SIEBEL_TEST.TEST_POC(
   ID numeric,
   NAME varchar2 (50),
   BIRTHDAY date
)
```

Every row change in Oracle is captured by GoldenGate and routed to Kafka topic as a text message in json format, i.e.

```json
{"table":"SIEBEL_TEST.TEST_POC","op_type":"I","op_ts":"2019-11-21 10:05:34.000000","current_ts":"2019-11-21T11:05:37.823000","pos":"00000000250000058833","tokens":{"TK_OPTYPE":"INSERT","SCN":"61042900"},"after":{"ID":1,"NAME":"Igor","BIRTHDAY":"2000-01-01 00:00:00"}}
```

This record contains necessary data for replication in target table:
1. Operation type (op_type): insert, update or delete
2. Operation order (pos). In fact as soon as messages are ordered in Kafka it is not necessary, but GPSS currently doesn't support this
3. Table name
4. Data itself, for both key and nonkey attributes (after)

It allows us to merge Kafka stream to target table in Greenplum using MERGE mode of GPSS job.
Please find here GPSS configuration file that allows to

Below are the following limitations and issues that we met during PoC and that are addressed to engineering:

## 1. Delete option
GPSS MERGE mode can insert and update records, but cannot delete. When we need to delete record when we receive message with op_type='D'. I think this functionality can be implemented in GPSS MERGE mode along with specifying the criteria for input message if this message should delete record.
In provided example there is a workaround that was implemented to simulate deletion. We keep technical `op_type` field in target table and treat it as regular non-key attribute. Then, when table is accessed by business user, it should be filtered with `op_type <> 'D'` condition.

## 2. Deduplication
When record is updated more than once in Oracle, or updated and then deleted, or deleted and then inserted same ID, etc there is more than one record with the same ID (key attribute) appears in a batch of messages processed by GPSS .
Below is the update query generated by GPSS for each processed batch
```sql
UPDATE "public"."test_poc" AS into_table
SET    "op_type" = ext."op_type",
       "name" = ext."name",
       "birthday" = ext."birthday"
FROM   gpsstmp_c77ada7f436d2bc0854a300154b1517c AS ext
WHERE  into_table."id" = ext."id"
```
Where `test_poc` is target table, `gpsstmp_c77ada7f436d2bc0854a300154b1517c` is temporary table containing batch data and `id` is a key attribute.
This update statement fails if temp table contains non-unique `id`s
As a workaround in PoC we added `rn` field in target table that keeps row_number through the same ID in a batch and when updating filter records with rn=1.
```yaml
MAPPING:
  ...
  - NAME: rn
    EXPRESSION: row_number() over (partition by jdata#>>'{after,ID}' order by jdata->>'pos' desc)
```

The other query generated by GPSS is for inserting new records, and it performs deduplication out of the box correctly with row_number() function:
```sql
INSERT INTO "public"."test_poc" ("op_type","id","name","birthday")
  (SELECT from_table."op_type",from_table."id",from_table."name",from_table."birthday" FROM
    (SELECT *, row_number() OVER (PARTITION BY "id") AS gpss_row_number FROM gpsstmp_e1ca57da0250e378da512909b37bc855)
        AS from_table
   LEFT OUTER JOIN "public"."test_poc" into_table
        ON into_table."id" = from_table."id"
   WHERE into_table."id" IS NULL AND gpss_row_number = 1)
```
However expression `row_number() OVER (PARTITION BY "id")` does not have ORDER BY clause. It means that row number is assigned randomly across set of records with the same ID and finally random record will have gpss_row_number = 1 and be inserted in target table. Normally, when processing messages from GoldenGate, the last one should be taken as it represents the latest state of record in source by the moment we are processing the batch.
As an optimization I would advice to do deduplication when inserting in `gpsstmp_...` table rather than in insert and update statements for target table to simplify insert and update queries and avoid calling expensive window function twice.
Summarizing on this point
1. Implement deduplication by key
2. Add ORDER BY clause in deduplication window statement and choose the *latest* value from batch based on position of record in Kafka queue.

## 3. Temporary table
Each batch processing creates temporary table:
```sql
CREATE TEMP TABLE gpsstmp_c77ada7f436d2bc0854a300154b1517c
  (
     like "public"."TEST_POC"
  ) ON COMMIT drop;
```
As soon as GPSS is intended to provide near-real time replication, those batches are processed frequently, i.e. once a second. That could cause catalog bloat. I advice to create table once on GPSS job start and reuse it, truncating before processing of eatch batch
