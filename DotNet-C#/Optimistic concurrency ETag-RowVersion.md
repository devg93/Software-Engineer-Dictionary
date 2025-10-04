მოკლედ: **Optimistic Concurrency** ნიშნავს — „ჩავთვალოთ, რომ კონფლიქტი იშვიათია“. ჩაწერის დროს ვამოწმებთ, შეიცვალა თუ არა ჩანაწერი მას შემდეგ, რაც წავიკითხეთ. ამისთვის ვიყენებთ ვერსიის მარკერს: **RowVersion/ETag**. თუ ვერსია არ ემთხვევა, ჩაწერა იბლოკება „ვიღაცამ უკვე შეცვალა“ მიზეზით.

# როგორ მუშაობს

- **RowVersion (SQL/EF Core):** ბაინარული „თაიმსტამპი“ (`rowversion/timestamp` სვეტი SQL Server-ში). EF Core აგდებს `DbUpdateConcurrencyException`-ს, თუ `WHERE ... AND RowVersion = @original` ვერ იპოვა ჩანაწერი.
    
- **ETag (HTTP/Cosmos/REST):** რესურსს აქვს `ETag` ჰედერი. Client აგზავნის `If-Match: "<etag>"`. თუ არ ემთხვევა, ვუბრუნებთ **412 Precondition Failed** (ან EF-ზე 409/ProblemDetails).
    

**Trade-offs:**  
✔ კონფლიქტი იშვიათია → არ გვჭირდება მძიმე locks; მაღალმასშტაბურია.  
✘ საჭიროა კონფლიქტების დამუშავება UI/ბიზნეს-ლოგიკაში (merge/retry/refresh).

---

## EF Core + SQL Server (RowVersion) — პრაქტიკული იმპლემენტაცია

**Entity & Mapping**

```csharp
public sealed class Order
{
    public Guid Id { get; private set; }
    public decimal Amount { get; private set; }
    public string Status { get; private set; } = "Created";

    // RowVersion concurrency token
    public byte[] RowVersion { get; private set; } = Array.Empty<byte>();

    public void UpdateAmount(decimal amount) => Amount = amount;
}

public sealed class AppDbContext : DbContext
{
    public DbSet<Order> Orders => Set<Order>();
    public AppDbContext(DbContextOptions<AppDbContext> o) : base(o) { }

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<Order>(e =>
        {
            e.HasKey(x => x.Id);
            e.Property(x => x.RowVersion)
                .IsRowVersion() // ValueGeneratedOnAddOrUpdate + ConcurrencyToken
                .IsConcurrencyToken();
        });
    }
}
```

**DTO RowVersion-ის უსაფრთხო გადაცემა (Base64)**

```csharp
public sealed record OrderDto(Guid Id, decimal Amount, string Status, string RowVersion); // Base64
```

**GET — დააბრუნე ETag და RowVersion**

```csharp
[HttpGet("{id:guid}")]
public async Task<ActionResult<OrderDto>> Get(Guid id, [FromServices] AppDbContext db, CancellationToken ct)
{
    var o = await db.Orders.FindAsync(new object?[] { id }, ct);
    if (o is null) return NotFound();

    var base64 = Convert.ToBase64String(o.RowVersion);
    Response.Headers.ETag = $"\"{base64}\"";

    return Ok(new OrderDto(o.Id, o.Amount, o.Status, base64));
}
```

**PUT/PATCH — If-Match/RowVersion ვალიდაცია და კონფლიქტის დამუშავება**

