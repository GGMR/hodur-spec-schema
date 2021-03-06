[circleci-badge]: https://circleci.com/gh/hodur-org/hodur-spec-schema.svg?style=shield&circle-token=4efa55e1e836d3613c886708b4246b488090b263
[circleci]: https://circleci.com/gh/hodur-org/hodur-spec-schema
[clojars-badge]: https://img.shields.io/clojars/v/hodur/spec-schema.svg
[clojars]: http://clojars.org/hodur/spec-schema
[github-issues]: https://github.com/hodur-org/hodur-spec-schema/issues
[graphviz-colors]: https://www.graphviz.org/doc/info/colors.html
[graphviz]: https://www.graphviz.org/
[hodur-engine-clojars-badge]: https://img.shields.io/clojars/v/hodur/engine.svg
[hodur-engine-clojars]: http://clojars.org/hodur/engine
[hodur-engine-definition]: https://github.com/hodur-org/hodur-engine#model-definition
[hodur-engine-started]: https://github.com/hodur-org/hodur-engine#getting-started
[hodur-engine]: https://github.com/hodur-org/hodur-engine
[license-badge]: https://img.shields.io/badge/license-MIT-blue.svg
[license]: ./LICENSE
[logo]: ./docs/logo-tag-line.png
[motivation]: https://github.com/hodur-org/hodur-engine/blob/master/docs/MOTIVATION.org
[plugins]: https://github.com/hodur-org/hodur-engine#hodur-plugins
[spec]: https://clojure.org/guides/spec
[status-badge]: https://img.shields.io/badge/project%20status-beta-brightgreen.svg

# Hodur Spec Schema

[![CircleCI][circleci-badge]][circleci]
[![Clojars][hodur-engine-clojars-badge]][hodur-engine-clojars]
[![Clojars][clojars-badge]][clojars]
[![License][license-badge]][license]
![Status][status-badge]

![Logo][logo]

Hodur is a descriptive domain modeling approach and related collection
of libraries for Clojure.

By using Hodur you can define your domain model as data, parse and
validate it, and then either consume your model via an API making your
apps respond to the defined model or use one of the many plugins to
help you achieve mechanical, repetitive results faster and in a purely
functional manner.


> This Hodur plugin provides the ability to generate [Clojure
> Spec][spec] schemas out of your Hodur model. You can then validate
> your data structures, generate random payloads, extend yours
> tests... you name it.

## Motivation

For a deeper insight into the motivations behind Hodur, check the
[motivation doc][motivation].

## Getting Started

Hodur has a highly modular architecture. [Hodur Engine][hodur-engine]
is always required as it provides the meta-database functions and APIs
consumed by plugins.

Therefore, refer the [Hodur Engine's Getting
Started][hodur-engine-started] first and then return here for
Datomic-specific setup.

After having set up `hodur-engine` as described above, we also need to
add `hodur/spec-schema`, a plugin that creates Lacinia Schemas out
of your model to the `deps.edn` file:

``` clojure
  {:deps {hodur/engine      {:mvn/version "0.1.6"}
          hodur/spec-schema {:mvn/version "0.1.5"}}}
```

You should `require` it any way you see fit:

``` clojure
  (require '[hodur-engine.core :as hodur])
  (require '[hodur-spec-schema.core :as hodur-spec])
```

Let's expand our `Person` model from the original getting started by
"tagging" the `Person` entity for Spec. You can read more about the
concept of tagging for plugins in the sessions below but, in short,
this is the way we, model designers, use to specify which entities we
want to be exposed to which plugins.

``` clojure
  (def meta-db (hodur/init-schema
                '[^{:spec/tag-recursive true}
                  Person
                  [^String first-name
                   ^String last-name]]))
```

The `hodur-spec-schema` plugin exposes a function called `schema` that
returns a vector with all the spec definitions your model needs:

``` clojure
  (def spec-schema (hodur-spec/schema meta-db))
```

When you inspect `spec-schema`, this is what you have:

``` clojure
  [(clojure.spec.alpha/def
     :my-app.core.person/last-name
     clojure.core/string?)
   (clojure.spec.alpha/def
     :my-app.core.person/first-name
     clojure.core/string?)
   (clojure.spec.alpha/def
     :my-app.core/person
     (clojure.spec.alpha/keys
      :req-un
      [:my-app.core.person/first-name
       :my-app.core.person/last-name]
      :opt-un
      []))]
```

As a convenience, `hodur-spec-schema` also provides a macro called
`defspecs` that already defines all your specs onto your registry:

``` clojure
  (hodur-spec/defspecs meta-db)
```

Once `defspecs` is run, you'll have three specs to use:

- `:my-app.core.person/last-name`
- `:my-app.core.person/first-name`
- `:my-app.core/person`

Therefore, we can use spec normally like:

``` clojure
  (require '[clojure.spec.alpha :as s])

  (s/valid? :my-app.core/person {:first-name "Jane"
                                 :last-name "Janet"}) ;; => true

  (s/valid? :my-app.core/person {:firs-name "Jane"
                                 :last-name "Janet"}) ;; => false
```

