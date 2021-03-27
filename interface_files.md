# Interface files RFC version 1.0 - DRAFT

## Abstract

Interface files specify connection details of services that are present in the
system. They unify the way services are configured, making it much easier to
connect client to the server.

## Motivation

There are various different daemons running on common operating systems. They
are providing services to the clients and to each-other. With the increasing
number of services, the maintainance burden increases. The administrator has to
configure each parameter for each service manually, double checking two
documentations, since configuration fields are often named differently. Even if
this is handled by package management, the same burden lies on the package
maintainer.

The aim of this RFC is to concentrate such configuration into a single file for
each interface. This enables the administrators to specify a file containing
the configuration. Further, since the files are versioned, they may help
discover incompatibility sooner.

## Format specification

The format of interface file is a strict subset of Toml format. It can be
parsed with any general Toml parser, but a simpler parser can be used if
needed.

Supported constructs:

* `key = "value"` with correctly escaped `value`, there may be any number of
                  spaces around `=`
* `key = true`, `key = false` - Toml boolean
* `key = 123` - Toml integer

Further, if advanced features are needed, it is allowed to use subsections
marked with square bracket (e.g. `[some_section]`) The top-most section must
begin at the beginning of the line.

Simple parser MUST implement above mentioned `key = value` records and MUST
stop when it hits left square bracket at the beginning of the line, ignoring
the rest. Simple parser MUST also stop if it doesn't support `_spec_version`
(see below).

The providing services SHOULD avoid extra configuration in subsection whenever
possible, in order to not force the consuming services into implementing full
Toml.

### Rationale

Toml format is well-established and easy to read and write for humans. However,
it's also quite complex and thus supports complex configuration. The advantage
of using a well-established format is no need to write yet another parser. On
the other hand, restricting the beginning of the file to subset of Toml makes
it possible for projects that don't want to introduce a new dependency to write
a very simple parser. As we show in the following section this restriction
doesn't cause any significant problems.

## Meta fields

The interface file MUST contain these fields:

* `_spec_vesion = "1.0"` - MUST be at the top of the file
* `_iface_name = "INTERFACE_NAME"`
* `_iface_version = "SEMVER"`

The interface file SHOULD contain these fields:

* `_instance_id = "UNIQUE IDENTIFIER OF THE INSTANCE"`

The interface file MAY contain these fields:

* `_impl_name = "IMPLEMENTATION NAME"`

These fields work as sanity check to detect incompatibility early.

### Specification version

Specification version is defined by field `_spec_version`. Major version of
specification increases if mandatory fields are renamed or removed. Minor
version increases if it puts new requirements on simple parser. In other words:

* Basic parser MUST stop if the version doesn't equal "1.0"
* Toml parser MAY always *parse* the whole file, but if major version is more
  than 1, further processing MUST be avoided.

### Interface name

The name of the interface is specified by field `_iface_name`. It's considered
a global identifier of the interface. All implementations MUST stop processing
the file if the name doesn't match what they expect.

The interface name MUST match this regex: `[a-z][a-z0-9_.]*`

### Interface version

The version of the interface is specified by field `_iface_version`. It MUST
follow semantic versioning:

* Major MUST be increased when a field is removed or renamed.
* Major MUST be increased when the communication protocol changed in
  backwards-incompatible manner.
* Minor SHOULD be increased when a field s added to the file.
* Minor SHOULD be increased when there's a backward-compatible change in the
  communication protocol.

If the service provider loads its configuration from interface file, it MUST
report an error and disable the interface if it can not provide the specified
version of the interface.

If the client loads its configuration from interface file, it MUST report an
error and disable the interface if it can't use the specified version of the
interface.

### Virtual: Merged version

Merged version can be tought of as the version of the whole file. The version is
computed from specification version and interface version using this pattern
`iface_major.spec_major.spec_minor.iface_minor`

While it may seem strange, this kind of versioning scheme allows for specifying
dependencies in a very robust way. Let's say a file has the spec version `1.0`
and interface version `2.3`. The resulting version is `2.1.0.3`.

* The simple parser implementation depends on >= 2.1.0.3 && < 2.1.1.0
* The full parser implementation depends on >= 2.1.0.3 && < 2.2.0.0

