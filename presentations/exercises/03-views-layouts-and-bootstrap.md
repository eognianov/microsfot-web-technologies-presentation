# Set 3 — Views, layouts and Bootstrap

Covers decks **05 Templates: HTML, CSS and Razor** and **06 View Models and Bootstrap**.

Students have seen HTML and CSS, server-side templates, the view-lookup convention, Razor
(`@`, `@if`, `@foreach`), typed views with `@model`, the layout and `_ViewStart`, partial
views, `ViewData`, view models, and Bootstrap's grid and components. This set replaces the
plain-text answers of Set 2 with real pages. The data is still the in-memory list; Set 4
moves it into a database without touching a single view.

## Time budget · 2 hours

| # | Task | Part | Min |
|---|---|---|---|
| 1 | Build an `Index` view with `@model` and `@foreach` | A · Razor | 15 |
| 2 | Build a `Details` view with the ingredients list | A · Razor | 12 |
| 3 | Add a "Quick" badge for recipes under 20 minutes | A · Razor | 8 |
| 4 | Customize the shared layout: navbar and footer | A · Razor | 12 |
| 5 | Extract a `_RecipeCard` partial view | A · Razor | 12 |
| 6 | Create a `RecipeListViewModel` | B · View models and Bootstrap | 18 |
| 7 | Set the page title with `ViewData` | B · View models and Bootstrap | 6 |
| 8 | Lay out a responsive card grid | B · View models and Bootstrap | 12 |
| 9 | Add Bootstrap components: badge, alert, button group | B · View models and Bootstrap | 15 |
| 10 | Check the layout at phone width | B · View models and Bootstrap | 10 |
| | **Total** | | **120** |

There are no optional tasks. If the group runs behind, the button group in Task 9 is the
part to drop.

## The project

```
RecipeBox/
├── Controllers/RecipesController.cs        ← Index, Details, ByCuisine, About return views
├── Models/
│   ├── Recipe.cs                           ← gains IsQuick(), Task 3
│   └── ViewModels/RecipeListViewModel.cs   ← new, Task 6
└── Views/
    ├── Recipes/
    │   ├── Index.cshtml                    ← new, Task 1
    │   ├── Details.cshtml                  ← new, Task 2
    │   └── About.cshtml                    ← new, Task 4
    └── Shared/
        ├── _Layout.cshtml                  ← edited, Task 4
        └── _RecipeCard.cshtml              ← new, Task 5
```

- **Starts from:** the end of Set 2. `RecipesController` answers in plain text at
  `/recipes`, `/recipes/{id:int}`, `/recipes/cuisine/{name}` and `/about`.
- **Ends with:** the same URLs answering with Bootstrap pages: a responsive card grid, a
  details page, a cuisine filter and a working layout on a phone.

`/recipes/json` stays as it is: it is data, not a page.

---

## Part A — Razor

### Task 1 — Build an `Index` view with `@model` and `@foreach` · 15 min

**Goal:** the controller hands a list to a view; the view turns it into HTML.

**Steps**

1. In `Index`, replace the `Content(...)` with `return View(matches);`.
2. Open `/recipes` and read the error. It lists the paths the framework searched.
3. Create `Views/Recipes/Index.cshtml`. Declare `@model List<RecipeBox.Models.Recipe>`.
4. Show a heading, the count, and a `<ul>` with one `<li>` per recipe: a link to its
   details page, its cuisine and its total minutes.

**Acceptance criteria**

- [ ] `/recipes` shows the template's navbar and footer around our list.
- [ ] Five list items; each title is a link to `/recipes/{id}`.
- [ ] `/recipes?cuisine=Italian` shows two items and says `2 recipe(s)`.
- [ ] View source contains no `@` and no `foreach`. Only HTML reaches the browser.

**Hints**

- The path in the error is the convention: `Views/` + controller name without
  `Controller` + action name + `.cshtml`.
- `@model` (lower case) declares the type, once. `Model` (upper case) is the object.
- `<a asp-action="Details" asp-route-id="@recipe.Id">` builds the URL from the route,
  like `Url.Action` in Set 2.
- `@recipe.TotalMinutes()` calls a method; Razor knows where the C# ends.

**Solution**

`Controllers/RecipesController.cs`

