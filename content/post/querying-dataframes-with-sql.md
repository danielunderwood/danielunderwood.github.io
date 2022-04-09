---
title: "Querying Dataframes With SQL"
date: 2022-04-04T21:04:32-04:00
draft: false
tags: [peg-parsing, pandas, python]
link: https://github.com/danielunderwood/sqldf
---

## Background

Not having a background in computer science, a lot of the formal concepts have escaped me until I have run across a need for them. One such concept that I have run across recently is parsing: given some sort of structured input and an associated grammar, make it do something useful.

Of course this is the basis of modern programming languages, but most resources on parsing begin with something simple such as a calculator. What else could we do with it?

## The Idea

The idea is to parse SQL statements into modifications to a pandas DataFrame. Some solutions to this already exist. [pandasql](https://github.com/yhat/pandasql/) offers this functionality by using `DataFrame.to_sql` to place the DataFrame into an in-memory SQLite database which it can query. Frameworks like PySpark also allow this type of querying, but have a much more complex architecture.

The parsing approach is a bit different: given a statement like `SELECT column FROM df` should transform into `df.get('column')` (or equivalently, `df['column']`) and other SQL clauses like `WHERE ...` and `GROUP BY ...` should turn into their respective DataFrame operations.

## Up and Running

While I can't claim to know the technical details of parsing, I do know that if you search for "peg parser generator $language" in your favorite search engine, you'll probably find something that lets you write a PEG grammar alongside some $language code and will generate a parser class/function.

Of course $language in the case of this little project is python since we're working with pandas. In that case, I found [pegen](https://github.com/we-like-parsers/pegen).  Install it with your favorite package manager and we're ready to go!

## Selecting Fields

The first part we want to deal with is `SELECT` statements. The simplest way to do this is with the following grammar:

```
start: 'SELECT' NAME { name.string }
```

We can try running it:

```shell
python -m pegen sqldf/sql.gram -o sqldf/parser.py \
  && python sqldf/parser.py - <<< "SELECT somecol"

Clean Grammar:
  start: 'SELECT' NAME
somecol
```

Good start! But what about in SQL where you can do something like `SELECT 1` to select out a constant? Well we can do that as well, though we'll have to have some concept of the difference in a column and a constant. For now, we can represent them by `{'column': 'colname'}` or `{'const': 1}`, respectively. With a bit of organization of the grammar, we end up with

```
start: select
select: 'SELECT' selectable { selectable }
selectable: NAME { {'column': name.string} }
  | NUMBER { {'const': float(number.string)} }
```

```shell
python -m pegen sqldf/sql.gram -o sqldf/parser.py \
  && python sqldf/parser.py - <<< "SELECT 1"

Clean Grammar:
  start: select
  select: 'SELECT' selectable
  selectable: NAME | NUMBER
{'const': 1.0}

```

Awesome, but what about multiple columns? This is pretty easy to handle as well since pegen has a [built-in expression](https://github.com/we-like-parsers/pegen#se) for separation by a field; we can use `','.selectable+`:

```
start: select
select: 'SELECT' selectables { selectables }
selectables: ','.selectable+
selectable: NAME { {'column': name.string} }
  | NUMBER { {'const': float(number.string)} }
```

```shell
python -m pegen sqldf/sql.gram -o sqldf/parser.py \
  && python sqldf/parser.py - <<< "SELECT a ,  1, c"

Clean Grammar:
  start: select
  select: 'SELECT' selectables
  selectables: ','.selectable+
  selectable: NAME | NUMBER
[{'column': 'a'}, {'const': 1.0}, {'column': 'c'}]

```

Now we have a nice little base for selecting out fields and constants. Let's try to integrate it with pandas.

## Pandas Integration

Let's start building a little file that can programatically parse things to I stop having to do it all via stdin. Imitating [`simple_parser_main`](https://github.com/we-like-parsers/pegen/blob/main/src/pegen/parser.py#L267) that the CLI uses, we can end up with something like the following:

```python
from io import StringIO

import pandas as pd
from pegen.tokenizer import Tokenizer, tokenize

from parser import GeneratedParser as SqlParser


def parse(sql):
    # generate_tokens particularly wants a readline method reference,
    # so we can use StringIO here to get that interface
    statement = StringIO(sql)
    tokengen = tokenize.generate_tokens(statement.readline)
    t = Tokenizer(tokengen)
    p = SqlParser(t)
    return p.start()


if __name__ == "__main__":
    print(parse("SELECT a, 1"))
```

Now we can load in some data and actually start indexing our DataFrame:

```python
def apply_sql(df, sql):
    parsed = parse(sql)
    columns = [c["column"] for c in parsed if "column" in c]
    if "*" in columns:
        reduced = df
    else:
        reduced = df[[c for c in columns if c != "*"]]
    for i, const in enumerate([c["const"] for c in parsed if "const" in c]):
        reduced[f"const{i}"] = const
    return reduced.reset_index()

  
  

if __name__ == "__main__":
    iris_url = "https://raw.githubusercontent.com/mwaskom/seaborn-data/master/iris.csv"
    iris = pd.read_csv(iris_url)
    reduced = apply_sql(iris, " ".join(sys.argv[1:]))
    print(reduced)
```

which produces a wonderful

```shell
python sqldf/sqldf.py SELECT sepal_length, species, 1
     sepal_length    species  const0
0             5.1     setosa     1.0
1             4.9     setosa     1.0
2             4.7     setosa     1.0
3             4.6     setosa     1.0
4             5.0     setosa     1.0
..            ...        ...     ...
145           6.7  virginica     1.0
146           6.3  virginica     1.0
147           6.5  virginica     1.0
148           6.2  virginica     1.0
149           5.9  virginica     1.0

[150 rows x 3 columns]
```

Profit! There are a couple imperfections here, such as `SELECT something AS alias` and the casing of `SELECT`, but I think this gives us solid ground to tackle a couple other situations: filtering and grouping.

## Filtering

Now that we have basic selects working, let's try conditions. In SQL, this is a where clause, such as the following:
- `WHERE value = 1`
- `WHERE value = 'string'`
- `WHERE value > 5`
- `WHERE value <= 1`
- `WHERE value IN (1, 2, 3)`

Provided we parse this in a vaguely python-like way, these correspond to operators in python's standard [operator module](https://docs.python.org/3/library/operator.html).

## Missing Pieces

From this stage, there are still a few missing pieces:
- Grouping and aggregations
- `SELECT ... FROM`
- Sorting

Aggregations will require parsing out function calls, `FROM` will require introspection of variables in the calling code, and I'm too lazy to work out sorting edge cases at the moment. Perhaps these will be the subject of a future post!

## Code

The code for this project and updates to it are available at https://github.com/danielunderwood/sqldf.