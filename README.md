# exccdata

[![Build Status](https://img.shields.io/travis/EXCCoin/exccdata.svg)](https://travis-ci.org/EXCCoin/exccdata)
[![Latest tag](https://img.shields.io/github/tag/EXCCoin/exccdata.svg)](https://github.com/EXCCoin/exccdata/tags)
[![Go Report Card](https://goreportcard.com/badge/github.com/EXCCoin/exccdata)](https://goreportcard.com/report/github.com/EXCCoin/exccdata)
[![ISC License](https://img.shields.io/badge/license-ISC-blue.svg)](http://copyfree.org)

The exccdata repository is a collection of golang packages and apps for [Decred](https://www.excc.org/) data collection, storage, and presentation.

## Repository overview

```none
../exccdata              The exccdata daemon.
├── api                 Package blockdata implements exccdata's own HTTP API.
│   ├── insight         Package insight implements the Insight API.
│   └── types           Package types includes the exported structures used by
|                         the exccdata and Insight APIs.
├── blockdata           Package blockdata is the primary data collection and
|                         storage hub, and chain monitor.
├── cmd
│   ├── rebuilddb       rebuilddb utility, for SQLite backend. Not required.
│   ├── rebuilddb2      rebuilddb2 utility, for PostgreSQL backend. Not required.
│   └── scanblocks      scanblocks utility. Not required.
├── exccdataapi          Package exccdataapi for golang API clients.
├── db
│   ├── agendadb        Package agendadb is a basic PoS voting agenda database.
│   ├── dbtypes         Package dbtypes with common data types.
│   ├── exccpg           Package exccpg providing PostgreSQL backend.
│   └── exccsqlite       Package exccsqlite providing SQLite backend.
├── dev                 Shell scripts for maintenance and deployment.
├── explorer            Package explorer, powering the block explorer.
├── mempool             Package mempool for monitoring mempool for transactions,
|                         data collection, and storage.
├── middleware          Package middleware provides HTTP router middleware.
├── notification        Package notification manages exccd notifications, and
|                         synchronous data collection by a queue of collectors.
├── public              Public resources for block explorer (css, js, etc.).
├── rpcutils            Package rpcutils contains helper types and functions for
|                         interacting with a chain server via RPC.
├── semver              Package semver.
├── stakedb             Package stakedb, for tracking tickets.
├── testutil            Package testutil provides some testing helper functions.
├── txhelpers           Package txhelpers provides many functions and types for
|                         processing blocks, transactions, voting, etc.
├── version             Package version describes the exccdata version.
└── views               HTML templates for block explorer.
```

## Requirements

* [Go](http://golang.org) 1.9.x or 1.10.x.
* Running `exccd` (>=1.3.0) synchronized to the current best block on the
  network. This is a strict requirement as testnet2 support is removed from
  exccdata v3.0.0.
* (Optional) PostgreSQL 9.6+, if running in "full" mode. v10.x is recommended
  for improved dump/restore formats and utilities.

## Installation

### Build from Source

The following instructions assume a Unix-like shell (e.g. bash).

* [Install Go](http://golang.org/doc/install)

* Verify Go installation:

      go env GOROOT GOPATH

* Ensure `$GOPATH/bin` is on your `$PATH`.
* Install `dep`, the dependency management tool. The current [released binary of
  `dep`](https://github.com/golang/dep/releases) is recommended, but the latest
  may be installed from git via:

      go get -u -v github.com/golang/dep/cmd/dep

* Clone the exccdata repository. It **must** be cloned into the following directory.

      git clone https://github.com/EXCCoin/exccdata $GOPATH/src/github.com/EXCCoin/exccdata

* Fetch dependencies, and build the `exccdata` executable.

      cd $GOPATH/src/github.com/EXCCoin/exccdata
      dep ensure -vendor-only
      # build exccdata executable in workspace:
      go build

To build with the git commit hash appended to the version, set it as follows:

```bash
go build -ldflags "-X github.com/EXCCoin/exccdata/version.CommitHash=`git describe --abbrev=8 --long | awk -F "-" '{print $(NF-1)"-"$NF}'`"
```

The sqlite driver uses cgo, which requires a C compiler (e.g. gcc) to compile
the C sources. On Windows this is easily handled with MSYS2
([download](http://www.msys2.org/) and install MinGW-w64 gcc packages).

Tip: If you receive other build errors, it may be due to "vendor" directories
left by dep builds of dependencies such as exccwallet. You may safely delete
vendor folders and run `dep ensure -vendor-only` again.

### Runtime resources

The config file, logs, and data files are stored in the application data folder,
which may be specified via the `-A/--appdata` and `-b/--datadir` settings.
However, the location of the config file may be set with `-C/--configfile`. If
encountering errors involving file system paths, check the permissions on these
folders to ensure that *the user running exccdata* is able to access these paths.

The "public" and "views" folders *must* be in the same folder as the `exccdata`
executable. Set read-only permissions as appropriate.

## Updating

First, update the repository (assuming you have `master` checked out):

    cd $GOPATH/src/github.com/EXCCoin/exccdata
    git pull origin master
    dep ensure -vendor-only
    go build

Look carefully for errors with `git pull`, and reset locally modified files if
necessary.

## Updating dependencies

`dep ensure -vendor-only` fetches project dependencies, without making changes
in manifest files `Gopkg.toml` and `Gopkg.lock`.

Call `dep ensure` to update project dependencies when introduce new imports.

For guides and reference materials about `dep`, see [the documentation](https://golang.github.io/dep).

The following FAQ explains [the difference between `Gopkg.toml` and `Gopkg.lock` files](https://github.com/golang/dep/blob/master/docs/FAQ.md#what-is-the-difference-between-gopkgtoml-the-manifest-and-gopkglock-the-lock).

## Upgrading Instructions 

*__Only necessary while upgrading from v2.x or below.__*  The database scheme
change from exccdata v2.x to v3.x does not permit an automatic migration. The
tables must be rebuilt from scratch:

1. Drop the old exccdata database, and create a new empty exccdata database.

```sql
 -- drop the old database
 DROP DATABASE exccdata;

-- create a new database with the same `pguser` set in the exccdata.conf
CREATE DATABASE exccdata OWNER exccdata;

-- grant all permissions to user exccdata
GRANT ALL PRIVILEGES ON DATABASE exccdata to exccdata;
 ```

2. Delete the exccdata data folder (i.e. corresponding to the `datadir` setting).
   By default, `datadir` is in `{appdata}/data`:

    * Linux:  `~/.exccdata/data`
    * Mac:  `~/Library/Application Support/Exccdata/data`
    * Windows:  `C:\Users\<your-username>\AppData\Local\Exccdata\data` (`%localappdata%\Exccdata\data`)
  
3. With exccd synchronized to the network's best block, start exccdata to begin
   the initial block data import.

## Getting Started

### Configuring PostgreSQL (IMPORTANT)

If you intend to run exccdata in "full" mode (i.e. with the `--pg` switch), which
uses a PostgreSQL database backend, it is crucial that you configure your
PostgreSQL server for your hardware and the exccdata workload.

Read [postgresql-tuning.conf](./db/exccpg/postgresql-tuning.conf) carefully for
details on how to make the necessary changes to your system. A helpful online
tool for determining good settings for your system is called
[PGTune](https://pgtune.leopard.in.ua/). **DO NOT** simply use this file in place
of your existing postgresql.conf or copy and paste these settings into the
existing postgresql.conf. It is necessary to edit postgresql.conf, reviewing all
the settings to ensure the same configuration parameters are not set in two
different places in the file.

On Linux, you may wish to use a unix domain socket instead of a TCP connection.
The path to the socket depends on the system, but it is commonly
/var/run/postgresql. Just set this path in `pghost`.

### Creating the Configuration File

Begin with the sample configuration file. With the default `appdata` directory
for the current user on Linux:

```bash
cp sample-exccdata.conf ~/.exccdata/exccdata.conf
```

Then edit exccdata.conf with your exccd RPC settings. See the output of `exccdata --help`
for a list of all options and their default values.

### Indexing the Blockchain

If exccdata has not previously been run with the PostgreSQL database backend, it
is necessary to perform a bulk import of blockchain data and generate table
indexes. *This will be done automatically by `exccdata`* on a fresh startup.

Alternatively (but not recommended), the PostgreSQL tables may also be generated
with the `rebuilddb2` command line tool:

* Create the exccdata user and database in PostgreSQL (tables will be created automatically).
* Set your PostgreSQL credentials and host in both `./cmd/rebuilddb2/rebuilddb2.conf`,
  and `exccdata.conf` in the location specified by the `appdata` flag.
* Run `./rebuilddb2` to bulk import data and index the tables.
* In case of irrecoverable errors, such as detected schema changes without an
  upgrade path, the tables and their indexes may be dropped with `rebuilddb2 -D`.

Note that exccdata requires that
[exccd](https://docs.excc.org/getting-started/user-guides/exccd-setup/) is
running with some optional indexes enabled.  By default, these indexes are not
turned on when exccd is installed. To enable them, set the following in
exccd.conf:

```ini
txindex=1
addrindex=1
```

If these parameters are not set, exccdata will be unable to retrieve transaction
details and perform address searches, and will exit with an error mentioning
these indexes.

### Starting exccdata

Launch the exccdata daemon and allow the databases to process new blocks. In
"lite" mode (without `--pg`), only a SQLite DB is populated, which usually
requires 30-60 minutes. In "full" mode (with `--pg`), concurrent synchronization
of both SQLite and PostgreSQL databases is performed, requiring from 3-12 hours.
See [System Hardware Requirements](#System-Hardware-Requirements) for more
information.

On subsequent launches, only blocks new to exccdata are processed.

```bash
./exccdata    # don't forget to configure exccdata.conf in the appdata folder!
```

Unlike exccdata.conf, which must be placed in the `appdata` folder or explicitly
set with `-C`, the "public" and "views" folders *must* be in the same folder as
the `exccdata` executable.

## System Hardware Requirements

The time required to sync in "full" mode varies greatly with system hardware and
software configuration. The most important factor is the storage medium on the
database machine. An SSD (preferably NVMe, not SATA) is strongly recommended if
you value your time and system performance.

### "lite" Mode (SQLite only)

Minimum:

* 1 CPU core
* 2 GB RAM
* HDD with 4GB free space

### "full" Mode (SQLite and PostgreSQL)

These specifications assume exccdata and postgres are running on the same machine.

Minimum:

* 1 CPU core
* 4 GB RAM
* HDD with 60GB free space

Recommend:

* 2+ CPU cores
* 7+ GB RAM
* SSD (NVMe preferred) with 60 GB free space

If PostgreSQL is running on a separate machine, the minimum "lite" mode
requirements may be applied to the exccdata machine, while the recommended
"full" mode requirements should be applied to the PostgreSQL host.

## exccdata Daemon

The root of the repository is the `main` package for the `exccdata` app, which
has several components including:

1. Block explorer (web interface).
2. Blockchain monitoring and data collection.
3. Mempool monitoring and reporting.
4. Database backend interfaces.
5. RESTful JSON API (custom and Insight) over HTTP(S).

### Block Explorer

After exccdata syncs with the blockchain server via RPC, by default it will begin
listening for HTTP connections on `http://127.0.0.1:7777/`. This means it starts
a web server listening on IPv4 localhost, port 7777. Both the interface and port
are configurable. The block explorer and the JSON APIs are both provided by the
server on this port. See [JSON REST API](#json-rest-api) for details.

Note that while exccdata can be started with HTTPS support, it is recommended to
employ a reverse proxy such as Nginx ("engine x"). See sample-nginx.conf for an
example Nginx configuration.

To save time and tens of gigabytes of disk storage space, exccdata runs by
default in a reduced functionality ("lite") mode that does not require
PostgreSQL. To enable the PostgreSQL backend (and the expanded functionality),
exccdata may be started with the `--pg` switch.

### Insight API (EXPERIMENTAL)

The [Insight API](https://github.com/bitpay/insight-api) is accessible via HTTP
at the `/insight/api` URL path prefix, and via WebSocket at the
`/insight/socket.io` URL path prefix.

The following endpoints are not implemented: `status`, `sync`, `peer`, `email`,
and `rates`.

### exccdata API

exccdata has its own JSON HTTP API in addition to the experimental Insight API
implementation. **exccdata's API endpoints are prefixed with `/api`** (e.g.
`http://localhost:7777/api/stake`).  The Insight API endpoints (not described in
this section) are prefixed with `/insight/api`.

#### Endpoint List

| Best block | Path | Type |
| --- | --- | --- |
| Summary | `/block/best` | `types.BlockDataBasic` |
| Stake info |  `/block/best/pos` | `types.StakeInfoExtended` |
| Header |  `/block/best/header` | `exccjson.GetBlockHeaderVerboseResult` |
| Hash |  `/block/best/hash` | `string` |
| Height | `/block/best/height` | `int` |
| Size | `/block/best/size` | `int32` |
| Subsidy | `/block/best/subsidy` | `types.BlockSubsidies` |
| Transactions | `/block/best/tx` | `types.BlockTransactions` |
| Transactions Count | `/block/best/tx/count` | `types.BlockTransactionCounts` |
| Verbose block result | `/block/best/verbose` | `exccjson.GetBlockVerboseResult` |

| Block X (block index) | Path | Type |
| --- | --- | --- |
| Summary | `/block/X` | `types.BlockDataBasic` |
| Stake info |  `/block/X/pos` | `types.StakeInfoExtended` |
| Header |  `/block/X/header` | `exccjson.GetBlockHeaderVerboseResult` |
| Hash |  `/block/X/hash` | `string` |
| Size | `/block/X/size` | `int32` |
| Subsidy | `/block/best/subsidy` | `types.BlockSubsidies` |
| Transactions | `/block/X/tx` | `types.BlockTransactions` |
| Transactions Count | `/block/X/tx/count` | `types.BlockTransactionCounts` |
| Verbose block result | `/block/X/verbose` | `exccjson.GetBlockVerboseResult` |

| Block H (block hash) | Path | Type |
| --- | --- | --- |
| Summary | `/block/hash/H` | `types.BlockDataBasic` |
| Stake info |  `/block/hash/H/pos` | `types.StakeInfoExtended` |
| Header |  `/block/hash/H/header` | `exccjson.GetBlockHeaderVerboseResult` |
| Height |  `/block/hash/H/height` | `int` |
| Size | `/block/hash/H/size` | `int32` |
| Subsidy | `/block/best/subsidy` | `types.BlockSubsidies` |
| Transactions | `/block/hash/H/tx` | `types.BlockTransactions` |
| Transactions count | `/block/hash/H/tx/count` | `types.BlockTransactionCounts` |
| Verbose block result | `/block/hash/H/verbose` | `exccjson.GetBlockVerboseResult` |

| Block range (X < Y) | Path | Type |
| --- | --- | --- |
| Summary array for blocks on `[X,Y]` | `/block/range/X/Y` | `[]types.BlockDataBasic` |
| Summary array with block index step `S` | `/block/range/X/Y/S` | `[]types.BlockDataBasic` |
| Size (bytes) array | `/block/range/X/Y/size` | `[]int32` |
| Size array with step `S` | `/block/range/X/Y/S/size` | `[]int32` |

| Transaction T (transaction id) | Path | Type |
| --- | --- | --- |
| Transaction details | `/tx/T` | `types.Tx` |
| Transaction details w/o block info | `/tx/trimmed/T` | `types.TrimmedTx` |
| Inputs | `/tx/T/in` | `[]types.TxIn` |
| Details for input at index `X` | `/tx/T/in/X` | `types.TxIn` |
| Outputs | `/tx/T/out` | `[]types.TxOut` |
| Details for output at index `X` | `/tx/T/out/X` | `types.TxOut` |
| Vote info (ssgen transactions only) | `/tx/T/vinfo` | `types.VoteInfo` |
| Serialized bytes of the transaction | `/tx/hex/T` | `string` |
| Same as `/tx/trimmed/T` | `/tx/decoded/T` | `types.TrimmedTx` |

| Transactions (batch) | Path | Type |
| --- | --- | --- |
| Transaction details (POST body is JSON of `types.Txns`) | `/txs` | `[]types.Tx` |
| Transaction details w/o block info | `/txs/trimmed` | `[]types.TrimmedTx` |

| Address A | Path | Type |
| --- | --- | --- |
| Summary of last 10 transactions | `/address/A` | `types.Address` |
| Number and value of spent and unspent outputs | `/address/A/totals` | `types.AddressTotals` |
| Verbose transaction result for last <br> 10 transactions | `/address/A/raw` | `types.AddressTxRaw` |
| Summary of last `N` transactions | `/address/A/count/N` | `types.Address` |
| Verbose transaction result for last <br> `N` transactions | `/address/A/count/N/raw` | `types.AddressTxRaw` |
| Summary of last `N` transactions, skipping `M` | `/address/A/count/N/skip/M` | `types.Address` |
| Verbose transaction result for last <br> `N` transactions, skipping `M` | `/address/A/count/N/skip/Mraw` | `types.AddressTxRaw` |

| Stake Difficulty (Ticket Price) | Path | Type |
| --- | --- | --- |
| Current sdiff and estimates | `/stake/diff` | `types.StakeDiff` |
| Sdiff for block `X` | `/stake/diff/b/X` | `[]float64` |
| Sdiff for block range `[X,Y] (X <= Y)` | `/stake/diff/r/X/Y` | `[]float64` |
| Current sdiff separately | `/stake/diff/current` | `exccjson.GetStakeDifficultyResult` |
| Estimates separately | `/stake/diff/estimates` | `exccjson.EstimateStakeDiffResult` |

| Ticket Pool | Path | Type |
| --- | --- | --- |
| Current pool info (size, total value, and average price) | `/stake/pool` | `types.TicketPoolInfo` |
| Current ticket pool, in a JSON object with a `"tickets"` key holding an array of ticket hashes | `/stake/pool/full` | `[]string` |
| Pool info for block `X` | `/stake/pool/b/X` | `types.TicketPoolInfo` |
| Full ticket pool at block height _or_ hash `H` | `/stake/pool/b/H/full` | `[]string` |
| Pool info for block range `[X,Y] (X <= Y)` | `/stake/pool/r/X/Y?arrays=[true\|false]`<sup>*</sup> | `[]apitypes.TicketPoolInfo` |

The full ticket pool endpoints accept the URL query `?sort=[true\|false]` for
requesting the tickets array in lexicographical order.  If a sorted list or list
with deterministic order is _not_ required, using `sort=false` will reduce
server load and latency. However, be aware that the ticket order will be random,
and will change each time the tickets are requested.

<sup>*</sup>For the pool info block range endpoint that accepts the `arrays` url query,
a value of `true` will put all pool values and pool sizes into separate arrays,
rather than having a single array of pool info JSON objects.  This may make
parsing more efficient for the client.

| Vote and Agenda Info | Path | Type |
| --- | --- | --- |
| The current agenda and its status | `/stake/vote/info` | `exccjson.GetVoteInfoResult` |

| Mempool | Path | Type |
| --- | --- | --- |
| Ticket fee rate summary | `/mempool/sstx` | `apitypes.MempoolTicketFeeInfo` |
| Ticket fee rate list (all) | `/mempool/sstx/fees` | `apitypes.MempoolTicketFees` |
| Ticket fee rate list (N highest) | `/mempool/sstx/fees/N` | `apitypes.MempoolTicketFees` |
| Detailed ticket list (fee, hash, size, age, etc.) | `/mempool/sstx/details` | `apitypes.MempoolTicketDetails` |
| Detailed ticket list (N highest fee rates) | `/mempool/sstx/details/N`| `apitypes.MempoolTicketDetails` |

| Other | Path | Type |
| --- | --- | --- |
| Status | `/status` | `types.Status` |
| Coin Supply | `/supply` | `types.CoinSupply` |
| Endpoint list (always indented) | `/list` | `[]string` |

All JSON endpoints accept the URL query `indent=[true|false]`.  For example,
`/stake/diff?indent=true`. By default, indentation is off. The characters to use
for indentation may be specified with the `indentjson` string configuration
option.

## Important Note About Mempool

Although there is mempool data collection and serving, it is **very important**
to keep in mind that the mempool in your node (exccd) is not likely to be exactly
the same as other nodes' mempool.  Also, your mempool is cleared out when you
shutdown exccd.  So, if you have recently (e.g. after the start of the current
ticket price window) started exccd, your mempool _will_ be missing transactions
that other nodes have.

## Command Line Utilities

### rebuilddb

`rebuilddb` is a CLI app that performs a full blockchain scan that fills past
block data into a SQLite database. This functionality is included in the startup
of the exccdata daemon, but may be called alone with rebuilddb.

### rebuilddb2

`rebuilddb2` is a CLI app used for maintenance of exccdata's `exccpg` database
(a.k.a. DB v2) that uses PostgreSQL to store a nearly complete record of the
Decred blockchain data. This functionality is included in the startup of the
exccdata daemon, but may be called alone with rebuilddb. See the
[README.md](./cmd/rebuilddb2/README.md) for `rebuilddb2` for important usage
information.

### scanblocks

scanblocks is a CLI app to scan the blockchain and save data into a JSON file.
More details are in [its own README](./cmd/scanblocks/README.md). The repository
also includes a shell script, jsonarray2csv.sh, to convert the result into a
comma-separated value (CSV) file.

## Helper Packages

`package exccdataapi` defines the data types, with json tags, used by the JSON
API.  This facilitates authoring of robust golang clients of the API.

`package dbtypes` defines the data types used by the DB backends to model the
block, transaction, and related blockchain data structures. Functions for
converting from standard Decred data types (e.g. `wire.MsgBlock`) are also
provided.

`package rpcutils` includes helper functions for interacting with a
`rpcclient.Client`.

`package stakedb` defines the `StakeDatabase` and `ChainMonitor` types for
efficiently tracking live tickets, with the primary purpose of computing ticket
pool value quickly.  It uses the `database.DB` type from
`github.com/EXCCoin/exccd/database` with an ffldb storage backend from
`github.com/EXCCoin/exccd/database/ffldb`.  It also makes use of the `stake.Node`
type from `github.com/EXCCoin/exccd/blockchain/stake`.  The `ChainMonitor` type
handles connecting new blocks and chain reorganization in response to notifications
from exccd.

`package txhelpers` includes helper functions for working with the common types
`exccutil.Tx`, `exccutil.Block`, `chainhash.Hash`, and others.

## Internal-use Packages

Packages `blockdata` and `exccsqlite` are currently designed only for internal
use internal use by other exccdata packages, but they may be of general value in
the future.

`blockdata` defines:

* The `chainMonitor` type and its `BlockConnectedHandler()` method that handles
  block-connected notifications and triggers data collection and storage.
* The `BlockData` type and methods for converting to API types.
* The `blockDataCollector` type and its `Collect()` and `CollectHash()` methods
  that are called by the chain monitor when a new block is detected.
* The `BlockDataSaver` interface required by `chainMonitor` for storage of
  collected data.

`exccpg` defines:

* The `ChainDB` type, which is the primary exported type from `exccpg`, providing
  an interface for a PostgreSQL database.
* A large set of lower-level functions to perform a range of queries given a
  `*sql.DB` instance and various parameters.
* The internal package contains the raw SQL statements.

`exccsqlite` defines:

* A `sql.DB` wrapper type (`DB`) with the necessary SQLite queries for
  storage and retrieval of block and stake data.
* The `wiredDB` type, intended to satisfy the `DataSourceLite` interface used by
  the exccdata app's API. The block header is not stored in the DB, so a RPC
  client is used by `wiredDB` to get it on demand. `wiredDB` also includes
  methods to resync the database file.

`package mempool` defines a `mempoolMonitor` type that can monitor a node's
mempool using the `OnTxAccepted` notification handler to send newly received
transaction hashes via a designated channel. Ticket purchases (SSTx) are
triggers for mempool data collection, which is handled by the
`mempoolDataCollector` class, and data storage, which is handled by any number
of objects implementing the `MempoolDataSaver` interface.

## Plans

See the GitHub issue tracker and the [project milestones](https://github.com/EXCCoin/exccdata/milestones).

## Contributing

Yes, please! See the CONTRIBUTING.md file for details, but here's the gist of it:

1. Fork the repo.
2. Create a branch for your work (`git branch -b cool-stuff`).
3. Code something great.
4. Commit and push to your repo.
5. Create a [pull request](https://github.com/EXCCoin/exccdata/compare).

**DO NOT merge from master to your feature branch; rebase.**

Before committing any changes to the Gopkg.lock file, you must update `dep` to
the latest version via:

    go get -u github.com/golang/dep/cmd/dep

**To update `dep` from the network, it is important to use the `-u` flag as
shown above.**

Note that all exccdata.org community and team members are expected to adhere to
the code of conduct, described in the CODE_OF_CONDUCT file.

Also, [come chat with us on Slack](https://slack.excc.org/)!

## License

This project is licensed under the ISC License. See the [LICENSE](LICENSE) file for details.
