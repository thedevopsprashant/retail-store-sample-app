# Retail Store Microservices Architecture - EKS Deployment Reference

> **Document Purpose**: Comprehensive technical reference for deploying and managing the retail store microservices on AWS EKS
>
> **Last Updated**: 2026-02-23
>
> **Target Audience**: DevOps Engineers, Platform Engineers, SREs

---

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Services Breakdown](#services-breakdown)
3. [Service Communication Flow](#service-communication-flow)
4. [Data Persistence Architecture](#data-persistence-architecture)
5. [Container Images & Registry](#container-images--registry)
6. [Security Configuration](#security-configuration)
7. [Health Checks & Observability](#health-checks--observability)
8. [Resource Requirements](#resource-requirements)
9. [EKS Deployment Checklist](#eks-deployment-checklist)
10. [Deployment Strategy](#deployment-strategy)
11. [Configuration Reference](#configuration-reference)
12. [Troubleshooting Guide](#troubleshooting-guide)

---

## Architecture Overview

This is a **cloud-native microservices application** built for AWS with 5 independent services communicating via HTTP/REST APIs.

```
                          ┌─────────────────┐
                          │   UI Service    │
                          │  (Java/Spring)  │
                          │    Port 8080    │
                          │  API Gateway    │
                          └────────┬────────┘
                                   │
                 ┌─────────────────┼─────────────────┬────────────┐
                 │                 │                 │            │
                 ▼                 ▼                 ▼            ▼
         ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
         │   Catalog    │  │     Cart     │  │   Checkout   │  │    Orders    │
         │   Service    │  │   Service    │  │   Service    │  │   Service    │
         │   (Go/Gin)   │  │(Java/Spring) │  │(Node/NestJS) │  │(Java/Spring) │
         │  Port 8080   │  │  Port 8080   │  │  Port 8080   │  │  Port 8080   │
         └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
                │                 │                 │                 │
                ▼                 ▼                 ▼                 ▼
         ┌──────────┐      ┌──────────┐      ┌──────────┐     ┌──────────────┐
         │  MySQL   │      │ DynamoDB │      │  Redis   │     │  PostgreSQL  │
         │   RDS    │      │          │      │ElastiCache     │     RDS      │
         │ (opt.)   │      │ (opt.)   │      │ (opt.)   │     │  (opt.)      │
         └──────────┘      └──────────┘      └──────────┘     └──────┬───────┘
                                                                      │
                                                                      ▼
                                                               ┌──────────────┐
                                                               │RabbitMQ / SQS│
                                                               │   (opt.)     │
                                                               └──────────────┘
```

**Design Principles**:
- Microservices pattern with clear service boundaries
- Stateless application design (state in external stores)
- Cloud-native (AWS-optimized with managed services)
- Container-first (Kubernetes/EKS deployment)
- Observable (Prometheus metrics, OpenTelemetry tracing)
- Secure by default (non-root containers, read-only filesystems)

---

## Services Breakdown

### 1. UI Service (Frontend/BFF - Backend for Frontend)

#### Technology Stack
- **Language**: Java 21
- **Framework**: Spring Boot 3.5.3, Spring WebFlux (reactive)
- **Template Engine**: Thymeleaf
- **Build Tool**: Maven
- **Base Image**: Amazon Linux 2023
- **Port**: 8080

#### Business Logic & Purpose
- Web-based frontend interface for the retail store
- Acts as an API gateway/BFF (Backend for Frontend)
- Aggregates data from all backend services
- Provides web UI for:
  - Product browsing and search
  - Shopping cart management
  - Checkout flow
  - Order history
- Includes optional AI chat feature (AWS Bedrock/OpenAI integration)

#### Service Communication
**Outbound Dependencies** (configured via environment variables):
- **Catalog Service**: `RETAIL_UI_ENDPOINTS_CATALOG` (default: `http://retail-store-catalog:80`)
- **Cart Service**: `RETAIL_UI_ENDPOINTS_CARTS` (default: `http://retail-store-cart:80`)
- **Orders Service**: `RETAIL_UI_ENDPOINTS_ORDERS` (default: `http://retail-store-orders:80`)
- **Checkout Service**: `RETAIL_UI_ENDPOINTS_CHECKOUT` (default: `http://retail-store-checkout:80`)

**Protocol**: HTTP/REST using Kiota-generated clients from OpenAPI specs

#### External Dependencies
- **Storage**: In-memory session state (stateless)
- **Optional AI**: AWS Bedrock or OpenAI for chat feature

#### Configuration Management
**Key Environment Variables**:
```bash
# Service Endpoints
RETAIL_UI_ENDPOINTS_CATALOG=http://retail-store-catalog:80
RETAIL_UI_ENDPOINTS_CARTS=http://retail-store-cart:80
RETAIL_UI_ENDPOINTS_ORDERS=http://retail-store-orders:80
RETAIL_UI_ENDPOINTS_CHECKOUT=http://retail-store-checkout:80

# UI Configuration
RETAIL_UI_THEME=default
RETAIL_UI_CHAT_ENABLED=false
RETAIL_UI_CHAT_PROVIDER=mock  # mock/bedrock/openai

# AI Chat (Optional)
RETAIL_UI_CHAT_BEDROCK_REGION=us-east-1
RETAIL_UI_CHAT_OPENAI_API_KEY=<secret>
```

#### Health Checks
- **Health**: `/actuator/health`
- **Readiness**: `/actuator/health/readiness`
- **Liveness**: `/actuator/health/liveness`
- **Metrics**: `/actuator/prometheus`
- **Framework**: Spring Boot Actuator

#### Container Configuration
- **Dockerfile**: `src/ui/Dockerfile`
- **Helm Chart**: `src/ui/chart/`
- **User**: Non-root (appuser:1000)
- **Security**: Read-only root filesystem, capabilities dropped
- **Resources**:
  - CPU Request: 128m
  - Memory Request: 512Mi
  - Memory Limit: 512Mi

#### API Endpoints (Internal)
- `GET /` - Home page
- `GET /catalogue` - Product catalog page
- `GET /cart` - Shopping cart page
- `GET /checkout` - Checkout page
- `GET /orders` - Orders page
- `GET /actuator/health` - Health check

---

### 2. Catalog Service (Product Catalog Management)

#### Technology Stack
- **Language**: Go 1.24.4
- **Framework**: Gin (HTTP web framework)
- **ORM**: GORM (Go Object Relational Mapper)
- **Build Tool**: Go modules
- **Base Image**: Amazon Linux 2023
- **Port**: 8080

#### Business Logic & Purpose
- Manages product catalog data (CRUD operations)
- Provides product search, filtering, and retrieval
- Supports product tags and categories
- Read-heavy workload pattern (optimized for queries)
- Sample data includes clothing items (shirts, jackets, etc.)

#### Service Communication
**Inbound**: REST API consumed by UI service

**API Endpoints**:
```
GET  /catalogue/products       - List products (supports filters: tags, order, size)
GET  /catalogue/product/:id    - Get product details by ID
GET  /catalogue/tags           - List all available product tags
GET  /catalogue/size           - Get total catalog size
GET  /health                   - Health check
GET  /metrics                  - Prometheus metrics
GET  /topology                 - Service topology info
```

#### External Dependencies
**Database**: MySQL (configurable)

**Storage Modes**:
1. **In-Memory** (default): No external dependencies, data reset on restart
2. **MySQL/RDS**: Persistent storage

**MySQL Configuration**:
```bash
RETAIL_CATALOG_PERSISTENCE_PROVIDER=mysql
RETAIL_CATALOG_PERSISTENCE_ENDPOINT=catalog-db.abc123.us-east-1.rds.amazonaws.com:3306
RETAIL_CATALOG_PERSISTENCE_DB_NAME=catalogdb
RETAIL_CATALOG_PERSISTENCE_USER=catalog_user
RETAIL_CATALOG_PERSISTENCE_PASSWORD=<secret>
RETAIL_CATALOG_PERSISTENCE_CONNECT_TIMEOUT=5s
```

#### Database Schema
```sql
-- Products table
CREATE TABLE products (
    id VARCHAR(40) PRIMARY KEY,
    name VARCHAR(100),
    description TEXT,
    image_url VARCHAR(255),
    price DECIMAL(10,2),
    count INT
);

-- Product tags table
CREATE TABLE product_tags (
    product_id VARCHAR(40),
    tag VARCHAR(50),
    FOREIGN KEY (product_id) REFERENCES products(id)
);
```

#### Configuration Management
**Environment Variables**:
```bash
PORT=8080
RETAIL_CATALOG_PERSISTENCE_PROVIDER=in-memory  # in-memory | mysql
RETAIL_CATALOG_PERSISTENCE_ENDPOINT=<hostname:port>
RETAIL_CATALOG_PERSISTENCE_DB_NAME=catalogdb
RETAIL_CATALOG_PERSISTENCE_USER=catalog_user
RETAIL_CATALOG_PERSISTENCE_PASSWORD=<secret>
RETAIL_CATALOG_PERSISTENCE_CONNECT_TIMEOUT=5s
GIN_MODE=release  # release for production
```

#### Health Checks
- **Health**: `/health` - Returns service status
- **Metrics**: `/metrics` - Prometheus format metrics
- **Topology**: `/topology` - Returns persistence provider info

#### Container Configuration
- **Dockerfile**: `src/catalog/Dockerfile`
- **Helm Chart**: `src/catalog/chart/`
- **User**: Non-root (appuser:1000)
- **Security**: Minimal capabilities
- **Resources**:
  - CPU Request: 256m
  - Memory Request: 256Mi
  - Memory Limit: 256Mi

#### Performance Characteristics
- Optimized for read operations
- Response time: <100ms for product queries
- Supports concurrent requests (Gin's goroutine model)
- Connection pooling for database

---

### 3. Cart Service (Shopping Cart Management)

#### Technology Stack
- **Language**: Java 21
- **Framework**: Spring Boot 3.5.3
- **Build Tool**: Maven
- **Base Image**: Amazon Linux 2023
- **Port**: 8080

#### Business Logic & Purpose
- Manages shopping cart operations (add, remove, update items)
- Session-based cart management
- Cart merging for anonymous-to-authenticated user transitions
- Stores item details with pricing snapshot
- Handles cart expiration and cleanup

#### Service Communication
**Inbound**: REST API consumed by UI service

**API Endpoints**:
```
GET    /carts/{customerId}                     - Get cart by customer ID
DELETE /carts/{customerId}                     - Delete cart
GET    /carts/{customerId}/merge?sessionId=    - Merge anonymous cart with user cart
GET    /carts/{customerId}/items               - List cart items
POST   /carts/{customerId}/items               - Add item to cart
DELETE /carts/{customerId}/items/{itemId}      - Remove item from cart
PATCH  /carts/{customerId}/items               - Update item quantity
GET    /actuator/health                        - Health check
GET    /actuator/prometheus                    - Metrics
```

#### External Dependencies
**Database**: DynamoDB (configurable)

**Storage Modes**:
1. **In-Memory** (default): No external dependencies, data reset on restart
2. **DynamoDB**: Persistent, scalable NoSQL storage

**DynamoDB Configuration**:
```bash
RETAIL_CART_PERSISTENCE_PROVIDER=dynamodb
RETAIL_CART_PERSISTENCE_DYNAMODB_ENDPOINT=https://dynamodb.us-east-1.amazonaws.com
RETAIL_CART_PERSISTENCE_DYNAMODB_TABLE_NAME=Items
RETAIL_CART_PERSISTENCE_DYNAMODB_CREATE_TABLE=false
```

#### DynamoDB Table Schema
```
Table Name: Items
Partition Key: id (String) - cart item ID
Sort Key: customerId (String) - customer/session ID

Attributes:
- id: String (PK)
- customerId: String (SK)
- productId: String
- productName: String
- quantity: Number
- unitPrice: Number
- totalPrice: Number
```

**Required IAM Permissions** (for IRSA):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query",
        "dynamodb:Scan"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/Items"
    }
  ]
}
```

#### Configuration Management
**Environment Variables**:
```bash
PORT=8080
RETAIL_CART_PERSISTENCE_PROVIDER=in-memory  # in-memory | dynamodb
RETAIL_CART_PERSISTENCE_DYNAMODB_ENDPOINT=https://dynamodb.us-east-1.amazonaws.com
RETAIL_CART_PERSISTENCE_DYNAMODB_TABLE_NAME=Items
RETAIL_CART_PERSISTENCE_DYNAMODB_CREATE_TABLE=false
```

#### Health Checks
- **Health**: `/actuator/health`
- **Readiness**: `/actuator/health/readiness`
- **Liveness**: `/actuator/health/liveness`
- **Metrics**: `/actuator/prometheus`

#### Container Configuration
- **Dockerfile**: `src/cart/Dockerfile`
- **Helm Chart**: `src/cart/chart/`
- **User**: Non-root (appuser:1000)
- **Security**: Read-only root filesystem
- **Resources**:
  - CPU Request: 256m
  - Memory Request: 512Mi
  - Memory Limit: 512Mi

#### Key Features
- **Cart Merging**: When anonymous user logs in, their session cart merges with their user cart
- **Price Snapshot**: Captures product price at time of adding to cart
- **Session Management**: Supports both authenticated and anonymous users
- **TTL Support**: Carts can expire after inactivity period

---

### 4. Orders Service (Order Management & History)

#### Technology Stack
- **Language**: Java 21
- **Framework**: Spring Boot 3.5.3, Spring Data JDBC
- **Database Migration**: Flyway
- **Build Tool**: Maven
- **Base Image**: Amazon Linux 2023
- **Port**: 8080

#### Business Logic & Purpose
- Manages order creation and retrieval
- Persists order history with line items
- Publishes order events to message queue for downstream processing
- Handles order lifecycle management
- Maintains audit trail of orders

#### Service Communication
**Inbound**: REST API consumed by Checkout service and UI service

**API Endpoints**:
```
POST   /orders              - Create new order
GET    /orders              - List orders (supports customerId filter)
GET    /orders/{id}         - Get order by ID
GET    /actuator/health     - Health check
GET    /actuator/prometheus - Metrics
```

**Outbound**: Event publishing to message queue

**Order Event Schema**:
```json
{
  "orderId": "550e8400-e29b-41d4-a716-446655440000",
  "customerId": "user123",
  "total": 149.98,
  "items": [
    {
      "productId": "prod-001",
      "productName": "Blue Shirt",
      "quantity": 2,
      "price": 74.99
    }
  ],
  "timestamp": "2026-02-23T10:30:00Z",
  "status": "CREATED"
}
```

#### External Dependencies

**Database**: PostgreSQL (configurable)

**Storage Modes**:
1. **In-Memory** (default): H2 database, data reset on restart
2. **PostgreSQL/RDS**: Persistent relational storage

**PostgreSQL Configuration**:
```bash
RETAIL_ORDERS_PERSISTENCE_PROVIDER=postgres
RETAIL_ORDERS_PERSISTENCE_POSTGRES_ENDPOINT=orders-db.abc123.us-east-1.rds.amazonaws.com:5432
RETAIL_ORDERS_PERSISTENCE_POSTGRES_DBNAME=ordersdb
RETAIL_ORDERS_PERSISTENCE_POSTGRES_USERNAME=orders_user
RETAIL_ORDERS_PERSISTENCE_POSTGRES_PASSWORD=<secret>
```

**Message Queue**: RabbitMQ or SQS (configurable)

**Messaging Modes**:
1. **In-Memory** (default): Events logged only, not published
2. **RabbitMQ**: AMQP-based message broker
3. **SQS**: AWS managed queue service

**RabbitMQ Configuration**:
```bash
RETAIL_ORDERS_MESSAGING_PROVIDER=rabbitmq
RETAIL_ORDERS_MESSAGING_RABBITMQ_ADDRESSES=amqp://rabbitmq.default.svc.cluster.local:5672
RETAIL_ORDERS_MESSAGING_RABBITMQ_USERNAME=guest
RETAIL_ORDERS_MESSAGING_RABBITMQ_PASSWORD=<secret>
```

**SQS Configuration**:
```bash
RETAIL_ORDERS_MESSAGING_PROVIDER=sqs
RETAIL_ORDERS_MESSAGING_SQS_TOPIC=arn:aws:sqs:us-east-1:123456789012:retail-orders
```

**Required IAM Permissions** (for IRSA with SQS):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:GetQueueUrl"
      ],
      "Resource": "arn:aws:sqs:*:*:retail-orders"
    }
  ]
}
```

#### Database Schema
```sql
-- Orders table
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    customer_id VARCHAR(255) NOT NULL,
    order_date TIMESTAMP NOT NULL,
    total DECIMAL(10,2) NOT NULL,
    status VARCHAR(50) NOT NULL
);

-- Order items table
CREATE TABLE order_items (
    id UUID PRIMARY KEY,
    order_id UUID NOT NULL,
    product_id VARCHAR(255) NOT NULL,
    product_name VARCHAR(255) NOT NULL,
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    total_price DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id)
);

-- Indexes
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_order_items_order ON order_items(order_id);
```

**Database Migrations**: Managed by Flyway
- Migration files location: `src/orders/src/main/resources/db/migration/`
- Runs automatically on application startup
- Version-controlled schema changes

#### Configuration Management
**Environment Variables**:
```bash
PORT=8080

# Database Configuration
RETAIL_ORDERS_PERSISTENCE_PROVIDER=in-memory  # in-memory | postgres
RETAIL_ORDERS_PERSISTENCE_POSTGRES_ENDPOINT=<hostname:port>
RETAIL_ORDERS_PERSISTENCE_POSTGRES_DBNAME=ordersdb
RETAIL_ORDERS_PERSISTENCE_POSTGRES_USERNAME=orders_user
RETAIL_ORDERS_PERSISTENCE_POSTGRES_PASSWORD=<secret>

# Messaging Configuration
RETAIL_ORDERS_MESSAGING_PROVIDER=in-memory  # in-memory | rabbitmq | sqs
RETAIL_ORDERS_MESSAGING_RABBITMQ_ADDRESSES=amqp://localhost:5672
RETAIL_ORDERS_MESSAGING_RABBITMQ_USERNAME=guest
RETAIL_ORDERS_MESSAGING_RABBITMQ_PASSWORD=<secret>
RETAIL_ORDERS_MESSAGING_SQS_TOPIC=arn:aws:sqs:region:account:queue-name
```

#### Health Checks
- **Health**: `/actuator/health` (includes DB connection check)
- **Readiness**: `/actuator/health/readiness`
- **Liveness**: `/actuator/health/liveness`
- **Metrics**: `/actuator/prometheus`

#### Container Configuration
- **Dockerfile**: `src/orders/Dockerfile`
- **Helm Chart**: `src/orders/chart/`
- **User**: Non-root (appuser:1000)
- **Security**: Read-only root filesystem
- **Resources**:
  - CPU Request: 256m
  - Memory Request: 512Mi
  - Memory Limit: 512Mi

#### Key Features
- **Event-Driven**: Publishes order events for downstream processing (e.g., fulfillment, notifications)
- **Transactional**: Database transactions ensure order consistency
- **Audit Trail**: Complete order history maintained
- **Schema Migration**: Flyway manages database versioning

---

### 5. Checkout Service (Checkout Orchestration)

#### Technology Stack
- **Language**: TypeScript (Node.js 20)
- **Framework**: NestJS (enterprise Node.js framework)
- **Build Tool**: Yarn
- **Base Image**: Amazon Linux 2023
- **Port**: 8080

#### Business Logic & Purpose
- Orchestrates checkout process
- Calculates shipping rates based on items and destination
- Calculates tax based on location
- Manages checkout session state
- Coordinates with Orders service to finalize purchases
- Provides shipping options (standard, express, priority)
- Generates payment tokens (mock implementation)

#### Service Communication
**Inbound**: REST API consumed by UI service

**API Endpoints**:
```
GET    /checkout/{customerId}         - Get checkout session
POST   /checkout/{customerId}/update  - Update checkout (shipping, payment)
POST   /checkout/{customerId}/submit  - Submit order (creates order via Orders API)
GET    /health                        - Health check
GET    /metrics                       - Prometheus metrics
GET    /topology                      - Service topology info
```

**Outbound**: REST API calls
- **Orders Service**: `RETAIL_CHECKOUT_ENDPOINTS_ORDERS` - To create final order

**Checkout Session Schema**:
```json
{
  "customerId": "user123",
  "items": [...],
  "subtotal": 149.98,
  "shipping": {
    "method": "standard",
    "cost": 5.99,
    "address": {...}
  },
  "tax": 12.00,
  "total": 167.97,
  "paymentToken": "tok_1234567890"
}
```

#### External Dependencies

**Cache/Session Store**: Redis (configurable)

**Storage Modes**:
1. **In-Memory** (default): No external dependencies, sessions reset on restart
2. **Redis/ElastiCache**: Distributed session cache for HA

**Redis Configuration**:
```bash
RETAIL_CHECKOUT_PERSISTENCE_PROVIDER=redis
RETAIL_CHECKOUT_PERSISTENCE_REDIS_URL=redis://checkout-cache.abc123.0001.use1.cache.amazonaws.com:6379
RETAIL_CHECKOUT_PERSISTENCE_REDIS_READER_URL=redis://checkout-cache-ro.abc123.0001.use1.cache.amazonaws.com:6379
```

**Redis Data Structure**:
```
Key Pattern: checkout:{customerId}
TTL: 1 hour (checkout sessions expire)
Value: JSON-serialized checkout session object
```

#### Configuration Management
**Environment Variables**:
```bash
PORT=8080

# Session Storage
RETAIL_CHECKOUT_PERSISTENCE_PROVIDER=in-memory  # in-memory | redis
RETAIL_CHECKOUT_PERSISTENCE_REDIS_URL=redis://localhost:6379
RETAIL_CHECKOUT_PERSISTENCE_REDIS_READER_URL=redis://localhost:6379

# Service Dependencies
RETAIL_CHECKOUT_ENDPOINTS_ORDERS=http://retail-store-orders:80

# Business Logic
RETAIL_CHECKOUT_SHIPPING_NAME_PREFIX=Super Duper Shipping
```

#### Health Checks
- **Health**: `/health` - NestJS Terminus health check
- **Metrics**: `/metrics` - Prometheus format metrics
- **Topology**: `/topology` - Returns persistence provider and dependencies

#### Container Configuration
- **Dockerfile**: `src/checkout/Dockerfile`
- **Helm Chart**: `src/checkout/chart/`
- **User**: Non-root (appuser:1000)
- **Security**: Minimal privileges
- **Resources**:
  - CPU Request: 128m
  - Memory Request: 256Mi
  - Memory Limit: 256Mi

#### Key Features
- **Session Management**: Maintains checkout state across requests
- **Shipping Calculation**: Dynamic shipping cost based on items and method
- **Tax Calculation**: Location-based tax calculation (mock implementation)
- **Payment Processing**: Token generation for payment gateway (mock)
- **Order Coordination**: Calls Orders service to finalize purchase

#### Shipping Methods
```typescript
const SHIPPING_METHODS = {
  standard: { days: '5-7', baseCost: 5.99 },
  express: { days: '2-3', baseCost: 12.99 },
  priority: { days: '1-2', baseCost: 24.99 }
};
```

---

## Service Communication Flow

### Request Flow: Product Browsing
```
User Browser → UI Service → Catalog Service → MySQL/In-Memory
                  ↓
            (renders HTML)
                  ↓
            User Browser
```

### Request Flow: Adding to Cart
```
User Browser → UI Service → Cart Service → DynamoDB/In-Memory
                  ↓
              (updates cart)
                  ↓
            User Browser
```

### Request Flow: Checkout & Order
```
User Browser → UI Service → Checkout Service → Redis/In-Memory
                                    ↓
                              (calculate totals)
                                    ↓
                            Orders Service → PostgreSQL/In-Memory
                                    ↓
                            Message Queue (SQS/RabbitMQ)
                                    ↓
                              (order created)
                                    ↓
                            User Browser (order confirmation)
```

### Service Dependency Matrix

| Service  | Depends On | Protocol | Purpose |
|----------|------------|----------|---------|
| UI | Catalog, Cart, Checkout, Orders | HTTP/REST | Aggregate data for frontend |
| Catalog | MySQL (opt.) | JDBC | Product data persistence |
| Cart | DynamoDB (opt.) | AWS SDK | Cart persistence |
| Checkout | Redis (opt.), Orders | HTTP/REST | Session cache, order creation |
| Orders | PostgreSQL (opt.), Queue (opt.) | JDBC, AMQP/SQS | Order persistence, events |

### Network Ports

All services expose:
- **Application Port**: 8080 (HTTP)
- **Service Port**: 80 (Kubernetes Service)

Kubernetes Service Discovery:
```bash
# DNS names within cluster
retail-store-ui.default.svc.cluster.local:80
retail-store-catalog.default.svc.cluster.local:80
retail-store-cart.default.svc.cluster.local:80
retail-store-checkout.default.svc.cluster.local:80
retail-store-orders.default.svc.cluster.local:80
```

---

## Data Persistence Architecture

### Persistence Strategy by Service

| Service | Development | Production | AWS Service | Data Type |
|---------|------------|------------|-------------|-----------|
| UI | Stateless | Stateless | N/A | None (session in memory) |
| Catalog | In-Memory | MySQL | RDS MySQL | Product catalog (read-heavy) |
| Cart | In-Memory | DynamoDB | DynamoDB | Shopping carts (key-value) |
| Checkout | In-Memory | Redis | ElastiCache | Checkout sessions (TTL) |
| Orders | In-Memory (H2) | PostgreSQL | RDS PostgreSQL | Orders & line items (relational) |
| Orders Events | In-Memory | SQS/RabbitMQ | Amazon SQS | Order events (async) |

### AWS Managed Services Setup

#### RDS MySQL (Catalog Service)
```terraform
resource "aws_db_instance" "catalog" {
  identifier           = "retail-catalog-db"
  engine              = "mysql"
  engine_version      = "8.0"
  instance_class      = "db.t3.micro"
  allocated_storage   = 20
  storage_encrypted   = true

  db_name  = "catalogdb"
  username = "catalog_user"
  password = var.catalog_db_password

  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.private.name

  backup_retention_period = 7
  skip_final_snapshot    = false

  tags = {
    Service = "catalog"
    Environment = "production"
  }
}
```

#### DynamoDB (Cart Service)
```terraform
resource "aws_dynamodb_table" "cart_items" {
  name           = "retail-cart-items"
  billing_mode   = "PAY_PER_REQUEST"  # On-demand scaling
  hash_key       = "id"
  range_key      = "customerId"

  attribute {
    name = "id"
    type = "S"
  }

  attribute {
    name = "customerId"
    type = "S"
  }

  ttl {
    attribute_name = "expiresAt"
    enabled        = true
  }

  point_in_time_recovery {
    enabled = true
  }

  tags = {
    Service = "cart"
    Environment = "production"
  }
}
```

#### ElastiCache Redis (Checkout Service)
```terraform
resource "aws_elasticache_replication_group" "checkout" {
  replication_group_id       = "retail-checkout-cache"
  replication_group_description = "Checkout session cache"

  engine               = "redis"
  engine_version       = "7.0"
  node_type           = "cache.t3.micro"
  number_cache_clusters = 2  # 1 primary + 1 replica

  parameter_group_name = "default.redis7"
  port                 = 6379

  subnet_group_name = aws_elasticache_subnet_group.private.name
  security_group_ids = [aws_security_group.redis.id]

  automatic_failover_enabled = true
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true

  tags = {
    Service = "checkout"
    Environment = "production"
  }
}
```

#### RDS PostgreSQL (Orders Service)
```terraform
resource "aws_db_instance" "orders" {
  identifier           = "retail-orders-db"
  engine              = "postgres"
  engine_version      = "15"
  instance_class      = "db.t3.micro"
  allocated_storage   = 20
  storage_encrypted   = true

  db_name  = "ordersdb"
  username = "orders_user"
  password = var.orders_db_password

  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.private.name

  backup_retention_period = 7
  skip_final_snapshot    = false

  tags = {
    Service = "orders"
    Environment = "production"
  }
}
```

#### Amazon SQS (Orders Events)
```terraform
resource "aws_sqs_queue" "orders" {
  name                       = "retail-orders-events"
  delay_seconds              = 0
  max_message_size          = 262144
  message_retention_seconds = 86400  # 1 day
  receive_wait_time_seconds = 10     # Long polling

  visibility_timeout_seconds = 30

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.orders_dlq.arn
    maxReceiveCount     = 3
  })

  tags = {
    Service = "orders"
    Environment = "production"
  }
}

resource "aws_sqs_queue" "orders_dlq" {
  name = "retail-orders-events-dlq"

  tags = {
    Service = "orders"
    Environment = "production"
  }
}
```

### Data Migration Strategy

#### Phase 1: Development (In-Memory Mode)
```bash
# All services use in-memory storage
# No external dependencies required
# Quick setup for development/testing
```

#### Phase 2: Staging (Mix Mode)
```bash
# Critical services use managed storage
# Catalog: MySQL RDS
# Orders: PostgreSQL RDS
# Non-critical services use in-memory
# Cart: In-memory
# Checkout: In-memory
```

#### Phase 3: Production (Full Managed Services)
```bash
# All services use AWS managed storage
# Catalog: MySQL RDS (Multi-AZ)
# Cart: DynamoDB (On-demand)
# Checkout: ElastiCache Redis (Multi-AZ)
# Orders: PostgreSQL RDS (Multi-AZ)
# Orders Events: SQS (FIFO optional)
```

---

## Container Images & Registry

### Container Registry
**AWS ECR**: `877009927033.dkr.ecr.ap-south-1.amazonaws.com`

### Image Tags
All services currently use tag: **`b85ff1a`**

### Image List
```bash
# UI Service
877009927033.dkr.ecr.ap-south-1.amazonaws.com/retail-store-ui:b85ff1a

# Catalog Service
877009927033.dkr.ecr.ap-south-1.amazonaws.com/retail-store-catalog:b85ff1a

# Cart Service
877009927033.dkr.ecr.ap-south-1.amazonaws.com/retail-store-cart:b85ff1a

# Checkout Service
877009927033.dkr.ecr.ap-south-1.amazonaws.com/retail-store-checkout:b85ff1a

# Orders Service
877009927033.dkr.ecr.ap-south-1.amazonaws.com/retail-store-orders:b85ff1a
```

### Pull Secret Configuration

Create Kubernetes secret for ECR authentication:

```bash
# Create ECR pull secret
kubectl create secret docker-registry regcred \
  --docker-server=877009927033.dkr.ecr.ap-south-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region ap-south-1) \
  --namespace=default
```

Or using IRSA (recommended):

```yaml
# ServiceAccount with ECR access via IRSA
apiVersion: v1
kind: ServiceAccount
metadata:
  name: retail-store-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::877009927033:role/retail-store-ecr-role
```

### CI/CD Pipeline

Images are built and pushed via GitHub Actions:

```yaml
# .github/workflows/build-push.yml
# Triggered on:
# - Push to main branch
# - Manual workflow dispatch
#
# Builds all 5 service images
# Tags with git commit SHA
# Pushes to ECR
# Updates Helm chart values
```

Recent commits updating charts:
```
5015639 - updated readme
25ef8c7 - Update ui Helm chart to b85ff1a
02d038e - Update orders Helm chart to b85ff1a
e64f431 - Update catalog Helm chart to b85ff1a
341c0a9 - Update cart Helm chart to b85ff1a
```

### Image Build Details

All Dockerfiles use multi-stage builds:

**Stage 1: Build**
- Compile source code
- Run tests
- Create artifacts

**Stage 2: Runtime**
- Minimal base image (Amazon Linux 2023)
- Copy artifacts from build stage
- Non-root user
- Security hardening

Example (Go service):
```dockerfile
FROM public.ecr.aws/docker/library/golang:1.24 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o catalog

FROM public.ecr.aws/amazonlinux/amazonlinux:2023
RUN yum install -y shadow-utils && \
    groupadd -r appuser -g 1000 && \
    useradd -u 1000 -r -g appuser -s /sbin/nologin appuser
USER appuser:appuser
COPY --from=builder /app/catalog /catalog
EXPOSE 8080
CMD ["/catalog"]
```

---

## Security Configuration

### Container Security

All containers implement security best practices:

#### 1. Non-Root User
```yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 1000
  runAsNonRoot: true
  fsGroup: 1000
```

#### 2. Read-Only Root Filesystem
```yaml
securityContext:
  readOnlyRootFilesystem: true
```

Writable volumes mounted only where necessary:
```yaml
volumeMounts:
  - name: tmp
    mountPath: /tmp
volumes:
  - name: tmp
    emptyDir: {}
```

#### 3. Dropped Capabilities
```yaml
securityContext:
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE  # Only if needed for port <1024
```

#### 4. Security Contexts (Pod Level)
```yaml
podSecurityContext:
  seccompProfile:
    type: RuntimeDefault
  fsGroup: 1000
  runAsNonRoot: true
```

### Network Security

#### Service-to-Service Communication
```yaml
# Services use ClusterIP (internal only)
apiVersion: v1
kind: Service
metadata:
  name: retail-store-catalog
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: catalog
```

#### Network Policies (Optional)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: catalog-netpol
spec:
  podSelector:
    matchLabels:
      app: catalog
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: ui  # Only UI can access Catalog
      ports:
        - protocol: TCP
          port: 8080
```

### Secrets Management

#### Database Credentials
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: catalog-db-secret
type: Opaque
stringData:
  username: catalog_user
  password: <generated-password>
  endpoint: catalog-db.abc123.us-east-1.rds.amazonaws.com
```

Referenced in deployment:
```yaml
env:
  - name: RETAIL_CATALOG_PERSISTENCE_USER
    valueFrom:
      secretKeyRef:
        name: catalog-db-secret
        key: username
  - name: RETAIL_CATALOG_PERSISTENCE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: catalog-db-secret
        key: password
```

#### AWS Secrets Manager Integration (Optional)
```bash
# Install External Secrets Operator
helm install external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets-system \
  --create-namespace

# Create ExternalSecret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: catalog-db-credentials
spec:
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: catalog-db-secret
  data:
    - secretKey: username
      remoteRef:
        key: retail/catalog/db
        property: username
    - secretKey: password
      remoteRef:
        key: retail/catalog/db
        property: password
```

### IAM Roles for Service Accounts (IRSA)

Enable fine-grained IAM permissions for pods:

#### Cart Service (DynamoDB Access)
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cart-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::877009927033:role/retail-cart-dynamodb-role
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cart
spec:
  template:
    spec:
      serviceAccountName: cart-sa  # Use IRSA
```

IAM Role Trust Policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::877009927033:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:sub": "system:serviceaccount:default:cart-sa"
        }
      }
    }
  ]
}
```

IAM Role Policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query",
        "dynamodb:Scan"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/retail-cart-items"
    }
  ]
}
```

#### Orders Service (SQS + RDS Access)
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orders-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::877009927033:role/retail-orders-role
```

IAM Role Policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:GetQueueUrl"
      ],
      "Resource": "arn:aws:sqs:*:*:retail-orders-events"
    }
  ]
}
```

### Security Scanning

#### Container Image Scanning
```bash
# ECR automatic scanning enabled
aws ecr put-image-scanning-configuration \
  --repository-name retail-store-ui \
  --image-scanning-configuration scanOnPush=true

# Manual scan
aws ecr start-image-scan \
  --repository-name retail-store-ui \
  --image-id imageTag=b85ff1a
```

#### Vulnerability Scanning with Trivy
```bash
# Scan images for vulnerabilities
trivy image 877009927033.dkr.ecr.ap-south-1.amazonaws.com/retail-store-ui:b85ff1a

# Scan Kubernetes manifests
trivy k8s --namespace default deployment/ui
```

---

## Health Checks & Observability

### Health Check Endpoints

#### Java Services (UI, Cart, Orders)
```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3
```

Health check response (healthy):
```json
{
  "status": "UP",
  "groups": ["liveness", "readiness"]
}
```

Health check response (unhealthy):
```json
{
  "status": "DOWN",
  "components": {
    "db": {
      "status": "DOWN",
      "details": {
        "error": "Connection refused"
      }
    }
  }
}
```

#### Go Service (Catalog)
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

Health check response:
```json
{
  "status": "healthy",
  "checks": {
    "database": "ok"
  }
}
```

#### Node.js Service (Checkout)
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

Health check response (NestJS Terminus):
```json
{
  "status": "ok",
  "info": {
    "redis": {
      "status": "up"
    }
  },
  "error": {},
  "details": {
    "redis": {
      "status": "up"
    }
  }
}
```

### Metrics (Prometheus)

All services expose Prometheus metrics:

#### Java Services
**Endpoint**: `/actuator/prometheus`

**Metrics**:
```
# Spring Boot Actuator metrics
http_server_requests_seconds_count
http_server_requests_seconds_sum
jvm_memory_used_bytes
jvm_gc_pause_seconds_count
process_cpu_usage
system_cpu_usage
jdbc_connections_active  # For DB services
```

#### Go Service
**Endpoint**: `/metrics`

**Metrics**:
```
# Custom application metrics
catalog_requests_total
catalog_requests_duration_seconds
catalog_products_count
go_goroutines
go_memstats_alloc_bytes
```

#### Node.js Service
**Endpoint**: `/metrics`

**Metrics**:
```
# NestJS Prometheus metrics
http_request_duration_seconds
http_requests_total
nodejs_heap_size_total_bytes
nodejs_heap_size_used_bytes
process_cpu_user_seconds_total
```

### Prometheus ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: retail-store-metrics
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      monitoring: "true"
  endpoints:
    - port: http
      path: /actuator/prometheus  # Java services
      interval: 30s
    - port: http
      path: /metrics  # Go/Node services
      interval: 30s
```

### Logging

All services log to stdout/stderr in structured format:

#### Java Services (Logback)
```json
{
  "timestamp": "2026-02-23T10:30:00.123Z",
  "level": "INFO",
  "thread": "http-nio-8080-exec-1",
  "logger": "com.example.cart.CartController",
  "message": "Cart created for customer: user123",
  "context": {
    "customerId": "user123",
    "traceId": "abc123def456",
    "spanId": "789ghi012jkl"
  }
}
```

#### Go Service (Structured Logging)
```json
{
  "time": "2026-02-23T10:30:00Z",
  "level": "info",
  "msg": "Product retrieved",
  "productId": "prod-001",
  "traceId": "abc123def456"
}
```

#### Node.js Service (Winston)
```json
{
  "timestamp": "2026-02-23T10:30:00.123Z",
  "level": "info",
  "message": "Checkout session created",
  "customerId": "user123",
  "traceId": "abc123def456"
}
```

### Distributed Tracing (OpenTelemetry)

All services support OpenTelemetry tracing:

#### Configuration
```yaml
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: http://otel-collector:4318
  - name: OTEL_SERVICE_NAME
    value: retail-store-ui
  - name: OTEL_TRACES_SAMPLER
    value: parentbased_traceidratio
  - name: OTEL_TRACES_SAMPLER_ARG
    value: "0.1"  # 10% sampling
```

#### AWS X-Ray Integration
```yaml
env:
  - name: OTEL_PROPAGATORS
    value: xray
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: http://xray-collector:4318
```

Trace example:
```
UI Service [100ms]
  ├── GET /catalogue [80ms]
  │   └── Catalog Service [70ms]
  │       └── MySQL Query [50ms]
  └── GET /carts/user123 [15ms]
      └── Cart Service [10ms]
          └── DynamoDB GetItem [5ms]
```

### Grafana Dashboards

#### Application Dashboard
- Request rate (requests/sec)
- Response time (p50, p95, p99)
- Error rate
- Active connections

#### Infrastructure Dashboard
- CPU usage
- Memory usage
- Network I/O
- Disk I/O

#### Business Metrics Dashboard
- Total orders
- Revenue
- Cart abandonment rate
- Checkout conversion rate

---

## Resource Requirements

### Per-Service Resources

| Service | CPU Request | CPU Limit | Memory Request | Memory Limit | Replicas (Min/Max) |
|---------|-------------|-----------|----------------|--------------|-------------------|
| UI | 128m | 500m | 512Mi | 512Mi | 2-10 |
| Catalog | 256m | 1000m | 256Mi | 256Mi | 2-10 |
| Cart | 256m | 1000m | 512Mi | 512Mi | 2-10 |
| Checkout | 128m | 500m | 256Mi | 256Mi | 2-5 |
| Orders | 256m | 1000m | 512Mi | 512Mi | 2-5 |

### Total Cluster Resources

**Minimum (Development)**:
- CPU: ~1.2 vCPU (requests)
- Memory: ~2.3 GB (requests)
- Nodes: 1x t3.medium (2 vCPU, 4GB)

**Production (HA)**:
- CPU: ~2.4 vCPU (requests, 2 replicas)
- Memory: ~4.6 GB (requests, 2 replicas)
- Nodes: 3x t3.medium (2 vCPU, 4GB each)
  - Allows rolling updates without downtime
  - Node failure tolerance

**Production (High Load)**:
- CPU: ~6 vCPU (requests, 5+ replicas)
- Memory: ~11 GB (requests, 5+ replicas)
- Nodes: 3x t3.large (2 vCPU, 8GB each) or auto-scaling

### Horizontal Pod Autoscaling (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ui-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ui
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 30
```

### Pod Disruption Budgets (PDB)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: ui-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: ui
```

### Resource Quotas (Per Namespace)

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: retail-store-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    persistentvolumeclaims: "10"
```

---

## EKS Deployment Checklist

### Prerequisites

- [ ] **EKS Cluster Provisioned**
  - Kubernetes version: 1.28+
  - VPC with private/public subnets
  - NAT Gateway for private subnet internet access
  - VPC CNI plugin configured

- [ ] **IAM OIDC Provider Enabled**
  ```bash
  eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster retail-store-cluster \
    --approve
  ```

- [ ] **kubectl Configured**
  ```bash
  aws eks update-kubeconfig \
    --region us-east-1 \
    --name retail-store-cluster
  ```

- [ ] **Helm 3 Installed**
  ```bash
  helm version
  ```

### Core Add-ons

- [ ] **AWS Load Balancer Controller**
  ```bash
  helm repo add eks https://aws.github.io/eks-charts
  helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=retail-store-cluster \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller
  ```

- [ ] **NGINX Ingress Controller** (alternative to ALB)
  ```bash
  helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
  helm install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    --create-namespace
  ```

- [ ] **Metrics Server** (for HPA)
  ```bash
  kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
  ```

- [ ] **EBS CSI Driver** (for persistent volumes)
  ```bash
  eksctl create addon \
    --name aws-ebs-csi-driver \
    --cluster retail-store-cluster \
    --service-account-role-arn arn:aws:iam::877009927033:role/AmazonEKS_EBS_CSI_DriverRole \
    --force
  ```

### Observability Stack

- [ ] **Prometheus & Grafana**
  ```bash
  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
  helm install prometheus prometheus-community/kube-prometheus-stack \
    --namespace monitoring \
    --create-namespace
  ```

- [ ] **AWS Distro for OpenTelemetry (ADOT)**
  ```bash
  eksctl create addon \
    --name adot \
    --cluster retail-store-cluster
  ```

- [ ] **CloudWatch Container Insights**
  ```bash
  aws eks update-cluster-config \
    --region us-east-1 \
    --name retail-store-cluster \
    --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
  ```

### Secrets & Configuration

- [ ] **ECR Pull Secret** (if not using IRSA)
  ```bash
  kubectl create secret docker-registry regcred \
    --docker-server=877009927033.dkr.ecr.ap-south-1.amazonaws.com \
    --docker-username=AWS \
    --docker-password=$(aws ecr get-login-password --region ap-south-1) \
    --namespace=default
  ```

- [ ] **External Secrets Operator** (optional)
  ```bash
  helm repo add external-secrets https://charts.external-secrets.io
  helm install external-secrets external-secrets/external-secrets \
    --namespace external-secrets-system \
    --create-namespace
  ```

### AWS Resources (Production Mode)

- [ ] **RDS MySQL** (for Catalog Service)
  - Instance: db.t3.micro or larger
  - Multi-AZ: Yes (production)
  - Storage: 20GB encrypted
  - Backup: 7 day retention

- [ ] **DynamoDB Table** (for Cart Service)
  - Table: retail-cart-items
  - Billing: On-demand
  - TTL: Enabled
  - PITR: Enabled

- [ ] **ElastiCache Redis** (for Checkout Service)
  - Node: cache.t3.micro or larger
  - Replication: 1 primary + 1 replica
  - Encryption: At-rest and in-transit

- [ ] **RDS PostgreSQL** (for Orders Service)
  - Instance: db.t3.micro or larger
  - Multi-AZ: Yes (production)
  - Storage: 20GB encrypted
  - Backup: 7 day retention

- [ ] **Amazon SQS** (for Orders Events)
  - Queue: retail-orders-events
  - DLQ: retail-orders-events-dlq
  - Visibility timeout: 30s

### IAM Roles for Service Accounts (IRSA)

- [ ] **Cart Service - DynamoDB Access**
  ```bash
  eksctl create iamserviceaccount \
    --name cart-sa \
    --namespace default \
    --cluster retail-store-cluster \
    --attach-policy-arn arn:aws:iam::877009927033:policy/RetailCartDynamoDBPolicy \
    --approve
  ```

- [ ] **Orders Service - SQS + RDS Access**
  ```bash
  eksctl create iamserviceaccount \
    --name orders-sa \
    --namespace default \
    --cluster retail-store-cluster \
    --attach-policy-arn arn:aws:iam::877009927033:policy/RetailOrdersSQSPolicy \
    --approve
  ```

- [ ] **UI Service - Bedrock Access** (if AI chat enabled)
  ```bash
  eksctl create iamserviceaccount \
    --name ui-sa \
    --namespace default \
    --cluster retail-store-cluster \
    --attach-policy-arn arn:aws:iam::877009927033:policy/RetailUIBedrockPolicy \
    --approve
  ```

### Networking

- [ ] **Security Groups**
  - EKS node security group allows all pod-to-pod traffic
  - RDS security group allows traffic from EKS nodes
  - ElastiCache security group allows traffic from EKS nodes

- [ ] **Network Policies** (optional, requires Calico/Cilium)
  ```bash
  kubectl apply -f k8s/network-policies/
  ```

- [ ] **Service Mesh** (optional, Istio)
  ```bash
  istioctl install --set profile=demo
  kubectl label namespace default istio-injection=enabled
  ```

### Application Deployment

- [ ] **Create Namespace**
  ```bash
  kubectl create namespace retail-store
  ```

- [ ] **Deploy Services** (Helm)
  ```bash
  # UI Service
  helm install ui ./src/ui/chart/ \
    --namespace retail-store \
    --set image.tag=b85ff1a

  # Catalog Service
  helm install catalog ./src/catalog/chart/ \
    --namespace retail-store \
    --set image.tag=b85ff1a \
    --set persistence.provider=mysql

  # Cart Service
  helm install cart ./src/cart/chart/ \
    --namespace retail-store \
    --set image.tag=b85ff1a \
    --set persistence.provider=dynamodb

  # Checkout Service
  helm install checkout ./src/checkout/chart/ \
    --namespace retail-store \
    --set image.tag=b85ff1a \
    --set persistence.provider=redis

  # Orders Service
  helm install orders ./src/orders/chart/ \
    --namespace retail-store \
    --set image.tag=b85ff1a \
    --set persistence.provider=postgres
  ```

- [ ] **Verify Deployments**
  ```bash
  kubectl get pods -n retail-store
  kubectl get svc -n retail-store
  kubectl get ingress -n retail-store
  ```

- [ ] **Configure Ingress**
  ```bash
  kubectl apply -f k8s/ingress.yaml
  ```

- [ ] **Test Application**
  ```bash
  # Get load balancer URL
  kubectl get ingress -n retail-store

  # Access application
  curl http://<alb-url>/
  ```

### Post-Deployment

- [ ] **Set up Monitoring Alerts**
  - High error rate
  - Pod crash loop
  - High latency
  - Resource exhaustion

- [ ] **Configure Log Aggregation**
  - CloudWatch Logs
  - Or EFK stack (Elasticsearch, Fluentd, Kibana)

- [ ] **Set up Backup Jobs**
  - RDS automated backups
  - DynamoDB point-in-time recovery
  - Velero for Kubernetes resources

- [ ] **Document Runbooks**
  - Deployment procedures
  - Rollback procedures
  - Incident response
  - Disaster recovery

---

## Deployment Strategy

### Phase 1: Development Environment (In-Memory Mode)

**Goal**: Quick setup for testing without AWS resource dependencies

**Steps**:
1. Deploy all services with `in-memory` persistence provider
2. Use port-forwarding to access UI
3. No external databases or queues required

**Helm Commands**:
```bash
# Deploy all services in development mode
helm install ui ./src/ui/chart/ \
  --set image.tag=b85ff1a \
  --set replicaCount=1

helm install catalog ./src/catalog/chart/ \
  --set image.tag=b85ff1a \
  --set replicaCount=1 \
  --set persistence.provider=in-memory

helm install cart ./src/cart/chart/ \
  --set image.tag=b85ff1a \
  --set replicaCount=1 \
  --set persistence.provider=in-memory

helm install checkout ./src/checkout/chart/ \
  --set image.tag=b85ff1a \
  --set replicaCount=1 \
  --set persistence.provider=in-memory

helm install orders ./src/orders/chart/ \
  --set image.tag=b85ff1a \
  --set replicaCount=1 \
  --set persistence.provider=in-memory \
  --set messaging.provider=in-memory

# Access application
kubectl port-forward svc/retail-store-ui 8080:80
# Open http://localhost:8080 in browser
```

**Pros**:
- Fast setup
- No AWS costs for databases
- Good for development/testing

**Cons**:
- Data lost on pod restart
- Not suitable for production
- Limited scalability

---

### Phase 2: Staging Environment (Hybrid Mode)

**Goal**: Test with production-like storage for critical services

**Steps**:
1. Provision AWS resources (RDS, DynamoDB)
2. Deploy services with mixed persistence modes
3. Critical services (Catalog, Orders) use external storage
4. Non-critical services (Cart, Checkout) use in-memory

**AWS Resources to Provision**:
```bash
# RDS MySQL for Catalog
aws rds create-db-instance \
  --db-instance-identifier retail-catalog-staging \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --allocated-storage 20 \
  --master-username catalog_user \
  --master-user-password <password>

# RDS PostgreSQL for Orders
aws rds create-db-instance \
  --db-instance-identifier retail-orders-staging \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --allocated-storage 20 \
  --master-username orders_user \
  --master-user-password <password>
```

**Helm Commands**:
```bash
# UI Service (stateless)
helm install ui ./src/ui/chart/ \
  --set image.tag=b85ff1a \
  --set replicaCount=2

# Catalog Service with MySQL
helm install catalog ./src/catalog/chart/ \
  --set image.tag=b85ff1a \
  --set replicaCount=2 \
  --set persistence.provider=mysql \
  --set persistence.mysql.endpoint=catalog-db.staging.us-east-1.rds.amazonaws.com:3306 \
  --set persistence.mysql.database=catalogdb \
  --set persistence.mysql.username=catalog_user \
  --set persistence.mysql.password=<secret>

# Cart Service (in-memory for staging)
helm install cart ./src/cart/chart/ \
  --set image.tag=b85ff1a \
  --set replicaCount=2 \
  --set persistence.provider=in-memory

# Checkout Service (in-memory for staging)
helm install checkout ./src/checkout/chart/ \
  --set image.tag=b85ff1a \
  --set replicaCount=2 \
  --set persistence.provider=in-memory

# Orders Service with PostgreSQL
helm install orders ./src/orders/chart/ \
  --set image.tag=b85ff1a \
  --set replicaCount=2 \
  --set persistence.provider=postgres \
  --set persistence.postgres.endpoint=orders-db.staging.us-east-1.rds.amazonaws.com:5432 \
  --set persistence.postgres.database=ordersdb \
  --set persistence.postgres.username=orders_user \
  --set persistence.postgres.password=<secret> \
  --set messaging.provider=in-memory
```

---

### Phase 3: Production Environment (Full Stack)

**Goal**: Production deployment with all managed AWS services

**AWS Resources**:

1. **RDS MySQL** (Catalog) - Multi-AZ
2. **DynamoDB** (Cart) - On-demand with PITR
3. **ElastiCache Redis** (Checkout) - Multi-AZ replication
4. **RDS PostgreSQL** (Orders) - Multi-AZ
5. **Amazon SQS** (Orders Events) - with DLQ

**Terraform Configuration** (see `terraform/` directory):
```bash
cd terraform/
terraform init
terraform plan
terraform apply
```

**Helm Values** (Production):

Create `values-production.yaml`:
```yaml
# Global settings
image:
  tag: b85ff1a
  pullPolicy: IfNotPresent

replicaCount: 3

resources:
  requests:
    cpu: 256m
    memory: 512Mi
  limits:
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

# Service-specific overrides
ui:
  replicaCount: 5
  chat:
    enabled: true
    provider: bedrock
    bedrock:
      region: us-east-1

catalog:
  persistence:
    provider: mysql
    mysql:
      endpoint: catalog-prod.abc123.us-east-1.rds.amazonaws.com:3306
      database: catalogdb

cart:
  persistence:
    provider: dynamodb
    dynamodb:
      tableName: retail-cart-items-prod

checkout:
  persistence:
    provider: redis
    redis:
      url: redis://checkout-prod.abc123.0001.use1.cache.amazonaws.com:6379
      readerUrl: redis://checkout-prod-ro.abc123.0001.use1.cache.amazonaws.com:6379

orders:
  persistence:
    provider: postgres
    postgres:
      endpoint: orders-prod.abc123.us-east-1.rds.amazonaws.com:5432
      database: ordersdb
  messaging:
    provider: sqs
    sqs:
      topic: arn:aws:sqs:us-east-1:877009927033:retail-orders-events-prod
```

**Deploy with Production Values**:
```bash
helm install retail-store ./helm/retail-store-umbrella/ \
  --namespace production \
  --create-namespace \
  --values values-production.yaml
```

**Verify Production Deployment**:
```bash
# Check pods
kubectl get pods -n production

# Check services
kubectl get svc -n production

# Check ingress
kubectl get ingress -n production

# Check HPA
kubectl get hpa -n production

# Check logs
kubectl logs -f deployment/ui -n production

# Check metrics
kubectl top pods -n production
```

---

### Rolling Updates

**Update Image Tag**:
```bash
# Update single service
helm upgrade ui ./src/ui/chart/ \
  --namespace production \
  --reuse-values \
  --set image.tag=c9d2e4f

# Update all services
helm upgrade retail-store ./helm/retail-store-umbrella/ \
  --namespace production \
  --values values-production.yaml \
  --set image.tag=c9d2e4f
```

**Rolling Update Strategy** (Deployment):
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

**Monitor Rollout**:
```bash
kubectl rollout status deployment/ui -n production
kubectl rollout history deployment/ui -n production
```

---

### Blue/Green Deployment

Use service label selectors to switch traffic:

**Blue Deployment** (current):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ui
spec:
  selector:
    app: ui
    version: blue  # Current version
```

**Green Deployment** (new):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ui-green
spec:
  selector:
    matchLabels:
      app: ui
      version: green
```

**Switch Traffic**:
```bash
# Update service selector to green
kubectl patch service ui -p '{"spec":{"selector":{"version":"green"}}}'

# Verify
kubectl get endpoints ui

# Rollback if needed
kubectl patch service ui -p '{"spec":{"selector":{"version":"blue"}}}'
```

---

### Canary Deployment (with Flagger)

Install Flagger:
```bash
helm repo add flagger https://flagger.app
helm install flagger flagger/flagger \
  --namespace istio-system \
  --set meshProvider=istio
```

Create Canary resource:
```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: ui
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ui
  service:
    port: 80
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99
        interval: 1m
```

---

## Configuration Reference

### Environment Variable Naming Convention

All services follow the pattern:
```
RETAIL_<SERVICE>_<CATEGORY>_<SUBCATEGORY>_<PARAMETER>
```

Examples:
```bash
RETAIL_CATALOG_PERSISTENCE_PROVIDER=mysql
RETAIL_CART_PERSISTENCE_DYNAMODB_TABLE_NAME=Items
RETAIL_ORDERS_MESSAGING_SQS_TOPIC=arn:aws:sqs:...
```

### Complete Configuration Matrix

| Service | Category | Parameter | Values | Default | Required |
|---------|----------|-----------|--------|---------|----------|
| **UI** |
| | Endpoints | CATALOG | URL | http://retail-store-catalog:80 | Yes |
| | Endpoints | CARTS | URL | http://retail-store-cart:80 | Yes |
| | Endpoints | ORDERS | URL | http://retail-store-orders:80 | Yes |
| | Endpoints | CHECKOUT | URL | http://retail-store-checkout:80 | Yes |
| | Theme | THEME | string | default | No |
| | Chat | ENABLED | bool | false | No |
| | Chat | PROVIDER | mock/bedrock/openai | mock | No |
| | Chat | BEDROCK_REGION | string | us-east-1 | Conditional |
| | Chat | OPENAI_API_KEY | string | - | Conditional |
| **Catalog** |
| | Persistence | PROVIDER | in-memory/mysql | in-memory | Yes |
| | Persistence | ENDPOINT | hostname:port | - | Conditional |
| | Persistence | DB_NAME | string | catalogdb | Conditional |
| | Persistence | USER | string | catalog_user | Conditional |
| | Persistence | PASSWORD | string | - | Conditional |
| | Persistence | CONNECT_TIMEOUT | duration | 5s | No |
| **Cart** |
| | Persistence | PROVIDER | in-memory/dynamodb | in-memory | Yes |
| | Persistence | DYNAMODB_ENDPOINT | URL | AWS default | No |
| | Persistence | DYNAMODB_TABLE_NAME | string | Items | Conditional |
| | Persistence | DYNAMODB_CREATE_TABLE | bool | false | No |
| **Checkout** |
| | Persistence | PROVIDER | in-memory/redis | in-memory | Yes |
| | Persistence | REDIS_URL | URL | - | Conditional |
| | Persistence | REDIS_READER_URL | URL | - | No |
| | Endpoints | ORDERS | URL | http://retail-store-orders:80 | Yes |
| | Shipping | NAME_PREFIX | string | Super Duper Shipping | No |
| **Orders** |
| | Persistence | PROVIDER | in-memory/postgres | in-memory | Yes |
| | Persistence | POSTGRES_ENDPOINT | hostname:port | - | Conditional |
| | Persistence | POSTGRES_DBNAME | string | ordersdb | Conditional |
| | Persistence | POSTGRES_USERNAME | string | orders_user | Conditional |
| | Persistence | POSTGRES_PASSWORD | string | - | Conditional |
| | Messaging | PROVIDER | in-memory/rabbitmq/sqs | in-memory | Yes |
| | Messaging | RABBITMQ_ADDRESSES | URL | - | Conditional |
| | Messaging | RABBITMQ_USERNAME | string | guest | Conditional |
| | Messaging | RABBITMQ_PASSWORD | string | - | Conditional |
| | Messaging | SQS_TOPIC | ARN | - | Conditional |

---

## Troubleshooting Guide

### Common Issues

#### 1. Pods Not Starting (ImagePullBackOff)

**Symptom**:
```bash
$ kubectl get pods
NAME                      READY   STATUS             RESTARTS   AGE
ui-5d4c8f9b8-abc12       0/1     ImagePullBackOff   0          2m
```

**Diagnosis**:
```bash
kubectl describe pod ui-5d4c8f9b8-abc12
# Look for: "Failed to pull image" or "unauthorized"
```

**Solutions**:
```bash
# Option 1: Fix ECR authentication
kubectl delete secret regcred
kubectl create secret docker-registry regcred \
  --docker-server=877009927033.dkr.ecr.ap-south-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region ap-south-1)

# Option 2: Use IRSA (recommended)
# Ensure service account has ECR pull permissions

# Option 3: Verify image exists
aws ecr describe-images \
  --repository-name retail-store-ui \
  --image-ids imageTag=b85ff1a \
  --region ap-south-1
```

---

#### 2. Service Unavailable / 503 Errors

**Symptom**:
```bash
$ curl http://ui.example.com
HTTP/1.1 503 Service Temporarily Unavailable
```

**Diagnosis**:
```bash
# Check pod status
kubectl get pods -l app=ui

# Check readiness probes
kubectl describe pod <pod-name> | grep -A 10 "Readiness"

# Check service endpoints
kubectl get endpoints ui
```

**Solutions**:
```bash
# If no endpoints:
# 1. Check pod labels match service selector
kubectl get pods --show-labels
kubectl get svc ui -o yaml | grep selector

# 2. Check readiness probe is passing
kubectl logs <pod-name>
kubectl exec <pod-name> -- curl localhost:8080/actuator/health/readiness

# 3. Check for startup delay
# Increase initialDelaySeconds in readinessProbe
```

---

#### 3. Database Connection Failures

**Symptom**:
```bash
# Logs show:
WARN  c.e.catalog.CatalogService - Failed to connect to database: Connection refused
```

**Diagnosis**:
```bash
# Check database credentials
kubectl get secret catalog-db-secret -o yaml

# Verify network connectivity
kubectl run -it --rm debug --image=mysql:8.0 --restart=Never -- \
  mysql -h catalog-db.abc123.us-east-1.rds.amazonaws.com -u catalog_user -p

# Check security groups
aws ec2 describe-security-groups --group-ids sg-xxxxx
```

**Solutions**:
```bash
# 1. Update security group to allow EKS node traffic
aws ec2 authorize-security-group-ingress \
  --group-id <rds-sg-id> \
  --protocol tcp \
  --port 3306 \
  --source-group <eks-node-sg-id>

# 2. Verify RDS endpoint is correct
aws rds describe-db-instances \
  --db-instance-identifier retail-catalog-db \
  --query 'DBInstances[0].Endpoint'

# 3. Check credentials
# Re-create secret with correct values
```

---

#### 4. High Memory Usage / OOMKilled

**Symptom**:
```bash
$ kubectl get pods
NAME                      READY   STATUS      RESTARTS   AGE
orders-7b8d5c9f-xyz89    0/1     OOMKilled   5          10m
```

**Diagnosis**:
```bash
# Check memory usage
kubectl top pod orders-7b8d5c9f-xyz89

# Check memory limits
kubectl describe pod orders-7b8d5c9f-xyz89 | grep -A 5 "Limits"

# Check logs before OOM
kubectl logs --previous orders-7b8d5c9f-xyz89
```

**Solutions**:
```bash
# Increase memory limits
helm upgrade orders ./src/orders/chart/ \
  --reuse-values \
  --set resources.limits.memory=1Gi

# Tune JVM settings (Java services)
helm upgrade orders ./src/orders/chart/ \
  --reuse-values \
  --set env.JAVA_OPTS="-Xmx768m -Xms256m"
```

---

#### 5. Slow Response Times

**Symptom**:
- API responses taking >5 seconds
- Timeouts in logs

**Diagnosis**:
```bash
# Check pod CPU/memory
kubectl top pods

# Check HPA status
kubectl get hpa

# Check database queries (if using RDS)
# - RDS Performance Insights
# - CloudWatch metrics

# Check application metrics
kubectl port-forward svc/orders 8080:80
curl localhost:8080/actuator/metrics/http.server.requests
```

**Solutions**:
```bash
# 1. Scale up replicas
kubectl scale deployment orders --replicas=5

# 2. Increase resource limits
# 3. Add database indexes
# 4. Enable caching
# 5. Optimize queries
```

---

#### 6. Cannot Access Application via Load Balancer

**Symptom**:
```bash
$ curl http://abc123.us-east-1.elb.amazonaws.com
curl: (6) Could not resolve host
```

**Diagnosis**:
```bash
# Check ingress
kubectl get ingress
kubectl describe ingress retail-store-ingress

# Check ALB creation
aws elbv2 describe-load-balancers

# Check target groups
aws elbv2 describe-target-health \
  --target-group-arn <tg-arn>
```

**Solutions**:
```bash
# 1. Verify AWS Load Balancer Controller is running
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

# 2. Check ingress annotations
kubectl get ingress retail-store-ingress -o yaml

# 3. Verify subnet tags for ALB
# Subnets must be tagged:
# kubernetes.io/role/elb = 1 (public subnets)
```

---

#### 7. Service Discovery Failures

**Symptom**:
```bash
# UI logs show:
ERROR c.e.ui.UIService - Failed to connect to http://retail-store-catalog:80: Name or service not known
```

**Diagnosis**:
```bash
# Test DNS resolution from pod
kubectl exec -it ui-xxx -- nslookup retail-store-catalog

# Check service exists
kubectl get svc retail-store-catalog

# Check endpoints
kubectl get endpoints retail-store-catalog
```

**Solutions**:
```bash
# 1. Verify CoreDNS is running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 2. Use fully qualified domain name
# Update UI config to use:
# http://retail-store-catalog.default.svc.cluster.local:80

# 3. Check network policies
kubectl get networkpolicies
```

---

### Debugging Commands Reference

```bash
# Pod debugging
kubectl get pods -A
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl exec -it <pod-name> -- /bin/sh

# Service debugging
kubectl get svc
kubectl describe svc <service-name>
kubectl get endpoints <service-name>

# Ingress debugging
kubectl get ingress
kubectl describe ingress <ingress-name>

# Resource usage
kubectl top nodes
kubectl top pods

# Events
kubectl get events --sort-by='.lastTimestamp'

# Network testing
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- /bin/sh
kubectl run -it --rm debug --image=busybox --restart=Never -- /bin/sh

# Database connectivity
kubectl run -it --rm mysql-client --image=mysql:8.0 --restart=Never -- \
  mysql -h <db-host> -u <user> -p

kubectl run -it --rm postgres-client --image=postgres:15 --restart=Never -- \
  psql -h <db-host> -U <user> -d <database>

# Redis connectivity
kubectl run -it --rm redis-client --image=redis:7 --restart=Never -- \
  redis-cli -h <redis-host>

# Port forwarding (for local testing)
kubectl port-forward svc/ui 8080:80
kubectl port-forward pod/ui-xxx 8080:8080

# Get all resources in namespace
kubectl get all -n <namespace>

# Helm debugging
helm list
helm status <release-name>
helm get values <release-name>
helm get manifest <release-name>
```

---

## Additional Resources

### Project Structure
```
retail-store-sample-app/
├── src/                           # Source code
│   ├── ui/                        # UI Service
│   │   ├── src/                   # Java source
│   │   ├── Dockerfile             # Container definition
│   │   └── chart/                 # Helm chart
│   ├── catalog/                   # Catalog Service
│   ├── cart/                      # Cart Service
│   ├── checkout/                  # Checkout Service
│   └── orders/                    # Orders Service
├── terraform/                     # Infrastructure as Code
│   ├── addons.tf                  # EKS add-ons
│   ├── vpc.tf                     # VPC configuration
│   ├── eks.tf                     # EKS cluster
│   └── rds.tf                     # RDS databases
├── k8s/                           # Kubernetes manifests
│   ├── namespaces/
│   ├── network-policies/
│   └── ingress/
├── helm/                          # Helm umbrella chart
│   └── retail-store-umbrella/
└── docs/                          # Documentation
```

### Helm Charts Location
- UI: `src/ui/chart/`
- Catalog: `src/catalog/chart/`
- Cart: `src/cart/chart/`
- Checkout: `src/checkout/chart/`
- Orders: `src/orders/chart/`

### Key Files
- Dockerfiles: `src/*/Dockerfile`
- Application code: `src/*/src/`
- Deployment configs: `src/*/chart/templates/deployment.yaml`
- Service configs: `src/*/chart/templates/service.yaml`
- Values: `src/*/chart/values.yaml`

### Git Information
- **Current Branch**: gitops
- **Main Branch**: main (use for PRs)
- **ECR Registry**: 877009927033.dkr.ecr.ap-south-1.amazonaws.com
- **Current Image Tag**: b85ff1a

---

## Next Steps

Based on your deployment phase:

### For Development:
1. Deploy in-memory mode
2. Test all services locally
3. Set up port-forwarding for UI

### For Staging:
1. Provision RDS MySQL (Catalog) and PostgreSQL (Orders)
2. Deploy with hybrid persistence
3. Run integration tests
4. Set up monitoring

### For Production:
1. Provision all AWS managed services
2. Set up IRSA for all services
3. Configure HPA and PDB
4. Deploy with production Helm values
5. Configure monitoring and alerting
6. Set up backup jobs
7. Document runbooks

---

**Document maintained by**: DevOps Team
**Questions?**: Please file an issue or contact the platform team.
