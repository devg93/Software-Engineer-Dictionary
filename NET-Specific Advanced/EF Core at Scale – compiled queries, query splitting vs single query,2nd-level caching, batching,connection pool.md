

# მთავარი გამოკლება (ერთი-მოთხრობით)

- **ჩვეულებრივი LINQ**: სადაც დაძახებ (e.g. `db.Users.Where(...).FirstOrDefaultAsync()`) — EF Core ყოველ ჯერზე გარდაქმნის (translate) LINQ expression-ს SQL-ად. ამ პროცედურას აქვს CPU-სამუშაო (expression parsing + query model building + SQL generation) — განსაკუთრებით ღირებულია, თუ ბევრი მოთხოვნაა და იგივე shape ხშირად მეორდება.
    
- **Compiled query** (`EF.CompileQuery` / `EF.CompileAsyncQuery`): ამ ტრანსლაციის ნაბიჯს აკეთებ **ერთჯერ** (პროცესის თქვენთვის) — EF ქმნის delegate-ს (ფუნქციას) რომელიც უკვე KNOWS როგორ აიღოს SQL-ი. შემდეგ ყოველ კლანზე უბრალოდ იძახებ იმას delegate-ს — არ აკეთებს expression→SQL ტრანსლაციას მეორედ. რეალური DB execution რჩება იგივე: compiled query _არ კოპირებს_ SQL execution-ს; ის მხოლოდ ამცირებს **კლიენტის (client-side) overhead**-ს.
    

# რა იცვლება და რა დარჩება ნაწილობრივ იგივე

- შეიცვლება: **კლიენტური CPU ხარჯი (translation)**, allocation-ები (expression trees) და latency თითო მოთხოვნაზე (მიმზიდველად p50/p95-ზე).
    
- არ იცვლება: **DB ფუნდამენტური execution** — SQL ივალდებულება და DB-side execution time არ უქვეითდება მხოლოდ compiled query–თ (მხოლოდ თუ DB plan/cache და სხვა მცენარეებიც მნიშვნელოვნად არ პატარავ).
    
- SQL Server-ის plan cache განსხვავდება: DB-ზე მაინც მიდის parameterized SQL და DB caches its plan independently — compiled query არ არეგულირებს DB-ს plan caching.
    

# როგორ მუშაობს პრაქტიკაში — step-by-step

1. ჩვეულებრივი LINQ: ყოველ `dbContext.Users.Where(...).FirstOrDefault()` => EF ააშენებს expression tree და გადაიყვანს SQL-ად **ყოველ გადაწყევ**; translation ტკბილობს CPU და allocations.
    
2. compiled query: ერთხელ ამონტაჟებ `EF.CompileQuery(...)` და შეინახავ `static readonly` delegate-ში. შემდეგ თითოივე request მხოლოდ იძახებ delegate-ს (გაეცემა DbContext და პარამეტრები) — translation დრო აღარ არის.
    

# რატომ მოგგები: როდის სარგებლობაშია დიდი განსხვავება

- დაბალი-ლატენსი, **high QPS** endpoints (მაგ., auth token lookup middleware — ყველა request ისტუმრებს DB-ს).
    
- პატარა, სწრაფი queries სადაც translation ძალზე მნიშვნელოვნად წონის DB execution-ზე (ანუ SQL სწრაფია, მაგრამ შემდეგკი translation იყენებს მნიშვნელობას).
    
- ბევრ კონტექსტში განმეორებადი shape (ფილტრები და projection არასდროს იცვლება).
    

# როდის არ ღირდა

- თუ query shape არის დინამიკური (runtime filters, dynamic sorting, conditional Includes) — compiled query არ ჯდება (ყოველჯერ ახალ shape სჭირდება).
    
- თუ კითხვა იშვიათია (one-off), translation cost არ წამგები — კომპილაცია სავსებით ზედმეტია.
    
