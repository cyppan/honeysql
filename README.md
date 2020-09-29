# Honey SQL [![CircleCI](https://circleci.com/gh/seancorfield/honeysql/tree/v2.svg?style=svg)](https://circleci.com/gh/seancorfield/honeysql/tree/v2)

SQL as Clojure data structures. Build queries programmatically -- even at runtime -- without having to bash strings together.

## Build

The latest stable version (1.0.444) on Clojars and on cljdoc:

[![Clojars Project](https://clojars.org/honeysql/latest-version.svg)](https://clojars.org/honeysql) [![cljdoc badge](https://cljdoc.org/badge/honeysql/honeysql?1.0.444)](https://cljdoc.org/d/honeysql/honeysql/CURRENT)

This project follows the version scheme MAJOR.MINOR.COMMITS where MAJOR and MINOR provide some relative indication of the size of the change, but do not follow semantic versioning. In general, all changes endeavor to be non-breaking (by moving to new names rather than by breaking existing names). COMMITS is an ever-increasing counter of commits since the beginning of this repository.

This is the README for the upcoming 2.x version of HoneySQL which provides a streamlined codebase and a simpler method for extending the DSL. It also supports SQL dialects out-of-the-box and will be extended to support vendor-specific language features over time (unlike the 1.x version).

## Note on code samples

All sample code in this README is automatically run as a unit test using
[seancorfield/readme](https://github.com/seancorfield/readme).

Note that while some of these samples show pretty-printed SQL, this is just for
README readability; honeysql does not generate pretty-printed SQL.

## Usage

```clojure
(require '[honey.sql :as sql]
         '[honey.sql.helpers :refer :all :as helpers])
```

Everything is built on top of maps representing SQL queries:

```clojure
(def sqlmap {:select [:a :b :c]
             :from   [:foo]
             :where  [:= :f.a "baz"]})
```

Column names can be provided as keywords or symbols (but not strings -- HoneySQL treats strings as values that should be lifted out of the SQL as parameters).

### `format`

`format` turns maps into `next.jdbc`-compatible (and `clojure.java.jdbc`-compatible), parameterized SQL:

```clojure
(sql/format sqlmap)
=> ["SELECT a, b, c FROM foo WHERE f.a = ?" "baz"]
```

HoneySQL is a relatively "pure" library, it does not manage your sql connection
or run queries for you, it simply generates SQL strings. You can then pass them
to a JDBC library, such as [`next.jdbc`](https://github.com/seancorfield/next-jdbc):

```clj
(jdbc/execute! conn (sql/format sqlmap))
```

> Note: you'll need to add your preferred JDBC library as a dependency in your project -- HoneySQL deliberately does not make that choice for you.

_The handling of namespace-qualified keywords is under review in 2.x._

By default, namespace-qualified keywords are treated as simple keywords: their namespace portion is ignored. This was the behavior in HoneySQL prior to the 0.9.0 release and has been restored since the 0.9.7 release as this is considered the least surprising behavior.
As of version 0.9.7, `format` accepts `:allow-namespaced-names? true` to provide the somewhat unusual behavior of 0.9.0-0.9.6, namely that namespace-qualified keywords were passed through into the SQL "as-is", i.e., with the `/` in them (which generally required a quoting strategy as well).
As of version 0.9.8, `format` accepts `:namespace-as-table? true` to treat namespace-qualified keywords as if the `/` were `.`, allowing `:table/column` as an alternative to `:table.column`. This approach is likely to be more compatible with code that uses libraries like [`next.jdbc`](https://github.com/seancorfield/next-jdbc) and [`seql`](https://github.com/exoscale/seql), as well as being more convenient in a world of namespace-qualified keywords, following the example of `clojure.spec` etc.

```clojure
(def q-sqlmap {:select [:foo/a :foo/b :foo/c]
               :from   [:foo]
               :where  [:= :foo/a "baz"]})
(sql/format q-sqlmap)
=> ["SELECT foo.a, foo.b, foo.c FROM foo WHERE foo.a = ?" "baz"]
```

### Vanilla SQL clause helpers

_The code behind this section is a work-in-progress._

There are also functions for each clause type in the `honey.sql.helpers` namespace:

```clojure
(-> (select :a :b :c)
    (from :foo)
    (where [:= :f.a "baz"]))
```

Order doesn't matter (for independent clauses):

```clojure
(= (-> (select :*) (from :foo))
   (-> (from :foo) (select :*)))
=> true
```

When using the vanilla helper functions, repeated clauses will be merged into existing clauses (where that makes sense):

```clojure
(-> sqlmap (select :d))
=> '{:from [:foo], :where [:= :f.a "baz"], :select [:a :b :c :d]}
```

If you want to replace a clause, you can `dissoc` the existing clause first, since this is all data:

```clojure
(-> sqlmap
    (dissoc :select)
    (select :*)
    (where [:> :b 10])
    sql/format)
=> ["SELECT * FROM foo WHERE (f.a = ?) AND (b > ?)" "baz" 10]
```

`where` will combine multiple clauses together using SQL's `AND`:

```clojure
(-> (select :*)
    (from :foo)
    (where [:= :a 1] [:< :b 100])
    sql/format)
=> ["SELECT * FROM foo WHERE (a = ?) AND (b < ?)" 1 100]
```

Column and table names may be aliased by using a vector pair of the original
name and the desired alias:

```clojure
(-> (select :a [:b :bar] :c [:d :x])
    (from [:foo :quux])
    (where [:= :quux.a 1] [:< :bar 100])
    sql/format)
=> ["SELECT a, b AS bar, c, d AS x FROM foo quux WHERE (quux.a = ?) AND (bar < ?)" 1 100]
```

In particular, note that `(select [:a :b])` means `SELECT a AS b` rather than
`SELECT a, b` -- `select` is variadic and does not take a collection of column names.

### Inserts

Inserts are supported in two patterns.
In the first pattern, you must explicitly specify the columns to insert,
then provide a collection of rows, each a collection of column values:

```clojure
(-> (insert-into :properties)
    (columns :name :surname :age)
    (values
     [["Jon" "Smith" 34]
      ["Andrew" "Cooper" 12]
      ["Jane" "Daniels" 56]])
    (sql/format {:pretty? true}))
=> ["
INSERT INTO properties (name, surname, age)
VALUES (?, ?, ?), (?, ?, ?), (?, ?, ?)
"
"Jon" "Smith" 34 "Andrew" "Cooper" 12 "Jane" "Daniels" 56]
```


Alternately, you can simply specify the values as maps; the first map defines the columns to insert,
and the remaining maps *must* have the same set of keys and values:

```clojure
(-> (insert-into :properties)
    (values [{:name "John" :surname "Smith" :age 34}
             {:name "Andrew" :surname "Cooper" :age 12}
             {:name "Jane" :surname "Daniels" :age 56}])
    (sql/format {:pretty? true}))
=> ["
INSERT INTO properties (name, surname, age)
VALUES (?, ?, ?), (?, ?, ?), (?, ?, ?)
"
"John" "Smith" 34
"Andrew" "Cooper"  12
"Jane" "Daniels" 56]
```

### Nested subqueries

The column values do not have to be literals, they can be nested queries:

```clojure
(let [user-id 12345
      role-name "user"]
  (-> (insert-into :user_profile_to_role)
      (values [{:user_profile_id user-id
                :role_id         (-> (select :id)
                                     (from :role)
                                     (where [:= :name role-name]))}])
      (sql/format {:pretty? true})))

=> ["
INSERT INTO user_profile_to_role (user_profile_id, role_id)
VALUES (?, (SELECT id FROM role WHERE name = ?))
"
12345
"user"]
```

```clojure
(-> (select :*)
    (from :foo)
    (where [:in :foo.a (-> (select :a) (from :bar))])
    sql/format)
=> ["SELECT * FROM foo WHERE (foo.a in (SELECT a FROM bar))"]
```

### Composite types

Composite types are supported:

```clojure
(-> (insert-into :comp_table)
    (columns :name :comp_column)
    (values
     [["small" (composite 1 "inch")]
      ["large" (composite 10 "feet")]])
    (sql/format {:pretty? true}))
=> ["
INSERT INTO comp_table (name, comp_column)
VALUES (?, (?, ?)), (?, (?, ?))
"
"small" 1 "inch" "large" 10 "feet"]
```

### Updates

Updates are possible too:

```clojure
(-> (helpers/update :films)
    (set {:kind "dramatic"
           :watched [:+ :watched 1]})
    (where [:= :kind "drama"])
    (sql/format {:pretty? true}))
=> ["
UPDATE films SET kind = ?, watched = (watched + ?)
WHERE kind = ?
"
"dramatic"
1
"drama"]
```

If you are trying to build a compound update statement (with `from` or `join`),
be aware that different databases have slightly different syntax in terms of
where `SET` should appear. The default above is to put `SET` before `FROM` which
is how PostgreSQL (and other ANSI-SQL dialects work). If you are using MySQL,
you will need to select the `:mysql` dialect in order to put the `SET` after
any `JOIN` clause.

### Deletes

Deletes look as you would expect:

```clojure
(-> (delete-from :films)
    (where [:<> :kind "musical"])
    (sql/format))
=> ["DELETE FROM films WHERE kind <> ?" "musical"]
```

If your database supports it, you can also delete from multiple tables:

```clojure
(-> (delete [:films :directors])
    (from :films)
    (join :directors [:= :films.director_id :directors.id])
    (where [:<> :kind "musical"])
    (sql/format {:pretty? true}))
=> ["
DELETE films, directors
FROM films
INNER JOIN directors ON films.director_id = directors.id
WHERE kind <> ?
"
"musical"]
```

If you want to delete everything from a table, you can use `truncate`:

```clojure
(-> (truncate :films)
    (sql/format))
=> ["TRUNCATE films"]
```

### Set operations

Queries may be combined within a :union, :union-all, :intersect or :except keyword:

```clojure
(sql/format {:union [(-> (select :*) (from :foo))
                     (-> (select :*) (from :bar))]})
=> ["SELECT * FROM foo UNION SELECT * FROM bar"]
```

### Functions

Keywords that begin with `%` are interpreted as SQL function calls:

```clojure
(-> (select :%count.*) (from :foo) sql/format)
=> ["SELECT count(*) FROM foo"]
```
```clojure
(-> (select :%max.id) (from :foo) sql/format)
=> ["SELECT max(id) FROM foo"]
```

### Bindable parameters

_This is not currently supported._

Keywords that begin with `?` are interpreted as bindable parameters:

```clojure
(-> (select :id)
    (from :foo)
    (where [:= :a :?baz])
    (sql/format :params {:baz "BAZ"}))
=> ["SELECT id FROM foo WHERE a = ?" "BAZ"]
```

### Miscellaneous

TODO: need to update this section to reflect how to select a function call, how
to identify inline parameter values, and how to add in raw SQL fragments!

```clojure
(def call-qualify-map
  (-> (select [[:foo :bar]] [[:raw "@var := foo.bar"]])
      (from :foo)
      (where [:= :a [:param :baz]] [:= :b [:inline 42]])))
```
```clojure
call-qualify-map
=> '{:where [:and [:= :a [:param :baz]] [:= :b [:inline 42]]]
     :from (:foo)
     :select [[[:foo :bar]] [[:raw "@var := foo.bar"]]]}
```
```clojure
(sql/format call-qualify-map {:params {:baz "BAZ"}})
=> ["SELECT foo(bar), @var := foo.bar FROM foo WHERE (a = ?) AND (b = 42)" "BAZ"]
```

#### PostGIS

A common example in the wild is the PostGIS extension to PostgreSQL where you
have a lot of function calls needed in code:

```clojure
(-> (insert-into :sample)
    (values [{:location [:ST_SetSRID
                         [:ST_MakePoint 0.291 32.621]
                         [:cast 4325 :integer]]}])
    (sql/format {:pretty? true}))
=> ["
INSERT INTO sample (location)
VALUES (ST_SetSRID(ST_MakePoint(?, ?), CAST(? AS integer)))
"
0.291 32.621 4326]
```

#### Raw SQL fragments

_This functionality is under review._

Raw SQL fragments that are strings are treated exactly as-is when rendered into
the formatted SQL string (with no parsing or parameterization). Inline values
will not be lifted out as parameters, so they end up in the SQL string as-is.

Raw SQL can also be supplied as a vector of strings and values. Strings are
rendered as-is into the formatted SQL string. Non-strings are lifted as
parameters. If you need a string parameter lifted, you must use `:param`
or the `param` helper.

```clojure
(-> (select :*)
    (from :foo)
    (where [:< :expired_at [:raw ["now() - '" 5 " seconds'"]]])
    (sql/format {:foo 5}))
=> ["SELECT * FROM foo WHERE expired_at < now() - '? seconds'" 5]
```

```clojure
(-> (select :*)
    (from :foo)
    (where [:< :expired_at [:raw ["now() - '" [:param :t] " seconds'"]]])
    (sql/format {:t 5}))
=> ["SELECT * FROM foo WHERE expired_at < now() - '? seconds'" 5]
```

#### Identifiers

To quote identifiers, pass the `:quoted true` option to `format` and they will
be quoted according to the selected dialect. If you override the dialect in a
`format` call, by passing the `:dialect` option, identifiers will be automatically
quoted. You can override the dialect and turn off quoting by passing `:quoted false`.
Valid `:dialect` options are `:ansi` (the default, use this for PostgreSQL),
`:mysql`, `:oracle`, or `:sqlserver`:

```clojure
(-> (select :foo.a)
    (from :foo)
    (where [:= :foo.a "baz"])
    (sql/format {:dialect :mysql}))
=> ["SELECT `foo`.`a` FROM `foo` WHERE `foo`.`a` = ?" "baz"]
```

#### Locking

_This is not implemented yet._

To issue a locking select, add a `:lock` to the query or use the lock helper. The lock value must be a map with a `:mode` value. The built-in
modes are the standard `:update` (FOR UPDATE) or the vendor-specific `:mysql-share` (LOCK IN SHARE MODE) or `:postresql-share` (FOR SHARE). The
lock map may also provide a `:wait` value, which if false will append the NOWAIT parameter, supported by PostgreSQL.

```clojure
(-> (select :foo.a)
    (from :foo)
    (where [:= :foo.a "baz"])
    (lock :mode :update)
    (sql/format))
=> ["SELECT foo.a FROM foo WHERE foo.a = ? FOR UPDATE" "baz"]
```

To support novel lock modes, implement the `format-lock-clause` multimethod.

To be able to use dashes in quoted names, you can pass ```:allow-dashed-names true``` as an argument to the ```format``` function.
```clojure
(sql/format
  {:select [:f.foo-id :f.foo-name]
   :from [[:foo-bar :f]]
   :where [:= :f.foo-id 12345]}
  :allow-dashed-names? true
  :quoting :ansi)
=> ["SELECT \"f\".\"foo-id\", \"f\".\"foo-name\" FROM \"foo-bar\" \"f\" WHERE \"f\".\"foo-id\" = ?" 12345]
```

### Big, complicated example

Here's a big, complicated query. Note that Honey SQL makes no attempt to verify that your queries make any sense. It merely renders surface syntax.

```clojure
(def big-complicated-map
  (-> (select :f.* :b.baz :c.quux [:b.bla "bla-bla"]
              [[:now]] [[:raw "@x := 10"]])
      (modifiers :distinct) ; this is not implemented yet
      (from [:foo :f] [:baz :b])
      (join :draq [:= :f.b :draq.x])
      (left-join [:clod :c] [:= :f.a :c.d])
      (right-join :bock [:= :bock.z :c.e])
      (where [:or
               [:and [:= :f.a "bort"] [:not= :b.baz [:param :param1]]]
               [:< 1 2 3]
               [:in :f.e [1 [:param :param2] 3]]
               [:between :f.e 10 20]])
      (group :f.a :c.e)
      (having [:< 0 :f.e])
      (order-by [:b.baz :desc] :c.quux [:f.a :nulls-first])
      (limit 50)
      (offset 10)))
```
```clojure
big-complicated-map
=> {:select [:f.* :b.baz :c.quux [:b.bla "bla-bla"]
             [[:now]] [[:raw "@x := 10"]]]
    :modifiers [:distinct]
    :from [[:foo :f] [:baz :b]]
    :join [:draq [:= :f.b :draq.x]]
    :left-join [[:clod :c] [:= :f.a :c.d]]
    :right-join [:bock [:= :bock.z :c.e]]
    :where [:or
             [:and [:= :f.a "bort"] [:not= :b.baz [:param :param1]]]
             [:< 1 2 3]
             [:in :f.e [1 [:param :param2] 3]]
             [:between :f.e 10 20]]
    :group-by [:f.a :c.e]
    :having [:< 0 :f.e]
    :order-by [[:b.baz :desc] :c.quux [:f.a :nulls-first]]
    :limit 50
    :offset 10}
```
```clojure
(sql/format big-complicated-map {:param1 "gabba" :param2 2})
=> ["
SELECT DISTINCT f.*, b.baz, c.quux, b.bla AS bla_bla, now(), @x := 10
FROM foo f, baz b
INNER JOIN draq ON f.b = draq.x
LEFT JOIN clod c ON f.a = c.d
RIGHT JOIN bock ON bock.z = c.e
WHERE ((f.a = ? AND b.baz <> ?) OR (? < ? AND ? < ?) OR (f.e in (?, ?, ?)) OR f.e BETWEEN ? AND ?)
GROUP BY f.a, c.e
HAVING ? < f.e
ORDER BY b.baz DESC, c.quux, f.a NULLS FIRST
LIMIT ?
OFFSET ?
"
"bort" "gabba" 1 2 2 3 1 2 3 10 20 0 50 10]
```
```clojure
;; Printable and readable
(= big-complicated-map (read-string (pr-str big-complicated-map)))
=> true
```

## Extensibility

_This needs a rewrite!_

You can define your own function handlers for use in `where`:

```clojure
(defmethod fmt/fn-handler "betwixt" [_ field lower upper]
  (str (fmt/to-sql field) " BETWIXT "
       (fmt/to-sql lower) " AND " (fmt/to-sql upper)))

(-> (select :a) (where [:betwixt :a 1 10]) sql/format)
=> ["SELECT a WHERE a BETWIXT ? AND ?" 1 10]
```

You can also define your own clauses:

```clojure
;; Takes a MapEntry of the operator & clause data, plus the entire SQL map
(defmethod fmt/format-clause :foobar [[op v] sqlmap]
  (str "FOOBAR " (fmt/to-sql v)))
```
```clojure
(sql/format {:select [:a :b] :foobar :baz})
=> ["SELECT a, b FOOBAR baz"]
```
```clojure
(require '[honeysql.helpers :refer [defhelper]])

;; Defines a helper function, and allows 'build' to recognize your clause
(defhelper foobar [m args]
  (assoc m :foobar (first args)))
```
```clojure
(-> (select :a :b) (foobar :baz) sql/format)
=> ["SELECT a, b FOOBAR baz"]

```

When adding a new clause, you may also need to register it with a specific priority so that it formats correctly, for example:

```clojure
(fmt/register-clause! :foobar 110)
```

If you do implement a clause or function handler for an ANSI SQL, consider submitting a pull request so others can use it, too. For non-standard clauses and/or functions, look for a library that extends `honeysql` for that specific database or create one, if no such library exists.

## Why does my parameter get emitted as `()`?

If you want to use your own datatype as a parameter then the idiomatic approach of implementing
`next.jdbc`'s [`SettableParameter`](https://cljdoc.org/d/seancorfield/next.jdbc/CURRENT/api/next.jdbc.prepare#SettableParameter)
or `clojure.java.jdbc`'s [`ISQLValue`](https://clojure.github.io/java.jdbc/#clojure.java.jdbc/ISQLValue) protocol isn't enough as `honeysql` won't correct pass through your datatype, rather it will interpret it incorrectly.

To teach `honeysql` how to handle your datatype you need to implement [`honeysql.format/ToSql`](https://github.com/seancorfield/honeysql/blob/a9dffec632be62c961be7d9e695d0b2b85732c53/src/honeysql/format.cljc#L94). For example:
``` clojure
;; given:
(defrecord MyDateWrapper [...]
  (to-sql-timestamp [this]...)
)

;; executing:
(hsql/format {:where [:> :some_column (MyDateWrapper. ...)]})
;; results in => "where :some_column > ()"

;; we can teach honeysql about it:
(extend-protocol honeysql.format/ToSql
  MyDateWrapper
  (to-sql [v] (to-sql (date/to-sql-timestamp v))))

;; allowing us to now:
(hsql/format {:where [:> :some_column (MyDateWrapper. ...)]})
;; which correctly results in => "where :some_column>?" and the parameter correctly set
```
## TODO

- [ ] Create table, etc.

## Extensions

* [For PostgreSQL-specific extensions falling outside of ANSI SQL](https://github.com/nilenso/honeysql-postgres)

## License

Copyright (c) 2020 Sean Corfield. HoneySQL 1.x was copyright (c) 2012-2020 Justin Kramer and Sean Corfield.

Distributed under the Eclipse Public License, the same as Clojure.
