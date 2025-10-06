# მოკლე შესავალი

ქვემოთ დალაგებულია **ლეირებად** (L0–L3) ყველა მნიშვნელოვანი ქეშირების მექანიზმი, თითო ლეირზე: **როგორი არის**, **რომელ შემთხვევებში უნდა გამოიყენო**, **ტრეიდ-ოფები** და **ვისაჩქარებელი/მომგვარებელი გზები**. ბოლოს ნახავ .NET/C# პრაქტიკულ მცირე კოდი-ნიმუშებს (IMemoryCache და Redis) და ქართულ ტერმინთა ლექსიკონს.

---

# L0 — Client / CDN / HTTP Cache

**რა არის:** სტატიკური რესურსები (JS/CSS, images) და HTTP response-ები CDN-ზე ან ბრაუზერზე კეშირება.  
**როდის გამოიყენო:** სტაბილური სტატიკური ფაილები და საჯარო read-heavy endpoints.  
**ტრეიდ-ოფი:** სწრაფი პასუხი, თუმცა ინვალიდაცია დაგვიანებით (purgе საჭირო).  
**მოგვარება:** `Cache-Control`, `ETag`, CDN purge API, `stale-while-revalidate`.

---

# L1 — In-Process / Local Cache (IMemoryCache)

**რა არის:** აპლიკაცია-პროცესში არსებული ძალიან სწრაფი cache (პამპლინგი მეხსიერებაში).  
**როდის:** როცა გჭირდება გამორჩეულად დაბალი ლატენცია single-node პირობებში (computed results, small objects).  
**პროც/კოდი (.NET):**

```csharp
// Program.cs
builder.Services.AddMemoryCache();

// გამოყენება
public class ProductLocalCache
{
    private readonly IMemoryCache _cache;
    public ProductLocalCache(IMemoryCache cache) => _cache = cache;

    public ProductDto? GetOrCreate(int id, Func<ProductDto?> factory)
    {
        string key = $"product:{id}:v1";
        return _cache.GetOrCreate(key, entry =>
        {
            entry.SlidingExpiration = TimeSpan.FromMinutes(2);
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
            return factory();
        }) as ProductDto;
    }
}
```

**ტრეიდ-ოფი:** არ სკალდება multi-node; მონაცემი წყვეტაზე იშლება.  
**მოგვარება:** გამოიყენე L1 როგორც მოკლე TTL-იანი fast layer და მიამაგრე L2-ს.

---

# L2 — Distributed Cache (Redis / Memcached)

**რა არის:** საერთო cache მრავალ აპლიკაციურ ინდუსტრიაზე — Redis ყველაზე გავრცელებულია.  
**როდის:** თუ გაქვს რამდენიმე ნოდი ან გჭირდება საერთო cache across instances.  
**პროდუქტიული რჩევა (.NET):**

```csharp
// Program.cs
builder.Services.AddStackExchangeRedisCache(opts =>
{
    opts.Configuration = builder.Configuration["Redis:ConnectionString"];
    opts.InstanceName = "myapp:";
});
```

**მაგალითი — L1+L2 Cache-Aside (Handler):**

```csharp
// მოკლე მაგალითი (ლოგიკა): L1(IMemoryCache) -> L2(IDistributedCache) -> DB
// დეტალური Handler ადრე მივეცი, აქ კონცეპტუალურია.
```

**ტრეიდ-ოფი:** ქსელური ლატენცია, კლასტერის ღირებულება; თუმცა სკალირებადია.  
**მოგვარება:** combine L1 short TTL + L2 medium TTL, გამოიყენე binary serialization (MessagePack) და მონიტორინგი (hits/misses).

---

# L3 — DB-side caching / Materialized Views / Read Replicas

**რა არის:** რთული შეკითხვების წინასწარ გამოთვლა DB-ში ან read-replica გამოყენება.  
**როდის:** როცა კითხვები ძალიან მძიმეა (aggs, joins) და кешის ინვალიდაცია გულდასმით კონტროლდება.  
**ტრეიდ-ოფი:** კონსისტენციის გამოწვევები (replication lag), რთული refresh ლოჯიკა.  
**მოგვარება:** CDC, event-driven invalidate ან scheduled refresh.

---

# ძირითადი პატერნები (Patterns) — ქართული ახსნა, ტრეიდ-ოფები და mitigations

### 1) Cache-Aside (Lazy populate)

**რა ხდება:** აპლიკაცია ჩეკავს cache-ს → MISS → DB → შევსება cache-ში → მონაცემის დაბრუნება.  
**პროზ:** სრული კონტროლი აპლიკაციაზე; მარტივი სადათ.  
**კონს:** cache stampede (როცა ბევრია concurrent MISS).  
**მოგვარება:** keyed semaphore (single node), RedLock (multi-node), jittered TTL, negative cache.

### 2) Read-Through / Write-Through

**რა ხდება:** cache provider-ი ავტომატურად იტვირთავს DB-დან; write-through-ში თითო write-ზე cache წერდება და მერე DB-ში synchronously.  
**პროზ:** აპლიკაციური ლოგიკა მარტივდება.  
**კონს:** write latency მატულობს; provider-ზე დამოკიდებულება.  
**მოგვარება:** მხოლოდ როცა provider მხარს უჭერს და latency მნიშვნელოვანია.

### 3) Write-Behind (Async write-back)

**რა ხდება:** გადის სწრაფად რეისპონს cache-ში; DB-ში წერის გარეშე/фонში.  
**პროზ:** სწრაფი პასუხი.  
**კონს:** data loss რისკი; რთული retry/confirm.  
**მოგვარება:** durable queue (Kafka/RabbitMQ), ack/transaction log.