## Model Definition

All Hodur plugins follow the [Model
Definition][hodur-engine-definition] as described on Hodur [Engine's
documentation][hodur-engine].

## Naming Conventions

For the sake of composability each of your entities, fields, and
parameters will have their own bespoke specs defined.

The convention is that each spec will have a fully-qualified name in
the namespace where `defspecs` is called pretty much as if a `::` was
used. Example:

``` clojure
  (ns my-app.core
    (:require [clojure.spec.alpha :as s]
              [hodur-engine.core :as hodur]
              [hodur-spec-schema.core :as hodur-spec]))

  (def meta-db (engine/init-schema
                '[^{:spec/tag-recursive true}
                  Person
                  [^String first-name
                   ^String last-name]]))

  (hodur-spec/defspecs meta-db)
  ;; => [:my-app.core.person/last-name
  ;;     :my-app.core.person/first-name
  ;;     :my-app.core/person]

  (s/valid? :my-app.core/person {:first-name "Jane"
                                 :last-name "Janet"}) ;; => true

  (s/valid? :my-app.core/person {:firs-name "Jane"
                                 :last-name "Janet"});; => false

  (s/explain :my-app.core/person {:firs-name "Jane"
                                  :last-name "Janet"})
  ;; prints out:
  ;; val: {:firs-name "Jane", :last-name "Janet"} fails spec: :hodur-spec-schema.core/person predicate: (contains? % :first-name)
```

## Influencing Names with Aliases and Prefix

Sometimes the default behavior of the naming convention above might
not suit you. There are two ways to affect the names.

The first one is to use `:prefix` on `defspecs`. It will override
the default namespace altogether. Example:

``` clojure
  (ns my-app.core
    (:require [clojure.spec.alpha :as s]
              [hodur-engine.core :as hodur]
              [hodur-spec-schema.core :as hodur-spec]))

  (def meta-db (engine/init-schema
                '[^{:spec/tag-recursive true}
                  Person
                  [^String first-name
                   ^String last-name]]))

  (hodur-spec/defspecs meta-db {:prefix :app})
  ;; => [:app.person/last-name
  ;;     :app.person/first-name
  ;;     :app/person]

  (s/valid? :app/person {:first-name "Jane"
                         :last-name "Janet"}) ;; => true
```

The second method is to use the marker `:spec/alias` or
`:spec/aliases` when defining entities, fields or parameters. Example:

``` clojure
  (ns my-app.core
    (:require [clojure.spec.alpha :as s]
              [hodur-engine.core :as hodur]
              [hodur-spec-schema.core :as hodur-spec]))

  (def meta-db (engine/init-schema
                '[^{:spec/tag-recursive true
                    :spec/alias :la/persona}
                  Person
                  [^{:spec/aliases [:a-persons/first-name
                                    :el/primo]}
                   ^String first-name
                   ^{:spec/aliases [:el/secondo]}
                   ^String last-name]]))

  (hodur-spec/defspecs meta-db)
  ;; => [:my-app.core.person/last-name
  ;;     :my-app.core.person/first-name
  ;;     :my-app.core/person
  ;;     :la/persona
  ;;     :a-persons/first-name
  ;;     :el/primo
  ;;     :el/secondo]

  (s/valid? :la/persona {:first-name "Jane"
                         :last-name "Janet"}) ;; => true

  (s/valid? :el/secondo "Janet") ;; => true
```

## Primitive Types

All Hodur primitive types have natural specs as described below:

| Hodur Type | Equivalent Spec |
|------------|-----------------|
| `String`   | `string?`       |
| `ID`       | `string?`       |
| `Integer`  | `integer?`      |
| `Boolean`  | `boolean?`      |
| `Float`    | `float?`        |
| `DateTime` | `inst?`         |

Other specs can be specified by using the `:spec/override` or
`:spec/extend` features described in more detail in the respective
section below.

## Cardinality

Multiple cardinalities are dealt with as expected. The following table
shows some examples:

| Hodur Cardinality      | Equivalent Spec                              |
|------------------------|----------------------------------------------|
| `nil` (none specified) | a single `<spec>`                            |
| `[0 n]`                | `s/coll-of <spec> :min-count 0`              |
| `[4 n]`                | `s/coll-of <spec> :min-count 4`              |
| `3`                    | `s/coll-of <spec> :count 3`                  |
| `[5 9]`                | `s/coll-of <spec> :min-count 5 :max-count 9` |
| `[n 7]`                | `s/coll-of <spec> :max-count 7`              |

## Interfaces

Hodur interfaces are supported. The approach taken is that the
resulting spec for the child entity is an `s/and` of itself and all of
its interfaces.

Take the following example:

``` clojure
  '[^:interface
    Animal
    [^String race]

    ^{:implements Animal}
    Person
    [^String first-name
     ^String last-name]]
```

The resulting high level specs would be `:app/animal` and
`:app/person` where `:app/person` needs to validate the keys in the
`Person` entity and also the keys on `Animal`.

## Enums and Unions

Hodur enums are spec'd as exact keywords or strings. Therefore the
hodur model below:

