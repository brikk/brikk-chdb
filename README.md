# brikk-chdb

An in-process Kotlin/JVM binding for embedded [chDB](https://github.com/chdb-io/chdb),
the ClickHouse execution engine. This is **not JDBC**: applications call a small
Kotlin session/query API instead of `Connection`, `Statement`, or `ResultSet`.
No separate ClickHouse server is required.

Extracted from [brikk-house](https://github.com/brikk/brikk-house), preserving
relevant Git history and existing `dev.brikk.house` Maven coordinates. Independent
of the [SQL parser/transpiler](https://github.com/brikk/brikk-sql).

## Modules and artifacts

| Module/artifact | Responsibility |
| --- | --- |
| `brikk-chdb` | Session/query API, copied-byte results, errors, native loader, and Java FFM binding. |
| `brikk-chdb-native-linux-x64` | Pinned native library packaged as a Linux x64 resource JAR. |
| `brikk-chdb-native-linux-arm64` | Linux ARM64 native resource JAR. |
| `brikk-chdb-native-macos-arm64` | macOS ARM64 native resource JAR. |
| `brikk-chdb-native-packaging` | Build-only Toolchain plugin; downloads, checksum-verifies, and packages native resources. Not published. |

Consumers need `brikk-chdb` and **exactly one matching native artifact**, or an
explicit native-library path. The native binary is extracted into a private,
checksum-verified cache and loaded inside the JVM. No downloads occur at runtime.
Windows is not currently supported. Native code remains native, not Kotlin bytecode.

Latest release: **`0.16.1`**. Snapshot: **`1.0.0-SNAPSHOT`**. Native engine pin:
**chdb-core v26.9.0**; platform manifests retain upstream URLs and SHA-256 checksums.

```kotlin
repositories { mavenCentral() }
dependencies {
    implementation("dev.brikk.house:brikk-chdb:0.16.1")
    runtimeOnly("dev.brikk.house:brikk-chdb-native-linux-x64:0.16.1")
}
```

Run on **JDK 25** with `--enable-native-access=ALL-UNNAMED`:

```kotlin
import dev.brikk.house.chdb.Chdb
import dev.brikk.house.chdb.ChdbOutputFormat

Chdb.open().use { session ->
    val result = session.query("SELECT 6 * 7", ChdbOutputFormat.CSV)
    println(result.text())
}
```

Results contain raw output bytes and query statistics. JSONCompact, JSONEachRow,
CSV, TSV, Parquet, and custom ClickHouse output formats are supported.
See the [design/ABI documentation](docs/design/brikk-chdb.md).

## Build and test

Use **Kotlin Toolchain 0.13.0**, with Kotlin explicitly pinned to **2.4.10**:

```sh
printf 'brikk.mavencentral.user=x\nbrikk.mavencentral.pass=x\n' > .env
./kotlin build
./kotlin test
./kotlin publish mavenLocal
```

Building all modules downloads and packages all three pinned native archives.
CI runs API tests and the Linux x64 packaged-resource extraction/ABI test.
The explicit-library query test is opt-in via `brikk.chdb.integrationLibrary`.
Do not interpret native loading/ABI checks as complete cross-platform query coverage.

## Publishing

This repository publishes **only the API and three native resource artifacts**.
POM URLs/SCM point to this repository; group/artifact IDs remain unchanged.

- Push to `main`: build/test/Maven Local gate, then publish the current snapshot
  to `https://central.sonatype.com/repository/maven-snapshots/`.
- Push `release/<version>` or manually dispatch the release workflow: the same
  gate, then signed Maven Central releases.
- Local snapshot: `./kotlin publish centralSnapshots`.
- Local release: `./publish-release.sh <version>`.

Required GitHub organization/repository secrets:
`KOTLIN_TOOLCHAIN_MAVENCENTRAL_USERNAME`, `KOTLIN_TOOLCHAIN_MAVENCENTRAL_PASSWORD`,
`KOTLIN_TOOLCHAIN_SIGNING_KEY`, and `KOTLIN_TOOLCHAIN_SIGNING_PASSPHRASE`.
The workflows accept the `KOTLIN_TOOLCHAIN_MAVEN_CENTRAL_*` credential aliases too.
Local snapshots read `brikk.mavencentral.user/pass` from `.env`; local releases
read `KOTLIN_TOOLCHAIN_*` environment variables. The launcher restores the
snapshot template after success or failure and invokes the four explicit Portal
tasks. Never republish an existing release version. SQL/chDB releases are independent.

## License and provenance

Apache-2.0; see [LICENSE](LICENSE) and [ATTRIBUTIONS.md](ATTRIBUTIONS.md).
Native libraries are fetched only at build/publish time and are not committed to Git.
