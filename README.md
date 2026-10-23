# goose

Container images with [pressly/goose](https://github.com/pressly/goose), the database migration tool. `goose` applies, rolls back and reports versioned SQL migrations on PostgreSQL, MySQL, SQLite, SQL Server, ClickHouse and other databases, and records the applied versions in a table of the database. It is compiled from the release tag with a current Go release on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when goose publishes a release, when the pinned Go release is raised and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by the goose project or Pressly. Report problems with the image in this repository and problems with goose itself [upstream](https://github.com/pressly/goose/issues).

## Quick start

Create a migration in `migrations/`. With `-s` the files are numbered in sequence instead of named after the current time:

```sh
mkdir migrations
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/goose \
  -dir migrations -s create create_users sql
```

This writes `migrations/00001_create_users.sql`. Without the `sql` at the end, `create` writes a Go migration, which these images cannot run (see [SQL migrations only](#sql-migrations-only)). Replace the placeholder queries in the file:

```sql
-- +goose Up
CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT NOT NULL);

-- +goose Down
DROP TABLE users;
```

Apply it to a SQLite database in your directory, then list which migrations are applied:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/goose \
  -dir migrations sqlite3 ./app.db up
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/goose \
  -dir migrations sqlite3 ./app.db status
```

For a PostgreSQL server on the same machine, put the connection string in `GOOSE_DBSTRING` in your shell and pass it with `-e`, so that the password stays out of the command line:

```sh
export GOOSE_DBSTRING='postgres://app:<password>@localhost:5432/app?sslmode=disable'
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --network host -e GOOSE_DBSTRING \
  ghcr.io/randomcontainers/goose -dir migrations postgres up
```

Other commands: `down` rolls back the newest migration, `redo` rolls it back and applies it again, `up-to <version>` and `down-to <version>` stop at a version, `reset` rolls back everything, `version` prints the current version, `validate` checks the files without a database, and `fix` renames timestamped files to sequential numbers. goose prints its results to stderr, the output of `status` and `version` included.

Without arguments the image prints `goose --help`. In CI, pin a major version tag such as `3` or a digest. A minor version tag such as `3.29` stops getting updates once the next minor release is built (see [Tags](#tags)).

## Databases and connection strings

The first argument after the options names the database. The image has every driver of upstream's `goose` command:

| Argument | Database | Connection string |
|---|---|---|
| `postgres`, `redshift` | PostgreSQL, Amazon Redshift | `postgres://user:password@host:5432/db?sslmode=disable` or `host=... user=... dbname=...` |
| `mysql`, `tidb`, `starrocks` | MySQL and MariaDB, TiDB, StarRocks | `user:password@tcp(host:3306)/db` |
| `sqlite3` | SQLite | a file path such as `./app.db` |
| `mssql` | SQL Server | `sqlserver://user:password@host:1433?database=db` |
| `azuresql` | Azure SQL Database with Microsoft Entra ID | `sqlserver://server.database.windows.net?database=db&fedauth=ActiveDirectoryDefault` |
| `clickhouse` | ClickHouse | `tcp://host:9000` or `clickhouse://user:password@host:9440/db?secure=true` |
| `turso` | Turso and libSQL | `libsql://db.turso.io?authToken=<token>` |
| `ydb` | YDB | `grpcs://host:2135/local?go_query_mode=scripting&go_fake_tx=scripting&go_query_bind=declare,numeric` |
| `vertica` | Vertica, deprecated upstream | `vertica://user:password@host:5433/db` |

`goose --help` also lists `spanner`, but upstream's command has no Spanner driver, so it does not work in these images either. SQLite is modernc.org/sqlite, a translation of SQLite to Go, so the binary needs no C library. For MySQL over TLS with your own CA, mount the CA certificate and pass its path with `-certfile`; `-ssl-cert` and `-ssl-key` add a client certificate.

The container has its own network, so `localhost` in a connection string is the container itself. On Linux, `--network host` reaches a server on your machine. On Docker Desktop, use the host name `host.docker.internal` instead. For a database in another container, add `--network <network>` with that container's network and use its name as the host.

## Settings from the environment

goose reads these variables:

| Variable | Instead of |
|---|---|
| `GOOSE_DRIVER` | the database argument, such as `postgres` |
| `GOOSE_DBSTRING` | the connection string |
| `GOOSE_MIGRATION_DIR` | `-dir` |
| `GOOSE_TABLE` | `-table`, the name of the version table (default `goose_db_version`) |

With `GOOSE_DRIVER` and `GOOSE_DBSTRING` set, the command is only `goose up`. `goose env` prints the values it sees.

goose also reads a `.env` file in the working directory, which is your directory mounted at `/work`, and loads every variable in it into its environment. Variables already set in the environment win over the file. If Docker Compose or another tool keeps a `.env` there, goose picks up all of it: the `GOOSE_` variables, the values that `-- +goose ENVSUB ON` substitutes into migrations, and the settings drivers read from the environment, such as `PGPASSWORD` or `PGSSLMODE` for PostgreSQL. `-env=none` skips the file, and `-env=<path>` reads another one.

## SQL migrations only

The images run SQL migrations. goose also supports migrations written in Go, but those have to be compiled into a program of your own that registers them, so `goose` stops with an error when it reaches a `.go` migration. The SQL annotations all work: `-- +goose StatementBegin` and `StatementEnd` around statements that contain semicolons, such as functions and triggers, `-- +goose NO TRANSACTION` for statements that cannot run in a transaction, and `-- +goose ENVSUB ON`, which replaces `${VAR}` with the value of an environment variable.

## What is in the image

- `goose` in `/usr/local/bin`, built without cgo, with all the drivers above. It is statically linked and loads no libraries.
- Nothing else beyond the base image, `ca-certificates` and `tini`. There are no database clients such as `psql` or `mysql` and no Go toolchain.

The Go version, the commit and every Go module with its version and checksum are in `/usr/local/share/randomcontainers/goose/buildinfo`.

## Default or slim

goose's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is a goose release such as `3.29.0`. `<minor>` and `<major>` are its shorter forms, `3.29` and `3`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current goose version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and `/cache` can be written by anyone. goose writes nothing there. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

`goose create` writes new migration files and `goose fix` renames them. With `sqlite3`, the database file is written where the connection string points, usually in the working directory.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new goose release and are rebuilt when the base image changes. This one carries the migrations, for running them as a job next to your application:

```dockerfile
FROM ghcr.io/randomcontainers/goose:slim-alpine@sha256:...
COPY migrations/ /migrations/
ENV GOOSE_MIGRATION_DIR=/migrations
```

Run it with `GOOSE_DRIVER` and `GOOSE_DBSTRING` set and `up` as the command. The entrypoint is `["tini", "--", "goose"]`; set your own `ENTRYPOINT` if your image runs something else, such as a shell script that builds `GOOSE_DBSTRING` from several secrets. To install more packages, switch to root with `USER root`, install them with `apt-get` or `apk add --no-cache`, and switch back with `USER 1000:1000`. To pick up new goose releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/goose/` holds the version, the source links, the build details, the license files, the MPL-2.0 module source and `runtime-deps`, the list of distro packages `goose` needs at run time beyond those in the base image (none).

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/goose:latest \
  --repo randomcontainers/goose --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/goose:latest --format '{{ json .SBOM }}'
```

goose publishes prebuilt binaries but no source tarball, and its tags are not signed, so each version is pinned to the commit of its tag. The build clones the `v<version>` tag of [pressly/goose](https://github.com/pressly/goose) and stops unless the tag points to the commit recorded in `package.yml`. The Go toolchain is downloaded from Go's release server (dl.google.com, as linked from go.dev/dl) and checked against the SHA-256 in the Dockerfile, and the Go modules come from the Go module proxy and are checked against goose's `go.sum`. The binary records the commit it was built from, and `buildinfo` lists it with the Go modules.

## Updates

The project checks the releases of [pressly/goose](https://github.com/pressly/goose/releases) every 15 minutes. A release is picked up once it is 24 hours old; pre-releases are skipped. The new version and the commit of its tag are then committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The Go toolchain is pinned in the Dockerfiles and raised by hand when Go publishes a security release, and each raise rebuilds the images of the current version. goose raises the Go version its `go.mod` requires from time to time, and a release that needs a newer Go than the pin does not build until the pin is raised.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_COMMIT=<commit from package.yml> \
  -t goose:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of packages Go builds in parallel.

## Licenses

goose is released under the [MIT License](https://github.com/pressly/goose/blob/main/LICENSE). The binary also contains:

- the Go standard library (BSD-3-Clause);
- the Go modules goose depends on, under MIT, BSD-3-Clause, Apache-2.0, MPL-2.0, BSD-2-Clause, ISC and MIT-0, among them the MySQL driver go-sql-driver/mysql under MPL-2.0;
- SQLite, which modernc.org/sqlite translates to Go. Its authors have dedicated it to the public domain; the SPDX identifier for these terms is `blessing`, after the blessing in SQLite's source files.

The image's license label is `MIT AND BSD-3-Clause AND Apache-2.0 AND MPL-2.0 AND BSD-2-Clause AND ISC AND MIT-0 AND blessing`. The license files are in `/usr/local/share/randomcontainers/goose/licenses/`, with one directory per Go module. modernc.org/sqlite and modernc.org/libc list the third-party code they include in their `LICENSE-3RD-PARTY.md`.

MPL-2.0 requires whoever distributes a binary built from MPL-2.0 files to make those files' source available to its recipients. The module zip of go-sql-driver/mysql is in `/usr/local/share/randomcontainers/goose/mpl-sources/`, downloaded from the Go module proxy and checked against goose's `go.sum`. `/usr/local/share/randomcontainers/goose/source` lists the goose commit and this zip. The packages from Ubuntu or Alpine keep their own licenses. The SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
