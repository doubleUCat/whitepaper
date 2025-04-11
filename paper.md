# Distributed API Ecosystem for Browser Plugin & Company Review Platform

## Executive Summary

This whitepaper outlines the architecture and implementation of a distributed API ecosystem designed to support a browser plugin and company review platform. The system utilizes a central master API with distributed self-hosted replicas to enhance availability, reduce hosting costs, and improve user experience. Key features include automatic instance discovery, data synchronization, integrity verification, and a user suggestion system for collaborative content development.

## 1. Introduction

### 1.1 Background and Motivation

Browser plugins often rely on centralized APIs, creating single points of failure and scaling challenges. This project implements a distributed architecture where the primary API can be supplemented by user-hosted instances, creating a resilient ecosystem that distributes load while maintaining data consistency and integrity.

### 1.2 Project Goals

- Reduce central infrastructure costs through distributed hosting
- Improve global availability and latency through geographically diverse instances
- Maintain data consistency across all instances
- Ensure data integrity and prevent tampering
- Enable community contributions while maintaining quality control
- Provide transparent performance metrics for all API instances

## 2. System Architecture

### 2.1 Overview

The system consists of three primary components:
1. **Central Master API**: Authoritative data source with admin capabilities
2. **Self-hosted API Instances**: Read-focused replicas that synchronize with the central API
3. **Registry Service**: Monitors and tracks all instances, providing metrics and discovery capabilities

### 2.2 Component Interaction Diagram

```
┌───────────────────┐      Register      ┌───────────────────┐
│  Self-hosted API  │──────────────────► │  Central Registry │
│    Instance 1     │◄─────────────────┐ │      Service      │
└───────────────────┘      Health       └─────────┬──────────┘
                           Checks                 │
┌───────────────────┐                            │
│  Self-hosted API  │                            │ Discovery
│    Instance 2     │                            │ API
└───────────────────┘                            │
                                                 ▼
┌───────────────────┐                   ┌───────────────────┐
│  Self-hosted API  │                   │  Browser Plugin   │
│    Instance 3     │◄──────────────────┤  & Website        │
└───────────────────┘   API Requests    │  (auto-selects    │
                                        │   best instance)  │
                                        └───────────────────┘
```

### 2.3 Data Flow

1. **Initial Setup**: The master API serves as the authoritative data source
2. **Instance Registration**: Self-hosted instances register with the central registry
3. **Health Monitoring**: The registry continuously monitors instance health and performance
4. **Data Synchronization**: Self-hosted instances periodically pull updates from the master API
5. **Client Discovery**: Clients (plugin or website) query the registry for optimal API instances
6. **Request Routing**: Clients connect to the best available instance based on performance metrics
7. **Fallback Mechanism**: Clients automatically fall back to the central API if needed

## 3. Technical Implementation

### 3.1 Database Schema

The database is logically divided into two schemas:

#### 3.1.1 Shared Schema (Synchronized)
Contains all business data that is synchronized across instances:
```sql
CREATE SCHEMA shared;

CREATE TABLE shared.companies (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    website VARCHAR(255),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE shared.products (
    id SERIAL PRIMARY KEY,
    company_id INTEGER REFERENCES shared.companies(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE shared.scores (
    id SERIAL PRIMARY KEY,
    company_id INTEGER REFERENCES shared.companies(id),
    product_id INTEGER REFERENCES shared.products(id) NULL,
    category VARCHAR(50) NOT NULL,
    score FLOAT NOT NULL,
    evidence TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE shared.reviews (
    id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES shared.products(id),
    content TEXT NOT NULL,
    author VARCHAR(255),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

#### 3.1.2 Instance Schema (Not Synchronized)
Contains instance-specific data, including authentication:
```sql
CREATE SCHEMA instance;

