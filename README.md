<!-- markdownlint-disable MD013 MD033 -->
<p align="center"><img width="90%" src="kong.png" /></p>

# Kong is a command-line parser for Go
[![](https://godoc.org/github.com/alecthomas/kong?status.svg)](http://godoc.org/github.com/alecthomas/kong) [![CircleCI](https://img.shields.io/circleci/project/github/alecthomas/kong.svg)](https://circleci.com/gh/alecthomas/kong) [![Go Report Card](https://goreportcard.com/badge/github.com/alecthomas/kong)](https://goreportcard.com/report/github.com/alecthomas/kong) [![Slack chat](https://img.shields.io/static/v1?logo=slack&style=flat&label=slack&color=green&message=gophers)](https://gophers.slack.com/messages/CN9DS8YF3)

<!-- TOC -->

1. [Introduction](#introduction)
2. [Help](#help)
3. [Command handling](#command-handling)
    1. [1. Switch on the command string](#1-switch-on-the-command-string)
    2. [2. Attach a `Run(...) error` method to each command](#2-attach-a-run-error-method-to-each-command)
4. [Hooks: BeforeResolve(), BeforeApply(), AfterApply() and the Bind() option](#hooks-beforeresolve-beforeapply-afterapply-and-the-bind-option)
5. [Flags](#flags)
6. [Commands and sub-commands](#commands-and-sub-commands)
7. [Branching positional arguments](#branching-positional-arguments)
8. [Terminating positional arguments](#terminating-positional-arguments)
9. [Slices](#slices)
10. [Maps](#maps)
11. [Custom named decoders](#custom-named-decoders)
12. [Custom decoders (mappers)](#custom-decoders-mappers)
13. [Supported tags](#supported-tags)
14. [Variable interpolation](#variable-interpolation)
15. [Modifying Kong's behaviour](#modifying-kongs-behaviour)
    1. [`Name(help)` and `Description(help)` - set the application name description](#namehelp-and-descriptionhelp---set-the-application-name-description)
    2. [`Configuration(loader, paths...)` - load defaults from configuration files](#configurationloader-paths---load-defaults-from-configuration-files)
    3. [`Resolver(...)` - support for default values from external sources](#resolver---support-for-default-values-from-external-sources)
    4. [`*Mapper(...)` - customising how the command-line is mapped to Go values](#mapper---customising-how-the-command-line-is-mapped-to-go-values)
    5. [`ConfigureHelp(HelpOptions)` and `Help(HelpFunc)` - customising help](#configurehelphelpoptions-and-helphelpfunc---customising-help)
    6. [`Bind(...)` - bind values for callback hooks and Run() methods](#bind---bind-values-for-callback-hooks-and-run-methods)
    7. [Other options](#other-options)

<!-- /TOC -->

<a id="markdown-introduction" name="introduction"></a>
## Introduction

Kong aims to support arbitrarily complex command-line structures with as little developer effort as possible.

To achieve that, command-lines are expressed as Go types, with the structure and tags directing how the command line is mapped onto the struct.

For example, the following command-line:

    shell rm [-f] [-r] <paths> ...
    shell ls [<paths> ...]

Can be represented by the following command-line structure:

```go
package main

import "github.com/alecthomas/kong"

var CLI struct {
  Rm struct {
    Force     bool `help:"Force removal."`
    Recursive bool `help:"Recursively remove files."`

    Paths []string `arg name:"path" help:"Paths to remove." type:"path"`
  } `cmd help:"Remove files."`

  Ls struct {
    Paths []string `arg optional name:"path" help:"Paths to list." type:"path"`
  } `cmd help:"List paths."`
}

func main() {
  ctx := kong.Parse(&CLI)
  switch ctx.Command() {
  case "rm <path>":
  case "ls":
  default:
    panic(ctx.Command())
  }
}
```

<a id="markdown-help" name="help"></a>
## Help

Help is automatically generated. With no other arguments provided, help will display a full summary of all available commands.

eg.

    $ shell --help
    usage: shell <command>

    A shell-like example app.

    Flags:
      --help   Show context-sensitive help.
      --debug  Debug mode.

    Commands:
      rm <path> ...
        Remove files.

      ls [<path> ...]
        List paths.

If a command is provided, the help will show full detail on the command including all available flags.

eg.

    $ shell --help rm
    usage: shell rm <paths> ...

    Remove files.

    Arguments:
      <paths> ...  Paths to remove.

    Flags:
          --debug        Debug mode.

      -f, --force        Force removal.
      -r, --recursive    Recursively remove files.

For flags with associated environment variables, the variable `${env}` can be
interpolated into the help string. In the absence of this variable in the help, 


<a id="markdown-command-handling" name="command-handling"></a>
## Command handling

There are two ways to handle commands in Kong.

<a id="markdown-1-switch-on-the-command-string" name="1-switch-on-the-command-string"></a>
### 1. Switch on the command string

When you call `kong.Parse()` it will return a unique string representation of the command. Each command branch in the hierarchy will be a bare word and each branching argument or required positional argument will be the name surrounded by angle brackets. Here's an example:

There's an example of this pattern [here](https://github.com/alecthomas/kong/blob/master/_examples/shell/main.go).

eg.

```go
package main

import "github.com/alecthomas/kong"

var CLI struct {
  Rm struct {
    Force     bool `help:"Force removal."`
    Recursive bool `help:"Recursively remove files."`

    Paths []string `arg name:"path" help:"Paths to remove." type:"path"`
  } `cmd help:"Remove files."`

  Ls struct {
    Paths []string `arg optional name:"path" help:"Paths to list." type:"path"`
  } `cmd help:"List paths."`
}

func main() {
  ctx := kong.Parse(&CLI)
  switch ctx.Command() {
  case "rm <path>":
  case "ls":
  default:
    panic(ctx.Command())
  }
}
```

This has the advantage that it is convenient, but the downside that if you modify your CLI structure, the strings may change. This can be fragile.

<a id="markdown-2-attach-a-run-error-method-to-each-command" name="2-attach-a-run-error-method-to-each-command"></a>
### 2. Attach a `Run(...) error` method to each command

A more robust approach is to break each command out into their own structs:

1. Break leaf commands out into separate structs.
2. Attach a `Run(...) error` method to all leaf commands.
3. Call `kong.Kong.Parse()` to obtain a `kong.Context`.
4. Call `kong.Context.Run(bindings...)` to call the selected parsed command.

Once a command node is selected by Kong it will search from that node back to the root. Each
encountered command node with a `Run(...) error` will be called in reverse order. This allows
sub-trees to be re-used fairly conveniently.

In addition to values bound with the `kong.Bind(...)` option, any values
passed through to `kong.Context.Run(...)` are also bindable to the target's
`Run()` arguments.

Finally, hooks can also contribute bindings via `kong.Context.Bind()` and `kong.Context.BindTo()`.

There's a full example emulating part of the Docker CLI [here](https://github.com/alecthomas/kong/tree/master/_examples/docker).

eg.

```go
type Context struct {
  Debug bool
}

type RmCmd struct {
  Force     bool `help:"Force removal."`
  Recursive bool `help:"Recursively remove files."`

  Paths []string `arg name:"path" help:"Paths to remove." type:"path"`
}

func (r *RmCmd) Run(ctx *Context) error {
  fmt.Println("rm", r.Paths)
  return nil
}

type LsCmd struct {
  Paths []string `arg optional name:"path" help:"Paths to list." type:"path"`
}

func (l *LsCmd) Run(ctx *Context) error {
  fmt.Println("ls", l.Paths)
  return nil
}

var cli struct {
  Debug bool `help:"Enable debug mode."`

  Rm RmCmd `cmd help:"Remove files."`
  Ls LsCmd `cmd help:"List paths."`
}

func main() {
  ctx := kong.Parse(&cli)
  // Call the Run() method of the selected parsed command.
  err := ctx.Run(&Context{Debug: cli.Debug})
  ctx.FatalIfErrorf(err)
}

```

<a id="markdown-hooks-beforeresolve-beforeapply-afterapply-and-the-bind-option" name="hooks-beforeresolve-beforeapply-afterapply-and-the-bind-option"></a>
## Hooks: BeforeResolve(), BeforeApply(), AfterApply() and the Bind() option

If a node in the grammar has a `BeforeResolve(...)`, `BeforeApply(...) error` and/or `AfterApply(...) error` method, those methods will be called before validation/assignment and after validation/assignment, respectively.

The `--help` flag is implemented with a `BeforeApply` hook.

Arguments to hooks are provided via the `Run(...)` method or `Bind(...)` option. `*Kong`, `*Context` and `*Path` are also bound and finally, hooks can also contribute bindings via `kong.Context.Bind()` and `kong.Context.BindTo()`.

eg.

```go
// A flag with a hook that, if triggered, will set the debug loggers output to stdout.
type debugFlag bool

func (d debugFlag) BeforeApply(logger *log.Logger) error {
  logger.SetOutput(os.Stdout)
  return nil
}

var cli struct {
  Debug debugFlag `help:"Enable debug logging."`
}

func main() {
  // Debug logger going to discard.
  logger := log.New(ioutil.Discard, "", log.LstdFlags)

  ctx := kong.Parse(&cli, kong.Bind(logger))

  // ...
}
```

<a id="markdown-flags" name="flags"></a>
## Flags

Any [mapped](#mapper---customising-how-the-command-line-is-mapped-to-go-values) field in the command structure *not* tagged with `cmd` or `arg` will be a flag. Flags are optional by default.

eg. The command-line `app [--flag="foo"]` can be represented by the following.

```go
type CLI struct {
  Flag string
}
```

<a id="markdown-commands-and-sub-commands" name="commands-and-sub-commands"></a>
## Commands and sub-commands

Sub-commands are specified by tagging a struct field with `cmd`. Kong supports arbitrarily nested commands.

eg. The following struct represents the CLI structure `command [--flag="str"] sub-command`.

```go
type CLI struct {
  Command struct {
    Flag string

    SubCommand struct {
    } `cmd`
  } `cmd`
}
```

If a sub-command is tagged with `default:"1"` it will be selected if there are no further arguments.

<a id="markdown-branching-positional-arguments" name="branching-positional-arguments"></a>
## Branching positional arguments

In addition to sub-commands, structs can also be configured as branching positional arguments.

This is achieved by tagging an [unmapped](#mapper---customising-how-the-command-line-is-mapped-to-go-values) nested struct field with `arg`, then including a positional argument field inside that struct _with the same name_. For example, the following command structure:

    app rename <name> to <name>

Can be represented with the following:

```go
var CLI struct {
  Rename struct {
    Name struct {
      Name string `arg` // <-- NOTE: identical name to enclosing struct field.
      To struct {
        Name struct {
          Name string `arg`
        } `arg`
      } `cmd`
    } `arg`
  } `cmd`
}
```

This looks a little verbose in this contrived example, but typically this will not be the case.

<a id="markdown-terminating-positional-arguments" name="terminating-positional-arguments"></a>
## Terminating positional arguments

If a [mapped type](#mapper---customising-how-the-command-line-is-mapped-to-go-values) is tagged with `arg` it will be treated as the final positional values to be parsed on the command line.

If a positional argument is a slice, all remaining arguments will be appended to that slice.

<a id="markdown-slices" name="slices"></a>
## Slices

Slice values are treated specially. First the input is split on the `sep:"<rune>"` tag (defaults to `,`), then each element is parsed by the slice element type and appended to the slice. If the same value is encountered multiple times, elements continue to be appended.

To represent the following command-line:

    cmd ls <file> <file> ...

You would use the following:

```go
var CLI struct {
  Ls struct {
    Files []string `arg type:"existingfile"`
  } `cmd`
}
```

<a id="markdown-maps" name="maps"></a>
## Maps

Maps are similar to slices except that only one key/value pair can be assigned per value, and the `sep` tag denotes the assignment character and defaults to `=`.

To represent the following command-line:

    cmd config set <key>=<value> <key>=<value> ...

You would use the following:

```go
var CLI struct {
  Config struct {
    Set struct {
      Config map[string]float64 `arg type:"file:"`
    } `cmd`
  } `cmd`
}
```

For flags, multiple key+value pairs should be separated by `mapsep:"rune"` tag (defaults to `;`) eg. `--set="key1=value1;key2=value2"`.

<a id="markdown-custom-named-decoders" name="custom-named-decoders"></a>
## Custom named decoders

Kong includes a number of builtin custom type mappers. These can be used by
specifying the tag `type:"<type>"`. They are registered with the option
function `NamedMapper(name, mapper)`.

| Name              | Description                                       |
|-------------------|---------------------------------------------------|
| `path`            | A path. ~ expansion is applied.                   |
| `existingfile`    | An existing file. ~ expansion is applied.         |
| `existingdir`     | An existing directory. ~ expansion is applied.    |


Slices and maps treat type tags specially. For slices, the `type:""` tag
specifies the element type. For maps, the tag has the format
`tag:"[<key>]:[<value>]"` where either may be omitted.


<a id="markdown-custom-decoders-mappers" name="custom-decoders-mappers"></a>
## Custom decoders (mappers)

Any field implementing `encoding.TextUnmarshaler` or `json.Unmarshaler` will use those interfaces
for decoding values.

For more fine-grained control, if a field implements the
[MapperValue](https://godoc.org/github.com/alecthomas/kong#MapperValue)
interface it will be used to decode arguments into the field.

<a id="markdown-supported-tags" name="supported-tags"></a>
## Supported tags

Tags can be in two forms:

1. Standard Go syntax, eg. `kong:"required,name='foo'"`.
2. Bare tags, eg. `required name:"foo"`

Both can coexist with standard Tag parsing.

Tag                    | Description
-----------------------| -------------------------------------------
`cmd`                  | If present, struct is a command.
`arg`                  | If present, field is an argument.
`env:"X"`              | Specify envar to use for default value.
`name:"X"`             | Long name, for overriding field name.
`help:"X"`             | Help text.
`type:"X"`             | Specify [named types](#custom-named-types) to use.
`placeholder:"X"`      | Placeholder text.
`default:"X"`          | Default value.
`default:"1"`          | On a command, make it the default.
`short:"X"`            | Short name, if flag.
`required`             | If present, flag/arg is required.
`optional`             | If present, flag/arg is optional.
`hidden`               | If present, command or flag is hidden.
`format:"X"`           | Format for parsing input, if supported.
`sep:"X"`              | Separator for sequences (defaults to ","). May be `none` to disable splitting.
`mapsep:"X"`           | Separator for maps (defaults to ";"). May be `none` to disable splitting.
`enum:"X,Y,..."`       | Set of valid values allowed for this flag.
`group:"X"`            | Logical group for a flag or command.
`xor:"X"`              | Exclusive OR group for flags. Only one flag in the group can be used which is restricted within the same command.
`prefix:"X"`           | Prefix for all sub-flags.
`set:"K=V"`            | Set a variable for expansion by child elements. Multiples can occur.
`embed`                | If present, this field's children will be embedded in the parent. Useful for composition.

<a id="markdown-variable-interpolation" name="variable-interpolation"></a>
## Variable interpolation

Kong supports limited variable interpolation into help strings, enum lists and
default values.

Variables are in the form:

    ${<name>}
    ${<name>=<default>}

Variables are set with the `Vars{"key": "value", ...}` option. Undefined
variable references in the grammar without a default will result in an error at
construction time.

Variables can also be set via the `set:"K=V"` tag. In this case, those variables will be available for that
node and all children. This is useful for composition by allowing the same struct to be reused.

When interpolating into flag or argument help strings, some extra variables
are defined from the value itself:

    ${default}
    ${enum}
    ${env}

eg.

```go
type cli struct {
  Config string `type:"path" default:"${config_file}"`
}

func main() {
  kong.Parse(&cli,
    kong.Vars{
      "config_file": "~/.app.conf",
    })
}
```

<a id="markdown-modifying-kongs-behaviour" name="modifying-kongs-behaviour"></a>
## Modifying Kong's behaviour

Each Kong parser can be configured via functional options passed to `New(cli interface{}, options...Option)`.

The full set of options can be found [here](https://godoc.org/github.com/alecthomas/kong#Option).

<a id="markdown-namehelp-and-descriptionhelp---set-the-application-name-description" name="namehelp-and-descriptionhelp---set-the-application-name-description"></a>
### `Name(help)` and `Description(help)` - set the application name description

Set the application name and/or description.

The name of the application will default to the binary name, but can be overridden with `Name(name)`.

As with all help in Kong, text will be wrapped to the terminal.

<a id="markdown-configurationloader-paths---load-defaults-from-configuration-files" name="configurationloader-paths---load-defaults-from-configuration-files"></a>
### `Configuration(loader, paths...)` - load defaults from configuration files

This option provides Kong with support for loading defaults from a set of configuration files. Each file is opened, if possible, and the loader called to create a resolver for that file.

eg.

```go
kong.Parse(&cli, kong.Configuration(kong.JSON, "/etc/myapp.json", "~/.myapp.json"))
```

[See the tests](https://github.com/alecthomas/kong/blob/master/resolver_test.go#L103) for an example of how the JSON file is structured.

<a id="markdown-resolver---support-for-default-values-from-external-sources" name="resolver---support-for-default-values-from-external-sources"></a>
### `Resolver(...)` - support for default values from external sources

Resolvers are Kong's extension point for providing default values from external sources. As an example, support for environment variables via the `env` tag is provided by a resolver. There's also a builtin resolver for JSON configuration files.

Example resolvers can be found in [resolver.go](https://github.com/alecthomas/kong/blob/master/resolver.go).

<a id="markdown-mapper---customising-how-the-command-line-is-mapped-to-go-values" name="mapper---customising-how-the-command-line-is-mapped-to-go-values"></a>
### `*Mapper(...)` - customising how the command-line is mapped to Go values

Command-line arguments are mapped to Go values via the Mapper interface:

```go
// A Mapper knows how to map command-line input to Go.
type Mapper interface {
  // Decode scan into target.
  //
  // "ctx" contains context about the value being decoded that may be useful
  // to some mapperss.
  Decode(ctx *MapperContext, scan *Scanner, target reflect.Value) error
}
```

All builtin Go types (as well as a bunch of useful stdlib types like `time.Time`) have mappers registered by default. Mappers for custom types can be added using `kong.??Mapper(...)` options. Mappers are applied to fields in four ways:

1. `NamedMapper(string, Mapper)` and using the tag key `type:"<name>"`.
2. `KindMapper(reflect.Kind, Mapper)`.
3. `TypeMapper(reflect.Type, Mapper)`.
4. `ValueMapper(interface{}, Mapper)`, passing in a pointer to a field of the grammar.

<a id="markdown-configurehelphelpoptions-and-helphelpfunc---customising-help" name="configurehelphelpoptions-and-helphelpfunc---customising-help"></a>
### `ConfigureHelp(HelpOptions)` and `Help(HelpFunc)` - customising help

The default help output is usually sufficient, but if not there are two solutions.

1. Use `ConfigureHelp(HelpOptions)` to configure how help is formatted (see [HelpOptions](https://godoc.org/github.com/alecthomas/kong#HelpOptions) for details).
2. Custom help can be wired into Kong via the `Help(HelpFunc)` option. The `HelpFunc` is passed a `Context`, which contains the parsed context for the current command-line. See the implementation of `PrintHelp` for an example.

<a id="markdown-bind---bind-values-for-callback-hooks-and-run-methods" name="bind---bind-values-for-callback-hooks-and-run-methods"></a>
### `Bind(...)` - bind values for callback hooks and Run() methods

See the [section on hooks](#BeforeApply-AfterApply-and-the-bind-option) for details.

<a id="markdown-other-options" name="other-options"></a>
### Other options

The full set of options can be found [here](https://godoc.org/github.com/alecthomas/kong#Option).
