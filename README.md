[![Build Status](https://github.com/dataddo/env/workflows/Go/badge.svg)](https://github.com/dataddo/env/actions)
[![Go Report Card](https://goreportcard.com/badge/github.com/dataddo/env)](https://goreportcard.com/report/github.com/dataddo/env)
[![PkgGoDev](https://pkg.go.dev/badge/go.dataddo.com/env)](https://pkg.go.dev/go.dataddo.com/env)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

```go
type config struct {
	Bool   bool   `env:"BOOL" default:"true"`
	Int    int    `env:"INT" default:"42"`
	String string `env:"STRING" default:"foo"`
}
```

# env

`env` is yet another package for parsing various data types from environment
variables. We decided to write this package as none of the available
packages met our needs. The closest one was
[envconfig](https://github.com/kelseyhightower/envconfig) but it has several
drawbacks, for example:

* The environment variables are optional by default. This is not aligned
  with our requirements as we believe that optional variables in deployment
  and environment configuration are bug prone practice. Writing
  `envconfig:required` annotation to every field is very annoying and
  pollutes the code.

* Variable name lookup policy is too complicated and therefore again bug prone.
  If there is variable named `VAR` loaded with prefix `PREFIX` and variable
  `PREFIX_VAR` does not exist in the environment, `envconfig` tries to read
  from variable `VAR` (without the prefix). If such variable exists in the
  environment by accident, completely different value is loaded and no one
  is notified about that.

* The name lookup is case insensitive which in general feels to magical and
  can again lead to errors by accident.

* On error, only the first one error is reported. This requires to re-run
  the program every time an error is resolved. This is also really annoying
  when deploying something for the first time or e.g. after the names of the
  variables have been refactored.

We would like to emphasize this piece of text is not aimed to be a rant and
we do not want to offend anyone. We rather want to give reasons why we have
decided to write the yet another package. We respect that use-case that
doesn't fit our needs can fit someone else's needs.

**It is worth mentioning that `env` does not have any external dependencies
beside standard library packages.**

## Usage

The usage is really straightforward. The environment variables are
identified using go annotations (`env` prefix followed by variable name). So
you can write your configuration structure like:

```go
type config struct {
	Foo
	Bar         `env:"BAR_"`
	Bool        *bool         `env:"BOOL"`
	Duration    time.Duration `env:"DURATION"`
	Inner       Foo           `env:"INNER_"`
	Int         int           `env:"INT"`
	IntSlice    *[]int        `env:"INT_SLICE"`
	String      string        `env:"STRING"`
	StringSlice []string      `env:"STRING_SLICE"`
}

type Foo struct {
	Foo string `env:"FOO"`
}

type Bar struct {
	Bar string `env:"BAR"`
}

```

As you can see, even structure embedding and composition is supported so you
can define arbitrarily complex structure. The naming scheme follows the
nesting, so for example the env variable corresponding to `Bar` string field
inside `Bar` struct will have name `BAR_BAR` (besides the global prefix, see
below). Nothing magical happens anywhere, e.g. the `_` separator is not
inserted automatically. Notice the `BAR_` prefix on `config.Bar`; without
the `_`, `BAR_BAR` would be `BARBAR`.

Similarly, no value will ever be loaded from a field which isn't env-tagged.
However, all struct data members (including the embedded ones) must be
exported to be recognized by `env`.

Embedded structs are traversed automatically (with no prefix, i.e. the
annotation is optional) to search for env-tagged fields. This is useful for
embedding of common configuration options.

Obviously the type of fields need not be defined types, i.e. it's possible
to write:

```go
type Foo struct {
    Bar struct {
        I int `env:"I"`
        J int `env:"J"`
    } `env:"BAR_"`
}
```

Finally, the configuration can be parsed and loaded like this:

```go
package main

import "go.dataddo.com/env"

func main() {
	var cfg config
	err := env.Load(&cfg, "PREFIX_")
}
```

All env variable names must be prefixed by `PREFIX_` in this example so the
final variable name will not be `BAR_BAR` but `PREFIX_BAR_BAR`. Empty prefix
is also allowed here.

### Default values

This fork supports default values defined in `default` tag.
All types are supported except the map, which is [a bit different](https://github.com/dataddo/env#map-keys-are-optional).

## Parsers

The actual parsing is driven by data-type of particular fields in the config
structure. Based on the data-type, concrete parser is chosen. The parser
look-up procedure goes as follows:

1. Default parser map is examined (see [default
   parsers](#list-of-default-parsers) for extensive list).  This map contains
   parsers for all primitive data types and also for some simple types from
   go base library (such as `time.Duration` or `url.URL`).
2. If the corresponding parser is not found in the default parsers map, we
   check if the type implements TextUnmarshaller interface and we use it for
   parsing the value.

For internal go composite types (such as pointers, slices or maps), we
provide built-in support. See below.

### Following pointers

When the value is a pointer (potentially a pointer to pointer, or pointer to
pointer to another pointer, etc.), we follow the pointer chain and
automatically initialize all intermediate pointers on the path. In this
respect, `env` behaves the same as the `json` package in the standard
library.

### Parsing slices

We also support parsing slices. If a struct field is declared as slice, the
corresponding value parsed from environment is treated as comma-separated
list of values loaded into the slice. This behavior is baked-in and is not
configurable.

The following rules apply to the slice parsing:

* Individual items are parsed recursively using the same rules according to
  their underlying data-type.
* The items are separated by comma. If one needs a comma to be present in a
  slice item, it must be escaped by back-slash, like this: `\,`. The same
  applies to the back-slash itself: `\\`. Generally, all characters prefixed
  by back-slash are expanded to the respective character after the
  back-slash.
* Double-quotes are also reserved for special use. Commas in double-quoted
  fields have no special meaning. If one needs a double-quote to be present
  in a slice item, it must be **always** escaped by back-slash like this:
  `\"`.
* All leading and trailing spaces at the item boundaries (before and after
  comma and also at the beginning and the end of the string) are ignored.
  Spaces inside double-quotes are never ignored.

### Parsing maps

Maps are treated in a special way. Map keys are bound to a suffix of the
environment variable name. Let's describe it by an example - suppose the
following code:

```go
type config struct {
	Map map[int]string `env:"MAP_"`
}

func main() {
	var c config
	err := env.Load(&cfg, "PREFIX_")
}
```

And the following environment variables exported:

```
$> export PREFIX_MAP_42=foo PREFIX_MAP_84=bar
```

This will produce the following map:

```go
map[int]string{
	42: "foo",
	84: "bar",
}
```

Individual map elements (both keys and values) are parsed recursively according
to their underlying data-types. Recursive parsing of maps is not supported.

#### Beware! Map keys are optional

There is one additional catch in the map parsing logic which you should be aware
of - environment variables defining map are implicitly **optional** and no
checking on their present is performed! For more information, please see [Map
keys are optional](#map-keys-are-optional) section below.

### List of default parsers

For these data-types, the parsing behavior is built-in (mostly using parsing
functions from standard library) and preferred over text unmarshaller:

- `Bool`
- `Float32`
- `Float64`
- `Int`
- `Uint`
- `Int8`
- `Uint8`
- `Int16`
- `Uint16`
- `Int32`
- `Uint32`
- `Int64`
- `Uint64`
- `String`
- `Regex`
- `Duration`
- `URL`
- `TextTemplate`


## Tests and examples

Please see our tests for more detailed examples.

## Contribution and bug reporting

If you would like to contribute, feel free to open a pull request here, on
GitHub. If the proposed changes will be reasonable, we will merge them to
upstream after proper review.

Also, if you would like to discuss anything related to this project, or you
would like to report a bug, please open a GitHub issue.

## Project status

The project was actively maintained by Showmax s.r.o., but it is not anymore.
Ex-Showmaxer adopted this project in Dataddo, and now it's Dataddo who uses it
internally and maintains it. It's possible that our vision of the project will
differ from the original one, but we will try to keep the project as backward
compatible as possible.

### Go version support

We support all [supported Go versions](https://go.dev/doc/devel/release#policy).
That means we support the last two major Go versions. With [Go's compatibility
promise](https://go.dev/doc/go1compat), there is no need to support more. 

## Design choices & Limitations

In this section, we will no longer describe `env` package behaviour, but we will
focus on specific design decisions and provide a reasoning behind them. For
those who are interested just in a quick start manual, feel free to skip this
section. Those of you who might want to propose some improvement of the package
or deeper understand why `env` package behaves in some potentially unexpected
way, this is the section which should provide you an explanation.

### Map keys are optional

As we described above, all environment variables defining maps are `optional`
even though we criticized `optional` variables at the very beginning of this
document. The reason of this behavior is the way map parsing is implemented. In
the case of all other data types (primitive types, arrays, `TextUnmarshaler`),
there exists a value of environment variable expressing an empty value - empty
string. As the operating system differentiates whether an environment variable
is not set or set to an empty string, we can say that all environment variables
are `required` and yet support parsing of an empty value. Unfortunately this
doesn't hold for maps.

In map parsing we denote every map key by environment variable presence, not by
its value! Consequence of this fact is that an empty map will be parsed from an
environment where no environment variable with a given prefix is present. As
parsing of an empty map is perfectly valid use-case, we cannot fail if no
environment variable is present and consequently environment variables defining
map content are `optional`. This optionality doesn't apply only to an empty map,
but in general to *any* map. If you specify `N` key-value pairs for the map and
you make a typo in one of those environment variables prefixes, the map will be
parsed with `N-1` keys and no error will be reported, because the parsing
algorithm simply cannot know you wanted to parse `N` values and not `N-1`
values. So again, even in the general case, the map cannot provide any strong
guarantees on environment variable presence.

The root cause of this is that we try to encode map in environment variables and
we are hitting limits of environment variable configuration. If we used a
JSON-based configuration or in general any other file-based configuration, we
would be able to express an empty map by an empty object. And we could `require`
the presence of the config file key even in case its content is empty. The same
again applies to the generic `N` and `N-1` problem. We could simply remedy it by
specifying `N` keys in the object and no typos would be possible as we would see
the `N` keys there. But environment variables are not generic enough to allow us
doing this. In other words, those properties of maps are not a bug, but a
consequence of environment variable based configuration parsing. For more
details, take a look at this GitHub
[issue](https://github.com/Showmax/env/issues/13) where this problematic was
discussed.
