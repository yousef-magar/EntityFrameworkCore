# 🧠 Entity Framework Core Guide (English + Arabic)

This guide explains how to use **Entity Framework Core** to connect your C# app with a database, step-by-step.

---

## 🔧 1. Create the Database Context Class

You need a class that connects your application with the database. This is called the **DbContext**.

```csharp
public class AppDbContext : DbContext
{
    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        options.UseSqlServer(@"Data Source=(localdb)\MSSQLLocalDB;Initial Catalog=YourDatabaseName;Integrated Security=True");
    }

    public DbSet<Employee> Employees { get; set; }
}
```

**شرح:**
هذه الكلاس `AppDbContext` تربط المشروع بقاعدة البيانات. نحدد فيها الاتصال ونضيف الجداول.

---

## 👨‍💼 2. Create a Model Class (مثال: Employee)

```csharp
public class Employee
{
    public int Id { get; set; }
    public string Name { get; set; }
}
```

> **ملاحظة:** اسم الكلاس مفرد ولكن في قاعدة البيانات يصبح جمع (Employees).

---

## 🏗️ 3. Create Migration and Database

### Create migration:

```powershell
PM> add-migration InitialCreate
```

### Apply migration to create the database:

```powershell
PM> update-database
```

### حذف migration:

```powershell
PM> remove-migration
```

**شرح:**
`migration` يعني إنشاء نسخة من التغييرات على الجداول. ثم نستخدم `update-database` لتنفيذها.

---

## ➕ 4. Insert Data into Database

```csharp
static void Main(string[] args)
{
    var _context = new AppDbContext();

    var employee = new Employee
    {
        Name = "Employee 1"
    };

    _context.Employees.Add(employee);
    _context.SaveChanges();
}
```

**ملاحظة:**
`Id` يتم توليده تلقائياً لأنه مفتاح أساسي (Primary Key).

---

## 🕐 5. الرجوع لنسخة سابقة

```powershell
PM> update-database -Migration:0
PM> remove-migration
```

---

## 🧾 6. Populate Data via Migration

```powershell
PM> add-migration PopulateData
PM> update-database
```

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    migrationBuilder.Sql("INSERT INTO Employees VALUES ('Employee 1')");
    migrationBuilder.Sql("INSERT INTO Employees VALUES ('Employee 2')");
}
```

```csharp
protected override void Down(MigrationBuilder migrationBuilder)
{
    migrationBuilder.Sql("DELETE FROM Employees WHERE Name = 'Employee 1'");
}
```

---

## 📁 7. Add Models Folder and Blog Class

```csharp
public class Blog
{
    public int Id { get; set; }

    [Required]
    public string Url { get; set; }
}
```

**Context:**

```csharp
public DbSet<Blog> Blogs { get; set; }
```

---

## 🧱 8. Use Fluent API to Configure Model

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Blog>()
        .Property(b => b.Url)
        .IsRequired();
}
```

---

## ⚙️ 9. Use Configuration Classes

### Add Configuration Class:

```csharp
public class BlogEntityTypeConfig : IEntityTypeConfiguration<Blog>
{
    public void Configure(EntityTypeBuilder<Blog> builder)
    {
        builder.Property(b => b.Url).IsRequired();
    }
}
```

### Register in DbContext:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfiguration(new BlogEntityTypeConfig());

    // Or automatically from assembly
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(BlogEntityTypeConfig).Assembly);
}
```

---

## 📄 10. Add `Post` Class with Relationships

```csharp
public class Post
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }

    public Blog Blog { get; set; }
}
```

```csharp
public class Blog
{
    public int Id { get; set; }
    public string Url { get; set; }

    public List<Post> Posts { get; set; }
}
```

**Migration Options on Delete:**

* `.OnDelete(DeleteBehavior.Cascade)` – حذف الأطفال مع الأب.
* `.OnDelete(DeleteBehavior.Restrict)` – خطأ إذا حاولت حذف الأب.

---

## 🧾 11. Add Audit Entity

```csharp
public class AuditEntry
{
    public int Id { get; set; }
    public string Username { get; set; }
    public string Action { get; set; }
}
```

```csharp
modelBuilder.Entity<AuditEntry>();
```

---

## 🛑 12. Ignore a Property

### Use Attribute:

```csharp
[NotMapped]
public List<Post> Posts { get; set; }
```

### Or Fluent API:

```csharp
modelBuilder.Entity<Blog>().Ignore(b => b.Posts);
```

---

## 🧯 13. Exclude a Table from Migration

```csharp
modelBuilder.Entity<Blog>()
    .ToTable("Blogs", b => b.ExcludeFromMigrations());