- თუ მთავარი ჭარბი ხარჯი DB execution time-ა (ძალიან მძიმე join/agg) — compiled query არ შეცვლის ამას.
    

# მცირე ტექნიკური მნიშვნელობა (practical caveats)

- **საკუთარი delegate** უნდა იყოს `static readonly` ან სხვაგან, რომ არ გადააკეთო და არ გამოიწვიო GC/რეკომპილაცია.
    
- Compiled delegate იღებს `DbContext` როგორც პარამეტრს — არ ინახავს/კერავს DbContext-ს თავის სტეიტში (Thread-safe).
    
- აქვს **sync** და **async** ვარიანტები: `EF.CompileQuery` (sync delegate) და `EF.CompileAsyncQuery` (async Task delegate + CancellationToken). აი იჩქარე async, თუ endpoint–ები იყენებენ async DB calls.
    
- გამოიყენე `AsNoTracking()` თუ არ გჭირდება tracking — მეტი performance synergy.
    
- არ იჩრდილო დაეყრდნო compiled queries როგორც ერთი რეცეპტი — combine them with caching/other optimizations where it makes sense.
    

# მოკლე მაგალითი (ჩვეულებრივი vs compiled) — კონცეპტუალური

- ჩვეულებრივი (მაგალითად):
    
    ```csharp
    var user = await db.Users.AsNoTracking().FirstOrDefaultAsync(u => u.Token == token);
    ```
    
    — EF-ს მოუწევს expression→SQL translation თითო შეთხვევაზე.
    
- compiled:
    
    ```csharp
    static readonly Func<AuthDbContext,string,User?> GetByToken =
        EF.CompileQuery((AuthDbContext db, string token)
            => db.Users.AsNoTracking().FirstOrDefault(u => u.Token == token));
    
    var user = GetByToken(dbContext, token);
    ```
    
    — translation შესრულებულია ერთხელ, შემდეგ კლიენტის ნაწილი გაჩერდა.
    

# რა მოგცემს ხელში (სურათი)

- პატარა, ხშირი კითხვების latency მცირდება; CPU spikes შემცირდება პიკის დროს; GC/allocs მცირდება.
    
- არ შეცვლის DB execution heavy cases; არ შეცვლის I/O-bound time.
    

# სწრაფი პატარა ჩეკლისტი — უნდა ვადებდე compiled query?

- ეს კითხვა იძახება ≥ სარბიელი RPS (მტავრია) და იგივე shape? — კი.
    
- ტრასნლაციის წილი არის measurable (profile) და არ უმნიშვნელო? — კი.
    
- არ არის დიდი runtime-დრონმადი filters/ordering? — თუ არა, მიდი.
    
- დაიწყე 1–2 hottest queries-ით, მეტრიკები (p95, CPU) before/after და გაფართოე.
    

---

===============================================================კარგი — ვჭრი პირდაპირ და პრაქტიკულად: ყველაფერი რაც უნდა იცოდე `2nd-level caching`-ზე (Redis/Distributed cache) — სტრატეგიები, trade-offs, invalidation მეთოდები, what to cache / what not to cache, და მოქმედი rollout-ჩეკლისტი — ყველაფერი ქართულად, კონციზური და პრაქტიკული.

# TL;DR

2nd-level cache = აპლიკაციის/ORM-ს მიღმა არსებული ზოგადი cache (Redis, Memcached) რომელსაც მრავალი ინსტანსი იყენებს. სასარგებლოა read-heavy, ტევადი მოთხოვნებისთვის, მაგრამ მთავარი პრობლემაა **invalidaton/consistency** — ეს უნდა დაგეგმნილი იყოს თავიდან.

# რა არის და რატომ გჭირდება

- მეორე-საფეხურიანი cache = დისტრიბუცირებული კეში (არ არის მხოლოდ per-process), რომელიც ინახავს DB-გან ასებულებს (entities, DTOs, query results), რათა შეამციროს DB-ქუერების რაოდენობა და latency.
    