### 4) Stale-while-revalidate (Serve stale, refresh background)

**რა ხდება:** აბრუნებს stale მნიშვნელობას და ფონში გაახლებს cache-ს.  
**პროზ:** latency ძალიან დაბალი; DB нагрузка ნაკლები.  
**კონს:** კლიენტს შეიძლება მოუვიდეს მწარე stale მონაცემი.  
**მოგვარება:** გამოიყენე სადაც მცირე staleness მისაღებია.

### 5) Negative Caching

**რა ხდება:** თუ DB-ში არაფერი არის — შენახავ “null” მოკლე TTL-ით.  
**მიზანი:** ბოტების/მცდარი რანდომ მოთხოვნებისგან DB-ის დაცვა.

### 6) Request Coalescing / Singleflight

**რა ხდება:** ბევრი concurrent request ერთი DB read-ით იხსნება.  
**Single-node:** KeyedSemaphore.  
**Multi-node:** Redis lock (SET NX PX) ან RedLock.

---

# Invalidation სტრატეგიები (როგორ ვოკინვალიდირებთ cache-ს)

1. **Explicit delete on write** — update/delete → DEL key. მარტივი მაგრამ race conditions შეიძლება იყოს.
    
2. **Key versioning (recommended)** — `product:123:v{version}`; ვერსია ინკრემენტდება update-ზე.
    
3. **Pub/Sub** — write → publish `invalidate:product:123`; ყველა instance იწვევს L1 purge.
    
4. **TTL** — combine with above for safety.
    
5. **Eventual Consistency / CQRS** — command side updates, read model async refresh.
    

---

# ოპერაციული და უსაფრთხოების ნოტები

- Redis უნდა იდგეს VPC-ში; გამოიყენე AUTH + TLS + ACL.
    
- ნუ კეშავ PII-ს plaintext-ად (encrypt თუ საჭიროა).
    
- მონიტორინგი: `cache.hits`, `cache.misses`, `evictions`, `latency`.
    
- შეანარჩუნე Redis persistence (AOF/RDB) თუ საჭიროა durability.
    

---

# ტერმინების ლექსიკონი (ქართულად, მოკლედ)

- **TTL** — რამდენ ხანს ინახება key-ი cache-ში.
    
- **Cache-Aside** — აპი თავად ამავსებს cache-ს (lazy).
    
- **Read-Through** — cache provider იტვირთავს DB-დან.
    
- **Write-Through** — write-ზე cache წერდება და მერე DB-ში sync.
    
- **Write-Behind** — write-ს cache-ში ინახავ, DB-ში ასინქრონულად წერს.
    
- **Stampede** — უამრავი concurrent MISS და DB-ზე ერთდროული ტრაფიკის აფეთქება.
    
- **Negative cache** — არარსებული ობიექტის short TTL cache.
    
- **RedLock** — Redis-ზე დაფუძნებული distributed lock ალგორითმი.
    
- **Key versioning** — key-ში ვერსიის ჩართვა invalidate-ის გასამარტივებლად.
    
- **Jitter** — TTL-ის მცირე შემთხვევითი ცვლილება avalanche თავიდან ასარიდებლად.
    
- **L1 / L2** — პირველი (local) და მეორე (distributed) cache ლეველები.
    

---

# პრაქტიკული მოკლე ჩეკლისტი (რა გააკეთო ახლავე)

-  განსაზღვრე L1 vs L2 keys და TTLs (L1 << L2).
    
-  დანიშნე key naming + versioning policy.
    
-  აუშვი metrics (hits/misses/evictions).
    
-  დააინსტრუმენტირე stampede mitigations (locks / refresh-ahead).
    
-  გამოიყენე negative cache და input validation ბოტებისგან დასაცავად.
    
-  serialization: MessagePack/Protobuf დიდი ობიექტებისთვის.
    
-  load-test (k6) და გაზომე DB QPS before/after.
    

---

# გრძელი მაგალითი (მოხერხებული two-level მეთოდი) — კონცეფტური კოდი

```csharp
// L1: IMemoryCache (short), L2: IDistributedCache (Redis, medium)
public async Task<T?> GetCached<T>(string key, Func<Task<T?>> fetch, TimeSpan l1, TimeSpan l2, CancellationToken ct)
{
    if (_memory.TryGetValue(key, out T local)) return local;
    var cached = await _distributed.GetStringAsync(key, ct);
    if (!string.IsNullOrEmpty(cached)) {
        var dto = JsonSerializer.Deserialize<T>(cached)!;
        _memory.Set(key, dto, l1);
        return dto;
    }

    // simple distributed lock (psuedo)
    var lockKey = $"{key}:lock";
    if (await AcquireDistributedLock(lockKey)) {
        try {
            // double-check
            cached = await _distributed.GetStringAsync(key, ct);
            if (!string.IsNullOrEmpty(cached)) { var dto = JsonSerializer.Deserialize<T>(cached)!; _memory.Set(key, dto, l1); return dto; }
            var fresh = await fetch();
            var serialized = JsonSerializer.Serialize(fresh);
            await _distributed.SetStringAsync(key, serialized, new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = l2 }, ct);
            _memory.Set(key, fresh, l1);
            return fresh;
        } finally { await ReleaseDistributedLock(lockKey); }
    }

    // თუ lock არ აიღეთ — დაბრუნდა stale ან small wait+retry
    await Task.Delay(50, ct);
    return await GetCached<T>(key, fetch, l1, l2, ct);
}
```

(შენიშვნა: ოპერაციულ production-ში გამოიყენე კარგად გადამუშავებული RedLock ბიბლიოთეკა და უსაფრთხო release.)

---

