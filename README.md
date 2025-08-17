# Potato.NET.SQLite

## 📌 What is **Potato.NET.SQLite**?

It is a **framework/library** that you created to simplify database connectivity in .NET applications.

* Provide **common utilities** for working with databases.
* Expose a **clean API** to your users, so they don’t have to configure EF Core themselves.
* Package EF Core and its dependencies **inside your own NuGet package** → when someone installs `Potato.NET.SQLite`, all required dependencies come automatically.

It is a lightweight framework DLL that wires up **Entity Framework Core + SQLite** for desktop apps (WPF/WinForms/Console).
You define your own `DbSet<T>`s (tables) in a small context class and start saving data immediately.

* ✔️ Works with **.NET 8**
* ✔️ Uses **EF Core 9** + **SQLite**
* ✔️ Simple configuration via `AppDbConfig`
* ✔️ Supports runtime `Database.Migrate()` (no need to run `Update-Database` manually)
* ✔️ Optional `TableMappings` for dynamic access to sets

---

## Installation

### Option A — NuGet (recommended)

In **Package Manager Console**:

```powershell
Install-Package Potato.NET.SQLite -Version 1.0.1
```

Or with **.NET CLI**:

```bash
dotnet add package Potato.NET.SQLite --version 1.0.1
```

> **Note:** `Potato.NET.SQLite` brings EF Core runtime dependencies.
> If you plan to **scaffold migrations** with EF tools (`Add-Migration`, `Update-Database`) in your app project, also install:
>
> ```powershell
> Install-Package Microsoft.EntityFrameworkCore.Design
> Install-Package Microsoft.EntityFrameworkCore.Tools
> ```
>
> (The runtime **does not** require these; they’re only for design-time commands.)

---

## Quick Start

### 1) Configure where the database file lives

```csharp
// Usually at app startup (Program.cs for console, App.xaml.cs for WPF)
AppDbConfig.DbDirectory = @"C:\PotatoMenuApp"; // any absolute folder path
AppDbConfig.DbName = "potato.db";

// Ensure the directory exists
Directory.CreateDirectory(AppDbConfig.DbDirectory);
```

### 2) Create your application context and tables

Create a **context** that inherits from `AppDbContext`, then declare your **DbSets** (tables).
Put this in your app project:

```csharp
public class MyAppDbContext : AppDbContext
{
    // Define your tables here
    public DbSet<YourModel> YourTable { get; set; }
    public DbSet<OtherModel> OtherTable { get; set; }
}

// Example models
public class YourModel
{
    public int Id { get; set; }          // PK
    public string Name { get; set; }
    public string Age { get; set; }
    public string Address { get; set; }
}

public class OtherModel
{
    public int Id { get; set; }          // PK
    public string Description { get; set; }
}
```

> `AppDbContext` (inside the library) already knows how to build the SQLite connection string from `AppDbConfig`.

### 3) Create/upgrade the database (apply migrations at runtime)

```csharp
using var db = new MyAppDbContext();
db.Database.Migrate(); // Ensures the database is created and upgraded to the latest schema
```

> Prefer `Migrate()` over `EnsureCreated()` for real apps, because it supports schema changes over time.

### 4) (Optional) Register table name mappings

If you want to access sets dynamically by string name elsewhere:

```csharp
AppDbConfig.RegisterTable<YourModel>("YourTable");
AppDbConfig.RegisterTable<OtherModel>("OtherTable");
```

### 5) Insert and read some data

```csharp
db.YourTable.Add(new YourModel { Name = "Test Data", Age = "IT", Address = "JAPAN" });
db.SaveChanges();

foreach (var item in db.YourTable)
{
    Console.WriteLine($"{item.Id}: {item.Name} / {item.Age} / {item.Address}");
}
```

---

## Where to put the startup code