```csharp
[HttpGet("")]
public IActionResult Index(string? cuisine)
{
    var matches = string.IsNullOrWhiteSpace(cuisine)
        ? _recipes
        : _recipes.Where(r => r.Cuisine == cuisine).ToList();

    return View(matches);
}
```

`Views/Recipes/Index.cshtml`

```cshtml
@model List<RecipeBox.Models.Recipe>

<h1>Recipes</h1>
<p class="text-muted">@Model.Count recipe(s)</p>

<ul>
    @foreach (var recipe in Model)
    {
        <li>
            <a asp-action="Details" asp-route-id="@recipe.Id">@recipe.Title</a>
            · @recipe.Cuisine · @recipe.TotalMinutes() min
        </li>
    }
</ul>
```

---

### Task 2 — Build a `Details` view with the ingredients list · 12 min

**Goal:** a page for one object, with a list nested inside it.

**Steps**

1. In `Details`, return `View(recipe)` when it is found. Keep the `NotFound()`, but drop
   its message: a browser shows a page now, not our text.
2. Create `Views/Recipes/Details.cshtml` with `@model RecipeBox.Models.Recipe`.
3. Show the title, a line with cuisine, prep, cook, total and servings, and the
   ingredients as a `<ul>`.
4. Add a link back to the list.

**Acceptance criteria**

- [ ] `/recipes/3` shows **Shopska Salad**, `15 min prep + 0 min cook = 15 min`, and five
      ingredients from Tomatoes to Sirene.
- [ ] The back link goes to `/recipes`.
- [ ] `/recipes/99` still returns **404**.

**Hints**

- `Model.Ingredients` is a `List<string>`: a `@foreach` over it, one `<li>` each.
- `<a asp-action="Index">` with no id links to the list.
- Give the back link `class="btn btn-outline-secondary"`. Bootstrap is already loaded by
  the layout.

**Solution**

`Controllers/RecipesController.cs`

```csharp
[HttpGet("{id:int}")]
public IActionResult Details(int id)
{
    var recipe = _recipes.FirstOrDefault(r => r.Id == id);
    if (recipe is null)
        return NotFound();

    return View(recipe);
}
```

`Views/Recipes/Details.cshtml`

```cshtml
@model RecipeBox.Models.Recipe

<h1>@Model.Title</h1>
<p class="text-muted">
    @Model.Cuisine · @Model.PrepMinutes min prep + @Model.CookMinutes min cook
    = <strong>@Model.TotalMinutes() min</strong> · serves @Model.Servings
</p>

<h2 class="h4 mt-4">Ingredients</h2>
<ul>
    @foreach (var ingredient in Model.Ingredients)
    {
        <li>@ingredient</li>
    }
</ul>

<a asp-action="Index" class="btn btn-outline-secondary">Back to all recipes</a>
```

---

### Task 3 — Add a "Quick" badge for recipes under 20 minutes · 8 min

**Goal:** markup that appears only when a condition holds, with the rule written once.

**Steps**

1. Add `public bool IsQuick() => TotalMinutes() < 20;` to `Recipe`.
2. In `Index.cshtml`, after each title, show `<span class="badge bg-success">Quick</span>`
   only when the recipe is quick.
3. Do the same next to the title in `Details.cshtml`.

**Acceptance criteria**

- [ ] Only Shopska Salad (15 min) has the badge, on the list and on its details page.
- [ ] Change Pancakes' prep to 2 minutes: it gains the badge with no view edit. Then
      change it back.
- [ ] The number 20 appears once in the whole project, in `Recipe.cs`.

**Hints**

- `@if (recipe.IsQuick()) { ... }` — inside the braces we are back in HTML.
- Why a method on the model, not `TotalMinutes() < 20` in both views? Two copies of a
  rule drift apart. Views display; they do not decide.
- "Quick" means *under* 20, as in Set 1 Task 4: 20 itself is Medium.

**Solution**

`Models/Recipe.cs`

```csharp
public int TotalMinutes() => PrepMinutes + CookMinutes;
public bool IsQuick() => TotalMinutes() < 20;
```

`Views/Recipes/Index.cshtml`, inside the `<li>`

```cshtml
<a asp-action="Details" asp-route-id="@recipe.Id">@recipe.Title</a>
@if (recipe.IsQuick())
{
    <span class="badge bg-success">Quick</span>
}
· @recipe.Cuisine · @recipe.TotalMinutes() min
```

