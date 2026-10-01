# Set 2 — Controllers and routing

Covers decks **03 Handling a Request** and **04 Routing and URL Design**.

Students have seen MVC and the request lifecycle, controllers and actions, the
`IActionResult` family (`Content`, `Json`, `NotFound`, `View`), model binding from the route
and the query string, the default route, attribute routing, route constraints and
`Url.Action`. There are no views yet: every action answers with plain text or JSON, so the
browser shows exactly what the server sent. Views arrive in Set 3.

## Time budget · 2 hours

| # | Task | Part | Min |
|---|---|---|---|
| 1 | Add a `RecipesController` with an `Index` action | A · Controllers | 10 |
| 2 | Return the catalog as plain text and as JSON | A · Controllers | 12 |
| 3 | Add a `Details(int id)` action | A · Controllers | 10 |
| 4 | Return 404 for an unknown recipe | A · Controllers | 8 |
| 5 | Filter by cuisine with a query-string parameter | A · Controllers | 12 |
| 6 | Add an `About` action that returns `Content` *(optional)* | A · Controllers | 5 |
| 7 | Give each action a clean URL with attribute routing | B · Routing | 20 |
| 8 | Add an `int` route constraint to the id | B · Routing | 10 |
| 9 | Add a by-cuisine route | B · Routing | 15 |
| 10 | Generate links with `Url.Action` | B · Routing | 18 |
| | **Total** | | **120** |
| | **Without the optional task** | | **115** |

Task 6 is optional; Task 7 gives `About` its URL, so skip both mentions if you skip it.

## The project

```
RecipeBox/
├── Controllers/
│   ├── HomeController.cs      ← from the template, untouched
│   └── RecipesController.cs   ← new, every task in this set
└── Models/
    └── Recipe.cs              ← from Set 1, untouched
```

- **Starts from:** the end of Set 1. `RecipeBox` runs and contains `Models/Recipe.cs`.
- **Ends with:** a `RecipesController` with six actions, reachable at clean URLs such as
  `/recipes/2` and `/recipes/cuisine/italian`, answering in plain text or JSON.

Run `dotnet watch` from the `RecipeBox` folder and leave it running; it rebuilds on every
save. Keep DevTools open on the **Network** tab: every task ends with a status code to
read.

The five recipes are the same as in Set 1. Task 1 copies them from the
`RecipeBoxBasics` catalog.

---

## Part A — Controllers

### Task 1 — Add a `RecipesController` with an `Index` action · 10 min

**Goal:** a controller the framework finds by convention, with our five recipes inside it.

**Steps**

1. Add `Controllers/RecipesController.cs`. Declare a `public class RecipesController`
   that inherits `Controller`, in the namespace `RecipeBox.Controllers`.
2. Inside it, declare `private static readonly List<Recipe> _recipes` and fill it with
   the five recipes. Copy them from `RecipeBoxBasics/Program.cs`, Task 10 of Set 1.
3. Add `public IActionResult Index()` that returns
   `Content($"RecipeBox has {_recipes.Count} recipes.")`.
4. Open `/Recipes` in the browser.

**Acceptance criteria**

- [ ] `/Recipes` shows `RecipeBox has 5 recipes.`
- [ ] DevTools shows **GET · 200 · text/plain**.
- [ ] `/Recipes/Index` shows the same thing, and `/recipes` (lower case) does too.

**Hints**

- `using RecipeBox.Models;` at the top, or `Recipe` is an unknown type.
- The URL is `/Recipes`, not `/RecipesController`. The framework strips the word
  `Controller`.
- `Index` is the default action of the default route. That is why `/Recipes` alone works.
- `static` keeps one list for the whole app. Without it, every request would build a
  fresh list.

**Solution**

```csharp
using Microsoft.AspNetCore.Mvc;
using RecipeBox.Models;

namespace RecipeBox.Controllers;

public class RecipesController : Controller
{
    // In-memory for now. Set 4 moves it into a database.
    private static readonly List<Recipe> _recipes = new()
    {
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
    };

    // GET /Recipes
    public IActionResult Index()
    {
        return Content($"RecipeBox has {_recipes.Count} recipes.");
    }
}
```

---

### Task 2 — Return the catalog as plain text and as JSON · 12 min

**Goal:** the same data in two formats, and the `Content-Type` header that tells the
browser which one it got.

**Steps**

1. Change `Index` to print one line per recipe, `id: title`, under a count line.
2. Add an action `All()` that returns `Json(_recipes)`.
3. Open both and compare them in DevTools.

**Acceptance criteria**

