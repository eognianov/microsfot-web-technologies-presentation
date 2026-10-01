# Set 5 — CRUD, debugging and deployment

Covers decks **09 Forms, CRUD and Validation** and **10 Debugging, Logging and
Deployment**.

Students have seen CRUD, the GET/POST action pair, tag helpers, server-side validation and
`ModelState`, POST/Redirect/GET, CSRF and the anti-forgery token, Edit and Delete,
scaffolding, the debugger, error pages per environment, `ILogger`, and `dotnet publish`.
This set makes RecipeBox writable, then treats it like software that ships: we debug it,
give it a production error page, log what it does and publish it.

## Time budget · 2 hours

| # | Task | Part | Min |
|---|---|---|---|
| 1 | Build a Create form with tag helpers | A · CRUD | 15 |
| 2 | Validate on the server and show the errors | A · CRUD | 12 |
| 3 | Redirect after POST (POST/Redirect/GET) | A · CRUD | 6 |
| 4 | Add the anti-forgery check and prove it works | A · CRUD | 6 |
| 5 | Build the Edit action pair | A · CRUD | 14 |
| 6 | Build Delete with a confirmation page | A · CRUD | 10 |
| 7 | Scaffold a controller and compare it with ours *(optional)* | A · CRUD | 12 |
| 8 | Use breakpoints to find a planted bug | B · Ship it | 12 |
| 9 | Add a custom error page for production | B · Ship it | 12 |
| 10 | Log recipe creation with `ILogger` | B · Ship it | 6 |
| 11 | Publish the app to a folder and run it | B · Ship it | 15 |
| | **Total** | | **120** |
| | **Without the optional task** | | **108** |

Task 7 is optional and works on a **copy** of the project, so nothing later depends on it.

## The project

```
RecipeBox/
├── Controllers/
│   ├── HomeController.cs             ← gains HttpStatus(code), Task 9
│   └── RecipesController.cs          ← Create, Edit, Delete, Scale; ILogger
├── Program.cs                        ← production error handling, Task 9
└── Views/
    ├── Home/HttpStatus.cshtml        ← new, Task 9
    └── Recipes/
        ├── Create.cshtml             ← new, Task 1
        ├── Edit.cshtml               ← new, Task 5
        ├── Delete.cshtml             ← new, Task 6
        └── _RecipeFields.cshtml      ← new, Task 5: the fields both forms share
../publish/                           ← created by Task 11, next to the project
```

- **Starts from:** the end of Set 4. RecipeBox reads five recipes from `recipebox.db`.
  The `Category` bonus is not required; if you did it, everything here still works.
- **Ends with:** a RecipeBox that creates, edits and deletes recipes safely, shows a
  friendly error page in production, logs what it changes, and runs from a published
  folder.

The form keeps `Ingredients` in a `<textarea>`, one per line. It is not a property the
form can bind directly, so the actions receive it as a separate `string? ingredientsText`
parameter and split it into the list.

---

## Part A — CRUD

### Task 1 — Build a Create form with tag helpers · 15 min

**Goal:** a form generated from the model, so the rules from Set 4 shape it without us
repeating them.

**Steps**

1. Add `Create()` at `GET /recipes/create`. It returns `View(new Recipe())`.
2. Create `Views/Recipes/Create.cshtml`. For each of `Title`, `Cuisine`, `PrepMinutes`,
   `CookMinutes`, `Servings` and `Difficulty`, add a `<label asp-for>`, an
   `<input asp-for>` and a `<span asp-validation-for>`.
3. Add a `<textarea name="ingredientsText">` and a **Save** button.
4. Add a **New recipe** button to the list page.
5. Open the form, view the page source, then click **Save**.

**Acceptance criteria**

- [ ] `/recipes/create` shows six labelled fields, a textarea and a button.
- [ ] The labels read `Prep minutes` and `Cook minutes`: the `[Display]` names.
- [ ] In the source, the `Title` input has `maxlength="100"` and the number fields have
      `type="number"`, though we typed neither.
