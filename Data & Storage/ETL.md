ძალიან კარგი შეკითხვა ტუღუშია ⚡

**ETL = Extract → Transform → Load**  
ანუ მონაცემების გადატანისა და მომზადების პროცესი ოპერაციული OLTP ბაზიდან ან სხვა წყაროებიდან ანალიტიკური OLAP გარემოსთვის (Data Warehouse).

---

### 📌 1. **Extract (ამოღება)**

- მონაცემების წამოღება **OLTP ან სხვა წყაროებიდან** (SQL Server, API, CSV, IoT device, Kafka).
    
- მოთხოვნაა, რომ მონაცემი ამოვიღოთ **სწორად და სრულად**, მაგრამ არა აუცილებლად ოპტიმიზებული ფორმით.
    
- .NET მაგალითი:
    
    ```csharp
    var orders = await _oltp.Orders
        .Where(o => o.CreatedAt >= lastSyncTime)
        .ToListAsync();
    ```
    

---

### 📌 2. **Transform (გადაყვანა/გაწმენდა)**

- ამოღებული მონაცემის **გადამუშავება**:
    
    - **გაწმენდა** – დუბლიკატების მოცილება, null ველების მართვა.
        
    - **სტანდარტიზაცია** – თარიღების ფორმატირება, ვალუტის კონვერტაცია.
        
    - **Aggregation** – მაგ. შეკვეთების დაჯამება დღეზე/კვარტალზე.
        
    - **Schema Mapping** – გადაყვანა OLTP → OLAP სტრუქტურაში.
        
- .NET მაგალითი:
    
    ```csharp
    var factSales = orders.Select(o => new FactSale
    {
        CustomerId = o.CustomerId,
        Amount = o.Total,
        DateKey = o.CreatedAt.Date,   // კონვერტაცია დღიურ Key-ზე
        ProductKey = MapProduct(o.ProductId)
    }).ToList();
    ```
    

---

### 📌 3. **Load (ჩატვირთვა)**

- უკვე ტრანსფორმირებული მონაცემების ჩაწერა **Data Warehouse-ში (OLAP)**.
    
- ჩვეულებრივ batching / bulk insert-ს იყენებენ (EF Core Bulk Extensions, SqlBulkCopy).
    
- .NET მაგალითი:
    
    ```csharp
    await _olap.FactSales.AddRangeAsync(factSales);
    await _olap.SaveChangesAsync();
    ```
    

---

### 🔄 მოკლედ ETL flow:

```
[ OLTP DB / Sources ]
        │
        ▼
   Extract (SQL/API/Files)
        │
        ▼
   Transform (Clean, Map, Aggregate)
        │
        ▼
   Load (Star Schema → OLAP Warehouse)
        │
        ▼
[ BI Tools / Analytics ]
```

---

### ⚙️ Production Considerations

- **Schedule**: ETL შეიძლება იყოს nightly job (SQL Agent, Hangfire, Quartz.NET) ან near-real-time (Kafka, RabbitMQ).
    
- **Data Quality**: აუცილებელია Validation Rules.
    
- **Incremental Load**: მხოლოდ ახალი ან შეცვლილი ჩანაწერები გადმოგვაქვს (CDC – Change Data Capture).
    
- **Error Handling**: ცალკე Log/Error Table → რომ ვიცოდეთ რა ჩავარდა.
    
- **Performance**: დიდ მასივზე → **bulk insert** ან **partitioned load**.
    

---
