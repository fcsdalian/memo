# Rust + HTMX + DaisyUI 数据库 CRUD 应用实现

下面是一个使用 Rust、HTMX 和 DaisyUI 构建的通用数据库 CRUD 应用程序的实现。这个实现支持多种数据库后端，并能动态处理不同的数据库表。

## 项目结构

```
.
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── models.rs
│   ├── database.rs
│   ├── handlers.rs
│   ├── templates/
│   │   ├── base.html
│   │   ├── list.html
│   │   ├── form.html
│   │   └── _partials/
│   │       ├── table.html
│   │       └── form_fields.html
├── static/
│   ├── css/
│   │   └── styles.css
│   └── js/
│       └── htmx.min.js
```

## Cargo.toml

```toml
[package]
name = "rust-htmx-crud"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4.0"
tokio = { version = "1.0", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
sqlx = { version = "0.7", features = ["runtime-tokio-native-tls", "postgres", "mysql", "sqlite", "any"] }
askama = "0.12"
dotenv = "0.15"
config = "0.13"
futures = "0.3"
thiserror = "1.0"
```

## 数据库配置 (database.rs)

```rust
use sqlx::{Pool, Postgres, MySql, Sqlite, Any};
use std::fmt;
use thiserror::Error;
use config::{Config, File};
use serde::Deserialize;

#[derive(Debug, Deserialize)]
pub struct DatabaseSettings {
    pub url: String,
    pub db_type: String,
}

#[derive(Debug, Error)]
pub enum DatabaseError {
    #[error("Failed to connect to database: {0}")]
    ConnectionError(#[from] sqlx::Error),
    #[error("Configuration error: {0}")]
    ConfigError(#[from] config::ConfigError),
    #[error("Unsupported database type: {0}")]
    UnsupportedDatabaseType(String),
}

pub enum DatabaseConnection {
    Postgres(Pool<Postgres>),
    MySql(Pool<MySql>),
    Sqlite(Pool<Sqlite>),
    Any(Pool<Any>),
}

impl DatabaseConnection {
    pub async fn new(settings: &DatabaseSettings) -> Result<Self, DatabaseError> {
        match settings.db_type.as_str() {
            "postgres" => {
                let pool = Pool::<Postgres>::connect(&settings.url).await?;
                Ok(DatabaseConnection::Postgres(pool))
            }
            "mysql" => {
                let pool = Pool::<MySql>::connect(&settings.url).await?;
                Ok(DatabaseConnection::MySql(pool))
            }
            "sqlite" => {
                let pool = Pool::<Sqlite>::connect(&settings.url).await?;
                Ok(DatabaseConnection::Sqlite(pool))
            }
            "any" => {
                let pool = Pool::<Any>::connect(&settings.url).await?;
                Ok(DatabaseConnection::Any(pool))
            }
            _ => Err(DatabaseError::UnsupportedDatabaseType(settings.db_type.clone())),
        }
    }

    pub async fn get_table_columns(&self, table_name: &str) -> Result<Vec<String>, DatabaseError> {
        let query = match self {
            DatabaseConnection::Postgres(pool) => {
                sqlx::query(
                    "SELECT column_name FROM information_schema.columns WHERE table_name = $1 ORDER BY ordinal_position"
                )
                .bind(table_name)
                .fetch_all(pool)
                .await?
            }
            DatabaseConnection::MySql(pool) => {
                sqlx::query(
                    "SELECT column_name FROM information_schema.columns WHERE table_name = ? ORDER BY ordinal_position"
                )
                .bind(table_name)
                .fetch_all(pool)
                .await?
            }
            DatabaseConnection::Sqlite(pool) => {
                sqlx::query("PRAGMA table_info(?)")
                    .bind(table_name)
                    .fetch_all(pool)
                    .await?
            }
            DatabaseConnection::Any(pool) => {
                // Try different database types
                if let Ok(rows) = sqlx::query(
                    "SELECT column_name FROM information_schema.columns WHERE table_name = $1 ORDER BY ordinal_position"
                )
                .bind(table_name)
                .fetch_all(pool)
                .await {
                    rows
                } else if let Ok(rows) = sqlx::query("PRAGMA table_info(?)")
                    .bind(table_name)
                    .fetch_all(pool)
                    .await {
                    rows
                } else {
                    return Err(DatabaseError::ConnectionError(sqlx::Error::Configuration(
                        "Failed to get table columns".into()
                    )));
                }
            }
        };

        let columns = query
            .into_iter()
            .map(|row| row.get::<String, _>("column_name"))
            .collect();

        Ok(columns)
    }
}

pub async fn load_database_config() -> Result<DatabaseSettings, DatabaseError> {
    let mut config = Config::default();
    config.merge(File::with_name("config"))?;

    let settings: DatabaseSettings = config.try_into()?;
    Ok(settings)
}
```

