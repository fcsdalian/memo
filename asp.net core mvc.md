# 具体实现的ASP.NET Core MVC BaseController示例

下面我将提供一个完整的、可直接使用的代码示例，包括BaseController、子控制器、DbContext配置和依赖注入设置。

## 1. 项目结构

```
- Controllers/
  - BaseController.cs
  - ProductsController.cs
  - CustomersController.cs
- Models/
  - Product.cs
  - Customer.cs
- Data/
  - AppDbContext.cs
```

## 2. 模型定义

### Product.cs
```csharp
namespace YourProject.Models
{
    public class Product
    {
        public int Id { get; set; }
        public string Name { get; set; }
        public decimal Price { get; set; }
        public string Description { get; set; }
    }
}
```

### Customer.cs
```csharp
namespace YourProject.Models
{
    public class Customer
    {
        public int Id { get; set; }
        public string FirstName { get; set; }
        public string LastName { get; set; }
        public string Email { get; set; }
    }
}
```

## 3. DbContext配置

### AppDbContext.cs
```csharp
using Microsoft.EntityFrameworkCore;
using YourProject.Models;

namespace YourProject.Data
{
    public class AppDbContext : DbContext
    {
        public AppDbContext(DbContextOptions<AppDbContext> options) : base(options)
        {
        }

        public DbSet<Product> Products { get; set; }
        public DbSet<Customer> Customers { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            // 可以在这里添加模型配置
            base.OnModelCreating(modelBuilder);
        }
    }
}
```

## 4. BaseController实现

### BaseController.cs
```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace YourProject.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public abstract class BaseController<TEntity> : ControllerBase where TEntity : class
    {
        protected readonly AppDbContext _context;

        public BaseController(AppDbContext context)
        {
            _context = context;
        }

        // GET: api/[controller]
        [HttpGet]
        public virtual async Task<ActionResult<IEnumerable<TEntity>>> GetAll()
        {
            return await _context.Set<TEntity>().ToListAsync();
        }

        // GET: api/[controller]/5
        [HttpGet("{id}")]
        public virtual async Task<ActionResult<TEntity>> GetById(int id)
        {
            var entity = await _context.Set<TEntity>().FindAsync(id);

            if (entity == null)
            {
                return NotFound();
            }

            return entity;
        }

        // POST: api/[controller]
        [HttpPost]
        public virtual async Task<ActionResult<TEntity>> Create(TEntity entity)
        {
            _context.Set<TEntity>().Add(entity);
            await _context.SaveChangesAsync();

            return CreatedAtAction(nameof(GetById), new { id = GetIdValue(entity) }, entity);
        }

        // PUT: api/[controller]/5
        [HttpPut("{id}")]
        public virtual async Task<IActionResult> Update(int id, TEntity entity)
        {
            if (id != GetIdValue(entity))
            {
                return BadRequest();
            }

            _context.Entry(entity).State = EntityState.Modified;

            try
            {
                await _context.SaveChangesAsync();
            }
            catch (DbUpdateConcurrencyException)
            {
                if (!EntityExists(id))
                {
                    return NotFound();
                }
                else
                {
                    throw;
                }
            }

            return NoContent();
        }

        // DELETE: api/[controller]/5
        [HttpDelete("{id}")]
        public virtual async Task<IActionResult> Delete(int id)
        {
            var entity = await _context.Set<TEntity>().FindAsync(id);
            if (entity == null)
            {
                return NotFound();
            }

            _context.Set<TEntity>().Remove(entity);
            await _context.SaveChangesAsync();

            return NoContent();
        }

        private bool EntityExists(int id)
        {
            return _context.Set<TEntity>().Find(id) != null;
        }

        private int GetIdValue(TEntity entity)
        {
            // 使用反射获取ID值（假设所有实体都有名为Id的属性）
            var property = typeof(TEntity).GetProperty("Id");
            if (property != null)
            {
                return (int)property.GetValue(entity);
            }
            throw new InvalidOperationException("Entity does not have an Id property");
        }
    }
}
```

## 5. 子控制器实现

### ProductsController.cs
```csharp
using Microsoft.AspNetCore.Mvc;
using YourProject.Data;
using YourProject.Models;
using System.Linq;
using System.Threading.Tasks;

namespace YourProject.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class ProductsController : BaseController<Product>
    {
        public ProductsController(AppDbContext context) : base(context)
        {
        }

        // 可以添加特定于Product的方法
        // GET: api/Products/Expensive
        [HttpGet("Expensive")]
        public async Task<ActionResult<IEnumerable<Product>>> GetExpensiveProducts()
        {
            return await _context.Set<Product>()
                .Where(p => p.Price > 100)
                .ToListAsync();
        }
    }
}
```

### CustomersController.cs
```csharp
using Microsoft.AspNetCore.Mvc;
using YourProject.Data;
using YourProject.Models;
using System.Linq;
using System.Threading.Tasks;

namespace YourProject.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class CustomersController : BaseController<Customer>
    {
        public CustomersController(AppDbContext context) : base(context)
        {
        }

        // 可以添加特定于Customer的方法
        // GET: api/Customers/ByName?name=John
        [HttpGet("ByName")]
        public async Task<ActionResult<IEnumerable<Customer>>> GetCustomersByName(string name)
        {
            return await _context.Set<Customer>()
                .Where(c => c.FirstName.Contains(name) || c.LastName.Contains(name))
                .ToListAsync();
        }
    }
}
```

## 6. 配置Startup/Program.cs

### 对于.NET 6+ (Program.cs)
```csharp
using Microsoft.EntityFrameworkCore;
using YourProject.Data;

var builder = WebApplication.CreateBuilder(args);

// 添加服务到容器
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddControllers();

var app = builder.Build();

// 配置HTTP请求管道
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

## 7. 使用示例

### 获取所有产品
```
GET /api/products
```

### 获取特定产品
```
GET /api/products/1
```

### 创建新产品
```
POST /api/products
Body: { "name": "Laptop", "price": 999.99, "description": "High-end laptop" }
```

### 获取高价产品
```
GET /api/products/expensive
```

### 按名称搜索客户
```
GET /api/customers/byname?name=John
```

## 高级扩展

### 添加通用过滤和排序

可以在BaseController中添加以下方法：

```csharp
protected async Task<ActionResult<IEnumerable<TEntity>>> GetFiltered(
    Expression<Func<TEntity, bool>> filter = null,
    Func<IQueryable<TEntity>, IOrderedQueryable<TEntity>> orderBy = null)
{
    IQueryable<TEntity> query = _context.Set<TEntity>();

    if (filter != null)
    {
        query = query.Where(filter);
    }

    if (orderBy != null)
    {
        return await orderBy(query).ToListAsync();
    }
    else
    {
        return await query.ToListAsync();
    }
}
```

然后在子控制器中使用：

```csharp
[HttpGet("filter")]
public async Task<ActionResult<IEnumerable<Product>>> GetFilteredProducts(decimal minPrice)
{
    return await GetFiltered(
        filter: p => p.Price >= minPrice,
        orderBy: q => q.OrderByDescending(p => p.Price)
    );
}
```

这个实现提供了完整的CRUD操作，同时保持了灵活性和可扩展性。你可以根据需要添加更多功能或修改现有功能。