`Views/Recipes/Details.cshtml`

```cshtml
<h1>
    @Model.Title
    @if (Model.IsQuick())
    {
        <span class="badge bg-success fs-6 align-middle">Quick</span>
    }
</h1>
```

---

### Task 4 — Customize the shared layout: navbar and footer · 12 min

**Goal:** change the frame of every page by editing one file.

**Steps**

1. Open `Views/Shared/_Layout.cshtml`.
2. Replace the navbar with a dark one: the brand and a **Recipes** link go to the recipe
   list, an **About** link goes to `/about`. Remove the Home and Privacy links.
3. Replace the footer text with `© <current year> RecipeBox · a Microsoft Web
   Technologies exercise`.
4. Turn `About` into a view: return `View(_recipes.Count)` and create
   `Views/Recipes/About.cshtml` with `@model int`.

**Acceptance criteria**

- [ ] Every page, `/recipes`, `/recipes/2` and `/about`, has the dark navbar and the new
      footer.
- [ ] The year in the footer is computed, not typed.
- [ ] Clicking the brand from any page goes to `/recipes`.
- [ ] `/about` says the catalogue has 5 recipes.

**Hints**

- `navbar-dark bg-dark` on the `<nav>`; the template's links have `text-dark` classes
  that need to go, or they disappear on the dark background.
- Use `asp-controller="Recipes" asp-action="Index"`, not `href="/recipes"`. The layout
  is shared with `HomeController`'s pages.
- `@DateTime.Now.Year` is the year.
- The toggler button needs `data-bs-target` pointing at the id of the collapsing `<div>`.
  Task 10 tests it.

**Solution**

`Views/Shared/_Layout.cshtml`, the `<nav>` and the `<footer>`

```cshtml
<nav class="navbar navbar-expand-sm navbar-dark bg-dark mb-4">
    <div class="container">
        <a class="navbar-brand" asp-controller="Recipes" asp-action="Index">RecipeBox</a>
        <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#mainNav"
                aria-controls="mainNav" aria-expanded="false" aria-label="Toggle navigation">
            <span class="navbar-toggler-icon"></span>
        </button>
        <div class="collapse navbar-collapse" id="mainNav">
            <ul class="navbar-nav">
                <li class="nav-item">
                    <a class="nav-link" asp-controller="Recipes" asp-action="Index">Recipes</a>
                </li>
                <li class="nav-item">
                    <a class="nav-link" asp-controller="Recipes" asp-action="About">About</a>
                </li>
            </ul>
        </div>
    </div>
</nav>
```

```cshtml
<footer class="border-top footer text-muted">
    <div class="container">
        &copy; @DateTime.Now.Year RecipeBox · a Microsoft Web Technologies exercise
    </div>
</footer>
```

`Controllers/RecipesController.cs`

```csharp
[HttpGet("/about")]
public IActionResult About()
{
    return View(_recipes.Count);
}
```

`Views/Recipes/About.cshtml`

```cshtml
@model int

<h1>About RecipeBox</h1>
<p>A small recipe catalogue with @Model recipes, built during the Microsoft Web Technologies course.</p>
```

---

### Task 5 — Extract a `_RecipeCard` partial view · 12 min

**Goal:** one fragment of markup, written once, dropped into any page.

**Steps**

1. Create `Views/Shared/_RecipeCard.cshtml` with `@model RecipeBox.Models.Recipe`.
2. Inside it, build a Bootstrap card: the title as a link, the cuisine, the Quick badge,
   and `25 min · serves 4`.
3. In `Index.cshtml`, replace the `<ul>` with a loop that renders the partial once per
   recipe.

**Acceptance criteria**

- [ ] `/recipes` shows five cards, one below the other.
- [ ] `Index.cshtml` no longer contains the word `badge`. That markup lives in the
      partial only.
- [ ] Each card's title still links to `/recipes/{id}`.

**Hints**

- `<partial name="_RecipeCard" model="recipe" />` — the name has no `.cshtml`.
- The leading underscore means "a building block, not a page".
- `Shared/` is the fallback folder, so every controller's views can find it.
- A card is `card` › `card-body` › `card-title` and `card-text`. Add `mb-3` for space
  between them.
