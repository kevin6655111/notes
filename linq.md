# LINQ

> C# 查詢運算子的速查與常見寫法。每個方法附最小可用的範例，後半是幾個實際情境的組合寫法。

## 目錄

- [⏳ 延遲執行](#deferred)
- [🔍 篩選與投影](#filter)
- [🔢 排序](#order)
- [🧮 聚合](#aggregate)
- [🔗 集合運算](#set)
- [📦 分組](#group)
- [🤝 Join](#join)
- [🎯 取得單一元素](#element)
- [✨ 產生序列](#generate)
- [🔄 型別轉換](#cast)
- [✅ 判斷](#predicate)
- [🛠️ 實用範例](#examples)
- [⚠️ 常見陷阱](#pitfalls)
- [📚 參考資料](#ref)

> 標記 `.NET 6+` 的方法是較新的多載，舊框架沒有。

---

<a id="deferred"></a>

## ⏳ 延遲執行

大多數 LINQ 運算子回傳的是**查詢本身**，不是結果。真正執行是在列舉的當下（`foreach`、`ToList()`、`Count()` 等）。

```csharp
var query = numbers.Where(n => n > 10);   // 這行沒有跑任何比對
var list  = query.ToList();               // 這行才真的執行
```

兩個實際影響：

- **來源變了，重新列舉會得到不同結果。** 同一個 `query` 跑兩次，中間改過集合就會不一樣。
- **重複列舉等於重複計算。** 要用多次就先 `ToList()` 收起來。

| 立即執行 | 延遲執行 |
|----------|----------|
| `ToList` `ToArray` `ToDictionary` `ToHashSet` | `Where` `Select` `OrderBy` `GroupBy` |
| `Count` `Sum` `Max` `Average` `Aggregate` | `Take` `Skip` `Distinct` `Concat` |
| `First` `Last` `Single` `ElementAt` `Any` `All` | `Cast` `OfType` `Reverse` `Zip` |

---

<a id="filter"></a>

## 🔍 篩選與投影

| 方法 | 說明 |
|------|------|
| `Where` | 篩選符合條件的項目 |
| `Select` | 把每個項目投影成新形狀 |
| `SelectMany` | 投影後把巢狀序列攤平成一層 |
| `Take(n)` | 取前 n 個 |
| `Skip(n)` | 跳過前 n 個 |
| `TakeWhile` / `SkipWhile` | 依條件取到 / 跳到第一個不符合為止 |
| `TakeLast(n)` / `SkipLast(n)` | 從尾端取 / 跳過 |
| `Chunk(size)` | 按大小切成多個子集合 `.NET 6+` |

```csharp
// Select 的第二個參數是索引
var indexed = names.Select((name, i) => new { Index = i, Name = name });

// SelectMany 攤平
var allTags = posts.SelectMany(p => p.Tags);

// Chunk 每 3 筆一組
foreach (int[] batch in Enumerable.Range(1, 10).Chunk(3))
    Console.WriteLine(string.Join(",", batch));
// 1,2,3 / 4,5,6 / 7,8,9 / 10
```

`TakeWhile` 與 `Where` 的差別：前者遇到第一個不符合就停，後者會掃完全部。

---

<a id="order"></a>

## 🔢 排序

| 方法 | 說明 |
|------|------|
| `OrderBy` / `OrderByDescending` | 主要排序鍵 |
| `ThenBy` / `ThenByDescending` | 次要排序鍵，接在 `OrderBy` 後面 |
| `Reverse` | 反轉現有順序，不做比較 |
| `Order()` / `OrderDescending()` | 直接用元素本身排序 `.NET 7+` |

```csharp
var sorted = people.OrderBy(p => p.DeptId)
                   .ThenByDescending(p => p.Salary);
```

```csharp
// 字串反轉
string str = "Hello World!";
string reversed = new string(str.Reverse().ToArray());
```

> 第二個排序鍵要用 `ThenBy`，連續寫兩個 `OrderBy` 會讓後者完全覆蓋前者。

---

<a id="aggregate"></a>

## 🧮 聚合

| 方法 | 說明 |
|------|------|
| `Count` / `LongCount` | 總數，可帶條件。`LongCount` 回傳 `long` |
| `Sum` `Average` `Max` `Min` | 數值聚合 |
| `MaxBy` / `MinBy` | 回傳**鍵值最大的那個元素本身** `.NET 6+` |
| `Aggregate` | 自訂累積邏輯 |

```csharp
var words = new List<string> { "apple", "banana", "cherry", "date", "elderberry" };
long total = words.LongCount();                  // 5
long over5 = words.LongCount(w => w.Length > 5); // 3
```

```csharp
string[] numbers = { "10007", "37", "299846234235" };
double average = numbers.Average(num => long.Parse(num));
```

`Max` 回傳最大的**鍵值**，`MaxBy` 回傳擁有該鍵值的**元素**：

```csharp
int topSalary   = emp.Max(e => e.Salary);     // 40000
Employee topEmp = emp.MaxBy(e => e.Salary);   // 那個人本身，.NET 6+
```

### Aggregate

```csharp
// 兩個參數：起始值 + 累積函式
int[] arr = { 4, 8, 8, 3, 9, 0, 7, 8, 2 };
int evenCount = arr.Aggregate(0, (total, next) => next % 2 == 0 ? total + 1 : total);
// 6
```

```csharp
// 三個參數：起始值 + 累積函式 + 結果轉換
string[] fruits = { "apple", "mango", "orange", "passionfruit", "grape" };
string longest = fruits.Aggregate(
    "banana",
    (longest, next) => next.Length > longest.Length ? next : longest,
    fruit => fruit.ToUpper());
// PASSIONFRUIT
```

```csharp
// 用 Aggregate 累積集合
var namesList = new List<string[]> {
    new[] { "Tommy" },
    new[] { "Terry", "Henriette", "Magnus", "Shu" },
    new[] { "Ajay", "Helge", "Henriette", "Cristina", "Lucio" }
};
var allNames = namesList.Aggregate(
    Enumerable.Empty<string>(),
    (current, next) => next.Length > 3 ? current.Union(next) : current);
```

---

<a id="set"></a>

## 🔗 集合運算

| 方法 | 說明 | 保留重複 |
|------|------|:--------:|
| `Concat` | 前後串接 | 是 |
| `Union` | 聯集 | 否 |
| `Intersect` | 交集 | 否 |
| `Except` | 差集，在 a 不在 b | 否 |
| `Distinct` | 去除重複 | — |
| `Zip` | 兩個序列依索引配對 | — |

`UnionBy` / `IntersectBy` / `ExceptBy` / `DistinctBy` 讓你指定比較鍵，不必實作 `IEqualityComparer` `.NET 6+`。

```csharp
var onlyInA = a.Except(b);                      // 存在於 a 但不在 b
var kept    = people.ExceptBy(idsToExclude, p => p.Id);
var common  = people.IntersectBy(otherIds, p => p.Id);
```

```csharp
var pairs = names.Zip(scores, (n, s) => $"{n}: {s}");
```

> `Concat` 與 `Union` 最容易搞混：要合併清單用 `Concat`，要合併並去重才用 `Union`。

---

<a id="group"></a>

## 📦 分組

`GroupBy` 回傳 `IGrouping<TKey, TElement>` 的序列，`Key` 是分組鍵，本身可當集合列舉。

```csharp
var stats = orders
    .GroupBy(o => o.CustomerId)
    .Select(g => new {
        Customer = g.Key,
        Count    = g.Count(),
        Total    = g.Sum(o => o.Amount)
    });
```

查詢語法的等價寫法：

```csharp
var stats = from o in orders
            group o by o.CustomerId into g
            select new { Customer = g.Key, Count = g.Count(), Total = g.Sum(x => x.Amount) };
```

`ToLookup` 與 `GroupBy` 類似，但**立即執行**且可以用索引子直接取某一組。

---

<a id="join"></a>

## 🤝 Join

| 寫法 | 對應 |
|------|------|
| `Join` | Inner Join |
| `GroupJoin` + `SelectMany` + `DefaultIfEmpty` | Left Join |

### Inner Join

```csharp
var result = from bk in books
             join o in orders on bk.BookId equals o.BookId
             select new { bk.BookId, bk.Name, o.PaymentMode };
```

```csharp
var result = books.Join(orders,
    bk => bk.BookId,
    o  => o.BookId,
    (bk, o) => new { bk.BookId, bk.Name, o.PaymentMode });
```

### Left Join

關鍵是 `into` 之後接 `DefaultIfEmpty()`，沒有配對到的左表項目才會被保留。

```csharp
var result = from bk in books
             join o in orders on bk.BookId equals o.BookId into grp
             from o in grp.DefaultIfEmpty(new Order())
             select new { bk.BookId, bk.Name, o.PaymentMode };
```

```csharp
var result = books
    .GroupJoin(orders, bk => bk.BookId, o => o.BookId,
        (bk, ordr) => new { Book = bk, Orders = ordr })
    .SelectMany(x => x.Orders.DefaultIfEmpty(new Order()),
        (x, o) => new { x.Book.BookId, x.Book.Name, o.PaymentMode });
```

> `DefaultIfEmpty()` 不給參數時，參考型別會拿到 `null`，後面取屬性就會 `NullReferenceException`。要嘛傳一個空物件進去，要嘛用 `o?.PaymentMode`。

---

<a id="element"></a>

## 🎯 取得單一元素

| 方法 | 找不到時 | 有多筆時 |
|------|----------|----------|
| `First` | 拋例外 | 取第一筆 |
| `FirstOrDefault` | 回傳預設值 | 取第一筆 |
| `Last` / `LastOrDefault` | 同上 | 取最後一筆 |
| `Single` | 拋例外 | **拋例外** |
| `SingleOrDefault` | 回傳預設值 | **拋例外** |
| `ElementAt(i)` | 拋例外 | — |
| `ElementAtOrDefault(i)` | 回傳預設值 | — |

```csharp
string[] names = { "Tommy", "Terry", "Henriette", "Magnus", "Shu" };
var random = new Random();
string name = names.ElementAt(random.Next(0, names.Length));
```

`...OrDefault` 系列可以指定預設值 `.NET 6+`：

```csharp
var item = list.FirstOrDefault(x => x.IsActive, fallbackItem);
```

> 預期「只會有一筆」時用 `Single`，資料有問題會立刻炸而不是默默取到第一筆。

---

<a id="generate"></a>

## ✨ 產生序列

```csharp
Enumerable.Range(20, 30);            // 20 開始、共 30 個：20~49
Enumerable.Repeat("重複10次", 10);    // 同一個值重複 10 次
Enumerable.Empty<string>();          // 空序列，當累積起始值用
```

| 方法 | 說明 |
|------|------|
| `Prepend` / `Append` | 在序列開頭 / 結尾加一個值 |
| `DefaultIfEmpty` | 序列為空時回傳只含預設值的單一元素序列 |

```csharp
var numbers = new List<int> { 1, 2, 3, 4 };
var withZero = numbers.Prepend(0);   // 0,1,2,3,4
```

> `Range` 的第二個參數是**數量不是結束值**，`Range(20, 30)` 是 20~49 而不是 20~30。

---

<a id="cast"></a>

## 🔄 型別轉換

| 方法 | 遇到型別不符 |
|------|--------------|
| `Cast<T>` | **拋 `InvalidCastException`** |
| `OfType<T>` | **略過**，只留下符合的 |

```csharp
var list = new ArrayList { "apple", "banana", "cherry" };
IEnumerable<string> all = list.Cast<string>();      // 有非字串就炸
IEnumerable<string> safe = list.OfType<string>();   // 只挑出字串
```

轉成實體集合：`ToList` `ToArray` `ToDictionary` `ToHashSet`。

> `ConvertAll` **不是 LINQ**，它是 `List<T>` 和陣列自己的方法，對 `IEnumerable<T>` 用不了。等價的 LINQ 寫法是 `Select(...).ToList()`。
>
> ```csharp
> var list = new List<int> { 1, 2, 3, 4 };
> list.ConvertAll(r => r.ToString());        // List<T> 的方法
> list.Select(r => r.ToString()).ToList();   // LINQ 的等價寫法
> ```

---

<a id="predicate"></a>

## ✅ 判斷

| 方法 | 說明 | 空序列時 |
|------|------|----------|
| `Any()` | 有沒有任何元素 | `false` |
| `Any(條件)` | 至少一個符合 | `false` |
| `All(條件)` | 全部符合 | **`true`** |
| `Contains(值)` | 是否包含某值 | `false` |
| `SequenceEqual` | 兩序列元素與順序是否完全相同 | — |

```csharp
string[] arr = { "apple", "banana", "cat" };
bool allHaveA = arr.All(pet => pet.Contains("a"));   // true
```

> **空序列的 `All` 回傳 `true`**，這是數學上的空真，不是 bug。條件再嚴格都會通過，寫驗證邏輯時要留意。
>
> 判斷「有沒有資料」用 `Any()` 而不是 `Count() > 0`，前者找到一個就停。

---

<a id="examples"></a>

## 🛠️ 實用範例

### 產生範圍內的奇數

```csharp
// 20 開始共 30 個（20~49），取其中的奇數
var oddNums = Enumerable.Range(20, 30).Where(x => x % 2 != 0);
```

### 產生範圍內的浮點數並查詢

```csharp
// 200~399 各除以 10 → 20.0 ~ 39.9
var rng = Enumerable.Range(200, 200).Select(x => x / 10f);

var first         = rng.First();                              // 20
var last          = rng.Last();                               // 39.9
var firstOver20   = rng.Where(f => f > 20).FirstOrDefault();  // 20.1
var lastUnder22   = rng.Where(f => f < 22).LastOrDefault();   // 21.9
var atIndex15     = rng.ElementAtOrDefault(15);               // 21.5
```

### 依索引切組，取每組的極值

`Select` 的索引多載配上整數除法，是不用 `Chunk` 也能分組的老寫法。

```csharp
var sequence = Enumerable.Range(200, 200).Select(x => x / 10f);

var grps = from x in sequence.Select((num, i) => new { Num = num, Grp = i / 10 })
           group x.Num by x.Grp into g
           select new { Min = g.Min(), Max = g.Max() };

var grpsLambda = sequence
    .Select((num, i) => new { Num = num, Grp = i / 10 })
    .GroupBy(g => g.Grp)
    .Select(s => new {
        s.Key,
        Min    = s.Min(x => x.Num),
        Max    = s.Max(x => x.Num),
        Counts = s.Count()
    });
```

> `.NET 6+` 可以直接用 `sequence.Chunk(10)`，不用自己算索引。

### 電子郵件每 3 筆一組

```csharp
string[] email = {
    "One@example.com",   "Two@example.com",  "Three@example.com", "Four@example.com",
    "Five@example.com",  "Six@example.com",  "Seven@example.com", "Eight@example.com"
};

var emailGrp = from i in Enumerable.Range(0, email.Length)
               group email[i] by i / 3;

var emailGrpLambda = email
    .Select((addr, i) => new { Grp = i / 3, Email = addr })
    .GroupBy(g => g.Grp)
    .Select(s => new {
        s.Key,
        Email = string.Join(";", s.Select(x => x.Email))
    });
```

### 找出各部門薪水最高的員工

```csharp
var emp = new List<Employee> {
    new() { EmpId = 1, DeptId = 1, Salary = 20000 },
    new() { EmpId = 2, DeptId = 1, Salary = 40000 },
    new() { EmpId = 3, DeptId = 2, Salary = 20000 },
    new() { EmpId = 4, DeptId = 1, Salary = 10000 },
};

// 查詢語法：用 let 先算出該組最高薪，避免重複計算
var highest = from e in emp
              group e by e.DeptId into g
              let topSalary = g.Max(m => m.Salary)
              select new {
                  Dept   = g.Key,
                  EmpId  = g.First(f => f.Salary == topSalary).EmpId,
                  Salary = topSalary
              };

// .NET 6+ 最簡潔的寫法
var highestByGroup = emp
    .GroupBy(e => e.DeptId)
    .Select(g => g.MaxBy(e => e.Salary));
```

> 上面查詢語法裡的 `let` 不只是好看。寫成 `g.First(f => f.Salary == g.Max(m => m.Salary))` 的話，`Max` 會在每次比對時重新算一遍整組。

### from-let-where

`let` 可以把中間結果存成變數，後面的 `where` 和 `select` 都能用。

```csharp
var arr = new[] { 5, 3, 4, 2, 6, 7 };
var sq = from num in arr
         let square = num * num
         where square > 10
         select new { num, square };
// 16, 25, 36, 49
```

### 交換字串中的單詞

```csharp
string name = "Teng, Kevin";
name = string.Join(",", name.Split(',').Reverse()).Trim();
// "Kevin,Teng"
```

> `Trim()` 只去掉頭尾空白。原本逗號後的那個空格會被搬到中間，結果是 `Kevin,Teng` 而不是 `Kevin, Teng`。要保留格式就先對每段 `Trim()` 再用 `", "` 接起來。

---

<a id="pitfalls"></a>

## ⚠️ 常見陷阱

| 陷阱 | 說明 |
|------|------|
| 重複列舉 | 延遲執行的查詢每次列舉都重跑一次。要用多次先 `ToList()` |
| 空序列的 `All` | 回傳 `true`，不是 `false` |
| `Count() > 0` | 會走訪整個序列，判斷有無資料用 `Any()` |
| `First` vs `Single` | 預期只有一筆時用 `Single`，資料異常才會被發現 |
| `DefaultIfEmpty()` 沒給值 | 參考型別拿到 `null`，Left Join 後取屬性會炸 |
| 連續兩個 `OrderBy` | 後者覆蓋前者，次要排序要用 `ThenBy` |
| `Range` 的第二個參數 | 是數量不是結束值 |
| `Cast<T>` 型別不符 | 直接拋例外，要略過用 `OfType<T>` |
| 在 `Select` 裡呼叫資料庫 | 對 `IQueryable` 而言，不能翻譯成 SQL 的運算會整段落回記憶體執行 |

---

<a id="ref"></a>

## 📚 參考資料

- [System.Linq 命名空間](https://learn.microsoft.com/zh-tw/dotnet/api/system.linq)
- [標準查詢運算子概觀](https://learn.microsoft.com/zh-tw/dotnet/csharp/programming-guide/concepts/linq/standard-query-operators-overview)
- [LINQ 總覽](https://learn.microsoft.com/dotnet/csharp/linq/)