```

---

## 📛 14. Change Table Name

### Using Attribute:

```csharp
[Table("Posts")]
public class Post { ... }
```

### Using Fluent API:

```csharp
modelBuilder.Entity<Post>()
    .ToTable("Posts");
```

---

## 🗃️ 15. Change Schema

### Attribute:

```csharp
[Table("Posts"), Schema = "blogging"]
```

### Fluent API:

```csharp
modelBuilder.HasDefaultSchema("blogging");
modelBuilder.Entity<Post>().ToTable("Posts", schema: "blogging");
```

---

## 🚫 16. Ignore Property (Again)

```csharp
[NotMapped]
public DateTime AddedOn { get; set; }
```

```csharp
modelBuilder.Entity<Blog>().Ignore(b => b.AddedOn);
```

---

## 🏷️ 17. Rename Column

### Attribute:

```csharp
[Column("BlogUrl")]
public string Url { get; set; }
```

### Fluent API:

```csharp
modelBuilder.Entity<Blog>()
    .Property(b => b.Url)
    .HasColumnName("BlogUrl");
```

---

## 🔤 18. Column Type, Max Length, Comment

### Column Type:

```csharp
[Column(TypeName = "varchar(200)")]
public string Url { get; set; }
```

### Fluent API:

```csharp
modelBuilder.Entity<Blog>()
    .Property(b => b.Url)
    .HasColumnType("varchar(200)");
```

### Max Length:

```csharp
[MaxLength(200)]
```

```csharp
modelBuilder.Entity<Blog>()
    .Property(b => b.Url)
    .HasMaxLength(200);
```

### Comment:

```csharp
[Comment("The URL of the blog")]
```

```csharp
modelBuilder.Entity<Blog>()
    .Property(b => b.Url)
    .HasComment("The URL of the blog");
```

---

## 🔑 19. Custom Primary Key

```csharp
modelBuilder.Entity<Book>()
    .HasKey(b => b.BookKey)
    .HasName("PK_BookKey");
```

---

## 📥 20. Set Default Values

```csharp
modelBuilder.Entity<Blog>()
    .Property(b => b.Rating)
    .HasDefaultValue(2);

modelBuilder.Entity<Blog>()
    .Property(b => b.CreatedOn)
    .HasDefaultValueSql("GETDATE()");
```

---

## 🔢 21. Computed Columns

```csharp
public class Author
{
    public int Id { get; set; }

    [Required, MaxLength(50)]
    public string FirstName { get; set; }

    [Required, MaxLength(50)]
    public string LastName { get; set; }

    [MaxLength(150)]
    public string DisplayName { get; set; }
}
```

```csharp
modelBuilder.Entity<Author>()
    .Property(a => a.DisplayName)
    .HasComputedColumnSql("[LastName] + ', ' + [FirstName]");
```

---

## 🆔 22. Identity Primary Key (Auto Increment)

```csharp
public class Category
{
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public byte Id { get; set; }

    public string Name { get; set; }
}
```

```csharp
modelBuilder.Entity<Category>()
    .Property(c => c.Id)
    .ValueGeneratedOnAdd();
```

---

# Entity Framework Core - Relationships & Indexes Guide
دليل مبسط لعلاقات الكيانات (Relationships) والفهارس (Indexes) في EF Core

---

## 🧩 One-to-One Relationship - علاقة واحد إلى واحد

```csharp
modelBuilder.Entity<Blog>()
    .HasOne(b => b.BlogIMG)
    .WithOne(i => i.Blog)
    .HasForeignKey<BlogIMG>(b => b.BlogForeignKey);
````

* هذا يعني أن كل `Blog` عنده صورة واحدة فقط `BlogIMG`.
* والعكس، كل `BlogIMG` مرتبط بمدونة واحدة فقط.
* نستخدم `HasForeignKey` لتحديد المفتاح الخارجي في جدول `BlogIMG`.

### الكلاسات:

```csharp
public class Blog
{
    public int ID { get; set; }
    public int URL { get; set; }

    public BlogIMG BlogIMG { get; set; }
}

public class BlogIMG
{
    public int ID { get; set; }
    public string IMG { get; set; }
    public string Caption { get; set; }

    public int BlogForeignKey { get; set; }

    [ForeignKey("BlogForeignKey")]
    public Blog Blog { get; set; }
}
```

---