* **Console app (Program.cs)**:

  ```csharp
  AppDbConfig.DbDirectory = @"C:\PotatoMenuApp";
  AppDbConfig.DbName = "potato.db";
  Directory.CreateDirectory(AppDbConfig.DbDirectory);

  using var db = new MyAppDbContext();
  db.Database.Migrate();
  // now use db...
  ```

* **WPF (App.xaml.cs → OnStartup)**:

  ```csharp
  protected override async void OnStartup(StartupEventArgs e)
  {
      base.OnStartup(e);

      AppDbConfig.DbDirectory = @"C:\PotatoMenuApp";
      AppDbConfig.DbName = "potato.db";
      Directory.CreateDirectory(AppDbConfig.DbDirectory);

      // Avoid blocking the UI thread
      await Task.Run(() =>
      {
          using var db = new MyAppDbContext();
          db.Database.Migrate();
      });

      new MainWindow().Show();
  }
  ```

---

## CRUD — Insert, Read, Update, Delete

All examples assume:

```csharp
using var db = new MyAppDbContext();
```

### Create (Insert)

```csharp
var row = new YourModel
{
    Name = "Alice",
    Age = "28",
    Address = "Osaka"
};

db.YourTable.Add(row);
db.SaveChanges();
```

### Read (Query)

```csharp
// All rows
var all = db.YourTable.ToList();

// Filter
var inJapan = db.YourTable.Where(x => x.Address == "JAPAN").ToList();

// Single by Id (returns null if not found)
var id5 = db.YourTable.FirstOrDefault(x => x.Id == 5);

// Projections (select specific fields)
var names = db.YourTable.Select(x => new { x.Id, x.Name }).ToList();
```

### Update (by Id)

```csharp
var target = db.YourTable.FirstOrDefault(x => x.Id == 5);
if (target != null)
{
    target.Address = "Tokyo";
    db.SaveChanges();
}
```

### Update (bulk with condition) — EF Core 7+/9+ feature

```csharp
// Set Address = "Tokyo" for everyone named "Alice"
db.YourTable
  .Where(x => x.Name == "Alice")
  .ExecuteUpdate(s => s.SetProperty(x => x.Address, x => "Tokyo"));
```

### Delete (by Id)

```csharp
var toDelete = db.YourTable.FirstOrDefault(x => x.Id == 5);
if (toDelete != null)
{
    db.YourTable.Remove(toDelete);
    db.SaveChanges();
}
```

### Delete (bulk with condition) — EF Core 7+/9+ feature

```csharp
// Delete all where Address is null/empty
db.YourTable
  .Where(x => string.IsNullOrEmpty(x.Address))
  .ExecuteDelete();
```

### Transactions (optional)

```csharp
using var tx = db.Database.BeginTransaction();
try
{
    db.YourTable.Add(new YourModel { Name = "Bob" });
    db.SaveChanges();

    db.OtherTable.Add(new OtherModel { Description = "Hello" });
    db.SaveChanges();

    tx.Commit();
}
catch
{
    tx.Rollback();
    throw;
}
```

---

## Migrations — two ways to manage schema changes

### A) **Runtime only** (simple)

When your models change (e.g., add a new property), you **create a migration** once, and then let the app apply it at runtime:

1. In Package Manager Console (pick the app project as **Default project**):

```powershell
Add-Migration AddAgeToYourModel -Context MyAppDbContext
```

2. You can either run:

```powershell
Update-Database -Context MyAppDbContext
```

**or** just start the app — the call to `db.Database.Migrate()` applies it automatically.

> If your `DbContext` lives in a separate class library, specify both projects:
>
> ```powershell
> Add-Migration AddAgeToYourModel -Context MyAppDbContext `
>   -Project YourDataProject -StartupProject YourAppProject
>
> Update-Database -Context MyAppDbContext `
>   -Project YourDataProject -StartupProject YourAppProject
> ```

### B) **Design-time every time** (classic)

If you prefer not to run `Migrate()` at runtime, you can always do:

```powershell
Add-Migration DescriptiveName -Context MyAppDbContext
Update-Database -Context MyAppDbContext
```

---

## Dynamic table access (optional)

If you registered mappings:

```csharp
AppDbConfig.RegisterTable<YourModel>("YourTable");
AppDbConfig.RegisterTable<OtherModel>("OtherTable");
```

You can resolve sets dynamically:

```csharp
// Using the built-in EF API
var type = AppDbConfig.TableMappings["YourTable"];  // typeof(YourModel)
var set = db.Set(type); // returns DbSet for that entity type
```

Or (if the base context exposes a helper like `GetDbSet<T>()`):

```csharp
var yourSet = db.GetDbSet<YourModel>();
```

---

## Troubleshooting

* **“No migrations were found…” when running `Update-Database`**
  You created the migration in a **different project** than your startup project. Use:

  ```powershell
  Add-Migration Name -Context MyAppDbContext -Project YourDataProject -StartupProject YourAppProject
  Update-Database -Context MyAppDbContext -Project YourDataProject -StartupProject YourAppProject
  ```

* **`Add-Migration` not recognized**
  Install tools in the **app** project:

  ```powershell
  Install-Package Microsoft.EntityFrameworkCore.Tools
  Install-Package Microsoft.EntityFrameworkCore.Design
  ```

* **`FileNotFoundException: Microsoft.EntityFrameworkCore.Sqlite`**
  Ensure the app project references EF Core SQLite (transitive usually works, but if needed add explicitly):

  ```powershell
  Install-Package Microsoft.EntityFrameworkCore.Sqlite
  ```

* **App hangs on `Database.Migrate()`**

  * Close DB browser tools; delete any `.db-wal` / `.db-shm` files near the DB.
  * Verify `AppDbConfig.DbDirectory` exists and is writable.
  * On WPF, run `Migrate()` off the UI thread (see WPF example above).
  * If migrations live in a different assembly, confirm the **correct context and assembly** via commands.

* **DB updated somewhere else**
  You might be writing to `bin\Debug\...` vs. another absolute path.
  Always set an explicit `AppDbConfig.DbDirectory` and `DbName`.

---

## License

Licensed under **Apache-2.0**.

---


### Minimal End-to-End Sample (all together)

```csharp
// 1) Configure DB location
AppDbConfig.DbDirectory = @"C:\PotatoMenuApp";
AppDbConfig.DbName = "potato.db";
Directory.CreateDirectory(AppDbConfig.DbDirectory);

// 2) Context + models (typically defined in your app project)
public class MyAppDbContext : AppDbContext
{
    public DbSet<YourModel> YourTable { get; set; }
    public DbSet<OtherModel> OtherTable { get; set; }
}

public class YourModel
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Age { get; set; }
    public string Address { get; set; }
}

public class OtherModel
{
    public int Id { get; set; }
    public string Description { get; set; }
}

// 3) Create/upgrade schema at runtime
using var db = new MyAppDbContext();
db.Database.Migrate();

// 4) Optional dynamic table name registration
AppDbConfig.RegisterTable<YourModel>("YourTable");
AppDbConfig.RegisterTable<OtherModel>("OtherTable");

// 5) CRUD
db.YourTable.Add(new YourModel { Name = "Test Data", Age = "IT", Address = "JAPAN" });
db.SaveChanges();

var first = db.YourTable.FirstOrDefault();
if (first != null)
{
    first.Address = "Tokyo";
    db.SaveChanges();
}

db.YourTable
  .Where(x => x.Name == "Test Data")
  .ExecuteUpdate(s => s.SetProperty(x => x.Age, _ => "29"));

db.YourTable
  .Where(x => x.Name == "Obsolete")
  .ExecuteDelete();

foreach (var item in db.YourTable)
{
    Console.WriteLine($"{item.Id}: {item.Name} / {item.Age} / {item.Address}");
}
```