- Inside the partial, `asp-action` needs `asp-controller="Recipes"` too: the partial
  cannot know which controller is rendering it.

**Solution**

`Views/Shared/_RecipeCard.cshtml`

```cshtml
@model RecipeBox.Models.Recipe

<div class="card mb-3">
    <div class="card-body">
        <h5 class="card-title">
            <a asp-controller="Recipes" asp-action="Details" asp-route-id="@Model.Id"
               class="text-decoration-none">@Model.Title</a>
        </h5>
        <p class="card-subtitle mb-2 text-muted">
            @Model.Cuisine
            @if (Model.IsQuick())
            {
                <span class="badge bg-success">Quick</span>
            }
        </p>
        <p class="card-text text-muted">@Model.TotalMinutes() min · serves @Model.Servings</p>
    </div>
</div>
```

`Views/Recipes/Index.cshtml`

```cshtml
@model List<RecipeBox.Models.Recipe>

<h1>Recipes</h1>
<p class="text-muted">@Model.Count recipe(s)</p>

@foreach (var recipe in Model)
{
    <partial name="_RecipeCard" model="recipe" />
}
```

---

## Part B — View models and Bootstrap

### Task 6 — Create a `RecipeListViewModel` · 18 min

**Goal:** give the list page exactly what it needs, in one typed object, and let
`ByCuisine` reuse the same page.

**Steps**

1. Create `Models/ViewModels/RecipeListViewModel.cs` with: `Recipes` (`List<Recipe>`),
   `TotalCount` (`int`), `Cuisine` (`string?`) and a computed `IsFiltered`.
2. In the controller, add a private method `BuildList(string? cuisine)` that filters
   case-blind, sorts by title and fills the view model.
3. `Index` returns `View(BuildList(cuisine))`. `ByCuisine` returns
   `View("Index", BuildList(name))`.
4. Change `Index.cshtml` to the new model, and show `2 of 5 recipes`.

**Acceptance criteria**

- [ ] `/recipes` shows `5 of 5 recipes`; `/recipes/cuisine/italian` and
      `/recipes?cuisine=Italian` both show `2 of 5 recipes`.
- [ ] The cards are in alphabetical order: Chicken Curry first, Spaghetti Carbonara last.
- [ ] `/recipes/cuisine/french` shows `0 of 5 recipes` (Task 9 makes that friendlier).
- [ ] Typing `@Model.Recipe` (no `s`) in the view is a **compile error**, not a blank.

**Hints**

- `public bool IsFiltered => Cuisine is not null;` is computed every time it is read; it
  stores nothing.
- `View("Index", vm)` names the view, because the action name, `ByCuisine`, has no view
  of its own.
- `BuildList` returns a value and touches no request data, the same shape as the
  methods in Set 1 Task 8.
- Store the cuisine as the data spells it, `Italian`, not as the URL did:
  `matches.FirstOrDefault()?.Cuisine ?? cuisine`. Task 7 shows why.
- `using RecipeBox.Models.ViewModels;` at the top of the controller.

**Solution**

`Models/ViewModels/RecipeListViewModel.cs`

```csharp
namespace RecipeBox.Models.ViewModels;

public class RecipeListViewModel
{
    public List<Recipe> Recipes { get; set; } = new();
    public int TotalCount { get; set; }
    public string? Cuisine { get; set; }

    public bool IsFiltered => Cuisine is not null;
}
```

`Controllers/RecipesController.cs`

```csharp
[HttpGet("")]
public IActionResult Index(string? cuisine)
{
    return View(BuildList(cuisine));
}

[HttpGet("cuisine/{name}")]
public IActionResult ByCuisine(string name)
{
    return View("Index", BuildList(name));
}

private RecipeListViewModel BuildList(string? cuisine)
{
    var matches = string.IsNullOrWhiteSpace(cuisine)
        ? _recipes
        : _recipes.Where(r => r.Cuisine.ToLower() == cuisine.ToLower()).ToList();

    return new RecipeListViewModel
    {
        Recipes    = matches.OrderBy(r => r.Title).ToList(),
        TotalCount = _recipes.Count,
        Cuisine    = string.IsNullOrWhiteSpace(cuisine) ? null : matches.FirstOrDefault()?.Cuisine ?? cuisine
    };
}
```

