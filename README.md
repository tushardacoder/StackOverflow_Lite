# StackOverflow Lite API

A Clean Architecture-based Q&A platform inspired by StackOverflow where users can ask questions, post answers, vote on content, and build reputation through community engagement.

---

# Features

## Authentication & Authorization

- User Registration
- JWT Authentication
- Secure Login System
- Protected Endpoints
- User Profile Retrieval
- Reputation Tracking

---

## Questions

Authenticated users can:

- Create questions
- Update their own questions
- Delete their own questions
- Browse all questions
- View question details
- Filter questions by tag
- Track question views using Redis
- Create tags
- Assign tags to questions
- Filter questions by tag

### Rules

- Question must contain:
  - Title
  - Description
- Only the author can edit/delete their question

---

## Answers

Authenticated users can:

- Post answers
- Edit their own answers
- Delete their own answers
- Retrieve answers for a specific question
- View accepted answers

### Rules

- Answer content cannot be empty
- Only the answer author can edit/delete
- Deleted answers cannot remain accepted

---

## Accepted Answers

Question authors can:

- Accept an answer
- Change accepted answer
- Remove accepted answer

### Rules

- Only question owner can accept answers
- Only one accepted answer per question
- Users cannot accept their own answers

---

## Voting System

Users can:

- Upvote questions
- Downvote questions
- Upvote answers
- Downvote answers
- Change their votes

### Rules

- One vote per user per content
- Users cannot vote on their own content

---

## Reputation System

Automatic reputation updates based on community actions.

| Action | Reputation |
|---|---|
| Question Upvote | +5 |
| Question Downvote | -1 |
| Answer Upvote | +10 |
| Answer Downvote | -2 |
| Accepted Answer | +15 |

### Reputation Consistency

- Vote changes reverse previous reputation effects
- Removing accepted answer removes +15 bonus
- Reputation never goes below zero

---





---

# Architecture

This project follows **Clean Architecture** principles with strict separation of concerns.

```text
Stackoverflow_lite/
│
├── Stackoverflow.Application
│   ├── Behaviours
│   ├── Contracts
│   ├── DTOS
│   ├── Features
│   └── DependencyInjection.cs
│
├── Stackoverflow.Domain
│   ├── Contracts
│   ├── Entities
│   └── DependencyInjection.cs
│
├── Stackoverflow.Infrastructure
│   ├── Data
│   ├── Extensions
│   ├── External
│   ├── Identity
│   ├── Repositories
│   ├── Security
│   ├── Services
│   ├── ApplicationUnitOfWork.cs
│   ├── Repository.cs
│   ├── UnitOfWork.cs
│   └── DependencyInjection.cs
│
├── Stackoverflow.Host
│   ├── Controllers
│   ├── Middleware
│   ├── Program.cs
│   ├── appsettings.json
│   └── DependencyInjection.cs
│
├── docker-compose.yml
└── Dockerfile.txt
```

---

# Technologies Used

- .NET 10
- ASP.NET Core Web API
- Entity Framework Core
- PostgreSQL
- Redis
- Docker
- JWT Authentication
- Clean Architecture
- CQRS Pattern
- MediatR

---

# Clean Architecture Layers

## Domain Layer

Contains:

- Entities
- Core business rules
- Domain contracts

No external dependencies.

---

## Application Layer

Contains:

- Commands
- Queries
- DTOs
- Validators
- Handlers
- CQRS logic

No infrastructure dependencies.

---

## Infrastructure Layer

Contains:

- EF Core
- Repository implementations
- Redis services
- JWT services
- Database configuration

Handles all external concerns.

---

## API Layer

Contains:

- Controllers
- Middleware
- API configuration

Controllers remain thin and delegate logic inward.

---

# Redis Integration
# Redis Cache & View Tracking

This project uses Redis for both caching and real-time analytics.

---

# Redis Connection

Redis runs inside Docker using the official Redis image.

## Docker Compose Configuration

```yaml
redis:

  image: redis:7

  container_name: stackoverflow_redis

  ports:
    - "6379:6379"
```

---

# Application Redis Connection

## appsettings.json

```json
{
  "Redis": {
    "ConnectionString": "redis:6379"
  }
}
```

---

# Redis Service Registration

```csharp
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration["Redis:ConnectionString"];
});
```

---

# Redis Insight

Redis Insight is used to monitor and inspect Redis data visually.

## Redis Insight Connection

| Setting | Value |
|---|---|
| Host | localhost |
| Port | 6379 |

---

# Redis Cache: Question Details

Question details are cached in Redis for faster retrieval.

## Redis Key

```text
question:a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44
```

---

## Cached JSON Data

```json
{
  "QuestionId": "a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44",
  "Title": "string",
  "Description": "string",
  "TagName": "string",
  "AcceptedAnswer": false,
  "CreatedAt": "2026-06-25T09:43:52.410205Z"
}
```

---

## Cache Expiration

Question cache expires after:

```text
10 minutes
```

---

## Example Cache Logic

```csharp
await _distributedCache.SetStringAsync(
    $"question:{question.QuestionId}",
    json,
    new DistributedCacheEntryOptions
    {
        AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
    });
```

---

## Purpose of Cache

- Faster question retrieval
- Reduced PostgreSQL queries
- Improved API performance
- Reduced database load

---

# Redis View Tracking

Redis also tracks question views in real time.

## Redis View Key

```text
question:views:a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44
```

---

## Example Value

```text
4
```

Meaning:

- The question has been viewed 4 times.

---

## View Counter Behavior

- No expiration time
- Persistent counter
- Incremented every time question details are opened

---

## Example View Tracking Logic

```csharp
await _redisDatabase.StringIncrementAsync(
    $"question:views:{questionId}");
```

---

# Why Redis For Views?

Redis is ideal for counters because:

- Extremely fast
- In-memory operations
- Atomic increments
- Handles high traffic efficiently

---

# Redis Keys Inside Redis Insight

## Question Cache Key

```text
question:a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44
```

### Type

```text
String
```

### TTL

```text
600 seconds
```

(10 minutes)

---

## Question View Counter Key

```text
question:views:a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44
```

### Type

```text
String
```

### TTL

```text
No Expiration
```

---

# Cache Flow

```text
Client Request
      ↓
Check Redis Cache
      ↓
Cache Hit → Return Cached Data
      ↓
Cache Miss
      ↓
Fetch From PostgreSQL
      ↓
Store In Redis (10 min)
      ↓
Return Response
```

---

# View Tracking Flow

```text
Question Opened
      ↓
Redis String Increment
      ↓
question:views:{questionId}
      ↓
Updated View Count
```

---

# API Endpoints

# Authentication

```http
POST /api/auth/register
POST /api/auth/login
```

---

# Questions

```http
POST   /api/questions
GET    /api/questions
PUT    /api/questions
GET    /api/questions/{id}
DELETE /api/questions/{id}

GET    /api/questions/tag/{tagName}
GET    /api/questions/my

GET    /api/questions/{questionId}/views

PUT    /api/questions/accept-answer
```

---

# Answers

```http
POST   /api/answers
PUT    /api/answers
DELETE /api/answers/{id}

GET    /api/answers/question/{questionId}
GET    /api/answers/my

PUT    /api/answers/accept-answer
```

---

# Votes

```http
POST /api/votes/question/upvote
POST /api/votes/question/downvote

POST /api/votes/answer/upvote
POST /api/votes/answer/downvote

GET  /api/votes/question/{questionId}
GET  /api/votes/myquestion

GET  /api/votes/available-answer-for-vote
```

---

# User Profile

```http
GET /api/userprofile
```

---

# Database Migration

## Add Migration

```bash
dotnet ef migrations add InitTeamModule --project Stackoverflow.Infrastructure --startup-project Stackoverflow.Host --context Stackoverflow.Infrastructure.Data.ApplicationDbContext --output-dir Data/Migrations
```

---

# Docker Support

The entire application runs using Docker Compose:

- API
- PostgreSQL
- Redis

---

# Docker Setup

## Build & Run Containers

```bash
docker compose up --build
```

---

## Stop Containers

```bash
docker compose down
```

---

# Docker Services

| Service | Port |
|---|---|
| API | 8080 |
| PostgreSQL | 5432 |
| Redis | 6379 |

---

# Docker Compose

```yaml
services:

  api:
    build:
      context: .
      dockerfile: Dockerfile.txt

    container_name: stackoverflow_api

    ports:
      - "8080:8080"

    depends_on:
      - postgres
      - redis

    environment:
      ASPNETCORE_URLS: http://+:8080

      ConnectionStrings__DefaultConnection: Host=postgres;Port=5432;Username=postgres;Password=softwaredev;Database=StackOverFlowDB

      Redis__ConnectionString: redis:6379

  postgres:
    image: postgres:16

    container_name: stackoverflow_postgres

    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: softwaredev
      POSTGRES_DB: StackOverFlowDB

    ports:
      - "5432:5432"

    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7

    container_name: stackoverflow_redis

    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

---

# Dockerfile

```dockerfile
# =========================
# Runtime image
# =========================

FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base

WORKDIR /app

EXPOSE 8080


# =========================
# Build image
# =========================

FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build

WORKDIR /src


# Copy source code
COPY . .


# Restore packages
RUN dotnet restore


# Publish application
RUN dotnet publish Stackoverflow.Host/Stackoverflow.Host.csproj \
    -c Release \
    -o /app/publish


# =========================
# Final image
# =========================

FROM base AS final

WORKDIR /app

COPY --from=build /app/publish .

ENTRYPOINT ["dotnet", "Stackoverflow.Host.dll"]
```

---

# Run Locally

## Prerequisites

- .NET 10 SDK
- PostgreSQL
- Redis
- Docker Desktop

---

## Clone Repository

```bash
git clone <your-repository-url>
```

---

## Run Application

```bash
docker compose up --build
```

API will be available at:

```text
http://localhost:8080
```

---

# Design Decisions

## CQRS Pattern

Commands and queries are separated for better maintainability and scalability.

---

## Repository Pattern

Repositories abstract persistence logic from business logic.

---

## Unit of Work

Ensures transactional consistency across operations.

---

## Redis Caching

Improves performance for frequently accessed question data.

---

# Future Improvements

- Refresh Tokens
- Role-Based Authorization
- Pagination
- Search Engine
- Notification System
- Rate Limiting
- Full Redis Distributed Cache
- SignalR Real-Time Notifications
- Elasticsearch Integration
- Unit & Integration Testing
- CI/CD Pipeline



---

# Author

Built as a backend engineering project to demonstrate:

- Clean Architecture
- Scalable API Design
- Redis Integration
- Dockerized Infrastructure
- CQRS + MediatR
- PostgreSQL + EF Core
- JWT Authentication
- Reputation & Voting Systems
