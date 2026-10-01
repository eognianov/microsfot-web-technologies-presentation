# Set 1 — HTTP and C# basics

Covers decks **01 How the Web Works** and **02 From Language to Framework**.

Students have seen HTTP, URLs, methods and status codes, and the C# essentials from
Session 02: variables and types, string interpolation, `if`/`switch`, loops, arrays,
`List<T>`, `Dictionary<K,V>`, methods, classes and properties, exceptions, and the four
LINQ operators. Nothing in this set goes beyond that, with three small exceptions that
are spelled out in the hints: `List.Contains`, `string.Join` and the `??` operator.

## Time budget · 2 hours

| # | Task | Part | Min |
|---|---|---|---|
| 1 | Read a real request in DevTools | A · HTTP | 10 |
| 2 | Trigger and identify 200, 301, 404 | A · HTTP | 10 |
| 3 | Write a recipe-scaling calculator | B · C# | 10 |
| 4 | Classify a recipe as quick, medium or long | B · C# | 8 |
| 5 | Print a shopping list with a loop *(optional)* | B · C# | 6 |
| 6 | Store ingredients in a `List<string>` | B · C# | 8 |
| 7 | Map units to grams with a `Dictionary` *(optional)* | B · C# | 8 |
| 8 | Move the logic into methods | B · C# | 12 |
| 9 | Create the `Recipe` class | B · C# | 10 |
| 10 | Hard-code a recipe catalog | B · C# | 8 |
| 11 | Query the catalog with LINQ | B · C# | 15 |
| 12 | Handle invalid input with `try`/`catch` | B · C# | 8 |
| 13 | Create the RecipeBox MVC project | C · Project | 7 |
| | **Total** | | **120** |
| | **Without the optional tasks** | | **106** |

Tasks 5 and 7 are optional: nothing later depends on them. Skip them if the group runs
behind.

## The project

```
RecipeBox/                 ← any folder you like
├── RecipeBoxBasics/       ← console app, Tasks 3–12
│   ├── Program.cs
│   └── Recipe.cs
└── RecipeBox/             ← MVC app, Task 13 → every later set
    └── Models/Recipe.cs   ← the class from Task 9, moved here
```

- **Starts from:** nothing. Only the .NET 10 SDK and an editor.
- **Ends with:** a console program that exercises every C# basic, and an MVC project
  called `RecipeBox` that runs and contains the `Recipe` class. Set 2 starts there.

Each C# task **adds to the bottom of `Program.cs`**; do not delete earlier tasks. Run
with `dotnet run` from the `RecipeBoxBasics` folder after each one.

### The sample data

Tasks 10–11 use these five recipes. The expected outputs below depend on them, so type
them exactly.

| Id | Title | Cuisine | Prep | Cook | Total | Serves | Ingredients |
|---|---|---|---|---|---|---|---|
| 1 | Pancakes | American | 10 | 15 | 25 | 4 | Flour, Milk, Eggs, Sugar |
| 2 | Spaghetti Carbonara | Italian | 10 | 15 | 25 | 2 | Spaghetti, Eggs, Pecorino, Guanciale, Black pepper |
| 3 | Shopska Salad | Bulgarian | 15 | 0 | 15 | 4 | Tomatoes, Cucumbers, Peppers, Onion, Sirene |
| 4 | Margherita Pizza | Italian | 30 | 12 | 42 | 2 | Flour, Tomatoes, Mozzarella, Basil |
| 5 | Chicken Curry | Indian | 20 | 40 | 60 | 4 | Chicken, Onion, Curry paste, Coconut milk, Rice |

---

## Part A — HTTP

### Task 1 — Read a real request in DevTools · 10 min

**Goal:** read an actual HTTP request and response instead of a diagram of one.

**Steps**

1. Open a new browser tab, then open DevTools (`F12`, or `Cmd+Option+I` on macOS) and
   switch to the **Network** tab.
2. Go to `https://www.wikipedia.org`.
3. Click the first row, the **document** request.
4. On a piece of paper, or in a text file, write down what the acceptance criteria ask for.

**Acceptance criteria**

- [ ] The method, the full URL and the status code of the document request.
- [ ] The `Content-Type` of its response.
- [ ] Three request headers, each with its value.
- [ ] The total number of requests the page made, from the status bar at the bottom.
- [ ] One request that is **not** the document (a stylesheet, script or image), with its
      type and status.

**Hints**

- If the list is empty, DevTools was opened after the page loaded. Reload with DevTools
  open.