- [ ] Clicking **Save** returns **405 Method Not Allowed**. There is no POST action yet.

**Hints**

- `asp-for="Title"` writes `name`, `id`, `value`, `type`, `maxlength` and the `data-val-*`
  rules, all from the model.
- `new Recipe()` rather than `View()`: the textarea reads `Model.Ingredients`, and a
  missing model would be `null`.
- `<form asp-action="Create" method="post">` points the form at the same URL.
- 405 is the server saying "I know this URL, but not with this method". The route exists
  only for GET.

**Solution**

`Controllers/RecipesController.cs`

```csharp
// GET /recipes/create
[HttpGet("create")]
public IActionResult Create() => View(new Recipe());
```

`Views/Recipes/Create.cshtml`

```cshtml
@model RecipeBox.Models.Recipe
@{
    ViewData["Title"] = "New recipe";
}

<h1>New recipe</h1>

<form asp-action="Create" method="post">
    <div asp-validation-summary="ModelOnly" class="text-danger"></div>

    <div class="mb-3">
        <label asp-for="Title" class="form-label"></label>
        <input asp-for="Title" class="form-control" />
        <span asp-validation-for="Title" class="text-danger"></span>
    </div>
    @* ... the same three lines for Cuisine, PrepMinutes, CookMinutes, Servings, Difficulty ... *@

    <div class="mb-3">
        <label for="ingredientsText" class="form-label">Ingredients, one per line</label>
        <textarea id="ingredientsText" name="ingredientsText" class="form-control" rows="5">@string.Join("\n", Model.Ingredients)</textarea>
    </div>

    <button type="submit" class="btn btn-primary">Save</button>
    <a asp-action="Index" class="btn btn-link">Cancel</a>
</form>
```

`Views/Recipes/Index.cshtml`, next to the heading

```cshtml
<a asp-action="Create" class="btn btn-primary">New recipe</a>
```

---

### Task 2 — Validate on the server and show the errors · 12 min

**Goal:** never trust what the browser sends. Check it on the server, and give the user's
input back with the errors next to it.

**Steps**

1. Add `Create` for `POST /recipes/create`. Bind a `Recipe` limited to the six form
   fields with `[Bind]`, plus a `string? ingredientsText`.
2. Split the text into `recipe.Ingredients`, one per line, skipping blank lines.
3. Add two rules of our own: at least one ingredient, and no duplicate title.
4. If `ModelState` is invalid, return the form with the recipe. Otherwise save it and,
   for now, `return View("Details", recipe)`.
5. Submit an empty form. Then a valid one: Banitsa, Bulgarian, 20, 40, 6, 2, with Filo
   pastry, Eggs, Sirene and Yoghurt.
6. Add the validation scripts and submit an empty form again.

**Acceptance criteria**

- [ ] Empty title → `The Title field is required.` next to the field.
- [ ] Servings `0` → `The field Servings must be between 1 and 20.`
- [ ] No ingredients → `Add at least one ingredient, one per line.` at the top.
- [ ] Title `Pancakes` → `A recipe with this title already exists.`
- [ ] After an error, everything typed is still in the form.
- [ ] Banitsa is saved and shows four ingredients. With the scripts added, empty fields
      are flagged **before** the form is sent.

**Hints**

- `[Bind("Title,Cuisine,PrepMinutes,CookMinutes,Servings,Difficulty")]` leaves out `Id`:
  a user who adds `Id=1` to the request cannot overwrite recipe 1.
- `text.Split('\n', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries)`
  splits, drops blank lines and trims spaces, including the `\r` Windows browsers send.
- `ModelState.AddModelError("", "...")` goes to the summary at the top;
  `AddModelError(nameof(Recipe.Title), "...")` goes next to the Title field.
- `return View(recipe)`, not `return View()`. Without the model, the user's typing is gone.
- Scripts: `@section Scripts { <partial name="_ValidationScriptsPartial" /> }` at the end of
  `Create.cshtml`. The server check stays: anyone can switch JavaScript off.

