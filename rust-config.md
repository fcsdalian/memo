在 Rust 项目中，配置文件 `config.yaml` 或 `config.toml` 的放置位置通常遵循以下约定：

## 标准配置文件路径

1. **开发环境**：
   - 直接放在项目根目录（与 `Cargo.toml` 同级）
   - 路径示例：`/your_project_root/config.yaml`

2. **生产环境**：
   - 通常放在 `/etc/your_app_name/` 目录
   - 或项目部署目录的 `config/` 子目录
   - 路径示例：`/etc/rust-htmx-crud/config.yaml`

## 具体实现方式

### 1. 基本配置加载代码

在 `main.rs` 或 `database.rs` 中添加配置加载逻辑：

```rust
use std::path::Path;

fn locate_config_file() -> Option<String> {
    // 尝试从多个可能的位置查找配置文件
    let paths = [
        "./config.yaml",                 // 开发环境 - 项目根目录
        "./config/config.yaml",         // 开发环境 - config子目录
        "/etc/rust-htmx-crud/config.yaml", // 生产环境
        "~/.config/rust-htmx-crud/config.yaml", // 用户级配置
    ];

    for path in &paths {
        if Path::new(path).exists() {
            return Some(path.to_string());
        }
    }
    None
}
```

### 2. 推荐的项目结构

```
your_project/
├── Cargo.toml
├── config/               # 开发环境配置目录
│   ├── config.yaml       # 主配置文件
│   └── config.prod.yaml  # 生产环境配置示例
├── src/
│   └── main.rs
└── .env                  # 可选的环境变量文件
```

### 3. 配置文件示例

`config.yaml` 内容示例：

```yaml
# 开发环境配置
database:
  url: "mssql://sa:yourpassword@localhost:1433/master?trust_server_certificate=true"
  db_type: "mssql"
  max_connections: 10

server:
  host: "127.0.0.1"
  port: 8080
```

或 `config.toml` 示例：

```toml
# 生产环境配置
[database]
url = "mssql://prod_user:ProdPass123@prod-db-server:1433/AppDB?encrypt=true"
db_type = "mssql"
max_connections = 30

[server]
host = "0.0.0.0"
port = 8080
```

### 4. 配置加载的最佳实践

```rust
use config::{Config, File, Environment};

pub fn load_config() -> Result<DatabaseSettings, Box<dyn std::error::Error>> {
    let config_path = std::env::var("APP_CONFIG")
        .unwrap_or_else(|_| "config/config.yaml".to_string());

    let settings = Config::builder()
        // 从配置文件加载
        .add_source(File::with_name(&config_path))
        // 允许环境变量覆盖 (例如 `APP_DATABASE__URL=mssql://...`)
        .add_source(Environment::with_prefix("APP").separator("__"))
        .build()?;

    settings.try_deserialize()
}
```

### 5. 运行时指定配置路径

启动应用时可以通过环境变量指定配置路径：

```bash
# 开发环境 (使用默认路径)
cargo run

# 生产环境 (指定自定义路径)
APP_CONFIG=/etc/myapp/config.prod.yaml cargo run --release
```

### 6. Docker 部署时的路径

在 Docker 中通常将配置文件挂载到容器内：

```dockerfile
FROM rust:latest

WORKDIR /app
COPY . .

# 默认配置文件路径
ENV APP_CONFIG=/app/config/config.prod.yaml

CMD ["cargo", "run", "--release"]
```

然后运行：

```bash
docker run -v ./config:/app/config my-app
```

## 重要提示

1. **安全警告**：永远不要将包含真实凭证的配置文件提交到版本控制
2. **推荐做法**：
   - 提交 `config.example.yaml` 到版本库
   - 将真实的 `config.yaml` 添加到 `.gitignore`
3. **多环境支持**：
   - `config.dev.yaml` - 开发环境
   - `config.staging.yaml` - 预发布环境
   - `config.prod.yaml` - 生产环境

通过这种结构，你的应用可以灵活地在不同环境中运行，同时保持配置的安全性和可维护性。
