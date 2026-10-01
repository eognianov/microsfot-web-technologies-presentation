# Set 4 — Database and EF Core

Covers decks **07 Data, Databases and ORMs** and **08 Code-First and Migrations**.

Students have seen tables, rows, keys and relationships, a working minimum of SQL, the
object–relational mismatch, EF Core's entity and `DbContext`, connection strings,
dependency injection, data annotations, migrations, seeding, async queries and the SQL that
EF Core logs. This set moves RecipeBox's `static` list into a SQLite file. By the end the
pages look exactly as they did after Set 3, and that is the point.

## Time budget · 2 hours

| # | Task | Part | Min |
|---|---|---|---|
| 1 | Sketch the `Recipes` table on paper | A · SQL | 6 |
| 2 | Write four SQL queries against a sample table | A · SQL | 12 |
| 3 | Install the EF Core SQLite packages | B · EF Core | 5 |
| 4 | Create `RecipeBoxContext` and register it with DI | B · EF Core | 12 |
| 5 | Add the connection string to `appsettings.json` | B · EF Core | 5 |
| 6 | Add data annotations | B · EF Core | 10 |
| 7 | Create and apply the first migration | B · EF Core | 12 |
| 8 | Seed five recipes | B · EF Core | 10 |
| 9 | Switch the controller from the list to the database | B · EF Core | 16 |
| 10 | Add a `Difficulty` property with a second migration | B · EF Core | 12 |
| 11 | Find the generated SQL in the console log | B · EF Core | 5 |
| 12 | Add a `Category` entity *(bonus)* | C · Bonus | 15 |
| | **Total** | | **120** |
| | **Without the bonus** | | **105** |

Task 12 is a bonus: Set 5 does not depend on it. Start it only if everything else is done.

## The project

```
RecipeBox/
├── Controllers/RecipesController.cs   ← gets the context injected, Task 9
├── Data/RecipeBoxContext.cs           ← new, Task 4
├── Migrations/                        ← generated, Tasks 7, 8, 10
├── Models/Recipe.cs                   ← annotations, Task 6 · Difficulty, Task 10
├── Program.cs                         ← one registration, Task 4
├── appsettings.json                   ← connection string, Task 5
└── recipebox.db                       ← created by Task 7
```

- **Starts from:** the end of Set 3. Pages are views fed by a `static` list.
- **Ends with:** the same pages, fed by a SQLite database that EF Core created from our
  `Recipe` class, with five seeded recipes and three migrations.

You need a way to look inside a SQLite file. Any one of these works:

- the `sqlite3` command-line tool (built into macOS and most Linux; on Windows, download
  the command-line tools from sqlite.org),
- the **SQLite Viewer** extension for VS Code,
- the free **DB Browser for SQLite** app.

---

## Part A — SQL

### Task 1 — Sketch the `Recipes` table on paper · 6 min

**Goal:** decide what a table looks like before any tool decides it for us.

**Steps**

1. On paper, draw a table called `Recipes` for our `Recipe` class.
2. For each column, write its name, a type (`INTEGER` or `TEXT`) and whether it may be
   empty.
3. Mark the primary key.
4. Decide what to do with `Ingredients`. A cell holds one value, and a recipe has five.

**Acceptance criteria**

- [ ] Seven columns, one per property of `Recipe`.
- [ ] `Id` is marked as the primary key.
- [ ] One written sentence on how `Ingredients` fits, with the trade-off.
- [ ] Fill in two rows from the sample data.

**Hints**

- Methods like `TotalMinutes()` are not columns. They are computed from columns.
- Two honest answers for `Ingredients`: a separate table with one row per ingredient, or
  all of them packed into one text cell. Task 7 shows which one EF Core picks.

**Solution**

```
Recipes
┌────┬──────────────┬──────────┬─────────────┬─────────────┬──────────┬──────────────────────────────┐
│ Id │ Title        │ Cuisine  │ PrepMinutes │ CookMinutes │ Servings │ Ingredients                  │
│ PK │ TEXT, req.   │ TEXT,req.│ INTEGER     │ INTEGER     │ INTEGER  │ TEXT, req.                   │
├────┼──────────────┼──────────┼─────────────┼─────────────┼──────────┼──────────────────────────────┤
│ 1  │ Pancakes     │ American │ 10          │ 15          │ 4        │ ["Flour","Milk","Eggs",...]  │
│ 3  │ Shopska Salad│ Bulgarian│ 15          │ 0           │ 4        │ ["Tomatoes","Cucumbers",...] │
└────┴──────────────┴──────────┴─────────────┴─────────────┴──────────┴──────────────────────────────┘
```