**Solution**

```csharp
// POST /recipes/create
[HttpPost("create")]
public async Task<IActionResult> Create(
    [Bind("Title,Cuisine,PrepMinutes,CookMinutes,Servings,Difficulty")] Recipe recipe,
    string? ingredientsText)
{
    recipe.Ingredients = ParseIngredients(ingredientsText);

    if (recipe.Ingredients.Count == 0)
        ModelState.AddModelError("", "Add at least one ingredient, one per line.");

    if (await _context.Recipes.AnyAsync(r => r.Title == recipe.Title))
        ModelState.AddModelError(nameof(Recipe.Title), "A recipe with this title already exists.");

    if (!ModelState.IsValid)
        return View(recipe);

    _context.Recipes.Add(recipe);
    await _context.SaveChangesAsync();

    return View("Details", recipe);      // temporary: Task 3 replaces it
}

private static List<string> ParseIngredients(string? text)
    => (text ?? "")
        .Split('\n', StringSplitOptions.RemoveEmptyEntries | StringSplitOptions.TrimEntries)
        .ToList();
```

`Views/Recipes/Create.cshtml`, at the end

```cshtml
@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

---

### Task 3 — Redirect after POST (POST/Redirect/GET) · 6 min

**Goal:** a page the user can reload, bookmark and share after saving.

**Steps**

1. Create a recipe, then press **F5** on the page Task 2 shows. Read what the browser asks.
   Confirm it.
2. Replace `return View("Details", recipe);` with a redirect to the new recipe's details
   page.
3. Create another recipe and press **F5** again.

**Acceptance criteria**

- [ ] Before: F5 asks to resubmit the form, and confirming sends the POST again. Our
      duplicate-title rule is the only thing that stops a second copy.
- [ ] After: saving lands on `/recipes/{new id}`, the address bar shows it, and F5 just
      reloads the page.
- [ ] DevTools shows **POST · 302**, then **GET · 200**.

**Hints**

- `RedirectToAction(nameof(Details), new { id = recipe.Id })`. `nameof` turns a renamed
  action into a compile error instead of a broken link.
- `recipe.Id` is set by the database during `SaveChangesAsync`, so it is ready to use
  right after.
- Tick **Preserve log** in the Network tab, or the 302 disappears from the list.

**Solution**

```csharp
_context.Recipes.Add(recipe);
await _context.SaveChangesAsync();

return RedirectToAction(nameof(Details), new { id = recipe.Id });
```

```
POST /recipes/create    302 Found    Location: /recipes/6
GET  /recipes/6         200 OK
```

---

### Task 4 — Add the anti-forgery check and prove it works · 6 min

**Goal:** see the token the form already carries, and make the server insist on it.

**Steps**

1. View the source of `/recipes/create` and find `__RequestVerificationToken`.
2. In DevTools **Elements**, delete that hidden `<input>`, fill in a valid recipe and save.
3. Add `[ValidateAntiForgeryToken]` to the POST action.
4. Repeat step 2.

**Acceptance criteria**

- [ ] The source has `<input name="__RequestVerificationToken" type="hidden" value="CfDJ8...">`.
- [ ] Without the attribute, the form with the token removed still saves. The server was
      not checking.
- [ ] With it, the same submission returns **400 Bad Request** and nothing is saved.
- [ ] A normal submission, with the token, still saves.

**Hints**

- A `<form>` with `method="post"` and a tag helper adds the token by itself. Checking it
  is our job.
- The rule for the rest of the course: **every** POST that changes data gets
  `[ValidateAntiForgeryToken]`.
- 400 means the client sent a bad request. Here, one that could have come from another
  site.

**Solution**

```csharp
// POST /recipes/create
[HttpPost("create")]
[ValidateAntiForgeryToken]
public async Task<IActionResult> Create(
    [Bind("Title,Cuisine,PrepMinutes,CookMinutes,Servings,Difficulty")] Recipe recipe,
    string? ingredientsText)
