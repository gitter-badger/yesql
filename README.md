# Yesql - Clojure SQL queries rethought.

Yesql is a Clojure library for _using_ SQL queries.

## Status

In development. I welcome your feedback.

## Rationale

You're writing Clojure. You need to write some SQL.

I think we're all agreed that this is a problem:

``` clojure
(query "SELECT * FROM users WHERE country_code = ?" "GB")
```

Unless these query strings are short, they quickly get hard to read
and hard to rewrite. Plus the lack of syntax highlighting is a pain.

But something like this is not the solution:

``` clojure
(select :*
        (from :users)
        (where (= :country_code "GB")))
```

Clojure is a great language for writing DSLs, but we don't need a new
one. SQL is already a mature DSL.  And s-expressions are great, but
here they're not adding anything. This is parens-for-parens sake.
(Don't agree? Wait until this extra syntax layer breaks down and you
start wrestling with a `(raw-sql)` function.)

So what's the solution? Keep the SQL as SQL. Have one file with your
query:

``` sql
SELECT *
FROM users
WHERE country_code = ?
```

...and then pull it in and use it as a regular Clojure function:

``` clojure
(defquery users-by-country "some/where/users_by_country.sql")
(users-by-country db-spec "GB")
```

By keeping the SQL and Clojure separate you get:

- No syntactic surprises. Your database doesn't stick to the SQL
  standard (because none of them do); Yesql doesn't care. You will
  never spend time hunting for "the equivalent sexp syntax". You will
  never need to fall back to a `(raw-sql "some('funky'::SYNTAX)")` function.
- Better editor support. Your editor probably already has great SQL
  support. By keeping the SQL as SQL, you get to use it.
- Team interoperability. Your DBAs can read and write the queries you
  use in your Clojure project.
- Easier performance tuning. Need to `EXPLAIN` that query plan? It's
  much easier when your query is ordinary SQL.
- Query reuse. Drop the same SQL files into other projects, because
  they're just plain ol' SQL. Share them as a submodule.

## Installation

[Leiningen](https://github.com/technomancy/leiningen) dependency information:

``` clojure
[yesql "0.1.0-SNAPSHOT"]
```

## Example usage

Create an SQL query. Note we can supply named parameters and a comment string:

```sql
-- Counts the users in a given country.
SELECT count(*) AS count
FROM user
WHERE country_code = :country_code
```

Make sure it's on the classpath. For this example, it's in
`src/some/where/`. Now we can use it in our Clojure program.

```clojure
; Import the SQL query as a function.
(require '[yesql.core :refer [defquery]])
(defquery users-by-country "some/where/users_by_country.sql")
```

Lo! It has automatic, useful docstrings in the REPL:

```clojure
(clojure.repl/doc users-by-country)

;=> -------------------------
;=> user/users-by-country
;=> ([db country_code])
;=>   Counts the users in a given country.
```

Now we can use it:
```clojure
; Define a database connection spec. (This is standard clojure.java.jdbc.)
(def db-spec {:classname "org.postgresql.Driver"
              :subprotocol "postgresql"
              :subname "//localhost:5432/demo"
              :user "me"})

; Use it standalone. Note that the first argument is the db-spec.
(users-by-country db-spec "GB")
;=> ({:count 58})

; Use it in a clojure.java.jdbc transaction.
(require '[clojure.java.jdbc :as sql])
(sql/db-transaction [connection db-spec]
                    {:limeys (users-by-country connection "GB") 
                     :yanks  (users-by-country connection "US")})
```

## License

Copyright © 2013 Kris Jenkins

Distributed under the Eclipse Public License, the same as Clojure.