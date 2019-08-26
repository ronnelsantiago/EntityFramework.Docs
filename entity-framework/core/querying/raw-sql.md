---
title: Raw SQL Queries - EF Core
author: rowanmiller
ms.date: 10/27/2016
ms.assetid: 70aae9b5-8743-4557-9c5d-239f688bf418
uid: core/querying/raw-sql
---
# Raw SQL Queries

Entity Framework Core allows you to drop down to raw SQL queries when working with a relational database. This can be useful if the query you want to perform can't be expressed using LINQ, or if using a LINQ query is resulting in inefficient SQL queries. Raw SQL queries can return entity types or, starting with EF Core 2.1, [query types](xref:core/modeling/query-types) that are part of your model.

> [!TIP]  
> You can view this article's [sample](https://github.com/aspnet/EntityFramework.Docs/tree/master/samples/core/Querying/Querying/RawSQL/Sample.cs) on GitHub.

## Basic raw SQL queries

You can use the `FromSqlRaw` extension method to begin a LINQ query based on a raw SQL query.

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var blogs = context.Blogs
    .FromSqlRaw("SELECT * FROM dbo.Blogs")
    .ToList();
```

Raw SQL queries can be used to execute a stored procedure.

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var blogs = context.Blogs
    .FromSqlRaw("EXECUTE dbo.GetMostPopularBlogs")
    .ToList();
```

## Passing parameters

As with any API that accepts SQL, it is important to parameterize any user input to protect against a SQL injection attack. `FromSqlRaw` performs simple string interpolation, so any parameters will be included as-is and must be manually sanitized; not doing so may expose your application to major security risks. To safely pass parameters, `FromSqlInterpolated` allows you to include placeholders in the SQL query string and then supply parameter values as additional arguments. Any parameter values you supply will automatically be converted to a `DbParameter` for safe execution. Using parameters with `FromSqlRaw` can still useful for dynamically building your SQL query, but should be used with care and never when simple parameterization is needed.

> [!WARNING]
> Prior to version 3.0, `FromSqlRaw` and `FromSqlInterpolated` were two overloads named `FromSql`. These two overloads have been obsoleted for safety reasons.

The following example passes a single parameter to a stored procedure. While this may look like `String.Format` syntax, the supplied value is wrapped in a parameter and the generated parameter name inserted where the `{0}` placeholder was specified.

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var user = "johndoe";

var blogs = context.Blogs
    .FromSqlInterpolated("EXECUTE dbo.GetMostPopularBlogsForUser {0}", user)
    .ToList();
```

This is the same query but using named interpolation syntax:

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var user = "johndoe";

var blogs = context.Blogs
    .FromSqlInterpolated($"EXECUTE dbo.GetMostPopularBlogsForUser {user}")
    .ToList();
```

You can also construct a DbParameter and supply it as a parameter value. Since a regular SQL parameter placeholder is used, rather than a string placeholder, `FromSqlRaw` can be safely used:

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var user = new SqlParameter("user", "johndoe");

var blogs = context.Blogs
    .FromSqlRaw("EXECUTE dbo.GetMostPopularBlogsForUser @user", user)
    .ToList();
```

This allows you to use named parameters in the SQL query string, which is useful when a stored procedure has optional parameters:

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var user = new SqlParameter("user", "johndoe");

var blogs = context.Blogs
    .FromSqlRaw("EXECUTE dbo.GetMostPopularBlogs @filterByUser=@user", user)
    .ToList();
```

## Composing with LINQ

If the SQL query can be composed on in the database, then you can compose on top of the initial raw SQL query using LINQ operators. SQL queries that can be composed on begin with the `SELECT` keyword.

The following example uses a raw SQL query that selects from a Table-Valued Function (TVF) and then composes on it using LINQ to perform filtering and sorting.

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var searchTerm = ".NET";

var blogs = context.Blogs
    .FromSqlInterpolated($"SELECT * FROM dbo.SearchBlogs({searchTerm})")
    .Where(b => b.Rating > 3)
    .OrderByDescending(b => b.Rating)
    .ToList();
```

## Change Tracking

Queries that use the `FromSql` methods follow the exact same change tracking rules as any other LINQ query in EF Core. For example, if the query projects entity types, the results will be tracked by default.

The following example uses a raw SQL query that selects from a Table-Valued Function (TVF), then disables change tracking with the call to `AsNoTracking`:

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var searchTerm = ".NET";

var blogs = context.Query<SearchBlogsDto>()
    .FromSqlInterpolated($"SELECT * FROM dbo.SearchBlogs({searchTerm})")
    .AsNoTracking()
    .ToList();
```

## Including related data

The `Include` method can be used to include related data, just like with any other LINQ query:

<!-- [!code-csharp[Main](samples/core/Querying/Querying/RawSQL/Sample.cs)] -->
``` csharp
var searchTerm = ".NET";

var blogs = context.Blogs
    .FromSqlInterpolated($"SELECT * FROM dbo.SearchBlogs({searchTerm})")
    .Include(b => b.Posts)
    .ToList();
```

## Limitations

There are a few limitations to be aware of when using raw SQL queries:

* The SQL query must return data for all properties of the entity or query type.

* The column names in the result set must match the column names that properties are mapped to. Note this is different from EF6 where property/column mapping was ignored for raw SQL queries and result set column names had to match the property names.

* The SQL query cannot contain related data. However, in many cases you can compose on top of the query using the `Include` operator to return related data (see [Including related data](#including-related-data)).

* `SELECT` statements passed to this method should generally be composable: If EF Core needs to evaluate additional query operators on the server (for example, to translate LINQ operators applied after `FromSql` methods), the supplied SQL will be treated as a subquery. This means that the SQL passed should not contain any characters or options that are not valid on a subquery, such as:
  * a trailing semicolon
  * On SQL Server, a trailing query-level hint (for example, `OPTION (HASH JOIN)`)
  * On SQL Server, an `ORDER BY` clause that is not accompanied of `OFFSET 0` OR `TOP 100 PERCENT` in the `SELECT` clause

* SQL statements other than `SELECT` are recognized automatically as non-composable. As a consequence, the full results of stored procedures are always returned to the client and any LINQ operators applied after `FromSql` methods are evaluated in-memory.
