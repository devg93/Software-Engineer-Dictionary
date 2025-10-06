

Redis scheduler — ეს არის გზა (მექანიზმი), სადაც პატარა სამუშაოები (jobs) ინახება Redis-ში მონიშნული დროთი და სპეციალური "ვორქერი" (პროსესს) მათ ასრულებს მაშინ, როცა დანიშნული დრო დგება.

### როგორ მუშაობს (შეზოლად):

1. შენ დგავ შენახავ სამუშაოს Redis-ის უჯრაში (მაგ. `ZSET`) და მასთან ვაყენებ „რანი ატ“ (runAt) — timestamp.
    
2. ვორქერი პერიოდულად იხედება და იპოვის სულისკვეთებულ (due) სამუშაოებს (რანი ატ <= ახლა).
    
3. ვორქერი ასრულებს მათ და წაშლის ჩანაწერს ან გადაადგილებს dead-letter ყუთში თუ წარუმატებელი იყო.
    

### მარტივი მაგალითი (რეალური გამოყენება)

- დაგეგმე "გამოხსენების გაგზავნა" ხვალ 10:00-ზე ან მოარიდე გადახდის retry 30 წამში.
    

C#-სტილით (ერთი ასო):

```csharp
await scheduler.ScheduleAsync("send-email-123", "{to: 'user@example.com'}", DateTime.UtcNow.AddMinutes(10));
```

### როდის აქვს აზრი

- კარგია: შეტყობინებები, email/SMS-delay, cache prewarm, მარტივი retry ლოჯიკა.
    
- არ არის კარგი: კრიტიკული ტრანზაქციები (ფინანსური), რთული მრავალსტეპიანი სამუშაოები — აქ უკეთესია DB-backed ან სპეციალური სისტემები (Hangfire/Quartz).
    

### მოკლედ — 1 წინადადებით

Redis scheduler = სწრაფი და მარტივი გზა დროით-დაგეგმილი სამუშაოების შესანახად და შესრულებისთვის, მაგრამ თუ საჭიროა მაღალი durability ან რთული retry semantics — გამოიყენე უფრო ძლიერი ეიდაბე/ქუ-პროდუქტი.

### ჩემი რეკომენდაცია (typical .NET production)

- თუ გინდა „სწრაფად და კონტროლირებადად“: **BackgroundService + ZSET + Lua + processing-set + dead-letter + metrics**.
    
- თუ გინდა უკეთესი multi-instance failover და ნაკლები საკუთარ ლოჯიკაზე ზრუნვა: **Redis Streams + consumer groups**.
    
- თუ გინდა GUI + retry/dashboard და ნაკლები საკუთარი ჯაგინგი: **Hangfire** (Redis/SQL storage).












# BackgroundService + ZSET + Lua + processing-set + dead-letter (სრულფასოვანი მაგალითი)

## კონცეფცია მოკლედ

1. Job მაკრო: პროდუქტის `job:{id}` (Hash) + `scheduler:jobs` (ZSET score = runAtUtc ms).
    
2. Worker Lua-ს გამოყენებით ატომურად იღებს `due` jobId-ებს (ZRANGEBYSCORE + ZREM), ტოვებს job payload-ს Hash-ში.
    
3. Worker მარყუჟობს `processing:{id}` (hash/SET) ან `processing:zset` მეტი metadata-თ (claimedAt, workerId) — ჭირდება reclaim logic.
    
4. ჩაწერე failures → `dead-letter` ZSET ან LIST.
    
5. Metrics: counters (processed, failed, queue_length) — აღწერილი როგორც interface (Prometheus/StatsD შესაძლებელია).
    

---

## პროექტის სტრუქტურა (შაბლონი)

- Api (Controller) → Schedule endpoint
    
- Application (ISchedulerService)
    
- Infrastructure.Redis (RedisScheduler + RedisWorker)
    
- Workers (RedisScheduledWorker : BackgroundService)
    
- Handlers (IJobHandler / JobProcessor)
    

---

### 1) DTO + Interface

```csharp
public record ScheduleRequest(string JobId, string Payload, DateTime RunAtUtc, int Attempt = 0);

public interface ISchedulerService
{
    Task EnqueueAsync(ScheduleRequest request, CancellationToken ct = default);
}
```

### 2) RedisScheduler — რეპოზე/ინფრასტრუქტურა