## 🔁 One-to-Many Relationship (Custom Key) - علاقة واحد إلى متعدد باستخدام مفتاح مخصص

```csharp
modelBuilder.Entity<RecordOfSale>()
    .HasOne(s => s.Car)
    .WithMany(c => c.SaleHistory)
    .HasForeignKey(s => s.CarLicensePlate)
    .HasPrincipalKey(c => c.LicensePlate);
```

* كل `Car` تحتوي على عدة `RecordOfSale`.
* المفتاح الأساسي هو `LicensePlate` وليس الـ `ID`.

### الكلاسات:

```csharp
public class Car
{
    public int CarID { get; set; }
    public string LicensePlate { get; set; }
    public string Make { get; set; }
    public string Model { get; set; }

    public List<RecordOfSale> SaleHistory { get; set; }
}

public class RecordOfSale
{
    public int RecordOfSaleID { get; set; }
    public DateTime DateSold { get; set; }
    public decimal Price { get; set; }

    public string CarLicensePlate { get; set; }
    public Car Car { get; set; }
}
```

---

## 🔁 One-to-Many Relationship - علاقة واحد إلى متعدد عادية

```csharp
modelBuilder.Entity<Blog>()
    .HasMany(b => b.Posts)
    .WithOne(p => p.Blog);
```

* كل `Blog` تحتوي على عدة `Posts`.
* كل `Post` مرتبط بمدونة واحدة فقط.

### الكلاسات:

```csharp
public class Blog
{
    public int ID { get; set; }
    public int URL { get; set; }

    public List<Post> Posts { get; set; }
}

public class Post
{
    public int PostID { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }

    public int BlogID { get; set; }
    public Blog Blog { get; set; }
}
```

---

## 🔁 Many-to-Many Relationship - علاقة متعدد إلى متعدد

```csharp
modelBuilder.Entity<Post>()
    .HasMany(p => p.Tags)
    .WithMany(t => t.Posts)
    .UsingEntity(j => j.ToTable("PostTags"));
```

* كل `Post` يحتوي على أكثر من `Tag`.
* وكل `Tag` ممكن يكون له أكثر من `Post`.
* يتم إنشاء جدول وسيط تلقائياً بإسم `PostTags`.

### الكلاسات:

```csharp
public class Post
{
    public int PostID { get; set; }
    public string Title { get; set; }
    public string Content { get; set; }

    public ICollection<Tag> Tags { get; set; }
}

public class Tag
{
    public string TagID { get; set; }

    public ICollection<Post> Posts { get; set; }
}
```

---

## 🔍 Index - فهرسة

```csharp
modelBuilder.Entity<Blog>()
    .HasIndex(b => b.URL);
```

أو باستخدام Data Annotations:

```csharp
[Index(nameof(URL))]
public class Blog
{
    public int ID { get; set; }
    public int URL { get; set; }

    public List<Post> Posts { get; set; }
}
```

* تساعد الفهارس (Indexes) في تسريع الاستعلامات في قاعدة البيانات.

---

## 🔍 Composite Index - فهرس مركب

```csharp
[Index(nameof(FirstName), nameof(LastName))]
public class Person
{
    public int ID { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
```

* فهرس يتكوّن من أكثر من عمود.
* ممتاز للبحث باستخدام أكثر من حقل.

---

## 🔐 Unique Index - فهرس فريد

```csharp
[Index(nameof(URL), IsUnique = true)]
public class Blog
{
    public int ID { get; set; }
    public int URL { get; set; }

    public List<Post> Posts { get; set; }
}
```

### تخصيص إضافي:

```csharp
modelBuilder.Entity<Person>()
    .HasIndex(p => new { p.FirstName, p.LastName })
    .IsUnique()
    .HasDatabaseName("Index_Url")
    .HasFilter("[URL] IS NOT NULL");
```

---

## 🔢 Sequences - تسلسل تلقائي

```csharp
modelBuilder.HasSequence<int>("OrderNumber")
    .StartsAt(10)
    .IncrementsBy(5);

modelBuilder.Entity<Order>()
    .Property(o => o.OrderNo)
    .HasDefaultValueSql("NEXT VALUE FOR OrderNumber");
```

* `Sequence` هو عداد خارجي تعيّنه بنفسك.
* يمكن استخدامه لتوليد أرقام تلقائياً بتسلسل معين.

### الكلاسات:

```csharp
public class Order
{
    public int ID { get; set; }
    public int OrderNo { get; set; }
    public double Amount { get; set; }
}
```

---

## ✅ ملاحظات عامة