## 模型定义 (models.rs)

```rust
use serde::{Deserialize, Serialize};
use sqlx::FromRow;

#[derive(Debug, Serialize, Deserialize, FromRow)]
pub struct DynamicRecord {
    #[serde(flatten)]
    pub fields: std::collections::HashMap<String, serde_json::Value>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct TableMetadata {
    pub name: String,
    pub columns: Vec<String>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct CrudResponse {
    pub success: bool,
    pub message: String,
    pub data: Option<serde_json::Value>,
}
```

## 请求处理器 (handlers.rs)

```rust
use actix_web::{web, HttpResponse, Responder};
use askama::Template;
use serde_json::json;
use sqlx::{Pool, Postgres, MySql, Sqlite, Any};
use std::collections::HashMap;
use crate::models::{DynamicRecord, TableMetadata, CrudResponse};
use crate::database::DatabaseConnection;

#[derive(Template)]
#[template(path = "list.html")]
struct ListTemplate {
    table: TableMetadata,
    records: Vec<DynamicRecord>,
}

#[derive(Template)]
#[template(path = "form.html")]
struct FormTemplate {
    table: TableMetadata,
    record: Option<DynamicRecord>,
}

pub async fn list_tables(
    db: web::Data<DatabaseConnection>,
) -> impl Responder {
    // Implement logic to list all tables in the database
    // This is database-specific
    unimplemented!()
}

pub async fn list_records(
    db: web::Data<DatabaseConnection>,
    table_name: web::Path<String>,
) -> impl Responder {
    let table_name = table_name.into_inner();
    
    let columns = match db.get_table_columns(&table_name).await {
        Ok(cols) => cols,
        Err(e) => return HttpResponse::InternalServerError().json(
            CrudResponse {
                success: false,
                message: format!("Failed to get table columns: {}", e),
                data: None,
            }
        ),
    };

    let query = format!("SELECT {} FROM {}", columns.join(", "), table_name);
    let records = match db {
        DatabaseConnection::Postgres(pool) => {
            sqlx::query_as::<_, DynamicRecord>(&query)
                .fetch_all(pool)
                .await
        }
        DatabaseConnection::MySql(pool) => {
            sqlx::query_as::<_, DynamicRecord>(&query)
                .fetch_all(pool)
                .await
        }
        DatabaseConnection::Sqlite(pool) => {
            sqlx::query_as::<_, DynamicRecord>(&query)
                .fetch_all(pool)
                .await
        }
        DatabaseConnection::Any(pool) => {
            sqlx::query_as::<_, DynamicRecord>(&query)
                .fetch_all(pool)
                .await
        }
    };

    let records = match records {
        Ok(recs) => recs,
        Err(e) => return HttpResponse::InternalServerError().json(
            CrudResponse {
                success: false,
                message: format!("Failed to fetch records: {}", e),
                data: None,
            }
        ),
    };

    let table_metadata = TableMetadata {
        name: table_name,
        columns,
    };

    let template = ListTemplate {
        table: table_metadata,
        records,
    };

    match template.render() {
        Ok(html) => HttpResponse::Ok().body(html),
        Err(e) => HttpResponse::InternalServerError().json(
            CrudResponse {
                success: false,
                message: format!("Template rendering error: {}", e),
                data: None,
            }
        ),
    }
}

pub async fn show_form(
    db: web::Data<DatabaseConnection>,
    path: web::Path<(String, Option<String>)>,
) -> impl Responder {
    let (table_name, record_id) = path.into_inner();
    
    let columns = match db.get_table_columns(&table_name).await {
        Ok(cols) => cols,
        Err(e) => return HttpResponse::InternalServerError().json(
            CrudResponse {
                success: false,
                message: format!("Failed to get table columns: {}", e),
                data: None,
            }
        ),
    };

    let record = if let Some(id) = record_id {
        let query = format!("SELECT {} FROM {} WHERE id = $1", columns.join(", "), table_name);
        let result = match db {
            DatabaseConnection::Postgres(pool) => {
                sqlx::query_as::<_, DynamicRecord>(&query)
                    .bind(id)
                    .fetch_one(pool)
                    .await
            }
            DatabaseConnection::MySql(pool) => {
                sqlx::query_as::<_, DynamicRecord>(&query)
                    .bind(id)
                    .fetch_one(pool)
                    .await
            }
            DatabaseConnection::Sqlite(pool) => {
                sqlx::query_as::<_, DynamicRecord>(&query)
                    .bind(id)
                    .fetch_one(pool)
                    .await
            }
            DatabaseConnection::Any(pool) => {
                sqlx::query_as::<_, DynamicRecord>(&query)
                    .bind(id)
                    .fetch_one(pool)
                    .await
            }
        };

        match result {
            Ok(rec) => Some(rec),
            Err(e) => return HttpResponse::InternalServerError().json(
                CrudResponse {
                    success: false,
                    message: format!("Failed to fetch record: {}", e),
                    data: None,
                }
            ),
        }
    } else {
        None
    };

    let table_metadata = TableMetadata {
        name: table_name,
        columns,
    };

    let template = FormTemplate {
        table: table_metadata,
        record,
    };

    match template.render() {
        Ok(html) => HttpResponse::Ok().body(html),
        Err(e) => HttpResponse::InternalServerError().json(
            CrudResponse {
                success: false,
                message: format!("Template rendering error: {}", e),
                data: None,
            }
        ),
    }
}

pub async fn create_or_update_record(
    db: web::Data<DatabaseConnection>,
    path: web::Path<String>,
    form: web::Form<HashMap<String, String>>,
) -> impl Responder {
    let table_name = path.into_inner();
    
    let columns = match db.get_table_columns(&table_name).await {
        Ok(cols) => cols,
        Err(e) => return HttpResponse::InternalServerError().json(
            CrudResponse {
                success: false,
                message: format!("Failed to get table columns: {}", e),
                data: None,
            }
        ),
    };

    let id = form.get("id").cloned();
    let is_update = id.is_some();

    let field_names: Vec<&str> = columns.iter().map(|s| s.as_str()).collect();
    let field_values: Vec<&str> = columns.iter()
        .filter(|&col| col != "id")
        .map(|col| form.get(col).map(|s| s.as_str()).unwrap_or_default())
        .collect();

    let query = if is_update {
        let set_clause = columns.iter()
            .filter(|&col| col != "id")
            .map(|col| format!("{} = ?", col))
            .collect::<Vec<_>>()
            .join(", ");
        
        format!("UPDATE {} SET {} WHERE id = ?", table_name, set_clause)
    } else {
        let columns_clause = columns.iter()
            .filter(|&col| col != "id")
            .map(|s| s.as_str())
            .collect::<Vec<_>>()
            .join(", ");
        
        let values_placeholders = columns.iter()
            .filter(|&col| col != "id")
            .map(|_| "?")
            .collect::<Vec<_>>()
            .join(", ");
        
        format!("INSERT INTO {} ({}) VALUES ({})", table_name, columns_clause, values_placeholders)
    };

    let mut query = sqlx::query(&query);
    
    for value in field_values {
        query = query.bind(value);
    }
    
    if is_update {
        query = query.bind(id.unwrap());
    }

    let result = match db {
        DatabaseConnection::Postgres(pool) => query.execute(pool).await,
        DatabaseConnection::MySql(pool) => query.execute(pool).await,
        DatabaseConnection::Sqlite(pool) => query.execute(pool).await,
        DatabaseConnection::Any(pool) => query.execute(pool).await,
    };

    match result {
        Ok(_) => HttpResponse::Ok().json(CrudResponse {
            success: true,
            message: if is_update { 
                "Record updated successfully".to_string() 
            } else { 
                "Record created successfully".to_string() 
            },
            data: None,
        }),
        Err(e) => HttpResponse::InternalServerError().json(CrudResponse {
            success: false,
            message: format!("Database error: {}", e),
            data: None,
        }),
    }
}

pub async fn delete_record(
    db: web::Data<DatabaseConnection>,
    path: web::Path<(String, String)>,
) -> impl Responder {
    let (table_name, record_id) = path.into_inner();
    
    let query = format!("DELETE FROM {} WHERE id = ?", table_name);
    let result = match db {
        DatabaseConnection::Postgres(pool) => {
            sqlx::query(&query)
                .bind(record_id)
                .execute(pool)
                .await
        }
        DatabaseConnection::MySql(pool) => {
            sqlx::query(&query)
                .bind(record_id)
                .execute(pool)
                .await
        }
        DatabaseConnection::Sqlite(pool) => {
            sqlx::query(&query)
                .bind(record_id)
                .execute(pool)
                .await
        }
        DatabaseConnection::Any(pool) => {
            sqlx::query(&query)
                .bind(record_id)
                .execute(pool)
                .await
        }
    };

    match result {
        Ok(_) => HttpResponse::Ok().json(CrudResponse {
            success: true,
            message: "Record deleted successfully".to_string(),
            data: None,
        }),
        Err(e) => HttpResponse::InternalServerError().json(CrudResponse {
            success: false,
            message: format!("Database error: {}", e),
            data: None,
        }),
    }
}
```