```csharp
using StackExchange.Redis;
using System.Text.Json;

public class RedisScheduler : ISchedulerService
{
    private readonly IDatabase _db;
    private const string ZKey = "scheduler:jobs";
    private const string JobHashPrefix = "job:";

    public RedisScheduler(IConnectionMultiplexer mux)
    {
        _db = mux.GetDatabase();
    }

    public async Task EnqueueAsync(ScheduleRequest request, CancellationToken ct = default)
    {
        var jobKey = JobHashPrefix + request.JobId;
        var jobJson = JsonSerializer.Serialize(request);
        // store payload in hash for inspection and retries
        await _db.HashSetAsync(jobKey, new HashEntry[] {
            new ("payload", jobJson),
            new ("attempt", request.Attempt),
            new ("createdAt", request.RunAtUtc.ToString("o"))
        });
        var score = new DateTimeOffset(request.RunAtUtc).ToUnixTimeMilliseconds();
        await _db.SortedSetAddAsync(ZKey, request.JobId, score);
    }
}
```

### 3) Lua სკრიპტი — ატომური pop (ZRANGEBYSCORE + ZREM)

(პირველად კოდი მოემზადება და Load გავაკეთებთ Start-ზე)

```lua
-- ARGV[1] = nowMs
-- ARGV[2] = limit
local key = KEYS[1]
local now = tonumber(ARGV[1])
local limit = tonumber(ARGV[2])
local ids = redis.call('ZRANGEBYSCORE', key, '-inf', now, 'LIMIT', 0, limit)
if (#ids == 0) then return {} end
for i=1,#ids do
  redis.call('ZREM', key, ids[i])
end
return ids
```

### 4) RedisScheduledWorker (BackgroundService) — core loop

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using StackExchange.Redis;
using System.Text.Json;

public class RedisScheduledWorker : BackgroundService
{
    private readonly IDatabase _db;
    private readonly IConnectionMultiplexer _mux;
    private readonly IJobProcessor _processor; // business logic
    private readonly ILogger<RedisScheduledWorker> _logger;
    private readonly LoadedLuaScript _popScript;
    private readonly string _workerId = Guid.NewGuid().ToString();
    private const string ZKey = "scheduler:jobs";
    private const string JobHashPrefix = "job:";
    private const string ProcessingKey = "processing";
    private const string DeadLetterKey = "dead-letter";

    private const int BatchSize = 20;
    private readonly TimeSpan PollInterval = TimeSpan.FromMilliseconds(500);

    public RedisScheduledWorker(IConnectionMultiplexer mux, IJobProcessor processor, ILogger<RedisScheduledWorker> logger)
    {
        _mux = mux;
        _db = mux.GetDatabase();
        _processor = processor;
        _logger = logger;

        var server = mux.GetServer(mux.GetEndPoints(true)[0]);
        var lua = @"
local key = KEYS[1]
local now = tonumber(ARGV[1])
local limit = tonumber(ARGV[2])
local ids = redis.call('ZRANGEBYSCORE', key, '-inf', now, 'LIMIT', 0, limit)
if (#ids == 0) then return {} end
for i=1,#ids do redis.call('ZREM', key, ids[i]) end
return ids
";
        _popScript = LuaScript.Prepare(lua).Load(server);
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Worker started. id={WorkerId}", _workerId);
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                var nowMs = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
                var result = (RedisValue[]?)await _db.ScriptEvaluateAsync(_popScript, new RedisKey[] { ZKey }, new RedisValue[] { nowMs, BatchSize });

                if (result != null && result.Length > 0)
                {
                    foreach (var rv in result)
                    {
                        var id = rv.ToString();
                        try
                        {
                            // Add to processing set with metadata (worker + claimedAt)
                            var processingEntry = JsonSerializer.Serialize(new { worker = _workerId, claimedAt = DateTimeOffset.UtcNow.ToString("o") });
                            await _db.HashSetAsync($"{ProcessingKey}:{id}", new HashEntry[] {
                                new("meta", processingEntry)
                            });

                            var jobKey = JobHashPrefix + id;
                            var payload = (await _db.HashGetAsync(jobKey, "payload")).ToString();
                            var req = JsonSerializer.Deserialize<ScheduleRequest>(payload);

                            // Process
                            await _processor.ProcessAsync(id, req?.Payload ?? payload, stoppingToken);

                            // On success: remove processing + job hash + metrics
                            await _db.KeyDeleteAsync($"{ProcessingKey}:{id}");
                            await _db.KeyDeleteAsync(jobKey);
                        }
                        catch (Exception ex)
                        {
                            _logger.LogError(ex, "Job {JobId} failed, moving to retry/dead-letter", id);
                            // naive retry: increment attempt and reschedule a few seconds later
                            var jobKey = JobHashPrefix + id;
                            var attemptVal = (int)(await _db.HashGetAsync(jobKey, "attempt") ?? 0);
                            attemptVal++;
                            await _db.HashSetAsync(jobKey, "attempt", attemptVal);

                            if (attemptVal > 5)
                            {
                                // send to dead-letter (store payload + lastError + attempt)
                                var payload = (await _db.HashGetAsync(jobKey, "payload")).ToString();
                                var dl = JsonSerializer.Serialize(new { id, payload, attempt = attemptVal, failedAt = DateTimeOffset.UtcNow });
                                await _db.ListLeftPushAsync(DeadLetterKey, dl);
                                await _db.KeyDeleteAsync(jobKey);
                                await _db.KeyDeleteAsync($"{ProcessingKey}:{id}");
                            }
                            else
                            {
                                // re-schedule with backoff (simple)
                                var next = DateTimeOffset.UtcNow.AddSeconds(Math.Pow(2, attemptVal));
                                await _db.SortedSetAddAsync(ZKey, id, next.ToUnixTimeMilliseconds());
                                await _db.KeyDeleteAsync($"{ProcessingKey}:{id}");
                            }
                        }
                    }
                    continue; // immediately poll next batch
                }

                await Task.Delay(PollInterval, stoppingToken);
            }
            catch (OperationCanceledException) { break; }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Worker loop error, sleeping briefly");
                await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
            }
        }
        _logger.LogInformation("Worker stopping.");
    }
}
```

### 5) IJobProcessor — მაგალითი (business handler)

```csharp
public interface IJobProcessor
{
    Task ProcessAsync(string jobId, string payload, CancellationToken ct = default);
}

public class MyJobProcessor : IJobProcessor
{
    private readonly ILogger<MyJobProcessor> _logger;
    public MyJobProcessor(ILogger<MyJobProcessor> logger) => _logger = logger;

    public Task ProcessAsync(string jobId, string payload, CancellationToken ct = default)
    {
        // აქ დაიტვირთება რეალური ლოჯიკა, მაგალითად: send email, call external API და ა.შ.
        _logger.LogInformation("Processing job {JobId} payload={Payload}", jobId, payload);
        return Task.CompletedTask;
    }
}
```

### 6) DI / Program.cs

```csharp
var mux = await ConnectionMultiplexer.ConnectAsync(configuration["Redis:ConnectionString"]);
builder.Services.AddSingleton<IConnectionMultiplexer>(mux);
builder.Services.AddSingleton<ISchedulerService, RedisScheduler>();
builder.Services.AddSingleton<IJobProcessor, MyJobProcessor>();
builder.Services.AddHostedService<RedisScheduledWorker>();
```

### 7) Controller (API შიგნით)

```csharp
[ApiController]
[Route("api/scheduler")]
public class SchedulerController : ControllerBase
{
    private readonly ISchedulerService _scheduler;

    public SchedulerController(ISchedulerService scheduler) => _scheduler = scheduler;

