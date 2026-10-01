# Exercises — RecipeBox

Five exercise sets, one after every two lecture decks. Each set fits in **2 hours** of
student work. All five build **one project, RecipeBox** — a small recipe catalogue with a
single `Recipe` entity. The domain is deliberately different from the lecture's
SimpleBlog, so students apply the ideas instead of copying the demo.

The project grows across the sets: a console app in Set 1 becomes the MVC app at the end
of Set 1, and every later set extends that same app.

| Set | Covers decks | File | Deck |
|---|---|---|---|
| 1 | 01–02 · HTTP and C# basics | [`01-http-and-csharp-basics.md`](01-http-and-csharp-basics.md) | [`ex01-http-and-csharp-basics.html`](../slides/ex01-http-and-csharp-basics.html) |
| 2 | 03–04 · Controllers and routing | [`02-controllers-and-routing.md`](02-controllers-and-routing.md) | [`ex02-controllers-and-routing.html`](../slides/ex02-controllers-and-routing.html) |
| 3 | 05–06 · Views, layouts and Bootstrap | [`03-views-layouts-and-bootstrap.md`](03-views-layouts-and-bootstrap.md) | [`ex03-views-layouts-and-bootstrap.html`](../slides/ex03-views-layouts-and-bootstrap.html) |
| 4 | 07–08 · Database and EF Core | [`04-database-and-ef-core.md`](04-database-and-ef-core.md) | [`ex04-database-and-ef-core.html`](../slides/ex04-database-and-ef-core.html) |
| 5 | 09–10 · CRUD, debugging and deployment | [`05-crud-debugging-and-deployment.md`](05-crud-debugging-and-deployment.md) | [`ex05-crud-debugging-and-deployment.html`](../slides/ex05-crud-debugging-and-deployment.html) |

## File template

Every set follows the same shape, so it can be scanned and timed like a session file:

- **Time budget** — one row per task, with optional (stretch) tasks marked.
- **The project** — folder layout and the state the set starts from and ends at.
- **Per task:** goal · steps · acceptance criteria · hints · solution.

Solutions are given in full and have been compiled and run against the .NET 10 SDK. In
the deck, each task has one slide with the brief and one with hints; the solution sits
behind a click on the hint slide.

## How the sets connect

| Set | Starts from | Ends with |
|---|---|---|
| 1 | nothing | a console app, and the `RecipeBox` MVC project with `Models/Recipe.cs` |
| 2 | Set 1 | `RecipesController` answering text and JSON at `/recipes`, `/recipes/{id:int}`, `/recipes/cuisine/{name}`, `/about` |
| 3 | Set 2 | Razor views, a `RecipeListViewModel` and a Bootstrap layout that holds at 375 px |
| 4 | Set 3 | the same pages fed by `recipebox.db`: three migrations, five seeded recipes |
| 5 | Set 4 | Create, Edit and Delete; a production error page; `ILogger`; a published build |

Each set's end state was built and run against the .NET 10 SDK (10.0.103) with EF Core
10.0.12, step by step in the order the tasks give.

Long solutions do not fit one hint slide. In the decks, those tasks get a third slide,
*Solution, continued*, right after the hints.
