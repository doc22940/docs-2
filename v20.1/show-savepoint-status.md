---
title: SHOW SAVEPOINT STATUS
summary: The SHOW SAVEPOINT STATUS statement lists the active savepoints in the current transaction.
toc: true
---

The `SHOW SAVEPOINT STATUS` [statement](sql-statements.html) lists the active [savepoints](savepoint.html) in the current [transaction](transactions.html).

## Required privileges

No [privileges](authorization.html#assign-privileges) are required to create or show a savepoint. However, privileges are required for each statement within a transaction.

## Synopsis

XXX: generate the diagram

<div>
  {% include {{ page.version.version }}/sql/diagrams/show_constraints.html %}
</div>

## Parameters

XXX: delete this?

Parameter | Description
----------|------------
`table_name` | The name of the table for which to show constraints.

## Response

The following fields are returned for each constraint.

Field | Description
------|------------
`savepoint_name` | The name of the savepoint.
`is_restart_savepoint` | Whether the savepoint is a [restart savepoint](savepoint.html#restart-savepoints).

## Example

{% include copy-clipboard.html %}
~~~ sql

~~~

{% include copy-clipboard.html %}
~~~ sql
SHOW SAVEPOINT STATUS;
~~~

~~~
  savepoint_name | is_restart_savepoint
-----------------+-----------------------
  foo            |        false
(1 row)
~~~

## See also

- [`SAVEPOINT`](savepoint.html)
- [`RELEASE SAVEPOINT`](release-savepoint.html)
- [`ROLLBACK`](rollback-transaction.html)
- [`BEGIN`](begin-transaction.html)
- [`COMMIT`](commit-transaction.html)
- [Transactions](transactions.html)
