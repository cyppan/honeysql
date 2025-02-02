# Extending HoneySQL

Out of the box, HoneySQL supports most standard ANSI SQL clauses
and expressions but where it doesn't support something you need
you can add new clauses, new operators, and new "functions" (or
"special syntax").

There are three extension points in `honey.sql` that let you
register formatters or behavior corresponding to clauses,
operators, and functions.

Built in clauses include: `:select`, `:from`, `:where` and
many more. Built in operators include: `:=`, `:+`, `:mod`.
Built in functions (special syntax) include: `:array`, `:case`,
`:cast`, `:inline`, `:raw` and many more.

## Registering a New Clause Formatter

`honey.sql/register-clause!` accepts a keyword (or a symbol)
that should be treated as a new clause in a SQL statement,
a "formatter", and a keyword (or a symbol) that identifies
an existing clause that this new one should be ordered before.

The formatter can either be a function
of two arguments or a previously registered clause (so
that you can easily reuse formatters).

The formatter function will be called with:
* The clause name (always as a keyword),
* The sequence of arguments provided.

The third argument to `register-clause!` allows you to
insert your new clause formatter so that clauses are
formatted in the correct order for your SQL dialect.
For example, `:select` comes before `:from` which comes
before `:where`. You can call `clause-order` to see what the
current ordering of clauses is.

> Note: if you call `register-clause!` more than once for the same clause, the last call "wins". This allows you to correct an incorrect clause order insertion by simply calling `register-clause!` again with a different third argument.

## Registering a New Operator

`honey.sql/register-op!` accepts a keyword (or a symbol) that
should be treated as a new infix operator.

By default, operators are treated as strictly binary --
accepting just two arguments -- and an exception will be
thrown if they are provided less than two or more than
two arguments. You can optionally specify that an operator
can take any number of arguments with `:variadic true`:

```clojure
(require '[honey.sql :as sql])

(sql/register-op! :<=> :variadic true)
;; and then use the new operator:
(sql/format {:select [:*], :from [:table], :where [:<=> 13 :x 42]})
;; will produce:
;;=> ["SELECT * FROM table WHERE ? <=> x <=> ?" 13 42]
```

If you are building expressions programmatically, you
may want your new operator to ignore "empty" expressions,
i.e., where your expression-building code might produce
`nil`. The built-in operators `:and` and `:or` ignore
such `nil` expressions. You can specify `:ignore-nil true`
to achieve that:

```clojure
(sql/register-op! :<=> :variadic true :ignore-nil true)
;; and then use the new operator:
(sql/format {:select [:*], :from [:table], :where [:<=> nil :x 42]})
;; will produce:
;;=> ["SELECT * FROM table WHERE x <=> ?" 42]
```

## Registering a New Function (Special Syntax)

`honey.sql/register-fn!` accepts a keyword (or a symbol)
that should be treated as new syntax (as a function call),
and a "formatter". The formatter can either be a function
of two arguments or a previously registered "function" (so
that you can easily reuse formatters).

The formatter function will be called with:
* The function name (always as a keyword),
* The sequence of arguments provided.

For example:

<!-- :test-doc-blocks/skip -->
```clojure
(sql/register-fn! :foo (fn [f args] ..))

(sql/format {:select [:*], :from [:table], :where [:foo 1 2 3]})
```

Your formatter function will be called with `:foo` and `(1 2 3)`.
It should return a vector containing a SQL string followed by
any parameters:

```clojure
(sql/register-fn! :foo (fn [f args] ["FOO(?)" (first args)]))

(sql/format {:select [:*], :from [:table], :where [:foo 1 2 3]})
;; produces:
;;=> ["SELECT * FROM table WHERE FOO(?)" 1]
```

In practice, it is likely that your formatter would call
`sql/sql-kw` on the function name to produce a SQL representation
of it and would call `sql/format-expr` on each argument:

```clojure
(defn- foo-formatter [f [x]]
  (let [[sql & params] (sql/format-expr x)]
    (into [(str (sql/sql-kw f) "(" sql ")")] params)))

(sql/register-fn! :foo foo-formatter)

(sql/format {:select [:*], :from [:table], :where [:foo [:+ :a 1]]})
;; produces:
;;=> ["SELECT * FROM table WHERE FOO(a + ?)" 1]
```