- მიზანი: გააუმჯობესო latency, throughput და შეამცირო DB-ს დატვირთვა, განსაკუთრებით როცა ბევრ წაკითხვას იღებ ერთსა და იმავე მონაცემებზე.
    

# კარგია (use-cases)

- სტატიკური/შეგნებულად იშვიათად მცვლადი მონაცემები: კატალოგი, კატეგორიები, საზღვრები, შტატები/ქვეყნები.
    
- UI widget-ები: დაფები, კონკრეტული read-models, ფილტრები, facet counts (თუ არ არის კრიტიკულად უმკაცრესი სინქრონიზაცია).
    
- Search facets / metadata, რომელიც შეიძლება იყოს eventual-consistent.
    
- Read-side CQRS projections, რომლებსაც არ წერენ ხშირად.
    
- Multi-instance APIs სადაც caching უნდა იყოს საერთო რესურსი.
    

# არ ღირს (anti-use)

- საბანკო სალდენები, მონეტარული/ტრანზაქციული მონაცემები, რეალური ინვენთორი/სტოკი flash-sale დროს — აუცილებელია strong consistency.
    
- პატარა data-set რომელიც ძალიან ხშირად იცვლება და invalidation რთულია.
    

# caching patterns (სტრატეგიულად)

1. **Cache-aside (lazy)** — აპლიკაცია ჯერ cache-ს უთვლის, თუ miss არის → DB → set cache. (ყველაზე გავრცელებული)
    
    - პლუსები: მარტივი, წაკითხვის control.
        
    - მინუსი: race conditions / stampede.
        
2. **Read-through / Write-through** — cache layer intercepts reads/writes; writes go to cache then to DB (sync).
    
    - პლუსი: consistency უკეთესი.
        
    - მინუსი: latency on write, infra complexity.
        
3. **Write-behind (async write-back)** — წერ შენს cache-ში, cache asynchronously flushes to DB.
    
    - პლუსი: write throughput დიდი.
        
    - მინუსი: data loss risk on failure; complexity.
        
4. **Event-driven invalidation** — თავისთავად გამორთეს writes → publish invalidation events (message bus) → all instances evict relevant keys.
    
    - საუკეთესო multi-instance consistency ჩათვლით.
        

# invalidation სტრატეგიები (შეჯიბრი)

- **TTL** (time-to-live): მარტივი, კარგი როცა data იცვლება პერიოდულად.
    
- **Event-driven eviction**: on DB write → publish "invalidate:key" → Redis evict. (პასუხი მაღალი სიმტკიცე)
    
- **Tag/region eviction**: ჯგუფური eviction (category:123:*). საჭირო Redis RedisJSON/Redis-modules ან library Unterstützung.
    
- **Versioned keys**: key includes version/hash of underlying set (e.g., `products:v3:123`) — change version on schema/semantic change.
    
- **Soft invalidation (stale-while-revalidate)**: serve stale value while background refresh runs.
    

# კონსისტენციის მოდელები და trade-offs

- **Strong consistency**: write-through or synchronous invalidation — მაღალი complexity და write latency.
    
- **Eventual consistency**: TTL or event invalidation — lower latency, but reads may be stale for a while.
    
- **Mostly-consistent with quick invalidation**: event invalidation + short TTL fallback — გადაბმა.
    

შენიშვნა: არჩევანი დამოკიდებულია ბიზნეს SLA-ზე: თუ read-after-write აუცილებელია — ნუ გამოიყენებ L2 cache-ს; თუ eventual consistency ჩართულია — გამოიყენე.

# cache key დიზაინი — წესები

- human-readable but deterministic: `<app>:<entity>:<id>` ან `<service>:query:<hash>`
    
- include tenant/environment (multi-tenant): `prod:tenant42:product:123`
    
- include version or schema tag when projection changes often.
    
- keep keys short enough (Redis key memory matters) but meaningful.
    

