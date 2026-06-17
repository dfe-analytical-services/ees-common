# EES Common

Shared .NET packages for use across [Explore Education Statistics (EES)](https://github.com/dfe-analytical-services/explore-education-statistics) projects, maintained by the Department for Education's analytical services team.

Packages in this repository target **.NET 8** and are published to [GitHub Packages](https://github.com/orgs/dfe-analytical-services/packages?repo_name=ees-common) whenever changes are merged into `main`.

## Packages

| Package | Description |
| ------- | ----------- |
| `GovUk.Education.ExploreEducationStatistics.Common.DuckDb` | Helpers and fixes for working with [DuckDB](https://duckdb.org/) in .NET, built on top of [DuckDB.NET](https://github.com/Giorgi/DuckDB.NET), with [Dapper](https://github.com/DapperLib/Dapper) and [MiniProfiler](https://miniprofiler.com/dotnet/) integration. |

## Installation

The packages are hosted on the `dfe-analytical-services` GitHub Packages NuGet feed. Add the feed as a package source, either directly in your local `NuGet.Config` file (`~/Users/<username>/AppData/Roaming/NuGet` on Windows):

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="dfe-analytical-services" value="https://nuget.pkg.github.com/dfe-analytical-services/index.json" />
  </packageSources>
</configuration>
```

or via the CLI:

```shell
dotnet nuget add source "https://nuget.pkg.github.com/dfe-analytical-services/index.json" \
  --name dfe-analytical-services \
  --username YOUR_GITHUB_USERNAME \
  --password YOUR_GITHUB_PAT \
  --store-password-in-clear-text
```

> [!NOTE]
> GitHub Packages requires authentication even for packages in public repositories. Use a personal access token (classic) with the `read:packages` scope - instructions to create a PAT can be found in the [GitHub documentation](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-personal-access-token-classic).

Then install the package:

```shell
dotnet add package GovUk.Education.ExploreEducationStatistics.Common.DuckDb
```

## GovUk.Education.ExploreEducationStatistics.Common.DuckDb

This package provides a thin layer over DuckDB.NET that patches functionality which doesn't work reliably and adds quality-of-life APIs for querying DuckDB from EES services.

### What it provides

- **`DuckDbConnection`** — a drop-in replacement for `DuckDBConnection` with factory methods for file-backed databases and a convenience method for executing non-query commands.
- **`ProfiledDuckDbConnection`** — a DuckDB connection wrapped with MiniProfiler, so queries appear in your application's profiling output.
- **`IDuckDbConnection`** — a common interface over both connection types (extending `IDbConnection`), making them easy to inject and swap in tests.
- **`DuckDbSqlBuilder` / `DuckDbDapperSqlBuilder`** — SQL builders based on [InterpolatedSql](https://github.com/Drizin/InterpolatedSql) that let you write parameterised SQL using ordinary C# string interpolation, with first-class Dapper execution support.
- **`DuckDbCommand` / `DuckDbParameter`** — patched command and parameter types that force the use of auto-incrementing *positional* parameters (`?`) instead of named parameters (`$param`), which [don't work correctly in all environments](https://github.com/Giorgi/DuckDB.NET/issues/178).

### Creating connections

```csharp
using GovUk.Education.ExploreEducationStatistics.Common.DuckDb;

// In-memory database (the default)
await using var connection = new DuckDbConnection();

// File-backed database
await using var fileConnection = DuckDbConnection.CreateFileConnection("data.duckdb");

// File-backed database in read-only mode
await using var readOnlyConnection = DuckDbConnection.CreateFileConnectionReadOnly("data.duckdb");

connection.Open();

// Convenience method for DDL / non-query statements
await connection.ExecuteNonQueryAsync("CREATE TABLE people (name VARCHAR, age INTEGER)");
```

### Querying with the SQL builder and Dapper

The `SqlBuilder` extension methods on `IDuckDbConnection` let you compose SQL with interpolated values that are automatically converted into query parameters — interpolated values are **never** concatenated directly into the SQL string:

```csharp
using GovUk.Education.ExploreEducationStatistics.Common.DuckDb;
using InterpolatedSql.Dapper;

var minAge = 18;

var adults = await connection
    .SqlBuilder($"SELECT * FROM people WHERE age >= {minAge}")
    .QueryAsync<Person>();
```

Builders can be composed incrementally before execution:

```csharp
var builder = connection.SqlBuilder($"SELECT * FROM people WHERE age >= {minAge}");

if (nameFilter is not null)
{
    builder += $" AND name LIKE {nameFilter}";
}

var results = await builder.QueryAsync<Person>();
```

Enumerable values are automatically expanded into a comma-separated list of parameter placeholders, which is convenient for `IN` clauses:

```csharp
string[] names = ["Alice", "Bob", "Carol"];

// Generates: SELECT * FROM people WHERE name IN (?, ?, ?)
var people = await connection
    .SqlBuilder($"SELECT * FROM people WHERE name IN ({names})")
    .QueryAsync<Person>();
```

### Bulk loading with appenders

`IDuckDbConnection` exposes DuckDB's native [appender](https://duckdb.org/docs/data/appender) API for efficient bulk inserts:

```csharp
using var appender = connection.CreateAppender("people");

foreach (var person in people)
{
    appender.CreateRow()
        .AppendValue(person.Name)
        .AppendValue(person.Age)
        .EndRow();
}
```

### Profiling queries with MiniProfiler

Use `ProfiledDuckDbConnection` in place of `DuckDbConnection` to have all DuckDB commands recorded by MiniProfiler (defaulting to `MiniProfiler.Current` if no profiler is supplied):

```csharp
await using var connection = new ProfiledDuckDbConnection("DataSource=data.duckdb");
```

Because both connection types implement `IDuckDbConnection`, you can register the profiled connection in dependency injection for local development and the plain connection elsewhere without changing consuming code.

### A note on parameters

Named parameters (e.g. `$name`) are deliberately disabled by this package because they behave inconsistently across environments (see [DuckDB.NET#178](https://github.com/Giorgi/DuckDB.NET/issues/178)). All parameters are bound positionally using `?` placeholders. This is handled for you when using the `SqlBuilder` APIs, but bear it in mind if you construct `DuckDbCommand` objects manually or use Dapper directly against the connection — supply parameters in the order they appear in the SQL.

## Development

### Requirements

- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)

### Building

Packages are only generated automatically on build if `GeneratePackageOnBuild` is enabled in the project configuration. This is currently disabled as packaging is managed by the CI workflow, so a local `Release` build will not currently produce the `.nupkg` files. If `GeneratePackageOnBuild` is enabled, the following commands can be used to create a package manually:

```shell
dotnet build
```

```shell
dotnet pack --configuration Release
```

### Releasing

Packages are versioned via the `<Version>` property in each project's `.csproj`. To release a new version, bump the version in the relevant project, then merge to `main` — CI packs the projects with the `Release` configuration and pushes the resulting packages to GitHub Packages, with release notes generated from the commit messages included in the merge.

## Contributing

Contributions are welcome — please see the [contributing guidelines](https://github.com/dfe-analytical-services/ees-common/blob/main/CONTRIBUTING.md) and [code of conduct](https://github.com/dfe-analytical-services/ees-common/blob/main/CODE_OF_CONDUCT.md). If you discover a security issue, please follow the [security policy](https://github.com/dfe-analytical-services/ees-common/security/policy) rather than opening a public issue.

## License

This project is licensed under the [MIT License](LICENSE.txt).

Crown copyright © Department for Education.
