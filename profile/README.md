# Teresa Solution Platform Architecture

This document provides an overview of the Teresa Solution Platform architecture and how its components work together to create a scalable, secure multi-tenant SaaS infrastructure.

## 🌟 System Overview

Teresa Solution is a comprehensive multi-tenant SaaS platform with three core components:

1. **API Gateway** - Entry point for all client requests
2. **Tenant Management Service** - Handles tenant provisioning and management
3. **Connection Pool Manager** - Manages database connections for multiple tenants

## 🏗️ Architecture Diagram

```
┌─────────────┐       HTTPS       ┌───────────────────────────────────┐         gRPC        ┌───────────────────────────┐
│             │                   │           API Gateway             │                     │  Tenant Management Service│
│   Clients   │◄─────────────────►│  ┌─────────┐ ┌────────┐ ┌──────┐  │◄───────────────────►│  - Tenant Provisioning    │
│ (Web/Mobile)│      (8080)       │  │   TLS   │ │  Auth  │ │ Rate │  │       (50051)       │  - Tenant Configuration   │
└─────────────┘                   │  │Terminator│ │Middleware│ │Limit │  │                     └───────────┬───────────────┘
                                 │  └─────────┘ └────────┘ └──────┘  │                              │
                                 │         │          │         │      │                              │ gRPC
                                 │         ▼          ▼         ▼      │                              ▼
                                 │       ┌─────────────────────────┐   │      gRPC          ┌───────────────────────────┐
                                 │       │   Multi-tenant Router   │───┼───────────────────►│ Connection Pool Manager   │
                                 │       └─────────────────────────┘   │      (50052)       │ - Database Connection Pools│
                                 │                   │                 │                     └───────────┬───────────────┘
                                 │                   ▼                 │                              │
                                 │       ┌─────────────────────────┐   │                              ▼
                                 │       │     Metrics Collector   │───┼──────► Prometheus   ┌───────────────────────────┐
                                 │       └─────────────────────────┘   │                     │      PostgreSQL DB        │
                                 └───────────────────────────────────┘                       │ (Tenant-specific Schemas) │
                                                                                           └───────────────────────────┘
```

## 🔄 Component Interactions

### Request Flow

1. **Client Request**:
   - Client sends HTTP/S request to the API Gateway
   - Request contains tenant identification in headers (X-Tenant-Subdomain)

2. **API Gateway Processing**:
   - Authenticates request using JWT tokens
   - Checks rate limits for tenant
   - Routes request to appropriate service (primarily Tenant Management Service)
   - Collects metrics for monitoring

3. **Tenant Management**:
   - Handles tenant-specific business logic
   - Manages tenant lifecycle (create, update, delete)
   - Provisions resources for new tenants

4. **Database Access**:
   - Tenant Management Service requests database connections from Connection Pool Manager
   - Connection Pool Manager maintains tenant-specific connection pools
   - Operations use tenant-specific database schemas for isolation

### Service Communication

All inter-service communication uses secure gRPC with TLS:

1. **API Gateway → Tenant Management**: 
   - Forwards client requests
   - Passes tenant context in metadata

2. **Tenant Management → Connection Pool Manager**:
   - Requests connections for tenant operations
   - Creates new pools during tenant provisioning
   - Retrieves statistics on pool usage

## 🔒 Security Model

Teresa Solution implements security at multiple levels:

1. **Network Security**:
   - TLS for all external and internal communications
   - Encrypted gRPC between services

2. **Authentication & Authorization**:
   - JWT validation at API Gateway
   - Role-based access controls
   - Tenant isolation throughout the system

3. **Data Security**:
   - Tenant data isolation in database schemas
   - Encryption of sensitive data at rest
   - Secure connection pool management

4. **Infrastructure Security**:
   - Prometheus monitoring and alerting
   - Health checks for all services
   - Graceful shutdown mechanisms

## 🧰 Development Setup

To set up the complete Teresa Solution platform locally:

