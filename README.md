![Icon](https://github.com/asherber/PetaPoco.SqlKata/raw/master/media/database-64.png)

# PetaPoco.SqlKata [![NuGet](https://img.shields.io/nuget/v/PetaPoco.SqlKata.svg)](https://nuget.org/packages/PetaPoco.SqlKata)

[PetaPoco](https://github.com/CollaboratingPlatypus/PetaPoco) is a handy micro-ORM, but the SQL builder that comes with it is extremely limited. This library lets you use [SqlKata](https://sqlkata.com) as a replacement query builder.

## Usage

### Basic

The simplest way to use this library is just to use a SqlKata `Query` object in place of a PetaPoco `Sql` object. There are extension methods on `IDatabase` which allow this for most of the basic query methods.

Note that there are no extension methods for `Single()` or `SingleOrDefault()`. This is because one of the overloads available in PetaPoco takes an `object` parameter, and this overload will resolve before an extension method with a more specific parameter type. The workaround is to use `Query().Single()`.

```csharp
public class MyTable
{
    property int ID { get; set; }
    property string Name { get; set; }
    property string Category { get; set; }
}

using (var db = new PetaPoco.Database(...))
{
    var query = new Query("MyTable").Select("ID", "Name", "Category").Where("Category", "foo");
    var records = db.Fetch<MyTable>(query);
}
```

Note that SqlKata requires a table name in order to render a query. However, this library can autogenerate the `SELECT` and `FROM` clauses from the target class, just like the PetaPoco `EnableAutoSelect` feature. 

```csharp
// This is equivalent to the example above
using (var db = new PetaPoco.Database(...))
{
    var query = new Query().Where("Category", "foo");
    var records = db.Fetch<MyTable>(query);
}
```

### Advanced

There are two other types of methods provided in this library which give you more control over how the SQL is generated. 

#### ToSql()

```csharp
// Build any SqlKata query
var query = new Query("MyTable")
    .Select("Field1", "Field2")
    .Where("Foo", "Bar");

// Turn it into a Sql object
var sql = query.ToSql();
```

Transforming a SqlKata `Query` into a SQL string (and then into a `Sql` object) requires a compiler. SqlKata comes with compilers for SQL Server, Postgres, MySql, Firebird, Oracle, and SQLite. For many simple queries, the generated SQL looks the same regardless of which compiler you use, but for certain queries the compiler will produce SQL tailored for that specific database. The compilers also know which characters to use to escape identifiers.

By default, `ToSql()` uses the SQL Server compiler. If you want to use a different compiler, there are a couple of different ways you can do so.

```csharp
// Specify the compiler for one SQL statement
var sql = query.ToSql(CompilerType.MySql);

// Change the default compiler for all SQL statements
SqlKataExtensions.DefaultCompiler = CompilerType.Postgres;
```

The `IDatabase` extension methods automatically pick a compiler based on the database's `Provider`. They do not use the `DefaultCompiler` property.

### Generate from POCO

Since part of the benefit of PetaPoco is that it understands information embedded in a POCO, there are also two extension methods to help do the same thing.

```csharp
public class MyTable
{
    property int ID { get; set; }
    [Column("NAME_FIELD")]
    property string Name { get; set; }
}

// These will all add the FROM and are equivalent to new Query("MyTable").
// If the class has a TableName property, that will be used instead.
var query = new Query().ForType<MyTable>();
var query = new Query().ForType(typeof(MyTable));
var query = new Query().ForObject(new MyTable());

// These will all add both the SELECT and the FROM:
// SELECT [ID], [NAME_FIELD] FROM [MyTable]
var query = new Query().GenerateSelect<MyTable>();  
var query = new Query().GenerateSelect(typeof(MyTable));
var query = new Query().GenerateSelect(new MyTable());
```

These methods need to have a PetaPoco `IMapper` in order to map class and property names to table and field names. By default, they use a `ConventionMapper`. They also have overloads that let you pass in your own `IMapper` instance, or you can assign a mapper to `SqlKataExtensions.DefaultMapper` to be used for all queries.

The `IDatabase` extension methods use PetaPoco to select the correct mapper based on the target type and the database's `DefaultMapper`. They do not use the `SqlKataExtensions.DefaultMapper` property.