`Views/Recipes/Index.cshtml`

```cshtml
@model RecipeBox.Models.ViewModels.RecipeListViewModel

<h1>Recipes</h1>
<p class="text-muted">@Model.Recipes.Count of @Model.TotalCount recipes</p>

@foreach (var recipe in Model.Recipes)
{
    <partial name="_RecipeCard" model="recipe" />
}
```

---

### Task 7 — Set the page title with `ViewData` · 6 min

**Goal:** the one legitimate use of `ViewData`: a small value the layout reads on every
page.

**Steps**

1. At the top of each view, in a `@{ }` block, set `ViewData["Title"]`.
2. Use `All recipes` or `Italian recipes` on the list, the recipe's title on details, and
   `About` on the about page.
3. Show the same title in the list page's `<h1>`.

**Acceptance criteria**

- [ ] The browser tab reads `All recipes - RecipeBox`, `Italian recipes - RecipeBox`,
      `Shopska Salad - RecipeBox` and `About - RecipeBox`.
- [ ] `/recipes/cuisine/italian` says `Italian`, with a capital I.
- [ ] No controller action sets `ViewData`.

**Hints**

- The layout already has `<title>@ViewData["Title"] - RecipeBox</title>`. We only fill
  the slot.
- `condition ? "a" : $"{x} b"` works inside the `@{ }` block.
- The capital I comes from Task 6's `matches.FirstOrDefault()?.Cuisine`.

**Solution**

`Views/Recipes/Index.cshtml`

```cshtml
@model RecipeBox.Models.ViewModels.RecipeListViewModel
@{
    ViewData["Title"] = Model.IsFiltered ? $"{Model.Cuisine} recipes" : "All recipes";
}

<h1>@ViewData["Title"]</h1>
```

`Views/Recipes/Details.cshtml` and `About.cshtml`

```cshtml
@{
    ViewData["Title"] = Model.Title;      // Details
}
@{
    ViewData["Title"] = "About";          // About
}
```

---

### Task 8 — Lay out a responsive card grid · 12 min

**Goal:** one column on a phone, two on a tablet, three on a laptop, with no media query
of our own.

**Steps**

1. In `Index.cshtml`, wrap the loop in `<div class="row g-4">`.
2. Wrap each partial in a column `<div>` with the right `col-*` classes.
3. In `_RecipeCard.cshtml`, change `mb-3` to `h-100`, so cards in a row are the same
   height.
4. Resize the browser window from narrow to wide.

**Acceptance criteria**

- [ ] Wide window: three cards per row, two rows.
- [ ] Below 992 px: two per row. Below 768 px: one per row.
- [ ] Cards in the same row have the same height.

**Hints**

- Read `col-12 col-md-6 col-lg-4` aloud: "full width; from medium up, half; from large
  up, a third". Breakpoints are minimum widths.
- `g-4` is the gutter between columns and rows. It replaces `mb-3`.
- The layout already gives us a `container`, so the order is container › row › col.

**Solution**

`Views/Recipes/Index.cshtml`

```cshtml
<div class="row g-4">
    @foreach (var recipe in Model.Recipes)
    {
        <div class="col-12 col-md-6 col-lg-4">
            <partial name="_RecipeCard" model="recipe" />
        </div>
    }
</div>
```

`Views/Shared/_RecipeCard.cshtml`

```cshtml
<div class="card h-100">
```

---

### Task 9 — Add Bootstrap components: badge, alert, button group · 15 min

**Goal:** three components, each with a job: label, warn, choose.

**Steps**

1. **Badge.** In the card, show the cuisine as `<span class="badge bg-secondary">`
   instead of plain text.
2. **Alert.** When the list is empty, show a yellow alert, `No recipes for French yet.`,
   with a link back to all recipes, instead of an empty grid.
3. **Button group.** Above the grid, show one button per cuisine plus **All**. Each one
   links to its `/recipes/cuisine/{name}` page. The current one is highlighted.

**Acceptance criteria**

- [ ] Every card has a grey cuisine badge; Shopska Salad also has the green Quick badge.
- [ ] `/recipes/cuisine/french` shows the alert, and its link goes to `/recipes`.
- [ ] The buttons read All, American, Bulgarian, Indian, Italian, in that order, and the
      list of cuisines comes from the data, not from typed strings.
