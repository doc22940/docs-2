---
title: SAVEPOINT
summary: Define a SAVEPOINT within the current transaction
toc: true
---

A savepoint is an optional placeholder inside a transaction that is used to allow all commands that are executed after the savepoint to be rolled back. When rolled back, the transaction state is restored to the state that existed prior to the savepoint, save in those cases listed in [Limitations](#limitations).

The `SAVEPOINT` statement has two uses in CockroachDB:

1. <span class="version-tag">New in v20.1</span>: Enables a client to partially roll back a partially completed transaction to an earlier status. In particular, the client can use multiple nested `SAVEPOINT`s, each with a different name. This functionality is often used by test suites for tools such as [ORMs used by application developers](hello-world-example-apps.html).

2. Defines the intent to retry [transactions](transactions.html) using the CockroachDB-provided function for client-side transaction retries. For more information, see [Transaction Retries](transactions.html#transaction-retries). Such "restart savepoints" must be the outermost savepoint in a transaction.

## Synopsis

<div>
  {% include {{ page.version.version }}/sql/diagrams/savepoint.html %}
</div>

## Required privileges

No [privileges](authorization.html#assign-privileges) are required to create a savepoint. However, privileges are required for each statement within a transaction.

## Parameters

Parameter | Description
--------- | -----------
name      | The name of the savepoint.

## Limitations

Limitations on CockroachDB's implementation of savepoints include:

- Savepoints used for [restarts](#restart-savepoints) must be the outermost savepoint in a transaction, and must be named `cockroach_restart` (unless restart savepoint renaming is enabled via the process described in [Customize the name of restart savepoints](#customizing-the-name-of-restart-savepoints)).

## Examples

### Basic usage

To establish a savepoint inside a transaction:

{% include copy-clipboard.html %}
~~~ sql
SAVEPOINT foo;
~~~

To roll back a transaction partially to a previously established savepoint:

{% include copy-clipboard.html %}
~~~ sql
ROLLBACK TO SAVEPOINT foo;
~~~

To forget a savepoint, and keep the effects of statements executed after the savepoint was established:

{% include copy-clipboard.html %}
~~~ sql
RELEASE SAVEPOINT foo;
~~~

For example, the transaction below will insert the values `1` and `3` into the table, but not `2`:

{% include copy-clipboard.html %}
~~~ sql
BEGIN;
    INSERT INTO table1 VALUES (1);
    SAVEPOINT my_savepoint;
    INSERT INTO table1 VALUES (2);
    ROLLBACK TO SAVEPOINT my_savepoint;
    INSERT INTO table1 VALUES (3);
COMMIT;
~~~

### Nested savepoints

#### Example 1

```
root@localhost:26257/defaultdb> begin;
begin;
Now adding input for a multi-line SQL transaction client-side (smart_prompt enabled).
Press Enter two times to send the SQL text collected so far to the server, or Ctrl+C to cancel.
You can also use \show to display the statements entered so far.
                             -> insert into kv values (2,1);
insert into kv values (2,1);
                             -> 

INSERT 1

Time: 1.2ms

savepoint my_savepoint;
root@localhost:26257/defaultdb  OPEN> savepoint my_savepoint;
SAVEPOINT

Time: 146µs

insert into kv values (3,1);
root@localhost:26257/defaultdb  OPEN> insert into kv values (3,1);
INSERT 1

Time: 1.243ms

savepoint my_savepoint2;
root@localhost:26257/defaultdb  OPEN> savepoint my_savepoint2;
SAVEPOINT

Time: 106µs

insert into kv values (4,1);
root@localhost:26257/defaultdb  OPEN> insert into kv values (4,1);
INSERT 1

Time: 635µs

release savepoint my_savepoint2;
root@localhost:26257/defaultdb  OPEN> release savepoint my_savepoint2;
RELEASE

Time: 117µs

rollback to savepoint my_savepoint;
root@localhost:26257/defaultdb  OPEN> rollback to savepoint my_savepoint;
ROLLBACK

Time: 126µs

select * from kv;
root@localhost:26257/defaultdb  OPEN> select * from kv;
  k | v
----+----
  1 | 1
  2 | 1
(2 rows)

Time: 654µs

commit;
root@localhost:26257/defaultdb  OPEN> commit;
COMMIT

Time: 2.149ms

root@localhost:26257/defaultdb> select * from kv;
select * from kv;
  k | v
----+----
  1 | 1
  2 | 1
(2 rows)

Time: 1.267ms
```

#### Example 2

```
create table kv (k int primary key, v int);

insert into kv (k, v) values (1, 1);

root@localhost:26257/defaultdb> begin;
begin;
Now adding input for a multi-line SQL transaction client-side (smart_prompt enabled).
Press Enter two times to send the SQL text collected so far to the server, or Ctrl+C to cancel.
You can also use \show to display the statements entered so far.
                             -> savepoint foo;
savepoint foo;
                             -> 

SAVEPOINT

Time: 124µs

update kv set v = v + 1 where k = 1;

UPDATE 1

Time: 784µs

select * from kv;
root@localhost:26257/defaultdb  OPEN> select * from kv;
  k | v
----+-----
  1 | 2
(1 row)

Time: 513µs

savepoint bar;
root@localhost:26257/defaultdb  OPEN> savepoint bar;
SAVEPOINT

Time: 146µs

update kv set v = v + 1 where k = 1;

UPDATE 1

Time: 863µs

select * from kv where k = 1;
root@localhost:26257/defaultdb  OPEN> select * from kv where k = 1;
  k | v
----+-----
  1 | 3
(1 row)

Time: 880µs

rollback to savepoint bar;
root@localhost:26257/defaultdb  OPEN> rollback to savepoint bar;
ROLLBACK

Time: 177µs

select * from kv where k = 1;
root@localhost:26257/defaultdb  OPEN> select * from kv where k = 1;
  k | v
----+-----
  1 | 2
(1 row)

Time: 463µs

commit;
root@localhost:26257/defaultdb  OPEN> commit;
COMMIT

Time: 1.353ms

select * from kv where k = 1;

  k | v
----+-----
  1 | 2
(1 row)

Time: 682µs
```

#### Multi-level commit / rollback

[`RELEASE SAVEPOINT`](release-savepoint.html) and [`ROLLBACK TO SAVEPOINT`](rollback-transaction.html) can both refer to a savepoint "higher" in the nesting hierarchy. When this occurs, all of the savepoints "under" the nesting are automatically released / rolled back too.

TO insert both 1 and 2:

{% include copy-clipboard.html %}
~~~ sql
BEGIN;
    SAVEPOINT foo;
    INSERT INTO kv VALUES (5,1);
    SAVEPOINT bar;
    INSERT INTO kv VALUES (6,1);
    RELEASE SAVEPOINT foo;
COMMIT;
~~~

This inserts nothing - both are rolled back:

{% include copy-clipboard.html %}
~~~ sql
BEGIN;
    SAVEPOINT foo;
    INSERT INTO kv VALUES (7,1);
    SAVEPOINT bar;
    INSERT INTO kv VALUES (8,1);
    ROLLBACK TO SAVEPOINT foo;
COMMIT;
~~~

This demonstrates that the name "bar" is not visible after it was rolled back over:

{% include copy-clipboard.html %}
~~~ sql
BEGIN;
    SAVEPOINT foo;
    SAVEPOINT bar;
    ROLLBACK TO SAVEPOINT foo;
    RELEASE SAVEPOINT bar; -- error: savepoint "bar" does not exist
COMMIT;
~~~

#### Savepoints and prepared statements

```
prepare a as select 1;
root@localhost:26257/defaultdb  OPEN> prepare a as select 1;
PREPARE

Time: 1.177ms

rollback to savepoint foo;
root@localhost:26257/defaultdb  OPEN> rollback to savepoint foo;
ROLLBACK

Time: 145µs

execute a; 
root@localhost:26257/defaultdb  OPEN> execute a; 
  ?column?
------------
         1
(1 row)

Time: 468µs

show savepoint status;
root@localhost:26257/defaultdb  OPEN> show savepoint status;
  savepoint_name | is_restart_savepoint
-----------------+-----------------------
  foo            |        false
(1 row)

Time: 427µs

rollback;
root@localhost:26257/defaultdb  OPEN> rollback;
ROLLBACK

Time: 384µs

root@localhost:26257/defaultdb> 
```

#### Savepoint name scoping

CockroachDB allows a `SAVEPOINT` statement to shadow an earlier savepoint with the same name. The name refers to the new savepoint until released (committed) or rolled back, after which the name refers to the previous savepoint.

{% include copy-clipboard.html %}
~~~ sql
root@localhost:26257/defaultdb> begin;
root@localhost:26257/defaultdb> begin;
Now adding input for a multi-line SQL transaction client-side (smart_prompt enabled).
Press Enter two times to send the SQL text collected so far to the server, or Ctrl+C to cancel.
You can also use \show to display the statements entered so far.
                             -> savepoint foo;
savepoint foo;
                             -> 

SAVEPOINT

Time: 196µs

update kv set v = v + 1 where k = 1;
root@localhost:26257/defaultdb  OPEN> update kv set v = v + 1 where k = 1;
UPDATE 1

Time: 1.885ms

select * from kv;
root@localhost:26257/defaultdb  OPEN> select * from kv;
  k | v
----+----
  1 | 2
  2 | 1
(2 rows)

Time: 753µs

savepoint foo;
root@localhost:26257/defaultdb  OPEN> savepoint foo;
SAVEPOINT

Time: 139µs

select * from kv;
root@localhost:26257/defaultdb  OPEN> select * from kv;
  k | v
----+----
  1 | 2
  2 | 1
(2 rows)

Time: 729µs

update kv set v = v + 1 where k = 1;
root@localhost:26257/defaultdb  OPEN> update kv set v = v + 1 where k = 1;
UPDATE 1

Time: 1.02ms

select * from kv;
root@localhost:26257/defaultdb  OPEN> select * from kv;
  k | v
----+----
  1 | 3
  2 | 1
(2 rows)

Time: 579µs

show savepoint status;
root@localhost:26257/defaultdb  OPEN> show savepoint status;
  savepoint_name | is_restart_savepoint
-----------------+-----------------------
  foo            |        false
  foo            |        false
(2 rows)

Time: 129µs

rollback to savepoint foo;
root@localhost:26257/defaultdb  OPEN> rollback to savepoint foo;
ROLLBACK

Time: 152µs

release savepoint foo;
root@localhost:26257/defaultdb  OPEN> release savepoint foo;
RELEASE

Time: 182µs

commit;
root@localhost:26257/defaultdb  OPEN> commit;
COMMIT

Time: 1.436ms
~~~

### Restart savepoints

{{site.data.alerts.callout_info}}
The example in this section refers to the method of using `SAVEPOINT` to implement [client-side transaction retries](transactions.html#transaction-retries). If you want to use general-purpose `SAVEPOINT`s, see [Nested savepoints](#nested-savepoints).
{{site.data.alerts.end}}

After you `BEGIN` the transaction, you must create the savepoint to identify that if the transaction contends with another transaction for resources and "loses", you intend to use [client-side transaction retries](transactions.html#transaction-retries).

A restart savepoint must be the outermost savepoint in the transaction.

Applications using `SAVEPOINT` must also include functions to execute retries with [`ROLLBACK TO SAVEPOINT `](rollback-transaction.html#retry-a-transaction).

#### restart savepoint - with nested savepoints underneath

root@localhost:26257/defaultdb> begin;
begin;
Now adding input for a multi-line SQL transaction client-side (smart_prompt enabled).
Press Enter two times to send the SQL text collected so far to the server, or Ctrl+C to cancel.
You can also use \show to display the statements entered so far.
                             -> savepoint cockroach_restart;
savepoint cockroach_restart;
                             -> 

SAVEPOINT

Time: 281µs

insert into kv(k,v) values (100,1);
root@localhost:26257/defaultdb  OPEN> insert into kv(k,v) values (100,1);
INSERT 1

Time: 841µs

savepoint foo;
root@localhost:26257/defaultdb  OPEN> savepoint foo;
SAVEPOINT

Time: 125µs

select * from kv;
root@localhost:26257/defaultdb  OPEN> select * from kv;
   k  | v
------+----
    1 | 2
    2 | 1
    3 | 1
    4 | 1
    5 | 1
    6 | 1
   10 | 1
  100 | 1
(8 rows)

Time: 875µs

rollback to savepoint foo;
root@localhost:26257/defaultdb  OPEN> rollback to savepoint foo;
ROLLBACK

Time: 184µs

select * from kv;
root@localhost:26257/defaultdb  OPEN> select * from kv;
   k  | v
------+----
    1 | 2
    2 | 1
    3 | 1
    4 | 1
    5 | 1
    6 | 1
   10 | 1
  100 | 1
(8 rows)

Time: 954µs

rollback to savepoint cockroach_restart;
root@localhost:26257/defaultdb  OPEN> rollback to savepoint cockroach_restart;
SAVEPOINT

Time: 115µs

select * from kv;
root@localhost:26257/defaultdb  OPEN> select * from kv;
  k  | v
-----+----
   1 | 2
   2 | 1
   3 | 1
   4 | 1
   5 | 1
   6 | 1
  10 | 1
(7 rows)

Time: 835µs

commit;
root@localhost:26257/defaultdb  OPEN> commit;
COMMIT

Time: 1.444ms

root@localhost:26257/defaultdb> select * from kv;
select * from kv;
  k  | v
-----+----
   1 | 2
   2 | 1
   3 | 1
   4 | 1
   5 | 1
   6 | 1
  10 | 1
(7 rows)

Time: 857µs

#### restart savepoint - common case

{% include copy-clipboard.html %}
~~~ sql
> BEGIN;
~~~

{% include copy-clipboard.html %}
~~~ sql
> SAVEPOINT cockroach_restart;
~~~

{% include copy-clipboard.html %}
~~~ sql
> UPDATE products SET inventory = 0 WHERE sku = '8675309';
~~~

{% include copy-clipboard.html %}
~~~ sql
> INSERT INTO orders (customer, sku, status) VALUES (1001, '8675309', 'new');
~~~

{% include copy-clipboard.html %}
~~~ sql
> RELEASE SAVEPOINT cockroach_restart;
~~~

{% include copy-clipboard.html %}
~~~ sql
> COMMIT;
~~~

### Showing savepoint status

Use the [`SHOW SAVEPOINT STATUS`](show-savepoint-status.html) statement to see how many savepoints are active in the current transaction:

{% include copy-clipboard.html %}
~~~ sql
SHOW SAVEPOINT STATUS;
~~~

~~~
  savepoint_name | is_restart_savepoint
-----------------+-----------------------
  foo            |        false
  bar            |        false
(2 rows)
~~~

Note that the `is_restart_savepoint` column will be true if the savepoint is a [restart savepoint](#restart-savepoints).

### Customizing the name of restart savepoints

{% include {{ page.version.version }}/misc/customizing-the-savepoint-name.md %}

## See also

- [`SHOW SAVEPOINT STATUS`](show-savepoint-status.html)
- [`RELEASE SAVEPOINT`](release-savepoint.html)
- [`ROLLBACK`](rollback-transaction.html)
- [`BEGIN`](begin-transaction.html)
- [`COMMIT`](commit-transaction.html)
- [Transactions](transactions.html)
- [Retryable transaction example code in Java using JDBC](build-a-java-app-with-cockroachdb.html)
- [CockroachDB Architecture: Transaction Layer](architecture/transaction-layer.html)
