# ⚙️ BUILD-TIME მოემსახურება 3 მთავარი კომპონენტი

## **1. MSBuild (Build Target Phase)**

➡ Re-run of MSBuild, ამჯერად რეალური Build  
➡ აკეთებს:

- NuGet Restore
    
- Compiled IL linking
    
- deps.json generation
    
- runtimeconfig.json გენერაცია
    
- publish folder შექმნა
    
- trimming
    
- AOT / R2R
    

📌 MSBuild = Build engine

---

## **2. NuGet Client (nuget.exe + dotnet restore)**

➡ ჩამოტვირთავს NuGet პაკეტებს NuGet gallery-დან: