---
layout: post
title: "T-Strings in Py3.14: Building SQL Utility for DuckDB"
date: 2026-07-02
---

Python 3.14 introduces **T-Strings** (PEP 750) — template strings that give you structured, programmable access to interpolated values at runtime. Instead of smashing strings together, `t"..."` literals produce a `Template` object that you can iterate over, inspect, and handle with full control.

I've been playing with this to build `tquery` — a thin wrapper around DuckDB that solves an ergonomic pain point I hit when writing duckdb queries.

## The Annoyance

DuckDB's Python API is clever: it scrapes your local and global scope to resolve names that look like table sources. So you can write:

```python
df = pd.DataFrame({"x": [1, 2, 3]})
conn.execute("SELECT * FROM df")
```

and DuckDB finds `df` automatically. Convenient, but it drives linters (and me) crazy — `df` looks unused, Pyright flags it, and you end up with `# noqa` comments everywhere or defensive `_ = df` stubs.

Explicit registration works but is noisy:

```python
conn.register("my_view", df)
conn.execute("SELECT * FROM my_view")
conn.unregister("my_view")
```

You're repeating names, and the connection between the Python variable and the SQL identifier is purely by convention.

## Enter `tquery`

With T-Strings, the variable reference is real — the linter sees `{df}` and knows it's used. Here's the core idea from `duckdb_relations.py`:

```python
from string.templatelib import Template, Interpolation, convert

RELATION_CLASSES = (duckdb.DuckDBPyRelation, pd.DataFrame, pl.DataFrame, ...)

def tquery_iteration1(connection, tsql: Template) -> duckdb.DuckDBPyRelation:
    query = ""
    for part in tsql:
        if isinstance(part, str):
            query += part
        elif isinstance(part, Interpolation):
            if isinstance(part.value, RELATION_CLASSES):
                rel_name = f"__tquery_{id(part.value)}_{part.expression}"
                connection.register(rel_name, part.value)
                query += f'"{rel_name}"' 
            else:
                raise Error(f"Unexpected interpolation {part}") 
    return connection.query(query)
```

Now:

```python
users = duckdb.sql("SELECT * FROM users")
orders = pd.DataFrame({"id": [1, 2], "user_id": [1, 1], "total": [100, 200]})

result = tquery_iteration1(conn, t"""
    SELECT u.*, o.total
    FROM {users} u
    JOIN {orders} o ON u.id = o.user_id
""")
```

Every interpolated variable is a real Python expression. Pyright sees `users` and `orders` as used. No unused-variable warnings. No manual registration. No name duplication.

What happens under the hood:
* **`{users}`** and **`{orders}`** — relation types get registered with a unique name, inserted into the SQL.

## Next step - parameters

The next logical step is to add params to `tquery` function... No wait! We can leverage Interpolations and automatically convert passed object into binded parameters.

 ```python
from string.templatelib import Template, Interpolation, convert

RELATION_CLASSES = (duckdb.DuckDBPyRelation, pd.DataFrame, pl.DataFrame, ...)

def tquery_iteration2(connection, tsql: Template) -> duckdb.DuckDBPyRelation:
    query = ""
    params = []
    for part in tsql:
        if isinstance(part, Interpolation):
            if ...:
                ...
            else:
                params.append(part.value)
                query += "?"
        ...
    return connection.query(query, params=params)
```

Now:

```python
users = duckdb.sql("SELECT * FROM users")
orders = pd.DataFrame({"id": [1, 2], "user_id": [1, 1], "total": [100, 200]})
min_amount = 50

result = tquery_iteration2(conn, t"""
    SELECT u.*, o.total
    FROM {users} u
    JOIN {orders} o ON u.id = o.user_id
    WHERE o.total > {min_amount}
""")
```

What happens under the hood for `min_amount`:
* **`{min_amount}`** — plain values become `?` bind parameters, automatically preventing sql injections!

## But... Sometimes I want to rawdog parts of SQL

A neat trick: when `Interpolation.conversion` is specified, let's convert interpolation to string and rawdog append to query.

```python
def tquery_iteration3(connection, tsql: Template) -> duckdb.DuckDBPyRelation:
    query = ""
    params = []
    for part in tsql:
        if isinstance(part, Interpolation):
            if part.conversion is not None:
                query += convert(part.value, part.conversion)
            ...
        ...
    return connection.query(query, params=params)
```

Now:

```python
users = duckdb.sql("SELECT * FROM users")
orders = pd.DataFrame({"id": [1, 2], "user_id": [1, 1], "total": [100, 200]})
min_amount = 50
fields = ", ".join(["u.name", "u.age", "o.total"])

result = tquery_iteration3(conn, t"""
    SELECT {fields!s}
    FROM {users} u
    JOIN {orders} o ON u.id = o.user_id
    WHERE o.total > {min_amount}
""")
```

## Composing Templates

T-Strings compose via concatenation. The `bins_to_case` helper in `resources/bins.py` builds dynamic `CASE` expressions by concatenating `t"..."` blocks:

```python
def bins_to_case(bins, field) -> Template:
    cases = t""
    for i, bin_pattern in enumerate(bins):
        if bin_pattern.startswith("<"):
            value = int(bin_pattern[1:])
            cases += t"\n    WHEN {field!s} < {value} THEN {bin_pattern}"
        elif ...:
            ...
    return t"CASE" + cases + t"\nEND"
```

Each `cases += t"..."` appends a `WHEN` clause. The resulting `Template` embeds into a `tquery` call:

```python
result = tquery(conn, t"""
    SELECT
        """ + case_expr + t""" AS age_group,
        COUNT(*) AS count
    FROM {data}
    GROUP BY 1
""")
```

You build SQL by writing full plain old SQL or extract more complex functionality in composable utilities.

## Why I like it

- **Linter-friendly** — no more unused-variable warnings because of DuckDB's magic scope scraping
- **Composable** — `Template` objects concatenate naturally
- **Familiar** — the syntax is almost identical to f-strings

I'm using this in production with Dagster pipelines and have been loving this for a while now.

## Full implementation

* [resources/tquery.py](https://github.com/sisidra/sisidra.github.io/blob/main/resources/tquery.py)
* [resources/bins.py](https://github.com/sisidra/sisidra.github.io/blob/main/resources/bins.py)

## Known limitations

* Collections are registered but not unregistered. As execution is delayed, there is no convenient way I can come up with to cleanup no longer used collections. Not an issue for me at this time as my use cases are small short lived data ETL pipelines.