This scheme also allows for phasing out old format specifications easily. When
a new version of specification is implemented in dependent software, they
increase maximum version, but providing software keeps its version. Once all
packages no longer dependd on old specification version, the interface file is
upgraded and the version number of providing software gets increased.

This scheme also allows for supporting multiple major versions of the protocol,
while they have the same specification version even with restricted syntax such
as Debian version specification.

Using Debian syntax, a package with full parser supporting all versions of `foo`
from 2.3 to 3.0 would depend on:
`foo (<<3.2.0.0), foo (<<2.2.0.0) | foo (>=3.1.0.0), foo (>=2.1.0.3)`

In case of simple parser, it would depend on:
`foo (<<3.2.1.0), foo (<<2.2.1.0) | foo (>=3.1.0.0), foo (>=2.1.0.3)`

Similarly, it's possible (although maybe clunky) to support an arbitrary number
of major versions by repeating the condition in the middle.

### Instance ID

The instance identifier allows de-duplicating services across different interfaces.
If a service provides two different alternative interfaces and the client supports both
then instance ID is mandatory and it's used to detect that there's only one service, not
two.

`lnd` and `lncli` are real-life examples where this is required.

The instance ID is considered free-form, but with these recommendations:

* In case of local service, use a path pointing to some file tied to that instance
  (e.g. /etc/systemd/system/something.service)
* If the remote instance has its own (natural) ID, use that one (e.g. public key)
* If none of the above applies, use 16 random bytes encoded as hex

### Implementation name

The name of implementation MAY be specified in `_impl_name` field. The service
provider MUST stop processing the file and disable the interface, if the name
doesn't match its name. The client MAY disable some features or enable
additional features, but MUST NOT disable the interface entirely, for any
specific implementation.

The implementation name must match regex: `[a-z][a-z0-9_]*`

#### Rationale

The primary purpose of this field is reporting and possibly workarounds/patches
for specific implementations, not compatibility checking. If an interface
specifies that certain features are required, then the compliant implementors
MUST implement all of them. In case a subset of features is desired, a new
interface with a different name must be specified.

## Custom fields

All custom fields MUST match regex `[a-z][a-z0-9_]*`. If the name of the field
is composed of several words, they SHOULD be separated by `_`. There are no
other restrictions.

### Blessed fields

In order to make interfacce files easy to understand and consistent, all
implementations SHOULD use following names whenever relevant:

* `port` - specifies TCP or UDP port, contains valid port number as Toml integer
* `unix_socket` - filesystem path to unix socket
* `auth_token` - string, arbitrary authentication token
* `cookie_file` - path to file containing authentication token

### Rationale

The regex above is what all popular programming languages accept as identifier.
This makes it easier to create tools for automatic code generation. While it
might happen that some languages have a different convention (such as
PascalCase or camelCase) when it comes to naming, the restriction actually
enables creating bijective mapping between field names in interface files and
field names used in code with idiomatic casing for the language.

Looking at many different projects, each has a different name for what's often
the same thing: port number (especially when there's more than one port
numbers), authentication token, unix socket... It's difficult to remember the
name for each project and one has to search for the name every time.

Recommending a particular name helps people remember the exact name and keeps
flexibility, since different names aren't prohibited.

While one might argue that a service could bind multiple ports, it's rarely the
case that the ports are used for the same interface. Since the interfaces are
different, they SHOULD use different interface files, each using field of the
name `port`. Same principle should be applied to other means of inter-process
communication.

## Extension: alternative implementations

The clients MAY support alternative implementations of the same interface or
even using alternative interfaces for the same task. In order to support such
scenario in a flexible way, the individual interface files or symlinks pointing
to them are placed in a common directory. A special file called `_default` is a
symlink pointing (directly or indirectly) to the preferred implementation.

The directory MUST be searched recursively, but `_default` files in
subdirectories MUST be ignored. `_default` file MUST NOT point to a directory.
This allows making union of per-user instances with system instances.

If two feature-equivalent interfaces are available they MUST be deduplicated
using `_instance_id`. The client is free to prefer any of the equivalent
interfaces and MAY be configurable.

The clients that don't support alternative implementations MUST be configured
to read the `_default` file.

## Example: Bitcoin ecosystem