- [ ] On `/recipes/cuisine/indian` the Indian button is highlighted; on `/recipes` All is.

**Hints**

- Add `List<string> AllCuisines` to the view model, filled with
  `_recipes.Select(r => r.Cuisine).Distinct().OrderBy(c => c).ToList()`.
- `@if (Model.Recipes.Count == 0) { alert } else { grid }`.
- A button group is `<div class="btn-group">` holding `<a class="btn ...">` links.
- `@(c == Model.Cuisine ? "active" : "")` inside the `class` attribute switches the
  highlight on. The parentheses tell Razor where the expression ends.
- `asp-route-name="@c.ToLower()"` fills the `{name}` segment: `/recipes/cuisine/indian`.

**Solution**

`Models/ViewModels/RecipeListViewModel.cs` and `BuildList`

```csharp
public List<string> AllCuisines { get; set; } = new();
```

```csharp
AllCuisines = _recipes.Select(r => r.Cuisine).Distinct().OrderBy(c => c).ToList()
```

`Views/Shared/_RecipeCard.cshtml`

```cshtml
<p class="card-subtitle mb-2">
    <span class="badge bg-secondary">@Model.Cuisine</span>
    @if (Model.IsQuick())
    {
        <span class="badge bg-success">Quick</span>
    }
</p>
```

`Views/Recipes/Index.cshtml`

```cshtml
<div class="btn-group mb-4" role="group" aria-label="Filter by cuisine">
    <a asp-action="Index"
       class="btn btn-outline-primary @(Model.IsFiltered ? "" : "active")">All</a>
    @foreach (var c in Model.AllCuisines)
    {
        <a asp-action="ByCuisine" asp-route-name="@c.ToLower()"
           class="btn btn-outline-primary @(c == Model.Cuisine ? "active" : "")">@c</a>
    }
</div>

@if (Model.Recipes.Count == 0)
{
    <div class="alert alert-warning">
        No recipes for <strong>@Model.Cuisine</strong> yet.
        <a asp-action="Index" class="alert-link">Show all recipes</a>
    </div>
}
else
{
    <div class="row g-4">
        @* ... the grid from Task 8 ... *@
    </div>
}
```

---

### Task 10 — Check the layout at phone width · 10 min

**Goal:** look at the site the way most visitors will, and fix what breaks.

**Steps**

1. Open DevTools and switch on the device toolbar (`Ctrl+Shift+M`, or `Cmd+Shift+M` on
   macOS). Pick a 375 px phone, such as iPhone SE.
2. Visit `/recipes`, `/recipes/3`, `/recipes/cuisine/french` and `/about`.
3. On each page, try to scroll sideways.
4. Open and close the navbar's menu button.
5. Fix what you find, then check again.

**Acceptance criteria**

- [ ] One card per row; nothing is cut off at the right edge.
- [ ] The navbar collapses into a menu button, and the button opens and closes the
      links.
- [ ] **No page scrolls sideways** at 375 px.
- [ ] Wider than 576 px, the menu button disappears and the links show inline.

**Hints**

- The template's `site.css` gives `.footer` `white-space: nowrap`: its text never wraps.
  Is our Task 4 footer short enough for 375 px?
- To find what is too wide, in the Console:
  `[...document.querySelectorAll('body *')].filter(e => e.getBoundingClientRect().right > innerWidth)`.
- `navbar-expand-sm` means "expanded from `sm`, 576 px, upward". Below that it
  collapses.
- If the menu button does nothing, the `data-bs-target` does not match the id of the
  collapsing `<div>`.

**Solution**

At 375 px the grid, cards, button group and navbar behave; the footer does not. Its text is
about 400 px wide and cannot wrap, so the whole page scrolls sideways by about 20 px.
Shorten it:

`Views/Shared/_Layout.cshtml`

```cshtml
<footer class="border-top footer text-muted">
    <div class="container">
        &copy; @DateTime.Now.Year RecipeBox
    </div>
</footer>
```

```
document.documentElement.scrollWidth   before: 396   after: 375
```

**State after this set:** every RecipeBox page is a Bootstrap view fed by a view model, and
works from 375 px up. The data is still the `static` list in the controller. Set 4 replaces
it with a database, and no view changes.
