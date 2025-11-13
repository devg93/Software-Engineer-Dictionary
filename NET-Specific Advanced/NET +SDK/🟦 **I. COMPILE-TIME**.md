

### 👉 „როცა კოდი იწერება და უნდა გადაიქცეს IL-ად“

### **რა ხდება ამ ეტაპზე:**

- C# → IL (Intermediate Language)
    
- Syntax check
    
- Type check
    
- Source Generators
    
- NuGet Restore (prepare)
    
- Errors / warnings
    

---

# ⚙️ **COMPILE-TIME მოემსახურება 3 მთავარი ინსტრუმენტი**

## **1. .NET SDK**

➡ თავად საკომპილაციო პლატფორმა  
➡ შეიცავს ყველა აუცილებელ tool-ს:

- `dotnet` CLI
    
- Roslyn compiler
    
- IL generator
    
- Source Generator engine
    
- NuGet resolver
    
- MSBuild scripts (targets & props)
    

📌 **SDK = Compile-Time Platform**  
❗ **Runtime არაა საჭირო ჯერ**

---

## **2. Roslyn Compiler (csc.exe / roslyn.dll)**

➡ რეალურად აკეთებს კომპილაციას C#-იდან IL-ში  
➡ ეშვება SDK-ს შიგნით

Roslyn:

- ქმნის Syntax Tree-ს
    
- Semantic Model
    
- გააკეთებს Type Checking-ს
    
- Emits: