# 1) إجراءات العلاقات (Referential Actions) بالـ Fluent API

**الفكرة:** تحديد سلوك الحذف/التحديث بين الجداول المرتبطة (Cascade / SetNull / Restrict/NoAction / ClientSetNull).

**الكود:**

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Order>()
        .HasOne(o => o.Customer)
        .WithMany(c => c.Orders)
        .HasForeignKey(o => o.CustomerId)
        .OnDelete(DeleteBehavior.Cascade);    // أو Restrict / NoAction / SetNull / ClientSetNull
}
```

**ملاحظات سريعة:**

* `Cascade`: حذف الأب يحذف الأبناء.
* `SetNull`: يحوّل FK في الأبناء إلى NULL.
* `Restrict/NoAction`: يمنع حذف الأب لو له أبناء.
* `ClientSetNull`: مثل SetNull لكن على مستوى التتبع بالعميل.

**تمرين عملي:**

1. أنشئ كيانين `Customer` و`Order` بعلاقة واحد-ل-متعدد.
2. جرّب `OnDelete(DeleteBehavior.Cascade)` ثم أنشئ Migration وطبّقها.
3. احذف Customer من DB وشاهد هل يتم حذف Orders تلقائيًا أم لا.

---

# 2) تضمين/استبعاد كيان (Include/Exclude Table) في EF Core

**الفكرة:** التحكم في ما يتم توليده في النموذج/المخطط:

* **استبعاد كيان كامل:**

  * Fluent: `modelBuilder.Ignore<SomeEntity>();`
  * Annotation: `[NotMapped]`.
* **استبعاد خاصية:** `[NotMapped]` على الخاصية.
* **تغيير اسم الجدول:** `.ToTable("MyTable")`.

**الكود:**

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Ignore<TempViewModel>(); // مش جدول
    modelBuilder.Entity<Student>().ToTable("tbl_Students");

    modelBuilder.Entity<Student>()
        .Ignore(s => s.TempCalculatedProperty); // خاصية غير مخزنة
}
```

**تمرين عملي:**

1. أضف كيان `ReportViewModel` واستخدم `Ignore` عشان مايتولدش جدول.
2. غيّر اسم جدول `Student` لـ `tbl_Students` وشاهد فرق الـ Migration.

---

# 3) التحكم في الكيانات بـ Data Annotations

**الأكثر استخدامًا:**
`[Key]`, `[Required]`, `[MaxLength]`, `[StringLength]`, `[Column(TypeName="decimal(18,2)")]`, `[ForeignKey]`, `[NotMapped]`, **وفي الإصدارات الحديثة** `[Index]`.

**الكود:**

```csharp
public class Product
{
    [Key]
    public int Id { get; set; }

    [Required, MaxLength(100)]
    public string Name { get; set; }

    [Column(TypeName = "decimal(10,2)")]
    public decimal Price { get; set; }

    // متاح في الإصدارات الحديثة من EF Core
    [Index(nameof(Name), IsUnique = true)]
    public string NormalizedName { get; set; }
}
```

**تمرين عملي:**

1. أضف `[Required]` و`[MaxLength]` لحقل اسم المنتج.
2. أنشئ Migration وشوف الأعمدة والقيود الناتجة في SQL.

---

# 4) التحكم في الكيانات بالـ Fluent API

**الفكرة:** كل ما تستطيع فعله بالـ Annotations يمكنك فعله وأكثر بالـ Fluent API، ويفضل للسيناريوهات المتقدمة وللفصل في ملفات Config.

**الكود (IEntityTypeConfiguration):**

```csharp
public class ProductConfig : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> b)
    {
        b.ToTable("Products");
        b.HasKey(p => p.Id);
        b.Property(p => p.Name).IsRequired().HasMaxLength(100);
        b.Property(p => p.Price).HasColumnType("decimal(10,2)");
        b.HasIndex(p => p.Name).IsUnique();
    }
}

// داخل DbContext
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfiguration(new ProductConfig());
}
```

**تمرين عملي:**

1. انقل إعدادات كيان موجود لملف `Config` منفصل.
2. قارن الـ Migration قبل/بعد.

---

# 5) الأعمدة الافتراضية والمحسوبة (Default & Computed Columns)

**الفكرة:**

* **Default:** قيمة افتراضية عند الإدراج.
* **Computed:** قيمة محسوبة في SQL (لا تُكتب يدويًا).

**الكود:**

```csharp
modelBuilder.Entity<Order>()
    .Property(o => o.CreatedAt)
    .HasDefaultValueSql("GETDATE()"); // Default

modelBuilder.Entity<Order>()
    .Property(o => o.TotalWithTax)
    .HasComputedColumnSql("[Total] * 1.14", stored: true); // Computed (stored/in memory)
```

**تمرين عملي:**

1. أضف `CreatedAt` بـ `HasDefaultValueSql("GETDATE()")`.
2. أضف عمود `TotalWithTax` محسوب من `Total`.
3. راقب السكريبت في Migration.

---

# 6) استخدام الفهارس (Indexing) في EF

**لماذا؟** تحسين الأداء في البحث والفرز والـ JOIN.

**الكود:**

```csharp
modelBuilder.Entity<User>()
    .HasIndex(u => u.Email)
    .IsUnique()
    .HasDatabaseName("IX_Users_Email");

// فهرس مركب + مفلتر (SQL Server)
modelBuilder.Entity<User>()
    .HasIndex(u => new { u.LastName, u.FirstName })
    .HasFilter("[IsDeleted] = 0");
```

**تمرين عملي:**

1. أضف فهرسًا فريدًا على `Email`.
2. أنشئ فهرسًا مركبًا على الاسم الأول/الأخير.

---

# 7) عمليات على الفهارس (إنشاء/حذف/تعديل) عبر الـ Migrations

**الفكرة:** أي تعديل في `.HasIndex` سينعكس في Migration:

* إضافة فهرس ⇒ `CreateIndex`
* حذف فهرس ⇒ `DropIndex`
* تعديل ⇒ Drop ثم Create جديد باسم مختلف.

**تمرين عملي:**

1. أضف فهرس، ثم عدّل اسمه أو خصائصه، وراقب الـ Migrations المتولدة.
2. نفّذ `Update-Database` وتأكد من وجود الفهرس في SSMS.

---

# 8) استخدام تسلسلات قاعدة البيانات (Database Sequences)

**الفكرة:** مولد أرقام مستقل عن الجداول (SQL Server Sequence).

**الكود:**

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.HasSequence<int>("OrderNumbers", schema: "shared")
        .StartsAt(1000)
        .IncrementsBy(1);

    modelBuilder.Entity<Order>()
        .Property(o => o.OrderNumber)
        .HasDefaultValueSql("NEXT VALUE FOR shared.OrderNumbers");
}
```

**بديل للمفاتيح:** HiLo

```csharp
modelBuilder.UseHiLo("EntityHiLoSequence", "shared");
```

**تمرين عملي:**

1. أنشئ Sequence واربِطه بعمود `OrderNumber`.
2. أدخل عدة سجلات وشاهد الترقيم المتتابع.

---

# 9) تحويل الـ Migration إلى SQL (Script)

**الأوامر:**

* Package Manager Console:

  ```powershell
  Script-Migration               # من أول Migration لآخر واحدة
  Script-Migration From To       # سكريبت بين ميجريشنين
  ```
* CLI:

  ```bash
  dotnet ef migrations script
  dotnet ef migrations script From To
  ```

**تمرين عملي:**

1. ولّد سكريبت آخر Migration واحفظه كملف `.sql`.
2. افتحه وتأكّد من أوامر `CREATE TABLE/ALTER TABLE...`.

---

# 10) Reverse Scaffold (من قاعدة بيانات إلى كود)

**الفكرة:** توليد الـ DbContext والكيانات من DB موجودة.

**الأمر (CLI):**

```bash
dotnet ef dbcontext scaffold "Server=.;Database=MyDb;Trusted_Connection=True;" Microsoft.EntityFrameworkCore.SqlServer -o Models -c AppDbContext -f
```

* `-o` مجلد الإخراج.
* `-c` اسم الكونتكست.
* `-f` للكتابة فوق الملفات.

**تمرين عملي:**

1. طبّق الأمر على قاعدة بيانات موجودة (جداول بسيطة).
2. راجع الـ Entities المتولدة والعلاقات.

---

# 11) إضافة Seed Data (بيانات مبدئية)

**الكود:**

```csharp
modelBuilder.Entity<Role>().HasData(
    new Role { Id = 1, Name = "Admin" },
    new Role { Id = 2, Name = "User" }
);
```

> تذكّر: يجب توفير المفاتيح الأساسية (Id) ثابتة.

**تمرين عملي:**

1. أضف `HasData` لأكتر من جدول.
2. أنشئ Migration؛ ستجد أوامر INSERT. نفّذها بـ `Update-Database`.

---

# 12) EF Core و LINQ

**الفكرة:** LINQ تُترجم إلى SQL (Query Translation).

* استخدم **Projection** لتقليل الأعمدة.
* استخدم `AsNoTracking` للقراءة السريعة.
* استخدم `Any/All/Count` بحذر.

**أمثلة:**

```csharp
var users = await _db.Users
    .Where(u => u.IsActive && u.CreatedAt >= DateTime.Today.AddDays(-30))
    .Select(u => new { u.Id, u.Email })
    .AsNoTracking()
    .ToListAsync();