```

```
token removed, no attribute     →  302, saved
token removed, with attribute   →  400 Bad Request, not saved
```

---

### Task 5 — Build the Edit action pair · 14 min

**Goal:** the same two-request dance as Create, starting from a filled-in form.

**Steps**

1. Move the fields of `Create.cshtml` into a partial, `Views/Recipes/_RecipeFields.cshtml`,
   and use it from `Create.cshtml`.
2. Add `Edit(int id)` at `GET /recipes/{id}/edit`: find the recipe or return 404, and
   show it.
3. Add `Edit` at `POST /recipes/{id}/edit`. Bind the same fields **plus** `Id`, check
   that the URL's id matches the form's, parse the ingredients, validate, update, save
   and redirect to details.
4. Create `Edit.cshtml` with a hidden `Id` field and the partial.
5. Add an **Edit** button to the details page.

**Acceptance criteria**

- [ ] `/recipes/6/edit` shows the form filled in, ingredients one per line.
- [ ] Changing the prep time to 25 and saving shows `65 min` on the details page.
- [ ] An empty title on Edit shows the same error as on Create.
- [ ] `/recipes/99/edit` → 404.
- [ ] `Create.cshtml` and `Edit.cshtml` contain no `<input>` of their own except the
      hidden `Id`.

**Hints**

- `<partial name="_RecipeFields" model="Model" />` — the partial lives next to the views
  that use it, in `Views/Recipes/`.
- `FindAsync(id)` looks a row up by primary key.
- `<input type="hidden" asp-for="Id" />` sends the id back in the form body, so the form
  says which recipe it edits. Model binding would also find `Id` in the URL, because the
  route parameter has the same name; the hidden field keeps the form self-contained.
- `id != recipe.Id` returns 400 when the URL and the form disagree about which recipe
  this is.
- `_context.Update(recipe)` marks every column as changed; `SaveChangesAsync` writes them.
- `DbUpdateConcurrencyException` means the row vanished while the user was typing: return
  404 if it is gone, otherwise re-throw.

**Solution**

```csharp
// GET /recipes/2/edit
[HttpGet("{id:int}/edit")]
public async Task<IActionResult> Edit(int id)
{
    var recipe = await _context.Recipes.FindAsync(id);
    if (recipe is null)
        return NotFound();

    return View(recipe);
}

// POST /recipes/2/edit
[HttpPost("{id:int}/edit")]
[ValidateAntiForgeryToken]
public async Task<IActionResult> Edit(
    int id,
    [Bind("Id,Title,Cuisine,PrepMinutes,CookMinutes,Servings,Difficulty")] Recipe recipe,
    string? ingredientsText)
{
    if (id != recipe.Id)
        return BadRequest();

    recipe.Ingredients = ParseIngredients(ingredientsText);

    if (recipe.Ingredients.Count == 0)
        ModelState.AddModelError("", "Add at least one ingredient, one per line.");

    if (!ModelState.IsValid)
        return View(recipe);

    try
    {
        _context.Update(recipe);
        await _context.SaveChangesAsync();
    }
    catch (DbUpdateConcurrencyException)
    {
        if (!await _context.Recipes.AnyAsync(r => r.Id == id))
            return NotFound();
        throw;
    }

    return RedirectToAction(nameof(Details), new { id });
}
```

`Views/Recipes/Edit.cshtml`

```cshtml
@model RecipeBox.Models.Recipe
@{
    ViewData["Title"] = $"Edit {Model.Title}";
}

<h1>Edit recipe</h1>

<form asp-action="Edit" asp-route-id="@Model.Id" method="post">
    <input type="hidden" asp-for="Id" />
    <partial name="_RecipeFields" model="Model" />
    <button type="submit" class="btn btn-primary">Save changes</button>
    <a asp-action="Details" asp-route-id="@Model.Id" class="btn btn-link">Cancel</a>
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

`Views/Recipes/_RecipeFields.cshtml` holds the validation summary, the six field groups and
the textarea, moved unchanged from Task 1. `Create.cshtml` becomes:

```cshtml
<form asp-action="Create" method="post">
    <partial name="_RecipeFields" model="Model" />
    <button type="submit" class="btn btn-primary">Save</button>
    <a asp-action="Index" class="btn btn-link">Cancel</a>
</form>
```

---

### Task 6 — Build Delete with a confirmation page · 10 min

**Goal:** deleting is two requests: a GET that asks, and a POST that does it.

**Steps**

1. Add `Delete(int id)` at `GET /recipes/{id}/delete`, showing a confirmation page.
2. Add `DeleteConfirmed(int id)` at `POST /recipes/{id}/delete`, answering to the action
   name `Delete`. Remove the recipe, save, redirect to the list.
3. Create `Delete.cshtml`: a red alert naming the recipe, and a form with a red **Delete**
   button.
4. Add a **Delete** button to the details page.

**Acceptance criteria**

- [ ] `/recipes/6/delete` names the recipe and its ingredient count, and deletes
      **nothing**.
- [ ] Clicking **Delete** removes it and lands on `/recipes`; `/recipes/6` is now 404.
- [ ] The POST has `[ValidateAntiForgeryToken]`.
- [ ] Nothing in the app deletes on a GET.

**Hints**

- C# cannot have two `Delete(int id)` methods. `[ActionName("Delete")]` lets
  `DeleteConfirmed` answer to the same URL.
- `btn-danger` for the button that destroys data, and only for that.
- A crawler follows every link it finds. A delete link would empty the database; a form
  POST it never submits.

**Solution**

```csharp
// GET /recipes/2/delete
[HttpGet("{id:int}/delete")]
public async Task<IActionResult> Delete(int id)
{
    var recipe = await _context.Recipes.AsNoTracking().FirstOrDefaultAsync(r => r.Id == id);
    if (recipe is null)
        return NotFound();

    return View(recipe);
}

// POST /recipes/2/delete
[HttpPost("{id:int}/delete"), ActionName("Delete")]
[ValidateAntiForgeryToken]
public async Task<IActionResult> DeleteConfirmed(int id)
{
    var recipe = await _context.Recipes.FindAsync(id);
    if (recipe is not null)
    {
        _context.Recipes.Remove(recipe);
        await _context.SaveChangesAsync();
    }

    return RedirectToAction(nameof(Index));
}
```

`Views/Recipes/Delete.cshtml`

```cshtml
@model RecipeBox.Models.Recipe
@{
    ViewData["Title"] = $"Delete {Model.Title}";
}

<h1>Delete recipe</h1>

<div class="alert alert-danger">
    Delete <strong>@Model.Title</strong> and its @Model.Ingredients.Count ingredients?
    This cannot be undone.
</div>

<form asp-action="Delete" asp-route-id="@Model.Id" method="post">
    <button type="submit" class="btn btn-danger">Delete</button>
    <a asp-action="Details" asp-route-id="@Model.Id" class="btn btn-link">Cancel</a>
</form>
```

`Views/Recipes/Details.cshtml`, at the bottom

```cshtml
<a asp-action="Edit" asp-route-id="@Model.Id" class="btn btn-primary">Edit</a>
<a asp-action="Delete" asp-route-id="@Model.Id" class="btn btn-outline-danger">Delete</a>
<a asp-action="Index" class="btn btn-link">Back to all recipes</a>
```

---

### Task 7 — Scaffold a controller and compare it with ours · 12 min · *optional*

**Goal:** read generated code we could not have read two sessions ago, and find where it
falls short.

**Steps**

1. Copy the whole `RecipeBox` folder to `RecipeBoxScaffold`, and work in the copy.
2. Install the generator and its packages:

   ```bash
   dotnet tool install --global dotnet-aspnet-codegenerator
   dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design
   dotnet add package Microsoft.EntityFrameworkCore.Tools
   ```

3. Generate a controller with views for `Recipe`:

   ```bash
   dotnet aspnet-codegenerator controller -name RecipesScaffoldController \
       -m Recipe -dc RecipeBoxContext -dbProvider sqlite \
       --relativeFolderPath Controllers --useDefaultLayout --referenceScriptLibraries
   ```

