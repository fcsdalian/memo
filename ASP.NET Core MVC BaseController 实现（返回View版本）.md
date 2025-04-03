# ASP.NET Core MVC BaseController 实现（返回View版本）

下面我将提供一个完整的MVC风格BaseController实现，包含视图返回、模型绑定等典型MVC功能。

## 1. 项目结构

```
- Controllers/
  - BaseController.cs
  - ProductsController.cs
  - CustomersController.cs
- Views/
  - Products/
    - Index.cshtml
    - Details.cshtml
    - Create.cshtml
    - Edit.cshtml
    - Delete.cshtml
  - Customers/
    - Index.cshtml
    - Details.cshtml
    - Create.cshtml
    - Edit.cshtml
    - Delete.cshtml
- Models/
  - Product.cs
  - Customer.cs
- Data/
  - AppDbContext.cs
```

## 2. BaseController实现

### BaseController.cs

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using System.Threading.Tasks;

namespace YourProject.Controllers
{
    public abstract class BaseController<TEntity> : Controller where TEntity : class
    {
        protected readonly AppDbContext _context;
        protected readonly string _viewsFolder;

        public BaseController(AppDbContext context, string viewsFolder)
        {
            _context = context;
            _viewsFolder = viewsFolder;
        }

        // GET: {Controller}/Index
        public virtual async Task<IActionResult> Index()
        {
            return View(await _context.Set<TEntity>().ToListAsync());
        }

        // GET: {Controller}/Details/5
        public virtual async Task<IActionResult> Details(int? id)
        {
            if (id == null)
            {
                return NotFound();
            }

            var entity = await _context.Set<TEntity>().FindAsync(id);
            if (entity == null)
            {
                return NotFound();
            }

            return View(_viewsFolder + "/Details", entity);
        }

        // GET: {Controller}/Create
        public virtual IActionResult Create()
        {
            return View(_viewsFolder + "/Create");
        }

        // POST: {Controller}/Create
        [HttpPost]
        [ValidateAntiForgeryToken]
        public virtual async Task<IActionResult> Create(TEntity entity)
        {
            if (ModelState.IsValid)
            {
                _context.Add(entity);
                await _context.SaveChangesAsync();
                return RedirectToAction(nameof(Index));
            }
            return View(_viewsFolder + "/Create", entity);
        }

        // GET: {Controller}/Edit/5
        public virtual async Task<IActionResult> Edit(int? id)
        {
            if (id == null)
            {
                return NotFound();
            }

            var entity = await _context.Set<TEntity>().FindAsync(id);
            if (entity == null)
            {
                return NotFound();
            }
            return View(_viewsFolder + "/Edit", entity);
        }