There are many services related to Bitcoin, which need to communicate between
each other. This example shows how they could be configured if all of them
supported interface files.

### Bitcoin Core (AKA bitcoind)

#### Provides

```toml
_spec_vesion = "1.0"
_iface_name = "bitcoin_rpc_mainnet"`
_iface_version = "1.0"`

port = 8331
cookie_file = "/var/run/bitcoin-mainnet/cookie"
```

```toml
_spec_vesion = "1.0"
_iface_name = "bitcoin_p2p_mainnet"`
_iface_version = "1.0"`

port = 8333
```
```toml
_spec_vesion = "1.0"
_iface_name = "bitcoin_zmq_block_mainnet"
_iface_version = "1.0"

port = 28332
```

```toml
_spec_vesion = "1.0"
_iface_name = "bitcoin_zmq_txmainnet"
_iface_version = "1.0"

port = 28333
```
### Bitcoin RPC proxy

#### Depends

* `bitcoin_rpc_mainnet`

#### Provides

```toml
_spec_vesion = "1.0"
_iface_name = "bitcoin_timechain_mainnet"
_iface_version = "1.0"

port = 8332
```

### Electrs

#### Depends

* `bitcoin_timechain_mainnet`

#### Provides

```toml
_spec_vesion = "1.0"
_iface_name = "electrum_server_mainnet"
_iface_version = "1.0"

port = 50001
```

### ElectrumX

#### Depends

* `bitcoin_timechain_mainnet`

#### Provides

```toml
_spec_vesion = "1.0"
_iface_name = "electrum_server_mainnet"
_iface_version = "1.0"

port = 50002
```

### Alternatives: `electrum_server_mainnet`

* `/etc/electrum-server-mainnet/electrs`
* `/etc/electrum-server-mainnet/electrumx`
* `/etc/electrum-server-mainnet/_default -> electrumx`

(ElectrumX has more features, but it's more resource-consuming.)

### LND

#### Depends

* `bitcoin_timechain_mainnet`
* `bitcoin_zmq_block_mainnet`
* `bitcoin_zmq_tx_mainnet`

#### Provides

```toml
_spec_vesion = "1.0"
_iface_name = "lnd_rest"
_iface_version = "1.0"

port = 8080
admin_macaroon = "/var/lib/lnd-mainnet/admin.macaroon"
invoice_macaroon = "/var/lib/lnd-mainnet/invoice.macaroon"
readonly_macaroon = "/var/lib/lnd-mainnet/readonly.macaroon"
```

```toml
_spec_vesion = "1.0"
_iface_name = "lnd_grpc"
_iface_version = "1.0"

port = 10009
```

```toml
_spec_vesion = "1.0"
_iface_name = "lightning_network_p2p"
_iface_version = "1.0"

port = 9735
```

### Alternatives: `lnd_rpc_mainnet`

* `/etc/lnd-mainnet/lnd.rest.clearnet`
* `/etc/lnd-mainnet/lnd.rest.tor`
* `/etc/lnd-mainnet/lnd.grpc.clearnet`
* `/etc/lnd-mainnet/lnd.grpc.tor`
* `/etc/lnd-mainnet/_default -> lnd.grpc.tor`

### Eclair

#### Depends

* `bitcoin_rpc_mainnet`
* `bitcoin_zmq_block_mainnet`
* `bitcoin_zmq_tx_mainnet`

#### Provides

```toml
_spec_vesion = "1.0"
_iface_name = "eclair_rest"
_iface_version = "1.0"

port = 8081
auth_token = "nbusr123"
```

```toml
_spec_vesion = "1.0"
_iface_name = "lightning_network_p2p"
_iface_version = "1.0"

port = 9736
```

### Alternatives: `ln_wallet_system_mainnet`

* `/etc/ln-wallet-system-mainnet/eclair.upstream`
* `/etc/ln-wallet-system-mainnet/eclair.turbo`
* `/etc/ln-wallet-system-mainnet/eclair.upstream+tor`
* `/etc/ln-wallet-system-mainnet/eclair.turbo+tor`
* `/etc/ln-wallet-system-mainnet/lnd`
* `/etc/ln-wallet-system-mainnet/lnd.tor`
* `/etc/ln-wallet-system-mainnet/_default -> eclair.turbo+tor`