- [ ] `/Recipes` shows `5 recipe(s)` and then five lines, from `1: Pancakes` to
      `5: Chicken Curry`.
- [ ] `/Recipes/All` shows a JSON array of five objects, each with its `ingredients`.
- [ ] DevTools shows `text/plain` for the first and `application/json` for the second.
- [ ] The JSON property names start with a lower-case letter: `title`, not `Title`.

**Hints**

- `Select` turns each recipe into a string, and `string.Join("\n", lines)` glues them
  with line breaks. Both are from Set 1.
- Do not name the action `Json`: the controller already has a method called `Json`, the
  one we are calling.
- The lower-case names are the JSON convention. The framework converts them for us.
- Firefox and Chrome both have a "pretty print" view for JSON. The raw response is the
  same.

**Solution**

```csharp
// GET /Recipes
public IActionResult Index()
{
    var lines = _recipes.Select(r => $"{r.Id}: {r.Title}");
    return Content($"{_recipes.Count} recipe(s)\n" + string.Join("\n", lines));
}

// GET /Recipes/All
public IActionResult All() => Json(_recipes);
```

```
$ curl -i http://localhost:5267/Recipes/All
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

[{"id":1,"title":"Pancakes","cuisine":"American","prepMinutes":10,...
```

---

### Task 3 — Add a `Details(int id)` action · 10 min

**Goal:** model binding: a value from the URL arrives as a method parameter.

**Steps**

1. Add `Details(int id)`. Find the recipe with `_recipes.First(r => r.Id == id)`.
2. Return three lines: title and cuisine, total minutes and servings, and the
   ingredients on one line.
3. Open `/Recipes/Details/2`, then `/Recipes/Details/99`.

**Acceptance criteria**

- [ ] `/Recipes/Details/2` shows `Spaghetti Carbonara (Italian)`,
      `25 min, serves 2` and its five ingredients.
- [ ] `/Recipes/Details/99` returns **500**. Write down the exception name from the
      error page. Task 4 fixes it.

**Hints**

- The default route ends in `{id?}`. The parameter is called `id`, so the value lands in
  it. Rename the parameter to `recipeId` and watch it stay `0`.
- `r.TotalMinutes()` is the method from Set 1.
- The 500 page is the *developer* exception page. Only we see it, because we run in
  Development.

**Solution**

```csharp
// GET /Recipes/Details/2
public IActionResult Details(int id)
{
    var recipe = _recipes.First(r => r.Id == id);

    return Content($"{recipe.Title} ({recipe.Cuisine})\n" +
                   $"{recipe.TotalMinutes()} min, serves {recipe.Servings}\n" +
                   $"Ingredients: {string.Join(", ", recipe.Ingredients)}");
}
```

```
/Recipes/Details/99  →  500
InvalidOperationException: Sequence contains no matching element
```

---

### Task 4 — Return 404 for an unknown recipe · 8 min

**Goal:** a missing recipe is not a crash. It is the client asking for something that is
not there.

**Steps**

1. Replace `First` with `FirstOrDefault`.
2. If the result is `null`, return `NotFound($"No recipe with id {id}.")`.
3. Reload `/Recipes/Details/99`.

**Acceptance criteria**

- [ ] `/Recipes/Details/99` returns **404** with the text `No recipe with id 99.`
- [ ] `/Recipes/Details/2` still returns 200.
- [ ] Nothing in the action can throw for any whole-number id.

**Hints**

- This is Set 1 Task 11's Banitsa question, now with an HTTP status code attached.
- `is null` reads better than `== null` and does the same thing here.
- `NotFound()` with no argument also works; the message just makes Task 8 easier to see.

**Solution**

```csharp
// GET /Recipes/Details/2
public IActionResult Details(int id)
{
    var recipe = _recipes.FirstOrDefault(r => r.Id == id);
    if (recipe is null)
        return NotFound($"No recipe with id {id}.");

    return Content($"{recipe.Title} ({recipe.Cuisine})\n" +
                   $"{recipe.TotalMinutes()} min, serves {recipe.Servings}\n" +
                   $"Ingredients: {string.Join(", ", recipe.Ingredients)}");
}
```

---

### Task 5 — Filter by cuisine with a query-string parameter · 12 min

**Goal:** an optional value from the query string, bound by name.

**Steps**

1. Give `Index` a parameter `string? cuisine`.
2. If it is empty, list every recipe. Otherwise list only the recipes with exactly that
   cuisine.
3. Try `/Recipes`, `/Recipes?cuisine=Italian` and `/Recipes?cuisine=French`.

**Acceptance criteria**