# cache-stampede და მისი პრევენცია

- **Locking / mutex**: first requester fetches from DB + sets cache; others wait/blocked.
    
- **Request coalescing**: single inflight fetch for same key.
    
- **Randomized TTL (jitter)**: avoid many keys expiring same moment.
    
- **Serve stale while refresh**: return stale and trigger background refresh.
    

# cache warming & preloading

- Preload hot keys on deploy or after eviction (bulk load into cache).
    
- Use background warmup jobs for known hot datasets (top N products).
    
- On deploy, invalidate or bump versions to avoid serving old format.
    

# metrics to track (მართვასთან)

- **Hit ratio** (primary). Target depends on case (>70–80% good).
    
- **Cache latency** (avg, p95).
    
- **DB queries avoided / saved per minute**.
    
- **Stale read incidents** (errors from business logic due to stale data).
    
- **Eviction rate / OOM events** on Redis.
    
- **Cache throughput (ops/sec)** and **memory usage**.
    
- **Invalidation event lag** (time between write and eviction across instances).
    

# monitoring & alerting

- Alert when hit ratio drops, when Redis memory > 80%, when invalidation lag > threshold, or when stale-read incidents increase.
    

# rollout checklist (პროგრესირებული)

1. **Pick pilot endpoints** — low-risk, read-heavy (catalog, lookups).
    
2. **Define invalidation** (TTL or event-driven). Start with TTL + event evict later.
    
3. **Implement cache-aside** initially (simpler).
    
4. **Add stampede protection** (mutex or serve stale).
    
5. **Monitor** hit ratio, DB QPS, staleness incidents.
    
6. **Gradually expand** to other endpoints. Use event-driven invalidation for hot mutable data.
    
7. **Document** keys, versioning, and fallback behavior.
    

# operational pitfalls & mitigation

- **Overcaching** (cache everything) → stale bugs; mitigate: selective caching and strong invalidation.
    
- **Key explosion / memory OOM** → use eviction policies, TTL, and compact keys.
    
- **Stampede** → use locks/jitter.
    
- **Complex invalidation** → prefer event-driven pipeline (message bus) instead of ad-hoc key deletions.
    
- **Testing missing** → create tests for stale reads and invalidation timing.
    
- **Security** → never cache PII without encryption or access controls; segregate tenants.
    

# პრაქტიკული მაგალითები (concrete scenarios)

1. **Product catalog page (good fit)**
    
    - Cache: product list per category, TTL = 5–15 min, event invalidation on product update (publish invalidate for product id and category).
        
    - Stampede: randomized TTL and mutex for refresh.
        
2. **Dashboard widget (good fit)**
    
    - Cache read-model for 30s–2min, serve stale while background refresh. Acceptable eventual consistency.
        
3. **Price & inventory (careful)**
    
    - Price might be cached if business logic allows brief staleness (e.g., price updates rare), but inventory for checkout must be authoritative (DB or atomic service) — read from cache only for display, not to reserve.
        

# მოკლე რჩევები (1-პარაგრაფიანი)

პირველად აირჩიე რამდენიმე არამკაცრი read-heavy endpoint; იმპლემენტირე cache-aside + short TTL (მაგ., 60–300s) + jitter და შეამოწმე hit ratio და stale incidents. თუ მონაცემები დინამიურად იცვლება ან მნიშვნელოვანია დატყვის სიმკაცრე, გადაიტან event-driven invalidation (write → publish invalidate) ან გამოიყენე write-through pattern მხოლოდ იქ, სადაც თქვენ მიდის strong consistency.

# მოკლე Checklist for Decision

- ამ endpoint-ზე read >> write? → cacheable.
    
- read-after-write strict requirement? → ხშირად არ cache.
    
- can we publish invalidation event on write? → yes → use event invalidation.
    
- can we accept small staleness window (seconds/minutes)? → yes → TTL ok.
    
===========================================================
- 