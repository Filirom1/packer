---
layout: "docs"
page_title: "Input Variables - HCL Configuration Language"
sidebar_current: "configuration-variables"
description: |-
  Input variables are parameters for Packer modules.
  This page covers configuration syntax for variables.
---

# Input Variables

Input variables serve as parameters for a Packer build, allowing aspects of the
build to be customized without altering the build's own source code.

When you declare variables in the build of your configuration, you can set
their values using CLI options and environment variables.

Input variable and local variable usage are introduced in the [_Variables
Guide_](/guides/hcl/variables).

-> **Note:** For brevity, input variables are often referred to as just
"variables" or "Packer variables" when it is clear from context what sort of
variable is being discussed. Other kinds of variables in Packer include
_environment variables_ (set by the shell where Packer runs) and _expression
variables_ (used to indirectly represent a value in an
[expression](./expressions.html)).

## Declaring an Input Variable

Each input variable accepted by a build must be declared using a `variable`
block :

```hcl
variable "image_id" {
  type = string
}

variable "availability_zone_names" {
  type    = list(string)
  default = ["us-west-1a"]
}

variable "docker_ports" {
  type = list(object({
    internal = number
    external = number
    protocol = string
  }))
  default = [
    {
      internal = 8300
      external = 8300
      protocol = "tcp"
    }
  ]
}
```

Or a less precise variables block:

```hcl
variables {
   foo = "value"
   my_secret = "foo"
}
```

The label after the `variable` keyword or a label of a `variables` block is a
name for the variable, which must be unique among all variables in the same
build. This name is used to assign a value to the variable from outside and to
reference the variable's value from within the build.

The `variable` block can optionally include a `type` argument to specify what
value types are accepted for the variable, as described in the following
section.

The `variable` declaration can also include a `default` argument. If present,
the variable is considered to be _optional_ and the default value will be used
if no value is set when calling the build or running Packer. The `default`
argument requires a literal value and cannot reference other objects in the
configuration.

## Using Input Variable Values

Within the build that declared a variable, its value can be accessed from
within [expressions](./expressions.html) as `var.<NAME>`, where `<NAME>`
matches the label given in the declaration block:

```hcl
source "googlecompute" "debian"  {
    zone = var.gcp_zone
    tags = var.gcp_debian_tags
}
```

The value assigned to a variable can be accessed only from expressions within
the folder where it was declared.

## Type Constraints

The `type` argument in a `variable` block allows you to restrict the [type of
value](./expressions.html#types-and-values) that will be accepted as the value
for a variable. If no type constraint is set then a value of any type is
accepted.

While type constraints are optional, we recommend specifying them; they serve
as easy reminders for users of the build, and allow Packer to return a helpful
error message if the wrong type is used.

Type constraints are created from a mixture of type keywords and type
constructors. The supported type keywords are:

* `string`
* `number`
* `bool`

The type constructors allow you to specify complex types such as collections:

* `list(<TYPE>)`
* `set(<TYPE>)`
* `map(<TYPE>)`
* `object({<ATTR NAME> = <TYPE>, ... })`
* `tuple([<TYPE>, ...])`

The keyword `any` may be used to indicate that any type is acceptable. For more
information on the meaning and behavior of these different types, as well as
detailed information about automatic conversion of complex types, see [Type
Constraints](./types.html).

If both the `type` and `default` arguments are specified, the given default
value must be convertible to the specified type.

## Input Variable Documentation

Because the input variables of a build are part of its user interface, you can
briefly describe the purpose of each variable using the optional `description`
argument:

```hcl
variable "image_id" {
  type        = string
  description = "The id of the machine image (AMI) to use for the server."
}
```

The description should concisely explain the purpose of the variable and what
kind of value is expected. This description string might be included in
documentation about the build, and so it should be written from the perspective
of the user of the build rather than its maintainer. For commentary for build
maintainers, use comments.

## Assigning Values to build Variables

When variables are declared in the build of your configuration, they can be set
in a number of ways:

* Individually, with the `-var` command line option.
* In variable definitions (`.pkrvars.hcl`) files, either specified on the
  command line or automatically loaded.
* As environment variables.

The following sections describe these options in more detail.

### Variables on the Command Line

To specify individual variables on the command line, use the `-var` option when
running the `packer build` command:

```
packer build -var="image_id=ami-abc123"
packer build -var='image_id_list=["ami-abc123","ami-def456"]'
packer build -var='image_id_map={"us-east-1":"ami-abc123","us-east-2":"ami-def456"}'
```

The `-var` option can be used any number of times in a single command.

### Variable Definitions (`.pkrvars.hcl`) Files

To set lots of variables, it is more convenient to specify their values in a
_variable definitions file_ (with a filename ending in either `.pkrvars.hcl` or
`.pkrvars.json`) and then specify that file on the command line with
`-var-file`:

```
packer build -var-file="testing.pkrvars"
```

A variable definitions file uses the same basic syntax as Packer language
files, but consists only of variable name assignments:

```hcl
image_id = "ami-abc123"
availability_zone_names = [
  "us-east-1a",
  "us-west-1c",
]
```

Packer also automatically loads a number of variable definitions files if they
are present:

* Any files with names ending in `.auto.pkrvars.hcl` or `.auto.pkrvars.json`.

Files whose names end with `.json` are parsed as JSON objects instead of HCL,
with the root object properties corresponding to variable names:

```json
{
  "image_id": "ami-abc123",
  "availability_zone_names": ["us-west-1a", "us-west-1c"]
}
```

### Environment Variables

As a fallback for the other ways of defining variables, Packer searches the
environment of its own process for environment variables named `PKR_VAR_`
followed by the name of a declared variable.

This can be useful when running Packer in automation, or when running a
sequence of Packer commands in succession with the same variables. For example,
at a `bash` prompt on a Unix system:

```
$ export PKR_VAR_image_id=ami-abc123
$ packer build gcp/debian/
...
```

On operating systems where environment variable names are case-sensitive,
Packer matches the variable name exactly as given in configuration, and so the
required environment variable name will usually have a mix of upper and lower
case letters as in the above example.

### Complex-typed Values

When variable values are provided in a variable definitions file, Packer's
[usual syntax](./expressions.html#structural-types) can be used to assign
complex-typed values, like lists and maps.

Some special rules apply to the `-var` command line option and to environment
variables. For convenience, Packer defaults to interpreting `-var` and
environment variable values as literal strings, which do not need to be quoted:

```
$ export PKR_VAR_image_id=ami-abc123
```

However, if a build variable uses a [type constraint](#type-constraints) to
require a complex value (list, set, map, object, or tuple), Packer will instead
attempt to parse its value using the same syntax used within variable
definitions files, which requires careful attention to the string escaping
rules in your shell:

```
$ export PKR_VAR_availability_zone_names='["us-west-1b","us-west-1d"]'
```

For readability, and to avoid the need to worry about shell escaping, we
recommend always setting complex variable values via variable definitions
files.

### Variable Definition Precedence

The above mechanisms for setting variables can be used together in any
combination. If the same variable is assigned multiple values, Packer uses the
_last_ value it finds, overriding any previous values. Note that the same
variable cannot be assigned multiple values within a single source.

Packer loads variables in the following order, with later sources taking
precedence over earlier ones:

* Environment variables
* Any `*.auto.pkrvars.hcl` or `*.auto.pkrvars.json` files, processed in lexical
  order of their filenames.
* Any `-var` and `-var-file` options on the command line, in the order they are
  provided.

~> **Important:** Variables with map and object values behave the same way as
other variables: the last value found overrides the previous values.