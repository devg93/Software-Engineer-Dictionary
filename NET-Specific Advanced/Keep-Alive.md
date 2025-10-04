
# შეჯამება

**HTTP Keep-Alive** (persistent connections) ნიშნავს, რომ ერთი TCP/TLS კავშირი მრავალ მოთხოვნა-პასუხზე მეორდება, რათა არ გაიხსნას ახალი სოკეტი ყოველ რექუესტზე. ეს ამცირებს latency-ს და CPU/ნეტის overhead-ს. H1-ში ამას მართავდა `Connection: keep-alive/close` ჰედერი; H2/H3-ში persistence ბუნებრივადაა და ეს ჰედერი აღარ გამოიყენება.

(ქიპ-აქაივი (HTTP Keep-Alive) — ეს არის მექანიზმი, რომელიც ერთსა და იმავე TCP (და TLS) შეერთებას „ცოცხლად“ ტოვებს რამდენიმე მოთხოვნა/პასუხის გასატანად. ამით თავიდან ვირიდებთ ყოველ მოთხოვნაზე ახალი კავშირის გახსნის ფასს (TCP 3-way handshake, TLS handshake), ვამცირებთ latency-ს და ვზოგავთ CPU/ქსელურ რესურსებს.)

---

# დეტალურად

## რატომ იყო საჭირო H1-ში

- **HTTP/1.0:** ნაგულისხმევად _არა_ persistent – თითო მოთხოვნაზე ერთჯერადი TCP.
    
- **HTTP/1.1:** ნაგულისხმევად **persistent**; `Connection: close` წყვეტს. ზოგჯერ იყენებდნენ `Connection: keep-alive`-საც explicit ქცევისთვის.  
    **სარგებელი:** ნაკლები TLS handshake, ნაკლები TCP slow-start, უკეთესი latency პატარა რესურსებზე.  
    **ნიუანსი:** პარალელიზაცია მაინც შეზღუდულია (არა-multiplexed) – ხშირად ბრაუზერები რამდენიმე პარალელურ TCP კავშირს ხსნიდნენ იმავე ჰოსტთან.
    

## H2/H3 ეპოქა

- **HTTP/2/3** უკვე **multiplexed** persistent კავშირებია: ერთი კავშირი → ბევრი პარალელური stream.
    
- `Connection: keep-alive/close` ჰედერი **არ გამოიყენება** (და H2/H3-ში ხშირად იგნორიც ხდება); idle დრო/გაწყვეტა მილევადია transport-ის/პინგების/სერვერის ლიმიტებით (e.g., HTTP/2 PING, QUIC keep-alive).
    

## რისკები/ტრედ-ოფები

- **რესურსები:** დიდი რაოდენობის idle persistent კავშირი ახარჯავს ფაილდესკრიპტორებს/მეხსიერებას.
    
- **H1 HoL:** Keep-Alive ვერ ხსნის head-of-line blocking-ს H1-ში; დიდი პასუხი მაინც დააკავებს რიგს.
    
- **LB/Proxy:** შეიძლება ტერმინირებდეს/რიუზდეს კავშირებს – ზედა ფენაში თქვენი აპლიკაცია ამას ვერ დაინახავს.
    

---

# .NET (ASP.NET Core/Kestrel & HttpClient)

## სერვერი — Kestrel

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.WebHost.ConfigureKestrel(k =>
{
    k.Limits.KeepAliveTimeout = TimeSpan.FromSeconds(120); // idle დრო ერთ კავშირზე
    k.Limits.MaxConcurrentConnections = 10_000;            // გლობალური ლიმიტი
    // H2/H3 ჩართვა სჯობს თუნდაც Keep-Alive-ზე მეტის მისაღებად
    k.ListenAnyIP(443, o =>
    {
        o.UseHttps();
        o.Protocols = HttpProtocols.Http1AndHttp2AndHttp3;
        o.Limits.Http2.MaxStreamsPerConnection = 200; // H2 multiplexing ლიმიტი
    });
});

var app = builder.Build();
app.MapGet("/", () => "ok");
app.Run();
```

**რჩევები:**

- H1-ზე: `KeepAliveTimeout` – ნუ გააკეთებ ზედმეტად დიდს; DoS/რესურსის გაჟღინთვას შეუწყობს.
    
- აწონე H2/H3 – იქ persistence+multiplexing გაცილებით ეფექტურია.
    

## კლიენტი — HttpClientFactory (SocketsHttpHandler)

```csharp
services.AddHttpClient("api")
    .ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
    {
        // კავშირების პულინგი == „Keep-Alive“ თანამედროვე სტილში
        PooledConnectionLifetime = TimeSpan.FromMinutes(10), // რეგულარული როტაცია
        PooledConnectionIdleTimeout = TimeSpan.FromMinutes(2), // idle გაწყვეტა
        KeepAlivePingPolicy = HttpKeepAlivePingPolicy.Always,  // .NET 7+
        KeepAlivePingDelay = TimeSpan.FromSeconds(30),
        KeepAlivePingTimeout = TimeSpan.FromSeconds(15),
        MaxConnectionsPerServer = int.MaxValue, // H1 fallback-ზე სასარგებლო
        AutomaticDecompression = DecompressionMethods.GZip | DecompressionMethods.Deflate
    })
    .ConfigureHttpClient(c =>
    {
        c.DefaultRequestVersion = HttpVersion.Version20; // სცადე H2/H3
        c.DefaultVersionPolicy = HttpVersionPolicy.RequestVersionOrHigher;
        c.Timeout = TimeSpan.FromSeconds(30);
    });
```

**რა ხდება აქ:**

- **Pooling** = კავშირების ხელახალი გამოყენება (Keep-Alive) ავტომატურად.
    
- **Ping-ები** ინარჩუნებს კავშირს და ამოავლენს გაწყვეტას ადრე.
    
- **Lifetime/IdleTimeout** — რისკების კონტროლი (LB-ის გვერდში ჩავარდნილი stale კავშირები, NAT timeouts, და ა.შ.).
    

---

# სად ვიყენოთ/რა გავზომოთ

- **API/gRPC:** ყოველთვის persistent + H2/H3.
    
- **High-QPS სერვისი:** გაზომე `connections.active`, `requests.per.second`, `p95/p99 latency`.
    
- **პროქსის მიღმა:** გადაამოწმე, რომ LB არ ტერმინირებს H2-ს და არ გიბრუნებს H1-ს (თორემ multiplexing/keep-alive სარგებელს კარგავ).
    

---

# სწრაფი ჩეკლისტი

-  H2/H3 ჩართული Kestrel-ზე (ALPN/HTTPS).
    
-  გონივარი `KeepAliveTimeout` სერვერზე; არა ზედმეტად დიდი.
    
-  HttpClientFactory + SocketsHttpHandler pooling/ping/lifetime დაყენებული.
    
-  Metrics/Tracing: idle კონექშენები, error rate, retransmits.
    
-  საჭიროებისას `MaxConnectionsPerServer` H1 fallback-ისთვის და `Http2.MaxStreamsPerConnection` H2-ზე.
    