- The **Headers** panel splits into *General*, *Response Headers* and *Request Headers*.
- The **Type** column tells you what each request fetched.

**Solution** *(values vary a little between browsers and visits)*

```
Method:        GET
URL:           https://www.wikipedia.org/
Status:        200 OK
Content-Type:  text/html

Request headers:
  accept:           text/html,application/xhtml+xml,...
  accept-language:  en-US,en;q=0.9,bg;q=0.8
  user-agent:       Mozilla/5.0 (...)

Total requests: about 15–25
Another request: a .css stylesheet — type "stylesheet", status 200
```

---

### Task 2 — Trigger and identify 200, 301, 404 · 10 min

**Goal:** see each of three status codes happen, and say what each one means.

**Steps**

1. In the Network tab, tick **Preserve log**, so a redirect does not wipe the list.
2. **200:** load `https://github.com`.
3. **301:** load `http://github.com`. Note that it is `http`, not `https`.
4. **404:** load `https://github.com/this-page-does-not-exist-recipebox`.
5. Optionally, do the same from a terminal with `curl -I`. It prints only the status
   line and the headers.

**Acceptance criteria**

- [ ] For each of the three, the URL you used and the status code you saw.
- [ ] For the 301, the value of the `Location` response header.
- [ ] One sentence per code saying **who** it blames, or that nothing went wrong: 2xx
      means fine, 3xx means "look elsewhere", 4xx is the client's mistake.

**Hints**

- Without *Preserve log*, the browser follows the redirect and you only see the final
  200.
- The 301 row has no body to look at; everything useful is in its headers.
- On Windows, type `curl.exe`; plain `curl` in old PowerShell is a different command.

**Solution**

```
curl -I https://github.com
HTTP/2 200                                   → found, here it is

curl -I http://github.com
HTTP/1.1 301 Moved Permanently
Location: https://github.com/                → look over there instead, forever

curl -I https://github.com/this-page-does-not-exist-recipebox
HTTP/2 404                                   → client asked for something that isn't there
```

---

## Part B — C# basics

### Task 3 — Write a recipe-scaling calculator · 10 min

**Goal:** variables, types, arithmetic and string interpolation, in the console
project that every C# task builds on.

**Steps**

1. Create the console project and run it once:

   ```bash
   dotnet new console -n RecipeBoxBasics -f net10.0
   cd RecipeBoxBasics
   dotnet run
   ```

2. Replace the contents of `Program.cs`. Declare a recipe name (`string`), the servings
   it is written for (`int`), the grams of flour (`double`) and the servings you want
   (`int`).
3. Calculate the scaled amount of flour and print both versions with string
   interpolation.

**Acceptance criteria**

- [ ] Four variables, each with the most fitting type.
- [ ] With 250 g for 4 servings scaled to 6, the program prints **375 g**, not 250.
- [ ] Output uses `$"..."` interpolation, not `+` concatenation.

**Hints**

- `6 / 4` is `1` in C#: an `int` divided by an `int` throws the remainder away. Convert
  one side to `double` first: `(double)targetServings / baseServings`.
- If you get 250, that is exactly what happened.

**Solution**

```csharp
string recipeName = "Pancakes";
int baseServings = 4;
double flourGrams = 250;
int targetServings = 6;

double factor = (double)targetServings / baseServings;
double scaledFlour = flourGrams * factor;

Console.WriteLine($"{recipeName} for {baseServings}: {flourGrams} g flour");
Console.WriteLine($"{recipeName} for {targetServings}: {scaledFlour} g flour");
```

```
Pancakes for 4: 250 g flour
Pancakes for 6: 375 g flour
```

---

### Task 4 — Classify a recipe as quick, medium or long · 8 min

**Goal:** make decisions with `if` / `else if` / `else`, then branch on the result with
`switch`.

**Steps**

1. Declare `int totalMinutes = 35;`.
2. Use `if` / `else if` / `else` to set a `speed` string: **Quick** under 20 minutes,
   **Medium** from 20 to 45, **Long** above 45.
3. Use a `switch` on `speed` to print a suggestion: Quick → "Good for a weeknight",
   Medium → "Good for the weekend", Long → "A Sunday project".

**Acceptance criteria**

- [ ] 15 → Quick, 35 → Medium, 90 → Long.
- [ ] The boundaries are right: **20 → Medium**, **45 → Medium**, **46 → Long**.
- [ ] The `switch` has a `default` branch.

**Hints**