Packing the ingredients into one `TEXT` cell is simple and fine for display. It makes "all
recipes with Eggs" a text search instead of a join. A separate `Ingredients` table with a
`RecipeId` foreign key is the textbook design; Task 12 shows the same shape with
categories.

---

### Task 2 — Write four SQL queries against a sample table · 12 min

**Goal:** the four questions from Set 1's LINQ task, asked in SQL.

**Steps**

1. Create a file `practice.sql` anywhere, with this content:

   ```sql
   CREATE TABLE Recipes (
       Id          INTEGER PRIMARY KEY,
       Title       TEXT    NOT NULL,
       Cuisine     TEXT    NOT NULL,
       PrepMinutes INTEGER NOT NULL,
       CookMinutes INTEGER NOT NULL,
       Servings    INTEGER NOT NULL
   );

   INSERT INTO Recipes VALUES (1, 'Pancakes',            'American',  10, 15, 4);
   INSERT INTO Recipes VALUES (2, 'Spaghetti Carbonara', 'Italian',   10, 15, 2);
   INSERT INTO Recipes VALUES (3, 'Shopska Salad',       'Bulgarian', 15,  0, 4);
   INSERT INTO Recipes VALUES (4, 'Margherita Pizza',    'Italian',   30, 12, 2);
   INSERT INTO Recipes VALUES (5, 'Chicken Curry',       'Indian',    20, 40, 4);
   ```

2. Load it: `sqlite3 practice.db < practice.sql`, then open it with `sqlite3 practice.db`.
3. Write one query per question:
   1. The title and cuisine of every recipe.
   2. Every column of the Italian recipes.
   3. The title and total minutes of recipes under 30 minutes, fastest first.
   4. How many recipes each cuisine has.

**Acceptance criteria**

- [ ] 1 → five rows, two columns.
- [ ] 2 → Spaghetti Carbonara and Margherita Pizza, all six columns.
- [ ] 3 → Shopska Salad 15, Pancakes 25, Spaghetti Carbonara 25.
- [ ] 4 → American 1, Bulgarian 1, Indian 1, Italian 2.

**Hints**

- In `sqlite3`, type `.headers on` and `.mode column` first for readable output, and
  `.quit` to leave.
- Strings use single quotes in SQL: `'Italian'`.
- A computed column can be named: `PrepMinutes + CookMinutes AS TotalMinutes`, and then
  sorted by that name.
- `COUNT(*)` with `GROUP BY Cuisine` counts per group.
- Pancakes and Carbonara both take 25 minutes. Add `, Title` to the `ORDER BY` to make
  their order certain.

**Solution**

```sql
-- 1 · every title and cuisine
SELECT Title, Cuisine FROM Recipes;

-- 2 · Italian recipes, every column
SELECT * FROM Recipes WHERE Cuisine = 'Italian';

-- 3 · under 30 minutes, fastest first
SELECT Title, PrepMinutes + CookMinutes AS TotalMinutes
FROM Recipes
WHERE PrepMinutes + CookMinutes < 30
ORDER BY TotalMinutes, Title;

-- 4 · recipes per cuisine
SELECT Cuisine, COUNT(*) AS Recipes FROM Recipes GROUP BY Cuisine;
```

```
Title                TotalMinutes          Cuisine    Recipes
-------------------  ------------          ---------  -------
Shopska Salad        15                    American   1
Pancakes             25                    Bulgarian  1
Spaghetti Carbonara  25                    Indian     1
                                           Italian    2
```

---

## Part B — EF Core

### Task 3 — Install the EF Core SQLite packages · 5 min

**Goal:** add the two packages and the command-line tool that the rest of the set uses.

**Steps**

1. Stop `dotnet watch`. From the `RecipeBox` folder:

   ```bash
   dotnet add package Microsoft.EntityFrameworkCore.Sqlite
   dotnet add package Microsoft.EntityFrameworkCore.Design
   dotnet tool install --global dotnet-ef
   ```