4. Open `RecipesScaffoldController.cs` next to our `RecipesController.cs`.

**Acceptance criteria**

- [ ] The generator made one controller and five views in `Views/RecipesScaffold/`.
- [ ] You can name, in the generated code: the action pairs, `ModelState.IsValid`,
      `[ValidateAntiForgeryToken]`, `RedirectToAction`, `NotFound()` and `[Bind]`.
- [ ] You found three differences that make ours better for RecipeBox.

**Hints**

- In Visual Studio: right-click `Controllers` → Add → New Scaffolded Item → MVC
  Controller with views, using Entity Framework.
- Without `-dbProvider sqlite`, the generator asks for the SQL Server package.
- Look at the `[Bind]` list, the `Ingredients` field in `Create.cshtml`, where Create
  redirects, and which rules it checks.

**Solution**

```csharp
// generated
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> Create([Bind("Id,Title,Cuisine,PrepMinutes,CookMinutes,Servings,Ingredients,Difficulty")] Recipe recipe)
{
    if (ModelState.IsValid)
    {
        _context.Add(recipe);
        await _context.SaveChangesAsync();
        return RedirectToAction(nameof(Index));
    }
    return View(recipe);
}
```

| | Scaffolded | Ours |
|---|---|---|
| `[Bind]` | includes `Id`: a client can choose the key | the six form fields only |
| Ingredients | one `<input asp-for="Ingredients">`, which cannot hold a list | a textarea, one per line, parsed |
| Own rules | none | at least one ingredient, no duplicate title |
| After Create | the list, so the new recipe has to be found | its own details page |
| URLs | `/RecipesScaffold/Edit/5` | `/recipes/5/edit` |

Scaffolding is a fast first draft. Delete the copy when done.

---

## Part B — Ship it

### Task 8 — Use breakpoints to find a planted bug · 12 min

**Goal:** find a wrong value by watching the program run, not by guessing.

**Steps**

1. Paste this action into `RecipesController`. It contains one bug.

   ```csharp
   // GET /recipes/2/scale/3
   [HttpGet("{id:int}/scale/{servings:int}")]
   public async Task<IActionResult> Scale(int id, int servings)
   {
       var recipe = await _context.Recipes.FindAsync(id);
       if (recipe is null)
           return NotFound();

       double factor = servings / recipe.Servings;
       return Content($"{recipe.Title} for {servings}: multiply every amount by {factor}");
   }
   ```

2. Open `/recipes/2/scale/3`. Carbonara serves 2, so the answer should be 1.5.
3. Start the app under the debugger (`F5` in VS Code with C# Dev Kit, or in Visual
   Studio). Set a breakpoint on the `double factor` line.
4. Request the URL again. When it stops, read **Locals**. Step over the line and read
   them again.
5. Fix the bug, and check `/recipes/1/scale/2`.

**Acceptance criteria**

- [ ] Before: `/recipes/2/scale/3` says `by 1`, and `/recipes/1/scale/2` says `by 0`.
- [ ] Paused on the line, Locals shows `servings = 3` and `recipe.Servings = 2`, both
      correct. After the step, `factor = 1`. The bug is on this line.
- [ ] After the fix: `by 1.5` and `by 0.5`.
- [ ] One sentence naming the bug. It has been seen before.

**Hints**

- Debugging is a method: reproduce, find the **first** wrong value, explain it, fix,
  verify.
- Expand `recipe` in Locals to see its properties. Or add `recipe.Servings` to **Watch**.
- `3 / 2` with two `int`s… Set 1 Task 3.
- Depending on your system's language settings, the fixed output may say `1,5`. Same
  number.

**Solution**

```csharp
double factor = (double)servings / recipe.Servings;
```

The bug is **integer division**: `int / int` throws the remainder away *before* the result
is stored in a `double`. `3 / 2` is `1`, `2 / 4` is `0`. Converting one side first makes
the division a `double` division.