        // POST: {Controller}/Edit/5
        [HttpPost]
        [ValidateAntiForgeryToken]
        public virtual async Task<IActionResult> Edit(int id, TEntity entity)
        {
            if (id != GetIdValue(entity))
            {
                return NotFound();
            }

            if (ModelState.IsValid)
            {
                try
                {
                    _context.Update(entity);
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
                return RedirectToAction(nameof(Index));
            }
            return View(_viewsFolder + "/Edit", entity);
        }

        // GET: {Controller}/Delete/5
        public virtual async Task<IActionResult> Delete(int? id)
        {
            if (id == null)
            {
                return NotFound();
            }

            var entity = await _context.Set<TEntity>().FindAsync(id);
            if (entity == null)
            {
                return NotFound();
            }

            return View(_viewsFolder + "/Delete", entity);
        }

        // POST: {Controller}/Delete/5
        [HttpPost, ActionName("Delete")]
        [ValidateAntiForgeryToken]
        public virtual async Task<IActionResult> DeleteConfirmed(int id)
        {
            var entity = await _context.Set<TEntity>().FindAsync(id);
            _context.Set<TEntity>().Remove(entity);
            await _context.SaveChangesAsync();
            return RedirectToAction(nameof(Index));
        }

        private bool EntityExists(int id)
        {
            return _context.Set<TEntity>().Find(id) != null;
        }

        private int GetIdValue(TEntity entity)
        {
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

## 3. 子控制器实现

### ProductsController.cs

```csharp
using Microsoft.AspNetCore.Mvc;
using YourProject.Data;
using YourProject.Models;
using System.Linq;
using System.Threading.Tasks;

namespace YourProject.Controllers
{
    public class ProductsController : BaseController<Product>
    {
        public ProductsController(AppDbContext context) 
            : base(context, "Products") // 指定视图文件夹名称
        {
        }

        // 可以覆盖基类方法或添加新方法
        public override async Task<IActionResult> Index()
        {
            // 添加特定于产品的逻辑
            var expensiveProducts = await _context.Set<Product>()
                .Where(p => p.Price > 100)
                .ToListAsync();
            
            return View("~/Views/Products/Index.cshtml", expensiveProducts);
        }

        // 添加特定于产品的方法
        public async Task<IActionResult> Search(string keyword)
        {
            var products = await _context.Set<Product>()
                .Where(p => p.Name.Contains(keyword) || p.Description.Contains(keyword))
                .ToListAsync();
            
            return View("~/Views/Products/Index.cshtml", products);
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
    public class CustomersController : BaseController<Customer>
    {
        public CustomersController(AppDbContext context) 
            : base(context, "Customers") // 指定视图文件夹名称
        {
        }

        // 可以覆盖基类方法或添加新方法
        public async Task<IActionResult> ActiveCustomers()
        {
            // 假设我们有一个IsActive属性
            var activeCustomers = await _context.Set<Customer>()
                .Where(c => c.IsActive)
                .ToListAsync();
            
            return View("~/Views/Customers/Index.cshtml", activeCustomers);
        }
    }
}
```

## 4. 视图示例 (Products/Index.cshtml)

```html
@model IEnumerable<YourProject.Models.Product>

@{
    ViewData["Title"] = "Products";
}

<h1>Products</h1>

<p>
    <a asp-action="Create">Create New</a>
</p>

<form asp-action="Search" method="get">
    <div class="form-group">
        <input type="text" name="keyword" class="form-control" />
        <input type="submit" value="Search" class="btn btn-primary" />
    </div>
</form>

<table class="table">
    <thead>
        <tr>
            <th>@Html.DisplayNameFor(model => model.Name)</th>
            <th>@Html.DisplayNameFor(model => model.Price)</th>
            <th>@Html.DisplayNameFor(model => model.Description)</th>
            <th></th>
        </tr>
    </thead>
    <tbody>
        @foreach (var item in Model) {
        <tr>
            <td>@Html.DisplayFor(modelItem => item.Name)</td>
            <td>@Html.DisplayFor(modelItem => item.Price)</td>
            <td>@Html.DisplayFor(modelItem => item.Description)</td>
            <td>
                <a asp-action="Edit" asp-route-id="@item.Id">Edit</a> |
                <a asp-action="Details" asp-route-id="@item.Id">Details</a> |
                <a asp-action="Delete" asp-route-id="@item.Id">Delete</a>
            </td>
        </tr>
        }
    </tbody>
</table>
```

## 5. 配置Startup/Program.cs

### Program.cs (.NET 6+)

```csharp
using Microsoft.EntityFrameworkCore;
using YourProject.Data;

var builder = WebApplication.CreateBuilder(args);

// 添加服务到容器
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddControllersWithViews();

var app = builder.Build();

// 配置HTTP请求管道
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthorization();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();
```

## 6. 使用说明

1. **路由**：
   - `/Products` - 显示产品列表
   - `/Products/Details/1` - 显示ID为1的产品详情
   - `/Products/Create` - 创建新产品表单
   - `/Products/Edit/1` - 编辑ID为1的产品

2. **自定义视图路径**：
   - 通过构造函数传入视图文件夹名称
   - 视图路径格式：`~/Views/{ControllerName}/{ActionName}.cshtml`

3. **扩展功能**：
   - 可以在子控制器中添加新的Action方法
   - 可以覆盖基类的Action方法实现自定义逻辑
   - 支持模型验证和防伪令牌

## 7. 高级用法

### 添加分页功能

修改BaseController中的Index方法：

```csharp
public virtual async Task<IActionResult> Index(int page = 1, int pageSize = 10)
{
    var query = _context.Set<TEntity>();
    var totalItems = await query.CountAsync();
    var items = await query.Skip((page - 1) * pageSize).Take(pageSize).ToListAsync();
    
    ViewBag.TotalItems = totalItems;
    ViewBag.Page = page;
    ViewBag.PageSize = pageSize;
    
    return View(_viewsFolder + "/Index", items);
}
```

然后在视图中添加分页控件：

```html
<div class="pagination">
    @for (int i = 1; i <= Math.Ceiling((double)ViewBag.TotalItems / ViewBag.PageSize); i++)
    {
        <a href="?page=@i&pageSize=@ViewBag.PageSize" class="@(i == ViewBag.Page ? "active" : "")">@i</a>
    }
</div>
```

这个实现提供了完整的MVC CRUD功能，同时保持了灵活性和可扩展性，适合传统的ASP.NET Core MVC应用程序开发。