## 主应用 (main.rs)

```rust
use actix_web::{web, App, HttpServer};
use dotenv::dotenv;
use std::sync::Arc;

mod models;
mod database;
mod handlers;
mod templates;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    dotenv().ok();
    
    let db_settings = database::load_database_config().await
        .expect("Failed to load database configuration");
    
    let db_connection = database::DatabaseConnection::new(&db_settings).await
        .expect("Failed to connect to database");
    
    let db_data = web::Data::new(db_connection);

    HttpServer::new(move || {
        App::new()
            .app_data(db_data.clone())
            .service(web::resource("/").route(web::get().to(handlers::list_tables)))
            .service(web::resource("/tables/{table_name}").route(web::get().to(handlers::list_records)))
            .service(web::resource("/tables/{table_name}/form/{record_id}").route(web::get().to(handlers::show_form)))
            .service(web::resource("/tables/{table_name}/form").route(web::get().to(handlers::show_form)))
            .service(web::resource("/tables/{table_name}").route(web::post().to(handlers::create_or_update_record)))
            .service(web::resource("/tables/{table_name}/{record_id}").route(web::delete().to(handlers::delete_record)))
            .service(actix_files::Files::new("/static", "static"))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

## 模板文件 (templates/)

### base.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Rust HTMX CRUD</title>
    <link href="https://cdn.jsdelivr.net/npm/daisyui@3.9.4/dist/full.css" rel="stylesheet" type="text/css" />
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="/static/js/htmx.min.js"></script>
    <link rel="stylesheet" href="/static/css/styles.css">
</head>
<body class="p-4">
    <div class="container mx-auto">
        <h1 class="text-3xl font-bold mb-6">Rust HTMX CRUD</h1>
        {% block content %}{% endblock %}
    </div>
</body>
</html>
```

