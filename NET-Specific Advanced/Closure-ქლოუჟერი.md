# 🔹 1. Closure = ფუნქცია + გარემო (environment)

💡 *საოპერაციო განმარტება:*

> **Closure** არის ობიექტი, რომელიც შეიცავს:
>
> 1. **კოდის პოინტერს (delegate / function pointer)** — რა უნდა შესრულდეს
> 2. **გარემოს სნაფშოთს (captured environment)** — რომელი ცვლადები იყო ხელმისაწვდომი ამ ფუნქციის შექმნის მომენტში

ანუ closure ≈ `(function pointer, captured variables[])`

---

## 🔹 2. რატომ არის საჭირო closure (ენების დიზაინის დონეზე)

ფუნქციები ობიექტებად რომ გადმოგვცემდეს, ენას უნდა შეეძლოს ამ ფუნქციისთვის გარე კონტექსტის შენახვა.
მაგალითად:

```csharp
int factor = 2;
Func<int, int> doubler = x => x * factor;
```

ფუნქცია `x => x * factor` **არ შეიძლება არსებობდეს მარტო** —
მას სჭირდება ინფორმაცია `factor`-ზე.
აქედან გამომდინარე კომპილერი ქმნის დამხმარე კლასს:

```csharp
class Closure_Display
{
    public int factor;
    public int Multiply(int x) => x * factor;
}
```

👉 ანუ: *function turned into object with environment*.

ესაა closure-ის მექანიზმი — **execution context becomes data structure**.

---

## 🔹 3. მეხსიერების მოდელი: stack → heap uplift

ნორმალური ლოკალური ცვლადი ცხოვრობს stack-ზე:

```
Main() stack frame
└── int factor = 2
```

მაგრამ როცა lambda ამას "იჭერს" (captures),
კომპილერი ამას აყენებს heap-ზე, რომ სიცოცხლე გაგრძელდეს lambda-სთან ერთად:

```
Heap:
  [Closure_Display]
      factor = 2
Stack:
  doubler → reference to Closure_Display
```

ამიტომ:

* closure ცოცხალია მანამ, სანამ მასზე reference არსებობს
* garbage collector ვერ წაშლის captured variable-ს, სანამ delegate არ გაქრება

---

## 🔹 4. Capturing semantics: by reference, not by value

C#–ში captured ცვლადები ინახება **reference–ით**, არა კოპიით.
ამიტომაც ხდება ეს მაგალითი:

```csharp
int i = 0;
Action act = () => Console.WriteLine(i);
i = 42;
act(); // Prints 42
```

🧠 რადგან lambda ინახავს reference-ს `i`–ზე, არა snapshot-ს.

---

## 🔹 5. closure-სა და execution-ის დინამიკა

როცა closure იქმნება, იგი ქმნის:

* Environment object (heap-ზე)
* Delegate რომელიც ამ ობიექტის method-ზე მიუთითებს

**როცა delegate გამოიძახება:**

1. CLR იღებს function pointer-ს
2. იღებს closure object-ს როგორც `this`
3. აწვდის მას execution context-ის სახით

ეს რეალურად კლასიკური `object + method` ინვოკაციაა.

---

## 🔹 6. Advanced concept — closures are “heap-lifted stack frames”

ენების დიზაინში closure ეწოდება **"heap-allocated activation record"**
ანუ როცა ფუნქციამ უნდა გადარჩეს თავისი stack frame-ის მიღმა,
კომპილერი მის activation record-ს (ლოკალურ ცვლადებს)
გადმოიტანს heap-ზე როგორც ობიექტს.

ესაა სწორედ closure-ის “გაშვებული” განმარტება:

> Closure არის heap-ზე გადატანილი stack frame, რომელიც ფუნქციას აძლევს საშუალებას იცოდეს თავისი წარმოშობის კონტექსტი მაშინაც კი, როცა ის კონტექსტი უკვე დასრულდა.

---

## 🔹 7. CLR დონეზე (IL მაგალითი)

თუ გავაანალიზებ lambda-ს IL კოდს:

```csharp
int a = 10;
Func<int> f = () => a + 5;
```

შექმნის დროს ვიღებთ რაღაც მსგავსს:

```
.class nested private auto ansi sealed beforefieldinit '<>c__DisplayClass0_0'
{
    .field public int32 a
    .method public hidebysig instance int32 '<Main>b__0'() { ... }
}
```

`f` სინამდვილეში არის delegate, რომელიც მიუთითებს
`<>c__DisplayClass0_0.<Main>b__0`-ზე,
და მისი “this” ფარავს captured field `a`-ს.

---

## 🔹 8. Closure vs Object-Oriented Thinking

| Closure           | Class Equivalent |
| ----------------- | ---------------- |
| Captured variable | Field            |
| Lambda            | Method           |
| Delegate          | Object reference |
| Execution context | Instance state   |

ანუ closure — ესაა დროებით გენერირებული ობიექტი, რომელიც იმოქმედებს როგორც კლასის ინსტანსი, ოღონდ შენ ეს ხელით არ გიწერია.

---

## 🔹 9. Practical summary

| დონე               | ახსნა                                                               |
| ------------------ | ------------------------------------------------------------------- |
| **Syntax-level**   | Lambda ხედავს გარე ცვლადს                                           |
| **Compiler-level** | იქმნება helper class environment-ით                                 |
| **Runtime-level**  | Lambda არის delegate, რომელიც მუშაობს ამ ობიექტზე (`this`)          |
| **Memory-level**   | Captured variables გადმოდის heap-ზე და ცოცხლობს delegate-სთან ერთად |