- `<` and `<=` are the whole difference at the boundaries. Test 20 and 45 on purpose.
- Every `case` in a `switch` statement ends with `break;`.
- Declare `string speed;` before the `if`, so all three branches can assign it.

**Solution**

```csharp
int totalMinutes = 35;
string speed;

if (totalMinutes < 20)
    speed = "Quick";
else if (totalMinutes <= 45)
    speed = "Medium";
else
    speed = "Long";

Console.WriteLine($"{totalMinutes} min -> {speed}");

switch (speed)
{
    case "Quick":  Console.WriteLine("Good for a weeknight");  break;
    case "Medium": Console.WriteLine("Good for the weekend");  break;
    case "Long":   Console.WriteLine("A Sunday project");      break;
    default:       Console.WriteLine("Unknown");               break;
}
```

---

### Task 5 — Print a shopping list with a loop · 6 min · *optional*

**Goal:** loop over an array two ways, with `for` and with `foreach`.

**Steps**

1. Declare an array of four ingredients: Flour, Milk, Eggs, Sugar.
2. Print them as a **numbered** list with a `for` loop.
3. Print them again as a bulleted list with `foreach`.

**Acceptance criteria**

- [ ] The numbered list starts at **1**, not 0.
- [ ] Adding a fifth ingredient to the array needs no other change to the code.

**Hints**

- Arrays count from 0, people count from 1. Print `i + 1`, but index with `i`.
- Use `ingredients.Length` as the loop limit, not the number 4.

**Solution**

```csharp
string[] ingredients = { "Flour", "Milk", "Eggs", "Sugar" };

Console.WriteLine("Shopping list:");
for (int i = 0; i < ingredients.Length; i++)
{
    Console.WriteLine($"{i + 1}. {ingredients[i]}");
}

foreach (string item in ingredients)
{
    Console.WriteLine($"- {item}");
}
```

---

### Task 6 — Store ingredients in a `List<string>` · 8 min

**Goal:** use a collection that grows and shrinks.

**Steps**

1. Create an empty `List<string>` called `shopping` and `Add` Flour, Milk, Eggs and
   Sugar.
2. Add Butter. Remove Sugar.
3. Add Salt **only if** it is not already in the list.
4. Print the count, then every item.

**Acceptance criteria**

- [ ] The output says **5 items**: Flour, Milk, Eggs, Butter, Salt.
- [ ] Running the "add Salt" code twice still leaves exactly one Salt.

**Hints**

- `list.Contains("Salt")` returns `true` or `false`. `!` flips it.
- A list uses `.Count`; an array uses `.Length`. Same idea, different name.

**Solution**

```csharp
List<string> shopping = new List<string>();
shopping.Add("Flour");
shopping.Add("Milk");
shopping.Add("Eggs");
shopping.Add("Sugar");

shopping.Add("Butter");
shopping.Remove("Sugar");

if (!shopping.Contains("Salt"))
{
    shopping.Add("Salt");
}

Console.WriteLine($"{shopping.Count} items:");
foreach (string item in shopping)
{
    Console.WriteLine($"- {item}");
}
```

---

### Task 7 — Map units to grams with a `Dictionary` · 8 min · *optional*

**Goal:** look values up by key, and handle a missing key.

**Steps**

1. Create a `Dictionary<string, double>` called `gramsPerUnit`: cup → 120,
   tbsp → 15, tsp → 5.
2. Print the whole table with `foreach`.
3. Convert a quantity and a unit to grams, for example `2` `cup`.
4. If the unit is not in the dictionary, print `Unknown unit: ...` instead of crashing.

**Acceptance criteria**

- [ ] 2 cup → **240 g**, and 3 tbsp → **45 g**.
- [ ] `pinch` prints `Unknown unit: pinch`; the program does not crash.

**Hints**

- `gramsPerUnit["pinch"]` throws `KeyNotFoundException`. Ask first with
  `gramsPerUnit.ContainsKey(unit)`.
- In a `foreach` over a dictionary, each item has `.Key` and `.Value`.

**Solution**

```csharp
Dictionary<string, double> gramsPerUnit = new()
{
    ["cup"]  = 120,
    ["tbsp"] = 15,
    ["tsp"]  = 5
};

foreach (var pair in gramsPerUnit)
    Console.WriteLine($"1 {pair.Key} = {pair.Value} g");

string unit = "cup";
double quantity = 2;

if (gramsPerUnit.ContainsKey(unit))
    Console.WriteLine($"{quantity} {unit} = {quantity * gramsPerUnit[unit]} g");
else
    Console.WriteLine($"Unknown unit: {unit}");
```

---

### Task 8 — Move the logic into methods · 12 min

**Goal:** name a piece of logic once and reuse it.

**Steps**

1. Write `ScaleServings(double amount, int fromServings, int toServings)`, which returns
   the scaled amount.
2. Write `TotalTime(int prepMinutes, int cookMinutes)`, which returns the sum.
3. Write `Classify(int totalMinutes)`, which returns "Quick", "Medium" or "Long", using
   the rules from Task 4.
4. Write `PrintList(List<string> items)`, which prints a numbered list and returns
   nothing.
5. Call each one once and print the result.

**Acceptance criteria**

- [ ] `ScaleServings(250, 4, 6)` → **375**.
- [ ] `TotalTime(10, 15)` → **25**.
- [ ] `Classify(TotalTime(20, 40))` → **Long**. One method's result feeds another.
- [ ] `PrintList(shopping)` prints the Task 6 list, numbered from 1.
- [ ] `PrintList` is `void`; the other three have a return type.

**Hints**

- Shape: `returnType Name(type param, ...) { ... return value; }`.
- A one-line method can use `=>` instead, as `IsRecent()` did in Session 02.
- In `ScaleServings`, multiply before dividing (`amount * toServings / fromServings`).
  `amount` is a `double`, so the division is not an integer division.
- In `Classify`, an early `return` replaces `else`: once a `return` runs, the method is
  done.

**Solution**

```csharp
double ScaleServings(double amount, int fromServings, int toServings)
    => amount * toServings / fromServings;

int TotalTime(int prepMinutes, int cookMinutes) => prepMinutes + cookMinutes;

string Classify(int totalMinutes)
{
    if (totalMinutes < 20) return "Quick";
    if (totalMinutes <= 45) return "Medium";
    return "Long";
}

void PrintList(List<string> items)
{
    for (int i = 0; i < items.Count; i++)
        Console.WriteLine($"{i + 1}. {items[i]}");
}

Console.WriteLine(ScaleServings(250, 4, 6));      // 375
Console.WriteLine(TotalTime(10, 15));             // 25
Console.WriteLine(Classify(TotalTime(20, 40)));   // Long
PrintList(shopping);
```

---

### Task 9 — Create the `Recipe` class · 10 min

**Goal:** describe a recipe once, as a blueprint, and build an object from it. **This
class moves into the MVC project in Task 13 and becomes a database table in Set 4.**

**Steps**

1. Add a new file, `Recipe.cs`, next to `Program.cs`.
2. Declare `public class Recipe` with these properties: `Id` (int), `Title` (string),
   `Cuisine` (string), `PrepMinutes` (int), `CookMinutes` (int), `Servings` (int),
   `Ingredients` (`List<string>`).
3. Add a method `TotalMinutes()` that returns prep plus cook.
4. In `Program.cs`, create a `pancakes` object with the values from the sample data,
   and print a one-line summary.

**Acceptance criteria**

- [ ] The class lives in its own file, `Recipe.cs`.
- [ ] String properties start as `""` and the list starts as an empty list, so a new
      `Recipe` never holds `null`.
- [ ] The output is `Pancakes (American)` on one line and `25 min, serves 4` on the next.
- [ ] `dotnet build` shows **0 warnings**.

**Hints**

- `{ get; set; }` makes a property readable and writable.
- Without `= "";` the compiler warns that a non-nullable string may be null. The
  warning is right.
- Object initializer: `new Recipe { Title = "Pancakes", ... }`.

**Solution**

`Recipe.cs`

```csharp
public class Recipe
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public string Cuisine { get; set; } = "";
    public int PrepMinutes { get; set; }
    public int CookMinutes { get; set; }
    public int Servings { get; set; }
    public List<string> Ingredients { get; set; } = new();

    public int TotalMinutes() => PrepMinutes + CookMinutes;
}
```

`Program.cs`

```csharp
var pancakes = new Recipe
{
    Id = 1,
    Title = "Pancakes",
    Cuisine = "American",
    PrepMinutes = 10,
    CookMinutes = 15,
    Servings = 4,
    Ingredients = new() { "Flour", "Milk", "Eggs", "Sugar" }
};

Console.WriteLine($"{pancakes.Title} ({pancakes.Cuisine})");
Console.WriteLine($"{pancakes.TotalMinutes()} min, serves {pancakes.Servings}");
```

---

### Task 10 — Hard-code a recipe catalog · 8 min

**Goal:** a list of objects. This is the shape every page of the MVC app will show.

**Steps**

1. Create a `List<Recipe>` called `recipes` holding all five recipes from the sample
   data. Reuse `pancakes` as the first one.
2. Print one line per recipe: id, title, total minutes and the `Classify` result from
   Task 8.

**Acceptance criteria**

- [ ] Five lines, in id order.
- [ ] Shopska Salad shows **15 min (Quick)** and Chicken Curry shows **60 min (Long)**.
- [ ] Changing one recipe's minutes changes its classification with no other edit.

**Hints**

- A collection initializer: `new() { item1, item2, ... }`.
- `Classify` from Task 8 takes an `int`, and `r.TotalMinutes()` is one.

**Solution**

```csharp
List<Recipe> recipes = new()
{
    pancakes,
    new Recipe { Id = 2, Title = "Spaghetti Carbonara", Cuisine = "Italian",   PrepMinutes = 10, CookMinutes = 15, Servings = 2,
                 Ingredients = new() { "Spaghetti", "Eggs", "Pecorino", "Guanciale", "Black pepper" } },
    new Recipe { Id = 3, Title = "Shopska Salad",       Cuisine = "Bulgarian", PrepMinutes = 15, CookMinutes = 0,  Servings = 4,
                 Ingredients = new() { "Tomatoes", "Cucumbers", "Peppers", "Onion", "Sirene" } },
    new Recipe { Id = 4, Title = "Margherita Pizza",    Cuisine = "Italian",   PrepMinutes = 30, CookMinutes = 12, Servings = 2,
                 Ingredients = new() { "Flour", "Tomatoes", "Mozzarella", "Basil" } },
    new Recipe { Id = 5, Title = "Chicken Curry",       Cuisine = "Indian",    PrepMinutes = 20, CookMinutes = 40, Servings = 4,
                 Ingredients = new() { "Chicken", "Onion", "Curry paste", "Coconut milk", "Rice" } }
};

foreach (Recipe r in recipes)
    Console.WriteLine($"#{r.Id} {r.Title} - {r.TotalMinutes()} min ({Classify(r.TotalMinutes())})");
```

```
#1 Pancakes - 25 min (Medium)
#2 Spaghetti Carbonara - 25 min (Medium)
#3 Shopska Salad - 15 min (Quick)
#4 Margherita Pizza - 42 min (Medium)
#5 Chicken Curry - 60 min (Long)
```

---

### Task 11 — Query the catalog with LINQ · 15 min

**Goal:** ask the catalog questions with `Where`, `Select`, `OrderBy` and
`FirstOrDefault`, without writing loops. Set 4 runs these same queries against a
database.

**Steps.** Answer each question with one LINQ expression and print the answer:

1. Which recipes take **under 30 minutes** in total?
2. What are the **titles** of the Italian recipes?
3. All recipes, **fastest first**.
4. Which recipes contain **Eggs**? Titles only.
5. Find the recipe titled **Banitsa**. Print `not found` if there is none.

**Acceptance criteria**

- [ ] 1 → Pancakes, Spaghetti Carbonara, Shopska Salad.
- [ ] 2 → `Spaghetti Carbonara, Margherita Pizza`, on one line.
- [ ] 3 → starts with Shopska Salad (15) and ends with Chicken Curry (60).
- [ ] 4 → `Pancakes, Spaghetti Carbonara`.
- [ ] 5 → `Banitsa: not found`. The program does **not** crash.
- [ ] No `for` or `if` used to filter. Loops are only used to print.

**Hints**

- Read `r => r.Cuisine == "Italian"` as "for each recipe `r`, is its cuisine Italian?"
- `Where` keeps whole recipes; `.Select(r => r.Title)` turns them into titles. Chain
  them.
- `string.Join(", ", titles)` puts a list on one line, comma-separated.
- `First` throws when nothing matches; `FirstOrDefault` returns `null`. Check for `null`
  before using the result.

**Solution**