- [ ] `/Recipes` still lists all five, under `5 recipe(s)`.
- [ ] `?cuisine=Italian` lists Spaghetti Carbonara and Margherita Pizza, under
      `2 recipe(s)`.
- [ ] `?cuisine=French` shows `0 recipe(s)` with status **200**, not 404: an empty
      search result is still a successful answer.
- [ ] `?cuisine=italian` (lower case) also shows 0. Note it; Task 9 handles case.

**Hints**

- `?cuisine=Italian` fills `string? cuisine` because the **names** match.
- `string.IsNullOrWhiteSpace(cuisine)` is true for a missing, empty or blank value.
- `condition ? a : b` picks one of two values. Both sides must be a `List<Recipe>`, so
  end the `Where` with `.ToList()`.

**Solution**

```csharp
// GET /Recipes            GET /Recipes?cuisine=Italian
public IActionResult Index(string? cuisine)
{
    var matches = string.IsNullOrWhiteSpace(cuisine)
        ? _recipes
        : _recipes.Where(r => r.Cuisine == cuisine).ToList();

    var lines = matches.Select(r => $"{r.Id}: {r.Title}");
    return Content($"{matches.Count} recipe(s)\n" + string.Join("\n", lines));
}
```

---

### Task 6 — Add an `About` action that returns `Content` · 5 min · *optional*

**Goal:** the smallest possible action, and one more URL for the routing tasks.

**Steps**

1. Add `About()` that returns one line describing the app, including the recipe count.
2. Open `/Recipes/About`.

**Acceptance criteria**

- [ ] `/Recipes/About` shows `RecipeBox: a small recipe catalogue with 5 recipes.`
- [ ] The count comes from `_recipes.Count`, not a typed 5.

**Hints**

- A one-line action can use `=>`, as in Set 1 Task 8.

**Solution**

```csharp
// GET /Recipes/About
public IActionResult About()
    => Content($"RecipeBox: a small recipe catalogue with {_recipes.Count} recipes.");
```

---

## Part B — Routing

### Task 7 — Give each action a clean URL with attribute routing · 20 min

**Goal:** URLs designed for people: lower case, no action names, the id as a path segment.

**Target**

| Action | Before | After |
|---|---|---|
| `Index` | `/Recipes` | `/recipes` |
| `All` | `/Recipes/All` | `/recipes/json` |
| `Details` | `/Recipes/Details/2` | `/recipes/2` |
| `About` | `/Recipes/About` | `/about` |

**Steps**

1. Put `[Route("recipes")]` on the controller class.
2. Put `[HttpGet("...")]` on each action, so the four URLs in the *After* column work.
3. Try every *After* URL, then every *Before* URL.

**Acceptance criteria**

- [ ] All four *After* URLs return 200. `/recipes?cuisine=Italian` still filters.
- [ ] `/Recipes/Details/2` now returns **404**. Write one sentence on why.
- [ ] `/about` works with no `recipes` in front of it.

**Hints**

- `[Route]` on the class is a prefix; `[HttpGet("...")]` on the action completes it.
  `[HttpGet("")]` means "the prefix itself".
- A leading `/` escapes the prefix: `[HttpGet("/about")]`.
- Once a controller has attribute routes, the default route no longer reaches it. That is
  the 404.
- The query string is not part of the route template. `Index(string? cuisine)` needs no
  change.

**Solution**

```csharp
[Route("recipes")]
public class RecipesController : Controller
{
    // ... _recipes unchanged ...

    [HttpGet("")]            // GET /recipes   and   /recipes?cuisine=Italian
    public IActionResult Index(string? cuisine) { ... }

    [HttpGet("json")]        // GET /recipes/json
    public IActionResult All() => Json(_recipes);

    [HttpGet("{id}")]        // GET /recipes/2
    public IActionResult Details(int id) { ... }

    [HttpGet("/about")]      // GET /about
    public IActionResult About() => ...;
}
```

`/Recipes/Details/2` is a 404 because the controller is attribute-routed now: the default
route `{controller}/{action}/{id?}` no longer reaches it.

---

### Task 8 — Add an `int` route constraint to the id · 10 min

**Goal:** a route that only matches the kind of value it can use.

**Steps**

1. Before changing anything, open `/recipes/banana`. Read the message.
2. Change `[HttpGet("{id}")]` to `[HttpGet("{id:int}")]`.
3. Open `/recipes/banana` again, then `/recipes/2` and `/recipes/json`.

**Acceptance criteria**

- [ ] Before: `/recipes/banana` → 404 with `No recipe with id 0.` Our action ran, with
      a made-up id.