* استخدم `Fluent API` في `OnModelCreating` للتحكم الكامل في العلاقات والفهارس.
* أو استخدم `Data Annotations` عند الحاجة بشكل بسيط.
* دايماً راجع أسماء الحقول وتأكد من تطابقها في العلاقات.

---
# 📘 Entity Framework Core Guide (EN + AR)

دليل مبسط للعمل مع Entity Framework Core يشمل كل الملاحظات المهمة من الدورة مع توضيحات إنجليزية بسيطة وترجمة عربية.

---

## 🧪 1. Data Seeding (إضافة بيانات تلقائية عند إنشاء قاعدة البيانات)

### ✅ الطريقة الأولى - باستخدام Fluent API:

```csharp
modelBuilder.Entity<Blog>()
    .HasData(new Blog { BlogID = 1, URL = "www.google" });

modelBuilder.Entity<Post>()
    .HasData(
        new Post { BlogID = 1, PostedID = 1, Title = "post1", Content = "cpn1" },
        new Post { BlogID = 1, PostedID = 2, Title = "post2", Content = "cpn2" }
    );
````

🎯 **الشرح**: البيانات هنا تضاف تلقائيًا وقت إنشاء القاعدة.

---

### ✅ الطريقة الثانية - Manual Seeding:

```csharp
class Program
{
    static void Main(string[] args)
    {
        SeedData();
    }

    public static void SeedData()
    {
        using (var context = new AppDbContext())
        {
            context.Database.EnsureCreated();

            var blog = context.Blogs.FirstOrDefault(b => b.Url == "www....");
            if (blog == null)
                context.Blogs.Add(new Blog { Url = "www...." });

            context.SaveChanges();
        }
    }
}
```

🗒️ **EnsureCreated()**: تتأكد أن القاعدة موجودة، إذا لم تكن موجودة سيتم إنشاؤها.

---

## 🧱 2. Working With Existing Database (العمل مع قاعدة بيانات موجودة)

### 🛠 Scaffold Existing DB:

```bash
PM> Scaffold-DbContext "Connection String" Microsoft.EntityFrameworkCore.SqlServer
```

🧭 **ملاحظات**:

* لفلترة الجداول: `-Tables Blogs,Posts`
* لتحديد مجلد لحفظ الكلاسات: `-OutputDir Models`
* لتغيير اسم السياق: `-Context ApplicationDbContext`
* لإجبار EF على استخدام Data Annotations: `-DataAnnotations`

---

## 🔍 3. Reading Data (قراءة البيانات)

```csharp
var stocks = _context.Stocks.ToList(); // كل البيانات

foreach (var stock in stocks)
    Console.WriteLine(stock.Name);
```

### 📌 Read One Record:

```csharp
var stock = _context.Stocks.Find(100); // يبحث باستخدام المفتاح الأساسي
Console.WriteLine($"{stock.ID} - {stock.Name}");

var stock = _context.Stocks.SingleOrDefault(s => s.ID == 100); // شرط مخصص
```

> ✅ `.SingleOrDefault()`: يرجع عنصر واحد أو null
> ❌ `.Single()` يرمي خطأ إذا لم يوجد عنصر

---

## 🔁 4. LINQ Queries & Filters

```csharp
var stocks = _context.Stocks
    .Where(s => s.ID > 500)
    .ToList();
```

### ✅ Methods:

* `.Any()`: هل يوجد أي عنصر؟
* `.All()`: هل كل العناصر تحقق الشرط؟
* `.First() / .FirstOrDefault()`
* `.Last() / .LastOrDefault()`
* `.OrderBy()` / `.OrderByDescending()`
* `.ThenBy()` / `.ThenByDescending()`
* `.Count()` / `.LongCount()`
* `.Sum()` / `.Average()` / `.Max()` / `.Min()`
* `.Append()` / `.Prepend()`
* `.Distinct()` – يرجع العناصر بدون تكرار
* `.Skip(0).Take(10)` – للصفحات (pagination)

---

## 📊 5. Grouping & Aggregates (التجميع والإحصائيات)

```csharp
var result = _context.Stocks
    .GroupBy(s => s.Industry)
    .Select(g => new {
        Industry = g.Key,
        Count = g.Count(),
        Sum = g.Sum(s => s.Balance)
    })
    .OrderByDescending(g => g.Count)
    .ToList();