```csharp
// 1 · under 30 minutes
var quick = recipes.Where(r => r.TotalMinutes() < 30);
foreach (Recipe r in quick) Console.WriteLine($"- {r.Title}");

// 2 · Italian titles
var italian = recipes.Where(r => r.Cuisine == "Italian").Select(r => r.Title);
Console.WriteLine($"Italian: {string.Join(", ", italian)}");

// 3 · fastest first
var byTime = recipes.OrderBy(r => r.TotalMinutes());
foreach (Recipe r in byTime) Console.WriteLine($"- {r.TotalMinutes()} min {r.Title}");

// 4 · with eggs
var withEggs = recipes.Where(r => r.Ingredients.Contains("Eggs")).Select(r => r.Title);
Console.WriteLine($"With eggs: {string.Join(", ", withEggs)}");

// 5 · Banitsa
Recipe? banitsa = recipes.FirstOrDefault(r => r.Title == "Banitsa");
if (banitsa == null) Console.WriteLine("Banitsa: not found");
else Console.WriteLine($"Banitsa: {banitsa.TotalMinutes()} min");
```

---

### Task 12 — Handle invalid input with `try`/`catch` · 8 min

**Goal:** catch the failures you can predict, and only those. In a web app an uncaught
one becomes a **500**.

**Steps**

1. Ask "How many servings?" and read the answer with `Console.ReadLine()`.
2. Turn it into an `int` with `int.Parse` and print the flour needed for that many
   servings, using `ScaleServings` from Task 8.
3. Wrap it in `try` / `catch` so bad input prints a friendly message instead of a stack
   trace.

**Acceptance criteria**

- [ ] `6` → `Flour for 6: 375 g`.
- [ ] `abc` and an empty line → `'abc' is not a whole number.`, with no crash.
- [ ] `99999999999` → its own message ("far too many servings"), with no crash.
- [ ] No empty `catch { }`, and no catch-all `catch (Exception)`.

**Hints**

- Run it from a terminal with `dotnet run`. The VS Code debug console cannot take
  typed input.
- `Console.ReadLine()` returns `string?`: it can be `null`. `?? ""` means "or an empty
  string if it is null".
- Type `99999999999` first, without a `catch` for it, and read which exception it
  throws. That name is what you catch.
- `int.TryParse` would avoid the exception altogether. That is the right choice in real
  code, but this task is about exceptions.

**Solution**

```csharp
Console.Write("How many servings? ");
string input = Console.ReadLine() ?? "";

try
{
    int servings = int.Parse(input);
    Console.WriteLine($"Flour for {servings}: {ScaleServings(250, 4, servings)} g");
}
catch (FormatException)
{
    Console.WriteLine($"'{input}' is not a whole number.");
}
catch (OverflowException)
{
    Console.WriteLine($"'{input}' is far too many servings.");
}
```

---

## Part C — The project

### Task 13 — Create the RecipeBox MVC project · 7 min

**Goal:** the app that Sets 2–5 grow, with our `Recipe` class already in it.

**Steps**

1. Go up one folder, out of `RecipeBoxBasics`, and create the MVC project:

   ```bash
   cd ..
   dotnet new mvc -n RecipeBox -f net10.0
   cd RecipeBox
   dotnet run
   ```

2. Open the URL the console prints, with DevTools open on the Network tab.
3. Copy `Recipe.cs` from `RecipeBoxBasics` into `RecipeBox/Models/`, and add
   `namespace RecipeBox.Models;` as its first line.
4. Stop the app with `Ctrl+C` and run `dotnet build`.

**Acceptance criteria**

- [ ] The home page loads, and DevTools shows the document as **GET · 200 ·
      text/html**.
- [ ] A made-up path such as `/nope` returns **404**, the same code as in Task 2, now
      from our own server.
- [ ] `Models/Recipe.cs` exists and starts with `namespace RecipeBox.Models;`.
- [ ] `dotnet build` → **0 errors, 0 warnings**.

**Hints**

- The port number is random per project. Use the URL the console prints, not one from
  a slide.
- Every file in `Models/` uses the `RecipeBox.Models` namespace. Open
  `ErrorViewModel.cs` and compare.
- From now on, run `dotnet watch` instead of `dotnet run`. It rebuilds on save.

**Solution**

`Models/Recipe.cs`

```csharp
namespace RecipeBox.Models;

public class Recipe
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public string Cuisine { get; set; } = "";
    public int PrepMinutes { get; set; }
    public int CookMinutes { get; set; }
    public int Servings { get; set; }
    public List<string> Ingredients { get; set; } = new();

    public int TotalMinutes() => PrepMinutes + CookMinutes;
}
```

```
$ dotnet run
Now listening on: http://localhost:5257       ← yours will differ

$ curl -I http://localhost:5257/
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8

$ curl -I http://localhost:5257/nope
HTTP/1.1 404 Not Found
```

**State after this set:** `RecipeBox` runs, shows the default home page, and contains
`Models/Recipe.cs`. Set 2 adds the `RecipesController`.
