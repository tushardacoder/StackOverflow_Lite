# StackOverflow Lite API

A Clean Architecture-based Q&A platform inspired by StackOverflow where users can ask questions, post answers, vote on content, and build reputation through community engagement.

---

# Features

## Authentication & Authorization

- User Registration
- JWT Authentication
- Secure Login System
- Protected Endpoints

### Rules

- Users must register before accessing protected APIs
- Registered users can log in using email and password
- Successful login returns a JWT access token
- JWT token must be included in authenticated requests

### Authorization Header

```http
Authorization: Bearer <jwt-token>
```

- Unauthenticated users cannot:
  - Create questions
  - Post answers
  - Vote on content
  - Accept answers
  - Access protected user data

- Only authenticated users can:
  - Create and manage their own content
  - Vote on questions and answers
  - View their profile and reputation

- Users can only modify or delete content they created themselves
- Invalid or expired JWT tokens are rejected
- Passwords are securely hashed before storing in the database
- Sensitive endpoints are protected using authorization middleware
- User identity is extracted from JWT claims
- Authentication state is stateless and token-based
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
  - TagName
- Only the author can edit/delete their question

---

## Answers

Authenticated users can:

- View Abailable question to answer
- Post answers
- Edit their own answers
- Delete their own answers
- Retrieve answers for a specific question
- View accepted answers
Question authors can:

- Accept an answer
- Change accepted answer

### Rules

- Answer content cannot be empty
- Only the answer author can edit/delete
- Deleted answers cannot remain accepted
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

## UserProfile Reputation System
 - User Profile Retrieval
 -  Reputation Tracking

- 
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

```
# Register API

## Endpoint

```http
POST /api/auth/register
```

---

## Request Body

```json
{
  "email": "tusharbasak041@gmail.com",
  "password": "Tushar@041"
}
```

---

## Example cURL Request

```bash
curl -X POST "https://localhost:5001/api/auth/register" \
-H "Content-Type: application/json" \
-d '{
  "email": "tusharbasak041@gmail.com",
  "password": "Tushar@041"
}'
```

---

## Success Response

```json
{
  "message": "User registered successfully"
}
```

---

## Error Responses

### Duplicate Email

```json
{
  "message": "User with this email already exists"
}
```

### Invalid Password

```json
{
  "message": "Password does not meet security requirements"
}
```

---

# Password Policy (ASP.NET Identity)

The password must follow these rules:

* Minimum **8 characters**
* At least one uppercase letter (`A-Z`)
* At least one lowercase letter (`a-z`)
* At least one number (`0-9`)
* At least one special character (`@`, `#`, `!`, etc.)

---

## Example Valid Passwords

```text
Tushar@041
Admin#2026
SecurePass1!
```

---

## Example Invalid Passwords

```text
tushar041      // No uppercase & special character
TUSHAR@041     // No lowercase letter
TusharPassword // No number & special character
12345678       // Weak password
```

---

## ASP.NET Identity Configuration

```csharp
builder.Services.Configure<IdentityOptions>(options =>
{
    options.Password.RequireDigit = true;
    options.Password.RequireLowercase = true;
    options.Password.RequireUppercase = true;
    options.Password.RequireNonAlphanumeric = true;
    options.Password.RequiredLength = 8;
    options.Password.RequiredUniqueChars = 1;
});
```

# Login API

## Endpoint

```
POST /api/auth/login
```

---

## Request Body

```json id="5njlwm"
{
  "email": "tusharbasak041@gmail.com",
  "password": "Tushar@041"
}
```

---

## Example cURL Request

```bash id="flc3ie"
curl -X POST "https://localhost:5001/api/auth/login" \
-H "Content-Type: application/json" \
-d '{
  "email": "tusharbasak041@gmail.com",
  "password": "Tushar@041"
}'
```

---

## Success Response

```json id="pn9g5z"
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyaWQiOiJkMzc3ZmIwNi00M2EzLTQyOTUtOWM4ZC05NjI2MjJhYzEzOTAiLCJlbWFpbCI6InR1c2hhcmJhc2FrMDQxQGdtYWlsLmNvbSIsImV4cCI6MTc4MjQ3OTE5N30.DX7v0CvONPtK3dY5rt4JOO_TbXe_oVPPDmq_kqPCftg"
}
```

---

## Error Responses

### Invalid Email

```json id="5tz6p4"
{
  "message": "User not found"
}
```

### Invalid Password

```json id="p91wzl"
{
  "message": "Invalid password"
}
```

---

# JWT Authentication

After successful login, the API returns a JWT token.

Use this token in the `Authorization` header for protected endpoints.

## Authorization Header Example

```http id="0t6q7d"
Authorization: Bearer your-jwt-token
```

---

## Example Protected Request

```bash id="bq98u4"
curl -X GET "https://localhost:5001/api/questions" \
-H "Authorization: Bearer your-jwt-token"
```

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

```
# Get User Profile API

## Endpoint

```http id="6n2i8v"
GET /api/userprofile
```

---

## Authorization

This endpoint requires JWT authentication.

```http id="4b5qrm"
Authorization: Bearer your-jwt-token
```

---

## Example cURL Request

```bash id="m1t4yo"
curl -X 'GET' \
  'http://localhost:8080/api/userprofile' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyaWQiOiJkMzc3ZmIwNi00M2EzLTQyOTUtOWM4ZC05NjI2MjJhYzEzOTAiLCJlbWFpbCI6InR1c2hhcmJhc2FrMDQxQGdtYWlsLmNvbSIsImV4cCI6MTc4MjQ3OTE5N30.DX7v0CvONPtK3dY5rt4JOO_TbXe_oVPPDmq_kqPCftg'
```

---

## Request URL

```text id="tr9xg5"
http://localhost:8080/api/userprofile
```

---

## Success Response

### Status Code

```http id="7cy9gq"
200 OK
```

### Response Body

```json id="0h0p0h"
{
  "userId": "d377fb06-43a3-4295-9c8d-962622ac1390",
  "reputationScore": 5
}
```

---

## Response Headers

```http id="zwqqea"
content-type: application/json; charset=utf-8
date: Fri, 26 Jun 2026 12:10:14 GMT
server: Kestrel
transfer-encoding: chunked
```

---

## Responses

| Status Code | Description |
| ----------- | ----------- |
| 200         | OK          |

```




# Database Migration

## Add Migration

```bash
dotnet ef migrations add InitTeamModule --project Stackoverflow.Infrastructure --startup-project Stackoverflow.Host --context Stackoverflow.Infrastructure.Data.ApplicationDbContext --output-dir Data/Migrations
```



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