### list.html

```html
{% extends "base.html" %}

{% block content %}
<div class="mb-6">
    <h2 class="text-2xl font-semibold mb-4">{{ table.name }}</h2>
    
    <div class="flex justify-end mb-4">
        <button class="btn btn-primary"
                hx-get="/tables/{{ table.name }}/form"
                hx-target="body">
            Add New Record
        </button>
    </div>

    <div id="records-table">
        {% include "_partials/table.html" %}
    </div>
</div>
{% endblock %}
```

### _partials/table.html

```html
<div class="overflow-x-auto">
    <table class="table table-zebra">
        <thead>
            <tr>
                {% for column in table.columns %}
                <th>{{ column }}</th>
                {% endfor %}
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            {% for record in records %}
            <tr>
                {% for column in table.columns %}
                <td>{{ record.fields.get(column).unwrap_or(&serde_json::Value::Null) }}</td>
                {% endfor %}
                <td class="flex space-x-2">
                    <button class="btn btn-sm btn-info"
                            hx-get="/tables/{{ table.name }}/form/{{ record.fields.get("id").unwrap_or(&serde_json::Value::String("".to_string())) }}"
                            hx-target="body">
                        Edit
                    </button>
                    <button class="btn btn-sm btn-error"
                            hx-delete="/tables/{{ table.name }}/{{ record.fields.get("id").unwrap_or(&serde_json::Value::String("".to_string())) }}"
                            hx-target="#records-table"
                            hx-confirm="Are you sure you want to delete this record?">
                        Delete
                    </button>
                </td>
            </tr>
            {% endfor %}
        </tbody>
    </table>
</div>
```

### form.html

```html
{% extends "base.html" %}

{% block content %}
<div class="card w-full bg-base-100 shadow-xl">
    <div class="card-body">
        <h2 class="card-title">
            {% if record.is_some() %}Edit{% else %}Create{% endif %} Record in {{ table.name }}
        </h2>
        
        <form hx-post="/tables/{{ table.name }}"
              hx-target="#records-table">
            <input type="hidden" name="id" 
                   value="{{ record.as_ref().and_then(|r| r.fields.get("id").and_then(|v| v.as_str()).unwrap_or("") }}" />
            
            <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                {% include "_partials/form_fields.html" %}
            </div>
            
            <div class="card-actions justify-end mt-6">
                <button type="button" class="btn btn-ghost"
                        hx-get="/tables/{{ table.name }}"
                        hx-target="body">
                    Cancel
                </button>
                <button type="submit" class="b
