
**იდეა:** თითო HTTP მოთხოვნას (და მასთან დაკავშირებულ შიდა ოპერაციებს) მივაბათ უნიკალური იდენტიფიკატორი.  
**რატომ:**

- _Traceability:_ ერთი მომხმარებლის ერთი მოქმედების ყველა ლოგი/ტრეისი ერთად ჩანს.
    
- _Distributed systems:_ სერვის A → B → C ჯაჭვში მარტივად მიყვები „ვინ გამოიწვია ვინ“.
    
- _Support:_ მომხმარებელმა მოგცა CorrelationId – Kibana/Datadog-ში მაშინვე პოულობ მთელ გზას.
    

**როგორ „მიყოლება“ ობიექტებს:**

- ASP.NET Core-ში Middleware აყენებს `X-Correlation-Id` ჰედერს Request/Response-ზე.
    
- Serilog `LogContext` ამ იდს „ამაგრებს“ ყველა ლოგზე ამ თხოვნის ამაორთი.
    
- OpenTelemetry/W3C Trace Context სტანდარტი იყენებს `traceparent`/`tracestate` ჰედერებს; `Activity.Current` შეიცავს TraceId/SpanId-ს, რაც ავტომატურად „მოგვაქვს“ downstream HTTP/gRPC/SQL ინსტრუმენტაციებში.
    
- Message brokers (RabbitMQ/Kafka)-ში CorrelationId გადაგყავს მესიჯის header-ებად, consumer-ში ისევ უკერავ `LogContext`-ს.
### წესები (კონცისი)

- თუ მოვიდა გარე `X-Correlation-Id` — **შენარჩუნე**; თუ არა — **შექმენი**.
    
- **ყოველ response-ში დააბრუნე** იგივე ჰედერი.
    
- **ყველა გამავალ ზარს მიაყოლე** (HTTP/gRPC) + **მესიჯის ჰედერებში ჩასვი** (RabbitMQ/Kafka).
    
- ლოგში ყოველთვის **ქონიე ველი** `CorrelationId` (Serilog `LogContext`).
    
- ერორებზე ალერტის ტექსტშიც ჩაამატე Correlation Id → სწრაფი ძიებისთვის.


### რატომ ფასობს ერთ სერვისშიც

1. **სწრაფი დიაგნოსტიკა:** Support-ს რომ მისწერენ — „დღეს 12:03-ზე ვერ გავაკეთე გადახდა“ — response-ის `X-Correlation-Id`-ით ერთ წამში იპოვი ზუსტად იმ მოთხოვნის ყველა ლოგს.
    
2. **კონკურენტული მოთხოვნები:** ერთსა და იმავე `UserId`-ზე პარალელურად შეიძლება მიდიოდეს რამდენიმე פעולה. `UserId` მარტო ვერ გიშველის; `CorrelationId` უბერთაობს კონკრეტულ მოთხოვნას.
    
3. **რეთრაი/დედლაინი/ქენსელი:** Polly retries, timeout-ები, circuit breaker—ერთი `CorrelationId`-ით შეადარებ ყველა ცდას, რა მოხდა ზუსტად იმ ოპერაციაზე.
    
4. **ბექგრაუნდ დავალებები:** HTTP მოთხოვნამ გააჩინა Hangfire/HostedService job — იმავე `CorrelationId`-ის ტარება log chain-ს ინარჩუნებს.
    
5. **Future-proofing:** ხვალ რომ დაშალო მონოლითი და დაამატო მეორე სერვისი (Payments, Notifications), propagation უკვე გაქვს „ჩონჩხში“.
    
6. **SecOps/Audit:** ინცინდენტის დროს კონკრეტული ქეისის trail სწრაფად იკვრება (და არ „ჩაიკარგება“ noise-ში).