var count = await _db.Orders.CountAsync(o => o.Total > 1000);
```

**تمرين عملي:**

1. اكتب استعلامات `Where`, `Select`, `GroupBy` وراقب SQL في الـ Output.
2. جرّب `AsNoTracking` ولاحظ فرق الأداء عند القراءة.

---

# 13) حفظ سجلات **بشكل عام** (Global Saving) – التدقيق (Auditing)

**الفكرة:** ضبط قيم مثل `CreatedAt`, `CreatedBy` تلقائيًا لكل الكيانات عند `SaveChanges`.

**الكود:**

```csharp
public interface IAuditable
{
    DateTime CreatedAt { get; set; }
    string CreatedBy { get; set; }
    DateTime? UpdatedAt { get; set; }
    string UpdatedBy { get; set; }
}

public override int SaveChanges()
{
    var entries = ChangeTracker.Entries<IAuditable>();
    var now = DateTime.UtcNow;
    var user = "system"; // استبدلها بالمستخدم الحالي

    foreach (var e in entries)
    {
        if (e.State == EntityState.Added)
        {
            e.Entity.CreatedAt = now;
            e.Entity.CreatedBy = user;
        }
    }
    return base.SaveChanges();
}
```

**تمرين عملي:**

1. طبّق الواجهة على كياناتك.
2. أضف سجل جديد وتحقق من القيم.

---

# 14) التحديث العام (Global Updating)

**الفكرة:** تعبئة `UpdatedAt/UpdatedBy` عند التعديل.

**الكود (تكملة السابق):**

```csharp
foreach (var e in entries)
{
    if (e.State == EntityState.Modified)
    {
        e.Entity.UpdatedAt = now;
        e.Entity.UpdatedBy = user;
    }
}
```

**تمرين عملي:**

1. عدّل أي سجل واحفظ.
2. تأكد من تحديث الحقول.

---

# 15) الحذف العام (Global Deleting) – **Soft Delete**

**الفكرة:** بدل ما نحذف فعليًا، نعلّم السجل بأنه محذوف (`IsDeleted = true`) + فلتر عام.

**الكود:**

```csharp
public interface ISoftDelete { bool IsDeleted { get; set; } }

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<User>().HasQueryFilter(u => !u.IsDeleted);
}