```csharp
public sealed record UpdateOrderAmount(decimal Amount, string RowVersion); // Base64 from client

[HttpPut("{id:guid}/amount")]
public async Task<IActionResult> UpdateAmount(Guid id, [FromBody] UpdateOrderAmount cmd,
    [FromServices] AppDbContext db, CancellationToken ct)
{
    var entity = await db.Orders.FirstOrDefaultAsync(x => x.Id == id, ct);
    if (entity is null) return NotFound();

    // მოთხოვნა: client-მა უნდა მოიტანოს მიმდინარე ვერსია
    if (string.IsNullOrWhiteSpace(cmd.RowVersion))
        return StatusCode(StatusCodes.Status428PreconditionRequired, "RowVersion (If-Match) is required.");

    // მიაბით entity-ს ორიგინალი ვერსია, რომ EF-მა WHERE-ში ჩასვას
    var original = Convert.FromBase64String(cmd.RowVersion);
    db.Entry(entity).Property(e => e.RowVersion).OriginalValue = original;

    entity.UpdateAmount(cmd.Amount);

    try
    {
        await db.SaveChangesAsync(ct);
    }
    catch (DbUpdateConcurrencyException)
    {
        // ვიღაცამ უკვე განაახლა — დააბრუნე 412/409 და უახლესი მდგომარეობა
        var current = await db.Orders.AsNoTracking().FirstAsync(x => x.Id == id, ct);
        var currentRv = Convert.ToBase64String(current.RowVersion);

        return Conflict(new
        {
            message = "Concurrency conflict. The resource was modified by another user.",
            current = new { current.Id, current.Amount, current.Status, RowVersion = currentRv }
        });
    }

    return NoContent();
}
```

**SQL მხარეს (Migration ციტატა):**

```sql
ALTER TABLE [Orders] ADD [RowVersion] rowversion NOT NULL; -- auto-maintained
```

---

## HTTP ETag first (თუ Read მოდელი/REST გაქვთ)

- **Server (GET):** `ETag: "base64RowVersion"`
    
- **Client (PUT/PATCH/DELETE):** `If-Match: "base64RowVersion"`
    
- **Server:** თუ არ ემთხვევა → **412 Precondition Failed**; წინააღმდეგ შემთხვევაში განახლება.
    

ASP.NET Core-ში ზემოთ ნაჩვენები RowVersion შეგიძლიათ პირდაპირ ETag-ად გამოიყენოთ (ერთადერთი წესი — _საიდუმლო არაფერია, უბრალოდ უნიკალური ვერსიის ბაიტებია_).

---

## Azure Cosmos DB (ETag) მოკლე ნიმუში

```csharp
var container = cosmosClient.GetContainer("app", "orders");

// Read gets ETag:
var read = await container.ReadItemAsync<OrderDoc>(id, new PartitionKey(id));
var etag = read.ETag;

// Update with optimistic concurrency:
var doc = read.Resource with { Amount = 123m };
var req = new ItemRequestOptions { IfMatchEtag = etag };

try
{
    await container.ReplaceItemAsync(doc, id, new PartitionKey(id), req);
}
catch (CosmosException ex) when (ex.StatusCode == HttpStatusCode.PreconditionFailed)
{
    // concurrency conflict → reread/merge/retry
}
```

---

## რა გავითვალისწინოთ პროდაქშენში

-  **გამოაჩინეთ ვერსია** (ETag/RowVersion) ყველა READ-ში.
    
-  **აიძულეთ If-Match** ყველა ცვლილებაზე (412/428 semantics).
    
-  **ჰენდლინგი UI-ში:** კონფლიქტის დროს აჩვენეთ დიფი/„Reload & Merge“.
    
-  **Retry with backoff** მხოლოდ იმ შემთხვევაში, თუ merge ავტომატურია (მაგ., idempotent PATCH).
    
-  **Audit trail** — ვინ/როდის შეცვალა.
    
-  **სარისკო რესურსებზე** განიხილეთ „compare-and-swap“ სემანტიკა მხოლოდ ნაწილობრივ ველებზე (PATCH + field-level concurrency).
    

თუ გინდა, შემოგೀತოვებ პატარა „ConcurrencyMiddleware“-ს (ფილტრს) და FluentValidation წესს, რომ Update commands აუცილებლად მოიტანდნენ `RowVersion`-ს/`If-Match`-ს, პლუს integration ტესტებს `DbUpdateConcurrencyException`-ისთვის.