- [ ] After: `/recipes/banana` → 404 with an **empty** body. No route matched, so our
      action never ran.
- [ ] `/recipes/2` and `/recipes/json` still work.

**Hints**

- `banana` cannot become an `int`, so model binding leaves `id` at its default, `0`.
- The empty 404 is the framework's own: nothing matched the URL.
- `/recipes/json` worked before the constraint too. A literal segment beats a parameter.
- A constraint is a routing filter, not input validation. Validation is Set 5.

**Solution**

```csharp
[HttpGet("{id:int}")]    // GET /recipes/2 — digits only
public IActionResult Details(int id) { ... }
```

```
before   /recipes/banana   404   No recipe with id 0.
after    /recipes/banana   404   (empty body: no route matched)
```

---

### Task 9 — Add a by-cuisine route · 15 min

**Goal:** a value in the path rather than the query string, and a URL that does not care
about case.

**Steps**

1. Add `ByCuisine(string name)` at `/recipes/cuisine/{name}`.
2. Return `Italian: Spaghetti Carbonara, Margherita Pizza`-style text, comparing cuisines
   without caring about upper or lower case.
3. If there are no matches, return 404 with `No {name} recipes.`

**Acceptance criteria**

- [ ] `/recipes/cuisine/italian`, `/recipes/cuisine/Italian` and
      `/recipes/cuisine/ITALIAN` all list the same two recipes.
- [ ] `/recipes/cuisine/french` → 404 with `No french recipes.`
- [ ] `/recipes/cuisine` with no name → 404.

**Hints**

- `ToLower()` on both sides makes the comparison case-blind:
  `r.Cuisine.ToLower() == name.ToLower()`.
- Why 404 here but 200 in Task 5? A path names a *thing*. `/recipes/cuisine/french`
  claims there is a French section, and there is not. A query string is a *search*, and
  an empty result is a valid answer.
- `{name}` with no `?` is required, so the route does not match without it.

**Solution**

```csharp
// GET /recipes/cuisine/italian
[HttpGet("cuisine/{name}")]
public IActionResult ByCuisine(string name)
{
    var titles = _recipes
        .Where(r => r.Cuisine.ToLower() == name.ToLower())
        .Select(r => r.Title)
        .ToList();

    if (titles.Count == 0)
        return NotFound($"No {name} recipes.");

    return Content($"{name}: {string.Join(", ", titles)}");
}
```

---

### Task 10 — Generate links with `Url.Action` · 18 min

**Goal:** ask the framework for a URL instead of typing it, so a route change cannot break
a link.

**Steps**

1. In `Index`, add each recipe's URL to its line: `1: Pancakes -> /recipes/1`. Build the
   URL with `Url.Action("Details", new { id = r.Id })`.
2. Change the class attribute to `[Route("dishes")]`. Reload `/dishes`.
3. Change it back to `[Route("recipes")]`.

**Acceptance criteria**

- [ ] `/recipes` shows `1: Pancakes -> /recipes/1` and so on, for all five.
- [ ] With `[Route("dishes")]`, `/dishes` shows `-> /dishes/1`. No other edit was needed.
- [ ] The string `"/recipes/"` appears nowhere in `RecipesController.cs`.

**Hints**

- `new { id = r.Id }` is an anonymous object. Its property names must match the route
  parameter names: `id`, as in `{id:int}`.
- `Url.Action` reads the routes we defined, so it knows the template is
  `recipes/{id:int}`.
- In Set 3 the same idea appears in views as `asp-action` and `asp-route-id`.

**Solution**

```csharp
// GET /recipes            GET /recipes?cuisine=Italian
[HttpGet("")]
public IActionResult Index(string? cuisine)
{
    var matches = string.IsNullOrWhiteSpace(cuisine)
        ? _recipes
        : _recipes.Where(r => r.Cuisine == cuisine).ToList();

    var lines = matches.Select(r => $"{r.Id}: {r.Title} -> {Url.Action("Details", new { id = r.Id })}");
    return Content($"{matches.Count} recipe(s)\n" + string.Join("\n", lines));
}
```

```
$ curl http://localhost:5267/recipes
5 recipe(s)
1: Pancakes -> /recipes/1
2: Spaghetti Carbonara -> /recipes/2
3: Shopska Salad -> /recipes/3
4: Margherita Pizza -> /recipes/4
5: Chicken Curry -> /recipes/5
```

**State after this set:** `RecipesController` answers at `/recipes`, `/recipes/json`,
`/recipes/{id:int}`, `/recipes/cuisine/{name}` and `/about`, in plain text or JSON. Set 3
replaces the text with HTML views.