public override int SaveChanges()
{
    foreach (var e in ChangeTracker.Entries<ISoftDelete>())
    {
        if (e.State == EntityState.Deleted)
        {
            e.State = EntityState.Modified;
            e.Entity.IsDeleted = true;
        }
    }
    return base.SaveChanges();
}
```

**تمرين عملي:**

1. أضف `IsDeleted` وفلتر عام.
2. احذف سجلًا وتأكد أنه اختفى من الاستعلامات لكنه موجود في الجدول.

---

# 16) تتبع مقابل عدم التتبع (Tracking vs No-Tracking)

* **Tracking (الافتراضي):** EF يتابع الكيانات لتحديثها لاحقًا.
* **AsNoTracking():** أسرع للقراءة فقط، لا يتم تتبع الكيانات.
* **AsNoTrackingWithIdentityResolution():** لا يكرر الكيانات المرجعية.

**أمثلة:**

```csharp
var tracked = await _db.Products.ToListAsync();              // تتبع
var readOnly = await _db.Products.AsNoTracking().ToListAsync(); // بدون تتبع
```

**تمرين عملي:**

1. اجلب نفس البيانات بـ Tracking وNo-Tracking.
2. حاول تعديل كيان من نتيجة No-Tracking وشاهد أنه لن يُحفظ إلا بعد Attach/Update.

---

# 17) التحميل النشِط (Eager Loading) – Include

**الفكرة:** جلب البيانات المرتبطة في نفس الاستعلام.

**الكود:**

```csharp
var orders = await _db.Orders
    .Include(o => o.Customer)
    .Include(o => o.Items)
    .ThenInclude(i => i.Product)
    .ToListAsync();
```

**تمرين عملي:**

1. استخدم `Include/ThenInclude` لجلب عميل/عناصر الطلب.
2. راقب SQL وعدد الـ JOINs.

---

# 18) التحميل الصريح (Explicit Loading)

**الفكرة:** تحميل العلاقات عند الطلب بعد جلب الكيان.

**الكود:**

```csharp
var order = await _db.Orders.FirstAsync();
await _db.Entry(order).Reference(o => o.Customer).LoadAsync();
await _db.Entry(order).Collection(o => o.Items).LoadAsync();
```

**تمرين عملي:**

1. اجلب كيان واحد ثم حمّل علاقاته يدويًا.
2. لاحظ عدد الاستعلامات المنفصلة.

---

# 19) التحميل الكسول (Lazy Loading)

**الفكرة:** تحميل العلاقات تلقائيًا عند الوصول للخاصية.
**المتطلبات (SQL Server مثال):**

* تثبيت الحزمة: `Microsoft.EntityFrameworkCore.Proxies`
* تفعيل البروكسيات:

```csharp
options.UseSqlServer(conn).UseLazyLoadingProxies();
```

* جعل الخصائص **virtual**.

**الكود:**

```csharp
public class Order
{
    public int Id { get; set; }
    public virtual Customer Customer { get; set; } // virtual
}
```

**تحذير:** قد يسبب N+1 Queries إن لم تنتبه.

**تمرين عملي:**

1. فعّل Lazy Loading وجرب الوصول لخاصية ملاحية خارج `Include`.
2. تابع الاستعلامات الخارجة.

---

# 20) المعاملات (Transactions) في EF Core

**الفكرة:** تنفيذ مجموعة عمليات كوحدة واحدة (إما الكل ينجح أو الكل يفشل).

**الكود:**

```csharp
using var tx = await _db.Database.BeginTransactionAsync();
try
{
    _db.Add(new Customer { Name = "Ali" });
    await _db.SaveChangesAsync();

    _db.Add(new Order { CustomerId = 1, Total = 500 });
    await _db.SaveChangesAsync();

    await tx.CommitAsync();
}
catch
{
    await tx.RollbackAsync();
    throw;
}
```

**بديل:** `TransactionScope` (مع الانتباه لـ async وتكوين البيئة).

**تمرين عملي:**

1. نفّذ عمليتين إدراج داخل Transaction.
2. افتعل خطأ قبل `Commit` وتأكد من عدم إدراج أيٍّ منهما.

---

## نصائح عامة

* أي تغيير في التكوين ⇒ **Migration** جديدة قبل `Update-Database`.
* راقب SQL الناتج (Logging) لفهم الأداء.
* استخدم الفهارس والـ Projection و`AsNoTracking` لتحسين الأداء.
* راقب سلوك العلاقات عند الحذف لتجنب فقدان بيانات غير مقصود.

لو تحب، أرتّب لك **مشروع نموذج** (Console/Minimal API) فيه كل هذه الأمثلة مجمعة بكيانات بسيطة (Customer/Order/Product/User…) عشان تجرّبها خطوة بخطوة.
