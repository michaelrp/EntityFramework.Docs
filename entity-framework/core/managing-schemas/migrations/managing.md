---
title: Managing Migrations - EF Core
author: bricelam
ms.author: bricelam
ms.date: 05/06/2020
uid: core/managing-schemas/migrations/managing
---
# Managing Migrations

As your model changes, migrations are added and removed as part of normal development, and the migration files are checked into your project's source control. To manage migrations, you must first install the [EF Core command-line tools](xref:core/miscellaneous/cli/index).

> [!TIP]
> If the `DbContext` is in a different assembly than the startup project, you can explicitly specify the target and startup projects in either the [Package Manager Console tools](xref:core/miscellaneous/cli/powershell#target-and-startup-project) or the [.NET Core CLI tools](xref:core/miscellaneous/cli/dotnet#target-project-and-startup-project).

## Add a migration

After your model has been changed, you can add a migration for that change:

### [.NET Core CLI](#tab/dotnet-core-cli)

```dotnetcli
dotnet ef migrations add AddBlogCreatedTimestamp
```

### [Visual Studio](#tab/vs)

``` powershell
Add-Migration AddBlogCreatedTimestamp
```

***

The migration name can be used like a commit message in a version control system. For example, you might choose a name like *AddBlogCreatedTimestamp* if the change is a new `CreatedTimestamp` property on your `Blog` entity.

Three files are added to your project under the **Migrations** directory:

* **XXXXXXXXXXXXXX_AddCreatedTimestamp.cs**--The main migrations file. Contains the operations necessary to apply the migration (in `Up`) and to revert it (in `Down`).
* **XXXXXXXXXXXXXX_AddCreatedTimestamp.Designer.cs**--The migrations metadata file. Contains information used by EF.
* **MyContextModelSnapshot.cs**--A snapshot of your current model. Used to determine what changed when adding the next migration.

The timestamp in the filename helps keep them ordered chronologically so you can see the progression of changes.

### Namespaces

You are free to move Migrations files and change their namespace manually. New migrations are created as siblings of the last migration. Alternatively, you can specify the namespace at generation time as follows:

### [.NET Core CLI](#tab/dotnet-core-cli)

```dotnetcli
dotnet ef migrations add InitialCreate --namespace Your.Namespace
```

### [Visual Studio](#tab/vs)

``` powershell
Add-Migration InitialCreate -Namespace Your.Namespace
```

***

## Customize migration code

While EF Core generally creates accurate migrations, you should always review the code and make sure it corresponds to the desired change; in some cases, it is even necessary to do so.

### Column renames

One notable example where customizing migrations is required is when renaming a property. For example, if you rename a property from `Name` to `FullName`, EF Core will generate the following migration:

```c#
migrationBuilder.DropColumn(
    name: "Name",
    table: "Customers");

migrationBuilder.AddColumn<string>(
    name: "FullName",
    table: "Customers",
    nullable: true);
```

EF Core is generally unable to know when the intention is to drop a column and create a new one (two separate changes), and when a column should be renamed. If the above migration is applied as-is, all your customer names will be lost. To rename a column, replace the above generated migration with the following:

```c#
migrationBuilder.RenameColumn(
    name: "Name",
    table: "Customers",
    newName: "FullName");
```

> [!TIP]
> The migration scaffolding process warns when an operation might result in data loss (like dropping a column). If you see that warning, be especially sure to review the migrations code for accuracy.

### Adding raw SQL

While renaming a column can be achieved via a built-in API, in many cases that is not possible. For example, we may want to replace existing `FirstName` and `LastColumn` properties with a single, new `FullName` property. The migration generated by EF Core will be the following:

``` csharp
migrationBuilder.DropColumn(
    name: "FirstName",
    table: "Customer");

migrationBuilder.DropColumn(
    name: "LastName",
    table: "Customer");

migrationBuilder.AddColumn<string>(
    name: "Name",
    table: "Customer",
    nullable: true);
```

As before, this would cause unwanted data loss. To transfer the data from the old columns, we rearrange the migrations and introduce a raw SQL operation as follows:

``` csharp
migrationBuilder.AddColumn<string>(
    name: "Name",
    table: "Customer",
    nullable: true);

migrationBuilder.Sql(
@"
    UPDATE Customer
    SET Name = FirstName + ' ' + LastName;
");

migrationBuilder.DropColumn(
    name: "FirstName",
    table: "Customer");

migrationBuilder.DropColumn(
    name: "LastName",
    table: "Customer");
```

### Arbitrary changes via raw SQL

Raw SQL can also be used to manage database objects that EF Core isn't aware of. To do this, add a migration without making any model change; an empty migration will be generated, which you can then populate with raw SQL operations.

For example, the following migration creates a SQL Server stored procedure:

```c#
migrationBuilder.Sql(
@"
    EXEC ('CREATE PROCEDURE getFullName
        @LastName nvarchar(50),
        @FirstName nvarchar(50)
    AS
        RETURN @LastName + @FirstName;')");
```

This can be used to manage any aspect of your database, including:

* Stored procedures
* Full-Text Search
* Functions
* Triggers
* Views

In most cases, EF Core will automatically wrap each migration in its own transaction when applying migrations. Unfortunately, some migrations operations cannot be performed within a transaction in some databases; for these cases, you may opt out of the transaction by passing `suppressTransaction: true` to `migrationBuilder.Sql`.

If the `DbContext` is in a different assembly than the startup project, you can explicitly specify the target and startup projects in either the [Package Manager Console tools](xref:core/miscellaneous/cli/powershell#target-and-startup-project) or the [.NET Core CLI tools](xref:core/miscellaneous/cli/dotnet#target-project-and-startup-project).

## Remove a migration

Sometimes you add a migration and realize you need to make additional changes to your EF Core model before applying it. To remove the last migration, use this command.

### [.NET Core CLI](#tab/dotnet-core-cli)

```dotnetcli
dotnet ef migrations remove
```

### [Visual Studio](#tab/vs)

``` powershell
Remove-Migration
```

***

After removing the migration, you can make the additional model changes and add it again.

> [!WARNING]
> Take care not to remove any migrations which are already applied to production databases. Not doing so will prevent you from being able to revert it, and may break the assumptions made by subsequent migrations.

## Listing migrations

You can list all existing migrations as follows:

```dotnetcli
dotnet ef migrations list
```

## Resetting all migrations

In some extreme cases, it may be necessary to remove all migrations and start over. This can be easily done by deleting your **Migrations** folder and dropping your database; at that point you can create a new initial migration, which will contain you entire current schema.

It's also possible to reset all migrations and create a single one without losing your data. This is sometimes called "squashing", and involves some manual work:

* Delete your **Migrations** folder
* Create a new migration and generate a SQL script for it
* In your database, delete all rows from the migrations history table
* Insert a single row into the migrations history, to record that the first migration has already been applied, since your tables are already there. The insert SEQL is the last operation in the SQL script generated above.