2. Open `RecipeBox.csproj`.

**Acceptance criteria**

- [ ] `RecipeBox.csproj` has two new `<PackageReference>` lines, both version `10.x`.
- [ ] `dotnet ef --version` prints a `10.x` version.
- [ ] `dotnet build` → 0 errors.

**Hints**

- If `dotnet tool install` says the tool is already installed, that is fine. Use
  `dotnet tool update --global dotnet-ef` to get the latest.
- If `dotnet ef` is "not found" right after installing, open a new terminal.
- A warning that the tools are older than the runtime is harmless.

**Solution**

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="10.0.12" />
  <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="10.0.12" />
</ItemGroup>
```

The patch number may be higher. `Design` also gets a few `PrivateAssets` lines; leave them.

---

### Task 4 — Create `RecipeBoxContext` and register it with DI · 12 min

**Goal:** the database as a C# object, built by the framework, never by us.

**Steps**

1. Create `Data/RecipeBoxContext.cs`: a class `RecipeBoxContext` that inherits
   `DbContext`, takes `DbContextOptions<RecipeBoxContext>` in its constructor, and has a
   `DbSet<Recipe> Recipes`.
2. In `Program.cs`, after `AddControllersWithViews()`, register it with
   `AddDbContext`, using SQLite and a connection string named `RecipeBox`.
3. `dotnet build`.

**Acceptance criteria**

- [ ] `RecipeBoxContext` lives in the `RecipeBox.Data` namespace.
- [ ] The word `new RecipeBoxContext` appears nowhere in the project.
- [ ] `dotnet build` → 0 errors, 0 warnings.
- [ ] The app still runs: nothing asks for the context yet.

**Hints**

- `public DbSet<Recipe> Recipes => Set<Recipe>();` is one table. Its name becomes the
  table name.
- `: base(options)` hands the options to `DbContext`. Our class does not decide which
  database it talks to.
- `Program.cs` needs `using Microsoft.EntityFrameworkCore;` and `using RecipeBox.Data;`.
- `builder.Configuration.GetConnectionString("RecipeBox")` reads Task 5's setting.

**Solution**

`Data/RecipeBoxContext.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using RecipeBox.Models;

namespace RecipeBox.Data;

public class RecipeBoxContext : DbContext
{
    public RecipeBoxContext(DbContextOptions<RecipeBoxContext> options)
        : base(options) { }

    public DbSet<Recipe> Recipes => Set<Recipe>();
}
```

`Program.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using RecipeBox.Data;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllersWithViews();

builder.Services.AddDbContext<RecipeBoxContext>(options =>
    options.UseSqlite(builder.Configuration.GetConnectionString("RecipeBox")));