```

---

## 🔗 6. Joins (العلاقات بين الجداول)

### ✅ Inner Join:

```csharp
var books = _context.Books
    .Join(_context.Authors,
          book => book.AuthorId,
          author => author.Id,
          (book, author) => new {
              BookId = book.Id,
              BookName = book.Name,
              AuthorName = author.Name
          });
```

### ✅ Multiple Joins (with Nationality):

```csharp
var books = _context.Books
    .Join(_context.Authors, b => b.AuthorId, a => a.Id, (b, a) => new {
        b.Id,
        b.Name,
        a.Name,
        a.NationalityId
    })
    .Join(_context.Nationalities, b => b.NationalityId, n => n.Id, (b, n) => new {
        b.Id,
        b.Name,
        b.Name,
        AuthorNationality = n.Name
    });
```

### ✅ Left Join with GroupJoin:

```csharp
var books = _context.Books
    .Join(_context.Authors, b => b.AuthorId, a => a.Id, (b, a) => new {
        b.Id, b.Name, a.Name, a.NationalityId
    })
    .GroupJoin(_context.Nationalities, a => a.NationalityId, n => n.Id, (a, nationalities) => new {
        Book = a,
        Nationality = nationalities
    })
    .SelectMany(x => x.Nationality.DefaultIfEmpty(), (x, nationality) => new {
        x.Book.Id,
        x.Book.Name,
        x.Book.Name,
        Nationality = nationality?.Name
    });
```

---

## ➕ 7. Add Records (إضافة بيانات)

```csharp
var nationality = new Nationality { Name = "New Nationality" };
_context.Nationalities.Add(nationality);
_context.SaveChanges();
```

### Add Many:

```csharp
_context.Nationalities.AddRange(new List<Nationality> {
    new Nationality { Name = "New 2" },
    new Nationality { Name = "New 3" },
});
```

### Add with Relationship:

```csharp
var author = new Author {
    Name = "New Author",
    Nationality = new Nationality { Name = "Nested Nationality" }
};
_context.Authors.Add(author);
_context.SaveChanges();
```

---

## 📝 8. Update Records (تعديل البيانات)

### الطريقة التقليدية:

```csharp
var nationality = _context.Nationalities.Find(10);
nationality.Name = "Updated Name";
_context.SaveChanges();
```

### باستخدام Update():

```csharp
var nationality = new Nationality { Id = 10, Name = "New Name" };
_context.Update(nationality);
_context.SaveChanges();
```

### باستخدام Entry:

```csharp
var current = _context.Nationalities.Find(10);
var updated = new Nationality { Id = 10, Name = "Updated Again" };
_context.Entry(current).CurrentValues.SetValues(updated);
```

---

## 🗑 9. Delete Records (حذف البيانات)

```csharp
var nationality = _context.Nationalities.Find(9);
_context.Nationalities.Remove(nationality);
_context.SaveChanges();
```

### Delete Many:

```csharp
var items = _context.Nationalities.Where(p => p.Id > 6 && p.Id <= 8).ToList();
_context.Nationalities.RemoveRange(items);
```

---

## ❌ Delete Related Data (مع علاقات)

```csharp
builder.Entity<Blog>()
    .HasMany(b => b.Posts)
    .WithOne(p => p.Blog)
    .OnDelete(DeleteBehavior.Restrict); // يمنع حذف تلقائي

var blog = _context.Blogs.Find(2);
var posts = _context.Posts.Where(p => p.BlogId == 2);

_context.RemoveRange(posts);
_context.Remove(blog);
_context.SaveChanges();
```

---

## 🔐 10. Transactions (التعامل مع العمليات المعقدة)

```csharp
using var transaction = _context.Database.BeginTransaction();

try
{
    _context.Blogs.Add(new Blog { Name = "First Blog" });
    _context.SaveChanges();

    transaction.CreateSavepoint("AddFirstBlog");

    _context.Blogs.Add(new Blog { Name = "Second Blog" });
    _context.SaveChanges();

    transaction.Commit();
}
catch (Exception)
{
    transaction.RollbackToSavepoint("AddFirstBlog");
}
```

---

## 📉 11. MinBy / MaxBy (الحد الأدنى والأقصى)

```csharp
var products = new List<Product> {
    new Product { Name = "P1", Price = 10 },
    new Product { Name = "P2", Price = 20 },
    new Product { Name = "P3", Price = 5 }
};

var cheapest = products.MinBy(p => p.Price);
var mostExpensive = products.MaxBy(p => p.Price);

Console.WriteLine($"Cheapest: {cheapest.Name}");
Console.WriteLine($"Most Expensive: {mostExpensive.Name}");
```

---

