

**Service Discovery** = „სახელით დარეკვა“.  
სერვისებს აღარ იმახსოვრებ IP:პორტებს. წერ `http://orders` და სისტემა თვითონ პოულობს ახლა სად და რომელ კონტეინერში chạy-ობს `orders`.

# რატომ გვჭირდება

- კონტეინერები ხშირად იცვლიან IP-ს (რესტარტი, scale up/down).
    
- შეიძლება ერთი სერვისი რამდენიმე ეგზემპლარად გარბოდეს.
    
- გვინდა ჯანსაღ/ცოცხალ ინსტანცზე წასვლა, არა მკვდარზე.
    

# შენს ქეისში (5 მიკროსერვისი Docker-ში)

Docker-ს უკვე აქვს **ჩაშენებული discovery** იმავე ქსელზე:  
**service name → კონტეინერის(ების) მისამართი.**  
ანუ `api`-დან შეგიძლია დაუძახო `http://orders:8080` და Docker DNS მიანიშნებს orders-ის კონტეინერს.

### მინიმალური `docker-compose.yml`

```yaml
version: "3.9"
services:
  api:
    build: ./api
    depends_on: [orders, payments]
    networks: [appnet]

  orders:
    build: ./orders
    ports: ["8080:8080"]
    networks: [appnet]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 10s
      timeout: 2s
      retries: 3

  payments:
    build: ./payments
    networks: [appnet]

networks:
  appnet: {}
```

### .NET მხრიდან (სახელით დარეკვა + retry/timeout)

```csharp
builder.Services.AddHttpClient("orders", c =>
    c.BaseAddress = new Uri("http://orders:8080")) // service name, არა IP
  .AddTransientHttpErrorPolicy(p => p.WaitAndRetryAsync(3, i => TimeSpan.FromMilliseconds(100 * i)))
  .AddPolicyHandler(Policy.TimeoutAsync<HttpResponseMessage>(TimeSpan.FromSeconds(3)));
```

ასე `api`-დან `orders`-ს ყოველთვის იპოვი—even თუ მისი IP შეიცვალა.

### Scale/Multiple replicas

```bash
docker compose up --scale orders=3 -d
```

Compose-ზე (user-defined network) სერვისის სახელი DNS round-robin-ით გადಮಕათავსებს მოთხოვნებს 3 კონტეინერს შორის.

# როდის გჭირდება დამატებითი Registry (Consul/Eureka) ან Kubernetes

- თუ კონტეინერები **რამდენიმე ჰოსტზე**ა გაფანტული (Compose अकेლა ვერ მართავს).
    
- გჭირდება **მძლავრი health-routing/mTLS/როლინგ-დეპლოი** — Kubernetes-ის `Service` და Ingress აკეთებს discovery+LB-ს.
    
- გინდა **გარე მომხმარებელს** მიაწოდო ერთიანი stable მისამართი მრავალი region/ჰოსტისთვის.
    

# მოკლე ჩეკლისტი (პროდაქშენი)

-  **სახელით დაძახება** (`http://service-name:port`) — ნუ hardcode-ავ IP-ებს.
    
-  **Health checks** (`/health`) Compose/K8s-ისთვის.
    
-  **HttpClientFactory + Polly** — retry, timeout, circuit breaker.
    
-  **Observability** — trace IDs (OpenTelemetry), ლოგები.
    
-  თუ ერთ ჰოსტზე აღარ ჩერდება სისტემა → გადადი **Kubernetes Service**-ებზე ( discovery “უბრალოდ მუშაობს”).
    

თუ გინდა, ჩამიწერე შენი services/პორტები და დაგიწერ ზუსტად შენს `docker-compose.yml`-ზე მორგებულ `Program.cs` რეგისტრაციებს (HttpClient-ები, Polly პოლიტიკები, health endpoints).