CREATE TABLE instance.users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE instance.config (
    key VARCHAR(255) PRIMARY KEY,
    value TEXT NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE instance.sync_status (
    id SERIAL PRIMARY KEY,
    last_sync_time TIMESTAMP WITH TIME ZONE,
    tables_synced INTEGER,
    status VARCHAR(50),
    error_message TEXT
);
```

### 3.2 Registry Service Schema

```sql
CREATE TABLE api_instances (
    id VARCHAR(36) PRIMARY KEY,
    url VARCHAR(255) NOT NULL UNIQUE,
    location VARCHAR(255),
    owner VARCHAR(255),
    version VARCHAR(50) NOT NULL,
    last_checked TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    uptime_percentage FLOAT DEFAULT 100.0,
    average_latency_ms INTEGER DEFAULT 0,
    status VARCHAR(20) DEFAULT 'online',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE health_checks (
    id SERIAL PRIMARY KEY,
    instance_id VARCHAR(36) REFERENCES api_instances(id),
    checked_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    latency_ms INTEGER NOT NULL,
    success BOOLEAN NOT NULL,
    error_message TEXT
);
```

### 3.3 Data Synchronization

Self-hosted instances synchronize data using the following process:

1. **Fetch Latest Timestamp**: Query the master API for the latest update timestamp
2. **Incremental Sync**: Download only records modified since the last synchronization
3. **Transaction-based Updates**: Apply changes in transactions to maintain consistency
4. **Tracking**: Record synchronization status for monitoring and troubleshooting

Example synchronization code:
```go
func (s *SyncService) SyncTable(ctx context.Context, tableName string) error {
    // Get the timestamp of the last sync for this table
    lastSync, err := s.getLastSyncTime(tableName)
    if err != nil {
        return err
    }
    
    // Request data changed since last sync
    url := fmt.Sprintf("%s/api/sync/%s?since=%s", 
        s.masterAPIURL, tableName, lastSync.Format(time.RFC3339))
    
    resp, err := s.client.Get(url)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    // Parse the response
    var changes struct {
        Records []map[string]interface{} `json:"records"`
        Deleted []int                   `json:"deleted"`
    }
    
    if err := json.NewDecoder(resp.Body).Decode(&changes); err != nil {
        return err
    }
    
    // Begin transaction
    tx, err := s.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    // Apply updates
    if err := s.applyChanges(ctx, tx, tableName, changes); err != nil {
        return err
    }
    
    // Update last sync time
    if err := s.updateSyncTime(ctx, tx, tableName); err != nil {
        return err
    }
    
    return tx.Commit()
}
```

### 3.4 Data Integrity Verification

To ensure data integrity across instances, the system implements digital signatures:

1. **Signing Process**: The master API signs critical data using RSA-SHA256
2. **Verification**: Clients verify signatures using the public key
3. **Key Distribution**: The public key is available through a dedicated endpoint
4. **Timestamp Verification**: Prevents replay attacks by including timestamps

### 3.5 Instance Health Monitoring

The registry continuously monitors instance health:

1. **Periodic Checks**: Probes each instance's health endpoint every minute
2. **Latency Measurement**: Records response time for performance metrics
3. **Status Updates**: Updates instance status based on health check results
4. **Exponential Backoff**: Reduces check frequency for consistently failing instances

## 4. User Contribution System

### 4.1 Overview

The platform allows users to suggest company data, reviews, and scores through a moderated submission system:

1. **Submission**: Users submit suggestions through web forms
2. **Queuing**: Suggestions enter a review queue
3. **Moderation**: Administrators review, edit, and approve/reject submissions
4. **Application**: Approved suggestions are applied to the database
5. **Notification**: Submitters may receive status updates (optional)

### 4.2 Suggestion Queue Schema

```sql
CREATE TABLE suggestion_queue (
    id SERIAL PRIMARY KEY,
    company_id INTEGER REFERENCES shared.companies(id),
    product_id INTEGER REFERENCES shared.products(id) NULL,
    suggestion_type VARCHAR(50) NOT NULL,
    content JSONB NOT NULL,
    submitter_email VARCHAR(255),
    submitter_name VARCHAR(255),
    submitter_ip VARCHAR(45),
    status VARCHAR(20) DEFAULT 'pending',
    mod_notes TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    reviewed_at TIMESTAMP WITH TIME ZONE NULL,
    reviewed_by VARCHAR(255) NULL
);
```

### 4.3 Moderation Workflow

1. **Initial Review**: Submissions are first scanned for spam/inappropriate content
2. **Content Verification**: Moderators verify accuracy of claims and supporting evidence
3. **Editing**: Moderators can edit submissions before approval
4. **Decision**: Moderators approve or reject with optional feedback
5. **Integration**: Approved content is integrated into the main database

## 5. Client Integration

### 5.1 Browser Plugin

The browser plugin intelligently connects to API instances:

1. **Instance Discovery**: Queries the registry for available instances
2. **Performance-based Selection**: Selects instance with lowest latency
3. **Local Caching**: Caches instance information to reduce discovery calls
4. **Automatic Failover**: Falls back to alternative instances if primary fails
5. **Manual Configuration**: Allows users to manually select preferred instances

### 5.2 Website Integration

The website similarly implements smart API selection:

1. **Server-side Selection**: For server-rendered pages, selection happens on the server
2. **Client-side Selection**: For client-rendered pages, JavaScript selects optimal instances
3. **Signature Verification**: Verifies data integrity client-side
4. **Refresh Mechanisms**: Periodically refreshes instance selection to adapt to changing conditions

## 6. Deployment and Operations

### 6.1 Containerization

Both master and self-hosted instances are packaged as Docker containers for easy deployment:

```Dockerfile
FROM golang:1.21-alpine
WORKDIR /app
COPY . .
RUN go build -o api
EXPOSE 8080
CMD ["./api"]
```

### 6.2 Environment Configuration

The system uses environment variables for configuration:

```
# Common variables
PORT=8080
DATABASE_URL=postgresql://user:password@localhost:5432/apidb
LOG_LEVEL=info

# Mode-specific variables
SERVER_MODE=master        # or "self-hosted"
SYNC_INTERVAL=3600        # seconds, for self-hosted only
MASTER_API_URL=https://api.example.com  # for self-hosted only
```

### 6.3 Monitoring and Alerting

The system includes comprehensive monitoring:

1. **Instance Health**: Dashboard showing all instance statuses
2. **Sync Status**: Monitoring of synchronization processes
3. **Performance Metrics**: Latency and throughput tracking
4. **Alerting**: Notifications for instance failures or sync issues

## 7. Security Considerations

### 7.1 Authentication and Authorization

1. **Instance Authentication**: Self-hosted instances authenticate to the master using API keys
2. **User Authentication**: Instance-specific user management with no credential sharing
3. **Role-based Access**: Different permission levels for administrators and users

### 7.2 Data Protection

1. **Isolation**: Authentication data is never synchronized between instances
2. **Encryption**: All communications use TLS encryption
3. **Signature Verification**: Critical data is digitally signed to prevent tampering

### 7.3 Rate Limiting and Abuse Prevention

1. **API Rate Limiting**: Prevents abuse of central synchronization endpoints
2. **Submission Limits**: Controls user contribution frequency
3. **IP-based Protections**: Guards against distributed attacks

## 8. Future Extensions

### 8.1 Feature Roadmap

1. **Contribution Reputation System**: Trust scoring for frequent contributors
2. **Geographic Routing**: Automatically route users to geographically close instances
3. **Advanced Caching**: Multi-level caching to further improve performance
4. **Conflict Resolution**: Enhanced mechanisms for handling data conflicts
5. **Analytics Integration**: Reporting on usage patterns across instances

### 8.2 Scaling Considerations

1. **Sharding**: Data sharding for extremely large datasets
2. **Federation Protocols**: Standard protocols for instance communication
3. **Plugin Marketplace**: Allow community-developed extensions

## 9. Conclusion

This distributed API ecosystem provides a robust foundation for browser plugin services. By distributing hosting responsibilities while maintaining data consistency and integrity, the system achieves cost efficiency, performance, and reliability goals. The collaborative content model enables community engagement while maintaining content quality through moderation.

The architecture's flexibility allows for future expansion and adapts to varying deployment scenarios, from individual self-hosting to enterprise-scale implementations.

---

## Appendix A: API Endpoints Reference

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/health` | GET | Health check endpoint |
| `/api/sync/:table` | GET | Synchronization endpoint for specific table |
| `/api/companies` | GET | List companies |
| `/api/companies/:id` | GET | Get company details |
| `/api/products` | GET | List products |
| `/api/products/:id` | GET | Get product details |
| `/api/suggestions` | POST | Submit a suggestion |
| `/api/admin/suggestions` | GET | List pending suggestions (admin only) |
| `/api/instances` | GET | List API instances |
| `/api/public-key` | GET | Fetch verification public key |

## Appendix B: Technology Stack

- **Backend**: Go
- **Database**: PostgreSQL
- **Container**: Docker
- **API Documentation**: OpenAPI/Swagger
- **Authentication**: JWT
- **Monitoring**: Prometheus/Grafana