```
/recipes/2/scale/3   Spaghetti Carbonara for 3: multiply every amount by 1.5
/recipes/1/scale/2   Pancakes for 2: multiply every amount by 0.5
```

---

### Task 9 — Add a custom error page for production · 12 min

**Goal:** developers see the stack trace; visitors see a friendly page. Both get the right
status code.

**Steps**

1. Add a temporary action that throws:
   `[HttpGet("boom")] public IActionResult Boom() => throw new InvalidOperationException("boom");`
2. Open `/recipes/boom` and `/recipes/99` in Development. Note what each shows.
3. Add `HttpStatus(int code)` to `HomeController` at `/error/{code:int}`, with a view that
   shows the code and a friendly sentence for 404 and for everything else.
4. In `Program.cs`, inside the `!IsDevelopment()` block, send exceptions to `/error/500`
   and status codes to `/error/{0}`.
5. Run in Production: `dotnet run -e ASPNETCORE_ENVIRONMENT=Production`. Open both URLs
   again, plus `/nope`.
6. Delete `Boom`.

**Acceptance criteria**

- [ ] Development: `/recipes/boom` shows the developer exception page with the stack
      trace; `/recipes/99` shows the browser's own, empty 404.
- [ ] Production: `/recipes/boom` shows our page with **500**; `/recipes/99` and `/nope`
      show it with **404**.
- [ ] DevTools still reports **500** and **404**. The page is friendly; the status code is
      honest.
- [ ] The console still logs the `boom` exception in Production.

**Hints**

- `ASPNETCORE_ENVIRONMENT=Production dotnet run` does **not** work: the `http` profile in
  `Properties/launchSettings.json` sets `Development` and wins. `dotnet run -e` overrides
  the profile, and is the same command in every shell.
- The console's `Hosting environment:` line says which one you got.
- Do not name the action `StatusCode`: the controller already has a method by that name.
- `UseStatusCodePagesWithReExecute` runs our page **inside** the original request, so the
  404 stays a 404. A redirect would turn it into a 302 and then a 200.

**Solution**

`Program.cs`

```csharp
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/error/500");
    app.UseStatusCodePagesWithReExecute("/error/{0}");
    app.UseHsts();
}
```

`Controllers/HomeController.cs`

```csharp
// GET /error/404 — reached by the error-handling middleware, not by links
[Route("error/{code:int}")]
public IActionResult HttpStatus(int code)
{
    return View(code);
}
```

`Views/Home/HttpStatus.cshtml`

```cshtml
@model int
@{
    ViewData["Title"] = Model == 404 ? "Not found" : "Something went wrong";
}

<div class="text-center py-5">
    <p class="display-1 fw-bold text-secondary">@Model</p>
    @if (Model == 404)
    {
        <h1 class="h3">We could not find that page.</h1>
        <p class="text-muted">The recipe may have been deleted, or the link is wrong.</p>
    }
    else
    {
        <h1 class="h3">Something went wrong on our side.</h1>
        <p class="text-muted">It has been logged. Please try again in a moment.</p>
    }
    <a asp-controller="Recipes" asp-action="Index" class="btn btn-primary">Back to the recipes</a>
</div>
```

```
$ dotnet run -e ASPNETCORE_ENVIRONMENT=Production
      Hosting environment: Production

/recipes/boom   500   Something went wrong on our side.
/recipes/99     404   We could not find that page.
/nope           404   We could not find that page.
```

---

### Task 10 — Log recipe creation with `ILogger` · 6 min

**Goal:** leave a searchable record of what the app changed and what it could not find.

**Steps**

1. Ask for an `ILogger<RecipesController>` in the controller's constructor, next to the
   context.
2. After a recipe is saved, log `Created recipe {RecipeId} ({Title}).` at Information.
3. When `Details` returns 404, log `Recipe {RecipeId} was not found.` at Warning.
4. Create a recipe, then open `/recipes/99`. Watch the terminal.

**Acceptance criteria**

