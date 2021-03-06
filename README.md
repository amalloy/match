match
====

An optimized pattern match and predicate dispatch library for Clojure. Currently the library only implements pattern matching. It supports Clojure 1.2.0 and later.

Usage
----

The fastest way to use this library is with Leiningen or Cake. Add the following to your project.clj dependencies:

```clojure
[match "0.1.0-SNAPSHOT"]
```

About
----

This library implements a pattern match compilation algorithm that uses the notion of "necessity" from lazy pattern matching.

For example the following pattern:

```clojure
(let [x true
      y true
      z true]
  (match [x y z]
     [_ false true] 1
     [false true _ ] 2
     [_ _ false] 3
     [_ _ true] 4))
```

expands into

```clojure
(cond
 (= y false) (cond
              (= z false) (let [] 3)
              (= z true) (let [] 1)
              :else (throw (java.lang.Exception. "No match found.")))
 (= y true) (cond
             (= x false) (let [] 2)
             :else (cond
                    (= z false) 3
                    (= z true) 4
                    :else (throw
                           (java.lang.Exception.
                            "No match found."))))
 :else (cond
        (= z false) (let [] 3)
        (= z true) (let [] 4)
        :else (throw (java.lang.Exception. "No match found."))))
```

Note that y gets tested first. Lazy pattern matching consistently gives compact decision trees. This means faster pattern matching. You can find out more in the top paper cited below.

Matching literals
----

```clojure
(let [x true
      y true
      z true]
  (match [x y z]
     [_ false true] 1
     [false true _ ] 2
     [_ _ false] 3
     [_ _ true] 4))
;; => 4
```

Wherever you would use a wildcard you can use a binding:

```clojure
(let [x 1 y 2 z 4]
  (match [x y z]
     [1 2 b] [:a0 b]
     [a 2 4] [:a1 a]))
;; => [:a0 4]
```

Seq matching
----

```clojure
(let [x [1 2 nil nil nil]]
   (match [x]
     [[1]]   :a0
     [[1 2]] :a1
     [[1 2 nil nil nil]] :a2))
;; => :a2
```

Seq patterns also support the familiar rest syntax from destructuring.

```clojure
(let [x '(1 2 3 4)]
  (match [x]
    [[1]] :a0
    [[_ 2 & [a & b]]] [:a1 a b]))
;; => [:a1 3 '(4)]
```

Map matching
----

```clojure
(let [x {:a 1 :b 1}]
   (match [x]
     [{:a _ :b 2}] :a0
     [{:a 1 :c _}] :a1
     [{:c 3 :d _ :e 4}] :a2))
;; => :a1
```

You can constrain map matching so that only maps with the exact key set will match:

```clojure
(match [x]
  [({:a _ :b 2} :only [:a :b])] :a0
  [{:a 1 :c c}] :a1
  [{:c 3 :d d :e 4}] :a2)
```

Special Syntax
----

The list syntax `()` is reserved for special uses. It does *not* match a literal list.

Or Patterns, Guards and As Patterns use this syntax.


Or Patterns
----

Or patterns are supported anywhere you would use a pattern:

```clojure
(let [x '(1 2 3)]
  (match [x y z ]
    [[1 (3 | 4) 3]] :a0
    [[1 (2 | 3) 3]] :a1))
;; => :a1
    
(let [x {:a 3}]
  (match [x y z ]
    [{:a (1 | 2)}] :a0
    [{:a (3 | 4)}] :a1))
;; => :a1
```

Guards
----

Guards are simple boolean tests. You can specify them like so:

```clojure
(let [y '(2 3 4 5)]
  (match [y]
    [[_ (a :when even?) _ _]] :a0
    [[_ (b :when [odd? div3?]) _ _]] :a1))
;; => :a1
```

As Patterns
----

Sometimes you'd like capture a part of the match with a binding:

```clojure
(let [v [[1 2]]]
  (match [v]
    [[3 1]] :a0
    [[([1 a] :as b)]] [:a1 a b]))
;; => [:a1 1 [1 2]]
```

Java Interop
----

By extending Java types to IMatchLookup, Java types can participate in map patterns:

```clojure
(extend-type java.util.Date
  IMatchLookup
  (val-at* [this k not-found]
    (case k
      :year    (.getYear this)
      :month   (.getMonth this)
      :date    (.getDate this)
      :hours   (.getHours this)
      :minutes (.getMinutes this)
      not-found)))

(let [d (java.util.Date. 2010 10 1 12 30)]
  (match [d]
    [{:year 2009 :month a}] [:a0 a]
    [{:year (2010 | 2011) :month b}] [:a1 b]))
;; => [:a1 10]
```

Note on Pattern Rows
----

A pattern row is delimited with `[]`, and is not a pattern itself.

For example, this syntax is illegal:

```clojure
(let [v 1]
  (match [v]
    ([1] :as w) :a0) ;; Illegal! [1] is a pattern row, not a pattern.
```

Matching single variables
---

`match.core/match-1` is sugar over `match` that allows pattern matching over a single variable, without
an "extra" pair of `[]` around the occurances and each pattern row.

```clojure
(let [x 3]
  (match-1 x
           1 :a0
           2 :a1
           :else :a2))
;=> :a2
```

This is equivalent to the following `match`.

```clojure
(let [x 3]
  (match [x]
         [1] :a0
         [2] :a1
         :else :a2))
;=> :a2
```

As shown, :else clauses are special, in that they are not implicitely wrapped in `[]`.


Road Map
----

A good chunk of Maranget's algorithm for pattern matching has been implemented. We would like to flesh out the pattern matching functionality. Once that work is done, we'll move on to predicate dispatch.

If you would like to see a more detailed account of what we're considering you can look here (https://github.com/swannodette/match/wiki/Design-Wiki).

Resources
----

The four most relevant papers:

* [Compiling Pattern Matching to Good Decision Trees](http://pauillac.inria.fr/~maranget/papers/ml05e-maranget.pdf)
* [Efficient Predicate Dispatch](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.47.4553)
* [Warnings for Pattern Matching](http://moscova.inria.fr/~maranget/papers/warn/index.html)
* [Extensible Pattern Matching for Extensible Languages](http://www.ccs.neu.edu/home/samth/ifl2010-slides.pdf)

Further reading:

* [Optimizing Pattern Matching](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.6.5507)
* [Pattern Guards And Transformational Patterns](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.35.8851)
* [Art of the Metaobject Protocol](http://mitpress.mit.edu/catalog/item/default.asp?ttype=2&tid=3925)
* [Custom Specializers in Object-Oriented Lisp](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.144.405&rep=rep1&type=pdf)

Copyright (C) 2011 David Nolen

Distributed under the Eclipse Public License, the same as Clojure.