1. **Clone repositories**:
   ```bash
   git clone https://github.com/teresa-solution/api-gateway.git
   git clone https://github.com/teresa-solution/tenant-management-service.git
   git clone https://github.com/teresa-solution/connection-pool-manager.git
   ```

2. **Generate TLS certificates**:
   ```bash
   mkdir -p certs
   openssl req -x509 -newkey rsa:4096 -keyout certs/key.pem -out certs/cert.pem -days 365 -nodes
   # Copy certificates to each service directory
   cp certs/* api-gateway/certs/
   cp certs/* tenant-management-service/certs/
   cp certs/* connection-pool-manager/certs/
   ```

3. **Set up PostgreSQL**:
   ```bash
   # Create main tenant registry database
   createdb -U postgres tenant_registry
   
   # Run schema creation script
   psql -U postgres -d tenant_registry -f tenant-management-service/scripts/schema.sql
   ```

4. **Set up Redis** (for Tenant Management Service):
   ```bash
   # Ensure Redis is running locally
   redis-cli ping
   ```

5. **Build and run services** (in order):
   ```bash
   # Build and run Connection Pool Manager
   cd connection-pool-manager
   go build -o conn-pool-manager ./cmd/server
   ./conn-pool-manager --port=50052 &
   
   # Build and run Tenant Management Service
   cd ../tenant-management-service
   go build -o tenant-management-service
   ./tenant-management-service --port=50051 --pool-mgr-addr=localhost:50052 &
   
   # Build and run API Gateway
   cd ../api-gateway
   go build -o api-gateway
   ./api-gateway --http-port=8080 --tenant-svc-addr=localhost:50051 --pool-mgr-addr=localhost:50052 &
   ```

## 📊 Monitoring Setup

1. **Configure Prometheus**:
   Create a `prometheus.yml` file:
   ```yaml
   global:
     scrape_interval:     15s
     evaluation_interval: 15s
   
   scrape_configs:
     - job_name: 'api-gateway'
       static_configs:
         - targets: ['localhost:8080']
     
     - job_name: 'tenant-management'
       static_configs:
         - targets: ['localhost:8081']
     
     - job_name: 'connection-pool-manager'
       static_configs:
         - targets: ['localhost:8082']
   ```

2. **Run Prometheus**:
   ```bash
   prometheus --config.file=prometheus.yml
   ```

3. **Access Prometheus dashboard**:
   Open http://localhost:9090 in your browser

## 🔍 Common Operations

### Creating a New Tenant

1. Client sends HTTP POST request to API Gateway:
   ```
   POST /v1/tenants
   Host: api.example.com
   Content-Type: application/json
   
   {
     "name": "Example Corp",
     "subdomain": "example",
     "contact_email": "admin@example.com"
   }
   ```

2. API Gateway forwards request to Tenant Management Service

3. Tenant Management Service:
   - Validates request
   - Creates tenant record
   - Requests new connection pool from Connection Pool Manager
   - Provisions tenant-specific database schema
   - Returns success response

4. New tenant is now accessible via:
   ```
   GET /v1/resources
   Host: api.example.com
   X-Tenant-Subdomain: example
   Authorization: Bearer <token>
   ```

### Monitoring Tenant Activity

1. **Prometheus Queries**:
   - Request rate by tenant: `sum(rate(http_requests_total{tenant!=""}[5m])) by (tenant)`
   - Error rate by tenant: `sum(rate(http_requests_total{status=~"5.."}[5m])) by (tenant)`
   - Database pool utilization: `pool_connections_active / pool_connections_total`

## 📚 Further Documentation

For more detailed information on each component, refer to the individual repository documentation:

- [API Gateway Documentation](https://github.com/teresa-solution/api-gateway)
- [Tenant Management Service Documentation](https://github.com/teresa-solution/tenant-management-service)
- [Connection Pool Manager Documentation](https://github.com/teresa-solution/connection-pool-manager)