- [ ] The terminal shows `info: RecipeBox.Controllers.RecipesController[0]` and
      `Created recipe 7 (Banitsa).`, with your id.
- [ ] `/recipes/99` shows `warn: ...` and `Recipe 99 was not found.`
- [ ] The log calls use `{RecipeId}` placeholders, not `$"..."` interpolation.

**Hints**

- `ILogger<T>` arrives by dependency injection, exactly like the context. Nothing to
  register.
- `{RecipeId}` is a named placeholder. The text looks the same as with `$"..."`, but a log
  system can search on `RecipeId = 7` across millions of lines.
- The log category is the class name, which is why `T` is the controller.

**Solution**

```csharp
private readonly RecipeBoxContext _context;
private readonly ILogger<RecipesController> _logger;

public RecipesController(RecipeBoxContext context, ILogger<RecipesController> logger)
{
    _context = context;
    _logger = logger;
}
```

```csharp
// in Create, after SaveChangesAsync
_logger.LogInformation("Created recipe {RecipeId} ({Title}).", recipe.Id, recipe.Title);

// in Details
if (recipe is null)
{
    _logger.LogWarning("Recipe {RecipeId} was not found.", id);
    return NotFound();
}
```

```
info: RecipeBox.Controllers.RecipesController[0]
      Created recipe 7 (Banitsa).
warn: RecipeBox.Controllers.RecipesController[0]
      Recipe 99 was not found.
```

---

### Task 11 — Publish the app to a folder and run it · 15 min

**Goal:** produce the thing that gets deployed, and run it without the SDK's help.

**Steps**

1. Stop the app. From the `RecipeBox` folder, publish next to the project, not inside it:

   ```bash
   dotnet publish -c Release -o ../publish
   ```

2. Open `../publish` and look at what is there.
3. Copy the database in, and start the app from the published folder:

   ```bash
   cd ../publish
   cp ../RecipeBox/recipebox.db .          # Windows: copy ..\RecipeBox\recipebox.db .
   dotnet RecipeBox.dll --urls http://localhost:5080
   ```

4. Open `http://localhost:5080/recipes`, `/recipes/2` and `/recipes/99`.

**Acceptance criteria**

- [ ] `publish/` holds `RecipeBox.dll`, `appsettings.json`, `wwwroot/` and the EF Core
      and SQLite DLLs, and **no `.cs` files**.
- [ ] The console says `Hosting environment: Production`, with no `-e` needed.
- [ ] The recipes load, styled; `/recipes/99` shows the Task 9 page with **404**.
- [ ] One sentence on why the database had to be copied.

**Hints**

- `-c Release` builds optimised code. `Debug` is for our own machine.
- Publishing **inside** the project folder makes the next build pick the published files
  up as content. Hence `../publish`.
- `SQLite Error 1: 'no such table: Recipes'` means the app found no database file and
  created an empty one. Copy it, or run the migrations against the new location.
- With no `launchSettings.json` in play, the environment defaults to Production.

**Solution**

```
$ dotnet publish -c Release -o ../publish
  RecipeBox -> .../publish/

$ ls ../publish
Microsoft.Data.Sqlite.dll                   RecipeBox.runtimeconfig.json
Microsoft.EntityFrameworkCore.dll           SQLitePCLRaw.core.dll
Microsoft.EntityFrameworkCore.Sqlite.dll    appsettings.json
RecipeBox.dll                               web.config
RecipeBox.deps.json                         wwwroot/
...                                         (no .cs files)

$ dotnet RecipeBox.dll --urls http://localhost:5080
      Hosting environment: Production

/recipes      200
/recipes/99   404   We could not find that page.
```

The connection string is `Data Source=recipebox.db`, a path relative to the folder the app
runs from. The published folder is a new place, so it needs its own copy. On a real server,
the connection string comes from the host's settings and points at a real database.

**State after this set:** RecipeBox is a complete small web app: list, filter, details,
create, edit and delete, validated and protected against CSRF, with a production error
page, structured logging and a published build. This is the shape of the final project.