var app = builder.Build();
```

---

### Task 5 — Add the connection string to `appsettings.json` · 5 min

**Goal:** tell the app where the database lives, in configuration rather than in code.

**Steps**

1. Add a `ConnectionStrings` section to `appsettings.json` with one entry, `RecipeBox`,
   pointing at `recipebox.db`.

**Acceptance criteria**

- [ ] The name in `appsettings.json` matches the name in `GetConnectionString(...)`
      exactly.
- [ ] The file is still valid JSON: the app starts.

**Hints**

- For SQLite the whole connection string is `Data Source=recipebox.db`: a file path,
  relative to the folder the app runs from.
- A missing comma between sections is the usual JSON mistake.
- A real database's connection string holds a password. Never commit one; Session 10
  comes back to this.

**Solution**

```json
{
  "ConnectionStrings": {
    "RecipeBox": "Data Source=recipebox.db"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

---

### Task 6 — Add data annotations · 10 min

**Goal:** write the rules for a recipe once, on the model, where the database (Task 7) and
the forms (Set 5) will both read them.

**Steps**

1. In `Recipe.cs`, add `using System.ComponentModel.DataAnnotations;`.
2. `Title`: required, 3 to 100 characters. `Cuisine`: required, at most 50.
3. `PrepMinutes` and `CookMinutes`: 0 to 600. `Servings`: 1 to 20.
4. Give the two minutes properties a friendly `[Display(Name = ...)]`.

**Acceptance criteria**

- [ ] `[Required]` and `[StringLength]` on both strings; `[Range]` on the three numbers.
- [ ] `dotnet build` → 0 warnings.
- [ ] The pages look the same as before. Annotations change nothing until something
      reads them.

**Hints**

- `[StringLength(100, MinimumLength = 3)]` sets both limits at once.
- `[Range(1, 20)]` includes both ends.
- Several attributes can stack on one property, one per line.

**Solution**

```csharp
using System.ComponentModel.DataAnnotations;

namespace RecipeBox.Models;

public class Recipe
{
    public int Id { get; set; }

    [Required]
    [StringLength(100, MinimumLength = 3)]
    public string Title { get; set; } = "";

    [Required]
    [StringLength(50)]
    public string Cuisine { get; set; } = "";

    [Range(0, 600)]
    [Display(Name = "Prep minutes")]
    public int PrepMinutes { get; set; }

    [Range(0, 600)]
    [Display(Name = "Cook minutes")]
    public int CookMinutes { get; set; }

    [Range(1, 20)]
    public int Servings { get; set; }

    public List<string> Ingredients { get; set; } = new();

    public int TotalMinutes() => PrepMinutes + CookMinutes;
    public bool IsQuick() => TotalMinutes() < 20;
}
```

---

### Task 7 — Create and apply the first migration · 12 min

**Goal:** let EF Core turn the class into a table, and read what it wrote before running
it.

**Steps**

1. `dotnet ef migrations add InitialCreate`
2. Open the new file in `Migrations/` that ends in `_InitialCreate.cs`. Read its `Up`
   method.
3. `dotnet ef database update`
4. Open `recipebox.db` and look at its tables.

**Acceptance criteria**

- [ ] `Up` creates a `Recipes` table with seven columns. `Title` has `maxLength: 100`,
      `Cuisine` has `maxLength: 50`, and both are `nullable: false`.
- [ ] `recipebox.db` exists and contains `Recipes` and `__EFMigrationsHistory`.
- [ ] `__EFMigrationsHistory` has one row, ending in `_InitialCreate`.
- [ ] Your Task 1 sketch: what did EF Core choose for `Ingredients`?

**Hints**

- `[StringLength]` became `maxLength`; `[Required]` became `nullable: false`. `[Range]`
  left no trace: a range is checked by the app, not by the table.
- `Ingredients` became one `TEXT` column. EF Core stores a `List<string>` as a JSON array
  in a single cell.
- Run `dotnet ef database update` a second time: nothing happens. The history table says
  it is already applied.
- `sqlite3 recipebox.db ".tables"` lists the tables from the terminal.

**Solution**

```csharp
migrationBuilder.CreateTable(
    name: "Recipes",
    columns: table => new
    {
        Id = table.Column<int>(type: "INTEGER", nullable: false)
            .Annotation("Sqlite:Autoincrement", true),
        Title = table.Column<string>(type: "TEXT", maxLength: 100, nullable: false),
        Cuisine = table.Column<string>(type: "TEXT", maxLength: 50, nullable: false),
        PrepMinutes = table.Column<int>(type: "INTEGER", nullable: false),
        CookMinutes = table.Column<int>(type: "INTEGER", nullable: false),
        Servings = table.Column<int>(type: "INTEGER", nullable: false),
        Ingredients = table.Column<string>(type: "TEXT", nullable: false)
    },
    constraints: table =>
    {
        table.PrimaryKey("PK_Recipes", x => x.Id);
    });
```

```
$ sqlite3 recipebox.db ".tables"
Recipes                __EFMigrationsHistory  __EFMigrationsLock
```

---

### Task 8 — Seed five recipes · 10 min

**Goal:** known starting data, recorded in the migration history like any other change.

**Steps**

1. In `RecipeBoxContext`, override `OnModelCreating` and call
   `modelBuilder.Entity<Recipe>().HasData(...)` with the five recipes. Copy them from the
   controller's `_recipes` list.
2. `dotnet ef migrations add SeedRecipes`, and read the new migration.
3. `dotnet ef database update`.

**Acceptance criteria**

- [ ] The `SeedRecipes` migration contains an `InsertData` call, not a `CreateTable`.
- [ ] `SELECT Id, Title FROM Recipes;` returns five rows.
- [ ] The `Ingredients` column of row 1 holds `["Flour","Milk","Eggs","Sugar"]`.

**Hints**

- `protected override void OnModelCreating(ModelBuilder modelBuilder)` — type
  `override` and let the editor complete it.
- Each seeded recipe **must** have its `Id` set. EF Core uses it to tell an insert from
  an update in later migrations.
- The controller still uses its own list. Task 9 switches it.

**Solution**

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Recipe>().HasData(
        new Recipe { Id = 1, Title = "Pancakes",            Cuisine = "American",  PrepMinutes = 10, CookMinutes = 15, Servings = 4,
                     Ingredients = new() { "Flour", "Milk", "Eggs", "Sugar" } },
        new Recipe { Id = 2, Title = "Spaghetti Carbonara", Cuisine = "Italian",   PrepMinutes = 10, CookMinutes = 15, Servings = 2,
                     Ingredients = new() { "Spaghetti", "Eggs", "Pecorino", "Guanciale", "Black pepper" } },
        new Recipe { Id = 3, Title = "Shopska Salad",       Cuisine = "Bulgarian", PrepMinutes = 15, CookMinutes = 0,  Servings = 4,
                     Ingredients = new() { "Tomatoes", "Cucumbers", "Peppers", "Onion", "Sirene" } },
        new Recipe { Id = 4, Title = "Margherita Pizza",    Cuisine = "Italian",   PrepMinutes = 30, CookMinutes = 12, Servings = 2,
                     Ingredients = new() { "Flour", "Tomatoes", "Mozzarella", "Basil" } },
        new Recipe { Id = 5, Title = "Chicken Curry",       Cuisine = "Indian",    PrepMinutes = 20, CookMinutes = 40, Servings = 4,
                     Ingredients = new() { "Chicken", "Onion", "Curry paste", "Coconut milk", "Rice" } }
    );
}
```

```
$ sqlite3 recipebox.db "SELECT Id, Title, Ingredients FROM Recipes;"
1|Pancakes|["Flour","Milk","Eggs","Sugar"]
2|Spaghetti Carbonara|["Spaghetti","Eggs","Pecorino","Guanciale","Black pepper"]
...
```

---

### Task 9 — Switch the controller from the list to the database · 16 min

**Goal:** ask for the context in the constructor, query it with the same LINQ, and watch
no view change.

**Steps**

1. Delete the `static` list from `RecipesController`.
2. Add a constructor that takes a `RecipeBoxContext` and stores it in a `private
   readonly` field.
3. Make every action `async Task<IActionResult>` and use the `...Async` methods:
   `ToListAsync`, `FirstOrDefaultAsync`, `CountAsync`.
4. Make `BuildList` async too, and build its query step by step.
5. Prove the data comes from the file: change a title in SQLite, reload, change it back.

**Acceptance criteria**

- [ ] `/recipes`, `/recipes/3`, `/recipes/cuisine/italian`, `/about` and `/recipes/json`
      look exactly as before.
- [ ] No file in `Views/` was edited.
- [ ] After `UPDATE Recipes SET Title = 'Fluffy Pancakes' WHERE Id = 1;` a reload shows
      **Fluffy Pancakes**, with no rebuild.
- [ ] `dotnet build` → 0 warnings.

**Hints**

- `using Microsoft.EntityFrameworkCore;` brings in the `...Async` methods.
- `var query = _context.Recipes.AsNoTracking();` then `query = query.Where(...)` only if
  a cuisine was given. Nothing runs until `ToListAsync()`.
- An `async` method returns `Task<T>`; the caller writes `await BuildList(cuisine)`.
- `r.Cuisine.ToLower() == cuisine.ToLower()` still works: EF Core translates `ToLower`
  into SQL.

**Solution**

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using RecipeBox.Data;
using RecipeBox.Models.ViewModels;

namespace RecipeBox.Controllers;

[Route("recipes")]
public class RecipesController : Controller
{
    private readonly RecipeBoxContext _context;

    public RecipesController(RecipeBoxContext context)
    {
        _context = context;
    }

    [HttpGet("")]
    public async Task<IActionResult> Index(string? cuisine)
    {
        return View(await BuildList(cuisine));
    }

    [HttpGet("json")]
    public async Task<IActionResult> All() => Json(await _context.Recipes.ToListAsync());

    [HttpGet("{id:int}")]
    public async Task<IActionResult> Details(int id)
    {
        var recipe = await _context.Recipes.FirstOrDefaultAsync(r => r.Id == id);
        if (recipe is null)
            return NotFound();

        return View(recipe);
    }

    [HttpGet("cuisine/{name}")]
    public async Task<IActionResult> ByCuisine(string name)
    {
        return View("Index", await BuildList(name));
    }

    [HttpGet("/about")]
    public async Task<IActionResult> About()
    {
        return View(await _context.Recipes.CountAsync());
    }

    private async Task<RecipeListViewModel> BuildList(string? cuisine)
    {
        var query = _context.Recipes.AsNoTracking();

        if (!string.IsNullOrWhiteSpace(cuisine))
            query = query.Where(r => r.Cuisine.ToLower() == cuisine.ToLower());

        var matches = await query.OrderBy(r => r.Title).ToListAsync();

        return new RecipeListViewModel
        {
            Recipes     = matches,
            TotalCount  = await _context.Recipes.CountAsync(),
            Cuisine     = string.IsNullOrWhiteSpace(cuisine) ? null : matches.FirstOrDefault()?.Cuisine ?? cuisine,
            AllCuisines = await _context.Recipes.Select(r => r.Cuisine).Distinct().OrderBy(c => c).ToListAsync()
        };
    }
}
```

---

### Task 10 — Add a `Difficulty` property with a second migration · 12 min

**Goal:** change the schema of a database that already holds data, without losing any.

**Steps**

1. Add `Difficulty` to `Recipe`: an `int` from 1 to 3, starting at 1.
2. Add a difficulty to each seeded recipe: Pancakes 1, Carbonara 2, Shopska 1, Pizza 3,
   Curry 2.
3. `dotnet ef migrations add AddDifficulty`. Read it.
4. `dotnet ef database update`.
5. Show `difficulty 2 / 3` on the details page.

**Acceptance criteria**

- [ ] The migration has one `AddColumn` and five `UpdateData` calls, and no
      `CreateTable` or `DropTable`.
- [ ] The five recipes are still there, each with its new difficulty.
- [ ] `/recipes/4` shows `difficulty 3 / 3`.
- [ ] `__EFMigrationsHistory` now has three rows.

**Hints**

- `[Range(1, 3)] public int Difficulty { get; set; } = 1;`
- The daily cycle: change the class, `migrations add`, **read it**, `database update`,
  run.
- Changing seed data is a schema change too, as far as migrations are concerned. That is
  where the `UpdateData` calls come from.
- Made a mistake before `database update`? `dotnet ef migrations remove` deletes the
  last migration.

**Solution**

`Models/Recipe.cs`

```csharp
[Range(1, 3)]
public int Difficulty { get; set; } = 1;
```

`Data/RecipeBoxContext.cs`, each seeded recipe gains one value

```csharp
new Recipe { Id = 1, Title = "Pancakes", ..., Servings = 4, Difficulty = 1, ... },
```

`Migrations/…_AddDifficulty.cs`

```csharp
migrationBuilder.AddColumn<int>(
    name: "Difficulty",
    table: "Recipes",
    type: "INTEGER",
    nullable: false,
    defaultValue: 0);

migrationBuilder.UpdateData(
    table: "Recipes",
    keyColumn: "Id",
    keyValue: 1,
    column: "Difficulty",
    value: 1);
// ... four more UpdateData calls
```

`Views/Recipes/Details.cshtml`

```cshtml
= <strong>@Model.TotalMinutes() min</strong> · serves @Model.Servings
· difficulty @Model.Difficulty / 3
```

---

### Task 11 — Find the generated SQL in the console log · 5 min

**Goal:** see what EF Core sent to the database for our LINQ.

**Steps**

1. Run the app and open `/recipes/cuisine/italian`.
2. In the terminal, find the `Executed DbCommand` entries for that request.
3. Open `/recipes/2` and find its query.
4. Run `dotnet ef migrations script` and skim the output.

**Acceptance criteria**

- [ ] You found a `SELECT ... WHERE lower("r"."Cuisine") = @...` for the cuisine page.
- [ ] You found a `SELECT ... WHERE "r"."Id" = @id LIMIT 1` for the details page.
- [ ] One sentence: how many queries does one list page run, and why?

**Hints**

- The lines start with `info: Microsoft.EntityFrameworkCore.Database.Command`.
- `LIMIT 1` is `FirstOrDefault`; `lower(...)` is `ToLower()`.
- `@ToLower` and `@id` are **parameters**: the value is sent separately from the SQL, never
  pasted into it.
- `migrations script` prints the SQL for every migration, from the first.

**Solution**

```
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (3ms) [Parameters=[@ToLower='?' (Size = 7)], CommandType='Text', CommandTimeout='30']
      SELECT "r"."Id", "r"."CookMinutes", "r"."Cuisine", "r"."Difficulty", "r"."Ingredients", ...
      FROM "Recipes" AS "r"
      WHERE lower("r"."Cuisine") = @ToLower
      ORDER BY "r"."Title"

info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      SELECT "r"."Id", ...
      FROM "Recipes" AS "r"
      WHERE "r"."Id" = @id
      LIMIT 1
```

A list page runs **three** queries: the matching recipes, the total count, and the distinct
cuisines for the button group. One per `...Async` call in `BuildList`.

---

## Part C — Bonus

### Task 12 — Add a `Category` entity with a one-to-many relationship · 15 min · *bonus*

**Goal:** a second table, a foreign key, and `Include` to load related rows.

**Steps**

1. Create `Models/Category.cs`: `Id`, a required `Name` of at most 50 characters, and a
   `List<Recipe> Recipes`.
2. In `Recipe`, add `int? CategoryId` and `Category? Category`.
3. Add `DbSet<Category> Categories` to the context, and seed three categories:
   Breakfast, Main course, Salad. Give each seeded recipe a `CategoryId`.
4. `dotnet ef migrations add AddCategories`, read it, `dotnet ef database update`.
5. Show the category name on the details page.

**Acceptance criteria**

- [ ] The migration creates a `Categories` table, adds a `CategoryId` column, an index
      and a foreign key.
- [ ] `/recipes/3` shows **Salad**; `/recipes/1` shows **Breakfast**.
- [ ] Without `.Include(r => r.Category)` in `Details`, every recipe shows
      **Uncategorised**. With it, the name appears.

**Hints**

- Pancakes 1, Carbonara 2, Shopska 3, Pizza 2, Curry 2.
- `int?` makes the relationship optional: a recipe may have no category.
- EF Core never loads what we did not ask for. `Include` turns into a `LEFT JOIN`; look
  for it in the log.
- `@(Model.Category?.Name ?? "Uncategorised")` handles both a missing category and a
  forgotten `Include`.

**Solution**

`Models/Category.cs`

```csharp
using System.ComponentModel.DataAnnotations;

namespace RecipeBox.Models;

public class Category
{
    public int Id { get; set; }

    [Required]
    [StringLength(50)]
    public string Name { get; set; } = "";

    public List<Recipe> Recipes { get; set; } = new();
}
```

`Models/Recipe.cs`

```csharp
public int? CategoryId { get; set; }        // foreign key
public Category? Category { get; set; }     // navigation property
```

`Data/RecipeBoxContext.cs`

```csharp
public DbSet<Category> Categories => Set<Category>();

// in OnModelCreating, before the recipes:
modelBuilder.Entity<Category>().HasData(
    new Category { Id = 1, Name = "Breakfast" },
    new Category { Id = 2, Name = "Main course" },
    new Category { Id = 3, Name = "Salad" }
);
// and each recipe gains CategoryId = 1, 2 or 3
```

`Controllers/RecipesController.cs`, in `Details`

```csharp
var recipe = await _context.Recipes
    .Include(r => r.Category)
    .FirstOrDefaultAsync(r => r.Id == id);
```

```
SELECT "r"."Id", ..., "c"."Id", "c"."Name"
FROM "Recipes" AS "r"
LEFT JOIN "Categories" AS "c" ON "r"."CategoryId" = "c"."Id"
WHERE "r"."Id" = @id
LIMIT 1
```

**State after this set:** RecipeBox reads from `recipebox.db`, created and seeded by three
migrations (four with the bonus). The pages are unchanged. Set 5 adds the forms that write
to it.