``` clojure
  '[^:enum
    Gender
    [FEMALE MALE]]
```

Will create two specs where one of them would be along the lines of
`:app.core.gender/female` where `#(= "FEMALE" (name %))` (one for
female and one for male).

The enum per se is an `s/or` between all of the enum's options.

If you need a different behavior, you can use `:spec/override`
described in the section below.

Hodur unions work similarly but the `s/or` is between the entities the
union refers to.

## Overriding and Extending

Specs can get very elaborate and Hodur models do not capture - nor
even try to capture - all the possibilities. Instead there are two
concepts in place: you can either override the spec that Hodur would
use or extend it.

Overriding is as simple as providing a marker `:spec/override` that
points to the function you want to use:

``` clojure
  '[MyEntity
    [{:spec/override keyword?}
     a-keyword-field]]
```

In the example above the spec for `a-keyword-field` will be simply
`keyword?`. You can also specify your own validation functions. Simply
make them fully qualified and make sure they have been required in the
correct context:

``` clojure
  '[User
    [{:spec/override my-app.user/email?}
     email]]
```

Then, just make sure you have something along these lines for your
email validation (or any other in fact):

``` clojure
  (ns my-app.user
    (:require [clojure.test.check.generators :as gen]))

  (defn email? [s]
    (let [email-regex #"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,63}$"]
      (re-matches email-regex s)))
```

Sometimes you are happy with the default spec used by Hodur but want
to extend it a bit. For instance, in the email example above you might
want to still make it a `string?` but also an email. By using the
marker `:spec/extend` you can automatically wrap the basic spec with
an `s/and`:

``` clojure
  '[User
    [{:type String
      :spec/extend my-app.user/email?}
     email]]
```

The resulting spec will be a `string?` `s/and` a
`my-app.validations/email?`.

## Custom Generators

Custom generators can be provided with the marker
`:spec/gen`. Example:

``` clojure
  '[User
    [{:type String
      :spec/extend my-app.user/email?
      :spec/gen my-app.user/gen-email}
     email]]
```

Then the hypothetical code below could validate and genarate out of a
set of possible emails:

``` clojure
  (ns my-app.user
    (:require [clojure.test.check.generators :as gen]))

  (defn email? [s]
    (let [email-regex #"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,63}$"]
      (re-matches email-regex s)))

  (defn email-gen []
    (gen/elements #{"asd@qwe.com" "qwe@asd.com" "foo@bar.edu" "bar@edu.br"}))
```

Once you have these in place, you can easily generate users like:

``` clojure
  (require '[clojure.spec.gen.alpha :as gen])
  (require '[clojure.spec.alpha :as s])

  (gen/sample (s/gen :app/user))
  ;; => [{:email "qwe@asd.com"}
  ;;     {:email "foo@bar.edu"}
  ;;     {:email "qwe@asd.com"}
  ;;     {:email "asd@qwe.com"}
  ;;     {:email "bar@edu.br"}]

  (s/valid? :app/user (gen/generate (s/gen :app/user))) ;; => true
```

## Parameters and Parameter Groups' Specs

Hodur parameters are each individually spec'd so that you are able to
run validations against specific entries in your functions.

In some situations though, it is also possible that you want to
validate the whole set of parameters as a group. This is particularly
useful if your parameters are set as a kind of argument map or ordered
tuple/vector.

Hodur's spec plugin will always create two specs for the parameter
group, one as a map and one as a tuple. What this means in practice is
that in the following example the specs `:app.core.user/avatar-url%`
and `:app.core.user/avatar-url-ordered%` are created.

`:app.core.user/avatar-url%` will represent a map that will include
the required entries `:max-width` and `:max-height`.

`:app.core.user/avatar-url-ordered%` will represent a tuple of two
integers (the first representing `:max-width` and the second
representing `:max-height`). As you can see, in this spec entry the
names of the parameters get lost. Another feature to notice is that
optional parameters are not supported in such case. This is as per
tuple spec.

``` clojure
  '[User
    [^String email
     ^String avatar-url [^Integer max-width
                         ^Integer max-height]]]
```

Special attention must be given to the naming convention here. A `%`
is added as a postfix to the name of the field the parameters
refer. For the ordered spec, a `-ordered%` is added. In the above
example, `:app.core.user/avatar-url` is the spec to the `avatar-url`
field (which happens to be a String - or `string?`),
`:app.core.user/avatar-url%` refers to the parameter group as a map,
and `:app.core.user/avatar-url-ordered%` refers to the parameter group
as a tuple..

You can also choose a different postfix when calling the `defspecs`
macro if `%` doesn't work for you. In the following example, instead
of `%`, `-params` will be used (for ordered specs, this will mean
`-ordered-params` will be used).

``` clojure
  (defspecs meta-db {:params-postfix "-params"})
```

## Bugs

If you find a bug, submit a [GitHub issue][github-issues].

## Help!

This project is looking for team members who can help this project
succeed! If you are interested in becoming a team member please open
an issue.

## License

Copyright © 2019 Tiago Luchini

Distributed under the MIT License (see [LICENSE][license]).