    [HttpPost("enqueue")]
    public async Task<IActionResult> Enqueue([FromBody] ScheduleRequest req, CancellationToken ct)
    {
        if (req == null) return BadRequest();
        await _scheduler.EnqueueAsync(req, ct);
        return Accepted();
    }
}
```

---

## ტესტირება + ოპერაციული რჩევები (პუნქტურად)

- Unit tests: RedisScheduler — რაღაც mocking შეუძლებელია პირდაპირი Redis გარეშე; გამოიყენე integration tests with Testcontainers Redis.
    
- Integration tests: ასევე worker behavior (pop+process+retry+dead-letter) — გამოიყენე ephemeral Redis container.
    
- Monitoring: publish metrics — queue length `ZCARD scheduler:jobs`, processing count `KEYS processing:*` და dead-letter length `LLEN dead-letter`.Expose to Prometheus.
    
- Failover: multi-instance რომ გქონდეს, ქვემოთ წაიკითხე: **risk** — ეს Lua pop approach ატომურად წაშლის zset member, ისე რომ სხვა instance არ მოიპოვებს — კარგია. მაგრამ თუ worker დაიქვე crashed გამწევის პროცესში — job უკვე ამოღებულია და შესაძლოა დაკარგე. ამიტომ საჭიროა სხვა მექანიკა: **claim + pending + visibility** ან გამოიყენე Streams. (ჩვენი code-ში retry არის გამოიყენებული: თუ worker შიგნით exception გამოვიდა, ჩვენ სქემატურად re-schedule ვაკეთებთ — მაგრამ თუ crash დენისამდე მოხდა, job ჩაიკარგება) — ამის გამოსასწორებლად საჭიროა ვერშენშტაბი: არ წაშალო zset პირდაპირ — არამედ move to processing zset და აკიდო timestamp; შემდეგ background reclaim ტასკი დაასხას stale processing entries უკან ან reschedule-ს. (სირთულეა, მაგრამ გასაკეთებელია.)
    

---

# Redis Streams + consumer groups — მოკლე მაგალითი

## როცა იყენებ Streams

- Create stream `XADD jobs * payload <json> runAt <ms>` და consumer group `XGROUP CREATE jobsGroup jobs $ MKSTREAM`.
    
- Consumers `XREADGROUP GROUP jobsGroup consumerId BLOCK 5000 COUNT 10 STREAMS jobs >` — ეს ბლოკინგი wait-ს იძლევა (შეიხვევა polling-ს).
    
- On process: `XACK jobsGroup <id>`; on fail: `XCLAIM` ან requeue.
    

### C# (StackExchange.Redis) snippet

```csharp
// create group (run once)
db.StreamCreateConsumerGroup("jobs", "jobsGroup", "$", createStream: true);

// consume loop
while(!ct.IsCancellationRequested) {
    var entries = db.StreamReadGroup("jobs", "jobsGroup", "consumer-1", ">", count:10, noAck:false);
    foreach(var entry in entries) {
        var id = entry.Id;
        var payload = entry["payload"];
        try {
            await processor.ProcessAsync(id, payload, ct);
            db.StreamAcknowledge("jobs", "jobsGroup", id);
        } catch {
            // don't ACK — will land in PEL, can XCLAIM later
        }
    }
}
```

### რატომ წაირჩიო Streams

- Built-in pending list (PEL) და XCLAIM → ძლიერი failover.
    
- BLOCKING read → არ გჭირდება aggressive polling.
    
- Slightly more complex but robust.
    

---

# Hangfire (GUI + retry/dashboard) — სწრაფი კონფიგურაცია

## წესი

- Add NuGet: `Hangfire.Core`, `Hangfire.AspNetCore`, და storage: `Hangfire.SqlServer` ან `Hangfire.Redis.StackExchange`.
    
- Configure in Program.cs:
    

```csharp
builder.Services.AddHangfire(cfg =>
    cfg.UseSqlServerStorage(configuration.GetConnectionString("HangfireConn"))); // or UseRedisStorage(...)
builder.Services.AddHangfireServer();
```

- Schedule job:
    

```csharp
// delayed
BackgroundJob.Schedule(() => Console.WriteLine("Hello delayed"), TimeSpan.FromMinutes(5));
// recurring
RecurringJob.AddOrUpdate("job-key", () => DoWork(), Cron.Daily);
```

- Hangfire provides dashboard `/hangfire` with retries, failed jobs, delete, requeue, etc.
    

## რატომ Hangfire

- Persistence, dashboard, retries, concurrency limits out-of-the-box.
    
- Trade-off: external dependency + storage overhead.
    

---

# შეჯამება და რჩევა მოკლედ

- თუ გინდა სრულ კონტროლსა და მაღალ throughput-ს — დაიწყე **BackgroundService + ZSET + Lua** და დაამატე: processing-set reclaim loop + metrics + dead-letter. მე ზემოთ წამომთვალა კეთილი საწყისი კოდი.
    
- თუ თავსარიდება Multi-instance failover-ს — **Redis Streams** არის უკეთესი არჩევანი (built-in PEL/XCLAIM).
    
- თუ გინდა მზა UI + retry + persistence — **Hangfire** ყველაზე სწრაფი გზა.
    

---

