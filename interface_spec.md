# Interface specification files

## Abstract

Interface specification files allow developers of applications using interface
file to specify their interfaces and fields in a way that can be processed by
machines. This enables automatic code generation and documentation generation.

## Motivation

Once there is a standard for configuring service interfaces, there's boilerplate
code that everyone has to write. Parsing the interface file is just the tip of
the iceberg, which can be easily solved by a library. Field validation,
documentation and connecting to the service is yet another code.

Further, additional overhead disincentivizes the implementation of interface
files, slowing down spreading of a useful standard.

Tooling that helps with these issues is needed and the tooling needs inputs.
This document specifies the input to such tooling.

## Specification format

As opposed to interface files themselves, their specification is full-blown
toml. Expresiveness of the language is preferred in this case over simplicity,
as the tooling will not be a dependency of the resulting software, but build
dependency.

## Meta fields

All mandatory fields from the interface are also mandatory fields of the
specification and their values match exactly:

* `_spec_vesion = "1.0"` (not required to be on top of file, but recommended)
* `_iface_name = "INTERFACE_NAME"`
* `_iface_version = "SEMVER"`

## Custom fields

Each custom field is speciefied as:

```toml
key = {
	type = "field_type",
	summary = "Short description of the field",
	doc = "Long documentation of the field",
	optional = true,
	default = VALUE,
	conflicts = ["other_key"],
	meta.tool.foo = "bar"
}
```

`type` MUST be defined, `summary` SHOULD be defined, it's perfectly fine to not
define `doc` if the meaning of the key is evident from `summary`.
`optional` has default value `false` and MAY be set to true to indicate that
the field is optional.
`default` can be used to specify recommended default value. It MUST NOT be
relied on by service provider and the client, but MAY be processed by tools.
In other words, presence of `default` implies `optional = false`.  This is to
ensure that the change in default value across versions doesn't affect older
clients. Change in default value would require major version increase otherwise.
`conflicts` is optional and lists fields that MUST NOT be specified at the same
time.
`meta.tool` allows users to configure specific `tool`, if needed. Tools that
don't need any special information MUST ignore it.

**Important: Changing optional field to mandatory is backwards-compatible
change with respect to clients, so *minor* version MUST be increased. Changing
mandatory field to optional is backwards-incompatible and *major* version MUST
be increased. This is counter-intuitive, but true, that's why there's this
warning.**

## Types

Types used in the specification represent semantic meaning of the field, not
underlying implementation. There are these types:

* `port_number` - TCP or UDP port number; represented as integer in Toml
* `unix_socket` - path pointing to Unix socket; represented as string in Toml
* `path` - any filesystem path; represented as string in Toml
* `secret_token` - A secret token; represented as string in Toml
* `string` - An arbitrary string
* `integer` - An arbitrary integer
* `bool` - Boolean

## Predefined fields

Predefined fields correspond to "Blessed fields" in the interface files. In
order to help with implementation, if a service is compliant with respect to
blessed fields, the field in interface specification MAY start with `-` to
indicate so. All tools MUST infer the type and documentation from the name.

In other words, writing:

```toml
_iface_name = "my_iface"
-port = {}
-unix_socket = {}
-auth_token = {}
-cookie_file = {}
```

is equivalent to:

```toml
_iface_name = "my_iface"
port = {
	type = "port_number",
	summary = "Port number of my_iface",
	doc = "The port number which uniquely identifies the service on the machine"
}
unix_socket = {
	type = "unix_socket",
	summary = "Path to the Unix socket of my_iface",
	doc = "The filesystem path which uniquely identifies the service on the machine"
}
auth_token = {
	type = "secret_token",
	summary = "Authentication secret for my_iface",
	doc = "Secret required for authentication with the service"
}
cookie_file = {
	type = "cookie_file",
	summary = "Path to cookie file of my_iface",
	doc = "Path to a file containing authentication secret."
}
```

When using predefined fields, all properties except `type` MAY be overriden
simply by specifying them. For example:

```toml
-port = {
	summary = "Very important port number"
}
```

## Implementations and tools

There are no implementations of this RFC. This section will contain them once
they are released.
