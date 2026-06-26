# StackOverflow Lite API

A Clean Architecture-based Q&A platform inspired by StackOverflow where users can ask questions, post answers, vote on content, and build reputation through community engagement.

---

# Features

## Authentication & Authorization

* User Registration
* JWT Authentication
* Secure Login System
* Protected Endpoints

### Rules

* Users must register before accessing protected APIs
* Registered users can log in using email and password
* Successful login returns a JWT access token
* JWT token must be included in authenticated requests

### Authorization Header

```http
Authorization: Bearer <jwt-token>
```

### Unauthenticated Users Cannot

* Create questions
* Post answers
* Vote on content
* Accept answers
* Access protected user data

### Authenticated Users Can

* Create and manage their own content
* Vote on questions and answers
* View their profile and reputation

### Additional Security Rules

* Users can only modify or delete content they created themselves
* Invalid or expired JWT tokens are rejected
* Passwords are securely hashed before storing in the database
* Sensitive endpoints are protected using authorization middleware
* User identity is extracted from JWT claims
* Authentication state is stateless and token-based

---

## Questions

Authenticated users can:

* Create questions
* Update their own questions
* Delete their own questions
* Browse all questions
* View question details
* Filter questions by tag
* Track question views using Redis
* Create tags
* Assign tags to questions

### Rules

* Question must contain:

  * Title
  * Description
  * TagName

* Only the author can edit/delete their question

---

## Answers

Authenticated users can:

* View available questions to answer
* Post answers
* Edit their own answers
* Delete their own answers
* Retrieve answers for a specific question
* View accepted answers

Question authors can:

* Accept an answer
* Change accepted answer

### Rules

* Answer content cannot be empty
* Only the answer author can edit/delete
* Deleted answers cannot remain accepted
* Only question owner can accept answers
* Only one accepted answer per question
* Users cannot accept their own answers

---

## Voting System

Users can:

* Upvote questions
* Downvote questions
* Upvote answers
* Downvote answers
* Change their votes

### Rules

* One vote per user per content
* Users cannot vote on their own content

---

# UserProfile Reputation System

* User Profile Retrieval
* Reputation Tracking
* Automatic reputation updates based on community actions

| Action            | Reputation |
| ----------------- | ---------- |
| Question Upvote   | +5         |
| Question Downvote | -1         |
| Answer Upvote     | +10        |
| Answer Downvote   | -2         |
| Accepted Answer   | +15        |

## Reputation Consistency

* Vote changes reverse previous reputation effects
* Removing accepted answer removes +15 bonus
* Reputation never goes below zero

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
└── Dockerfile
```

---

# Technologies Used

* .NET 10
* ASP.NET Core Web API
* Entity Framework Core
* PostgreSQL
* Redis
* Docker
* JWT Authentication
* Clean Architecture
* CQRS Pattern
* MediatR

---

# Clean Architecture Layers

## Domain Layer

Contains:

* Entities
* Core business rules
* Domain contracts

No external dependencies.

---

## Application Layer

Contains:

* Commands
* Queries
* DTOs
* Validators
* Handlers
* CQRS logic

No infrastructure dependencies.

---

## Infrastructure Layer

Contains:

* EF Core
* Repository implementations
* Redis services
* JWT services
* Database configuration

Handles all external concerns.

---

## API Layer

Contains:

* Controllers
* Middleware
* API configuration

Controllers remain thin and delegate logic inward.

---

# Redis Integration

## Redis Cache & View Tracking

This project uses Redis for both caching and real-time analytics.

---

## Redis Connection

Redis runs inside Docker using the official Redis image.

### Docker Compose Configuration

```yaml
redis:
  image: redis:7
  container_name: stackoverflow_redis
  ports:
    - "6379:6379"
```

---

## Application Redis Connection

### appsettings.json

```json
{
  "Redis": {
    "ConnectionString": "redis:6379"
  }
}
```

---

## Redis Service Registration

```csharp
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration["Redis:ConnectionString"];
});
```

---

## Redis Insight

Redis Insight is used to monitor and inspect Redis data visually.

### Redis Insight Connection

| Setting | Value     |
| ------- | --------- |
| Host    | localhost |
| Port    | 6379      |

---

## Redis Cache: Question Details

Question details are cached in Redis for faster retrieval.

### Redis Key

```text
question:a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44
```

### Cached JSON Data

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

### Cache Expiration

```text
10 minutes
```

### Example Cache Logic

```csharp
await _distributedCache.SetStringAsync(
    $"question:{question.QuestionId}",
    json,
    new DistributedCacheEntryOptions
    {
        AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
    });
```

### Purpose of Cache

* Faster question retrieval
* Reduced PostgreSQL queries
* Improved API performance
* Reduced database load

---

# Redis View Tracking

Redis also tracks question views in real time.

## Redis View Key

```text
question:views:a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44
```

## Example Value

```text
4
```

Meaning:

* The question has been viewed 4 times

### View Counter Behavior

* No expiration time
* Persistent counter
* Incremented every time question details are opened

### Example View Tracking Logic

```csharp
await _redisDatabase.StringIncrementAsync(
    $"question:views:{questionId}");
```

### Why Redis For Views?

Redis is ideal for counters because:

* Extremely fast
* In-memory operations
* Atomic increments
* Handles high traffic efficiently

---

# API Endpoints

## Authentication

### Register API

#### Endpoint

```http
POST /api/auth/register
```

#### Request Body

```json
{
  "email": "tusharbasak041@gmail.com",
  "password": "Tushar@041"
}
```

#### Example cURL Request

```bash
curl -X POST "https://localhost:5001/api/auth/register" \
-H "Content-Type: application/json" \
-d '{
  "email": "tusharbasak041@gmail.com",
  "password": "Tushar@041"
}'
```

#### Success Response

```json
{
  "message": "User registered successfully"
}
```

#### Error Responses

##### Duplicate Email

```json
{
  "message": "User with this email already exists"
}
```

##### Invalid Password

```json
{
  "message": "Password does not meet security requirements"
}
```

---

## Password Policy (ASP.NET Identity)

The password must follow these rules:

* Minimum 8 characters
* At least one uppercase letter
* At least one lowercase letter
* At least one number
* At least one special character

### ASP.NET Identity Configuration

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

---

## Login API

### Endpoint

```http
POST /api/auth/login
```

### Request Body

```json
{
  "email": "tusharbasak041@gmail.com",
  "password": "Tushar@041"
}
```

### Success Response

```json
{
  "token": "your-jwt-token"
}
```

---

# Questions Endpoints

```http
# Questions API Documentation

## Base URL

```http
http://localhost:8080/api/questions
```

---

# Endpoints

## Create Question

### `POST /api/questions`

Create a new question.

### Authorization

Bearer Token Required

### Request Body

```json
{
  "title": "string12",
  "description": "string123566",
  "tagName": "string"
}
```

### cURL Request

```bash
curl -X 'POST' \
  'http://localhost:8080/api/questions' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
  "title": "string12",
  "description": "string123566",
  "tagName": "string"
}'
```

### Success Response

```json
"3393e3d9-e43a-4230-a542-92d3ba9b9694"
```

---

# Get All Questions

## `GET /api/questions`

Retrieve all questions.

### Authorization

Bearer Token Required

### cURL Request

```bash
curl -X 'GET' \
  'http://localhost:8080/api/questions' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

### Success Response

```json
[
  {
    "questionId": "a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44",
    "title": "string",
    "description": "string",
    "tagName": "string",
    "acceptedAnswer": false,
    "createdAt": "2026-06-25T09:43:52.410205Z"
  }
]
```

---

# Update Question

## `PUT /api/questions`

Update an existing question.

### Authorization

Bearer Token Required

### Request Body

```json
{
  "questionId": "3393e3d9-e43a-4230-a542-92d3ba9b9694",
  "title": "string12",
  "description": "string12",
  "tagName": "string"
}
```

### cURL Request

```bash
curl -X 'PUT' \
  'http://localhost:8080/api/questions' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
  "questionId": "3393e3d9-e43a-4230-a542-92d3ba9b9694",
  "title": "string12",
  "description": "string12",
  "tagName": "string"
}'
```

### Success Response

```json
{
  "message": "Question updated successfully."
}
```

---

# Get Question By ID

## `GET /api/questions/{id}`

Retrieve a single question by ID.

### Path Parameter

| Parameter | Type | Description |
| --------- | ---- | ----------- |
| id        | UUID | Question ID |

### cURL Request

```bash
curl -X 'GET' \
  'http://localhost:8080/api/questions/3393e3d9-e43a-4230-a542-92d3ba9b9694' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

### Success Response

```json
{
  "questionId": "3393e3d9-e43a-4230-a542-92d3ba9b9694",
  "title": "string12",
  "description": "string12",
  "tagName": "string",
  "acceptedAnswer": false,
  "createdAt": "2026-06-26T12:30:24.323177Z"
}
```

---

# Get Questions By Tag

## `GET /api/questions/tag/{tagName}`

Retrieve questions filtered by tag name.

### Path Parameter

| Parameter | Type   | Description |
| --------- | ------ | ----------- |
| tagName   | string | Tag name    |

### cURL Request

```bash
curl -X 'GET' \
  'http://localhost:8080/api/questions/tag/string' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

### Success Response

```json
[
  {
    "questionId": "a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44",
    "title": "string",
    "description": "string",
    "tagName": "string",
    "acceptedAnswer": false,
    "createdAt": "2026-06-25T09:43:52.410205Z"
  }
]
```

---

# Get My Questions

## `GET /api/questions/my`

Retrieve all questions created by the authenticated user.

### Authorization

Bearer Token Required

### cURL Request

```bash
curl -X 'GET' \
  'http://localhost:8080/api/questions/my' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

### Success Response

```json
[
  {
    "questionId": "a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44",
    "title": "string",
    "description": "string",
    "tagName": "string",
    "acceptedAnswer": false,
    "createdAt": "2026-06-25T09:43:52.410205Z"
  }
]
```

---

# Delete Question

## `DELETE /api/questions/{id}`

Delete a question by ID.

### Path Parameter

| Parameter | Type | Description |
| --------- | ---- | ----------- |
| id        | UUID | Question ID |

### Authorization

Bearer Token Required

### cURL Request

```bash
curl -X 'DELETE' \
  'http://localhost:8080/api/questions/3393e3d9-e43a-4230-a542-92d3ba9b9694' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

### Success Response

```json
{
  "message": "Question deleted successfully."
}
```

---

# Accept Answer

## `PUT /api/questions/accept-answer`

Accept an answer for a question. Only question owner can accept and answer and change accepted answer as per wish

### Authorization

Bearer Token Required

### Request Body

```json
{
  "answerId": "a44e0f03-2d17-4da9-bf93-ad5d92badf7e"
}
```

### cURL Request

```bash
curl -X 'PUT' \
  'http://localhost:8080/api/questions/accept-answer' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
  "answerId": "a44e0f03-2d17-4da9-bf93-ad5d92badf7e"
}'
```

### Success Response

```json
{
  "message": "Answer accepted successfully."
}
```

---

# Get Accepted Answer & Other Answers

## `GET /api/questions/question/{questionId}`

Retrieve the accepted answer and other answers for a question.

### Path Parameter

| Parameter  | Type | Description |
| ---------- | ---- | ----------- |
| questionId | UUID | Question ID |

### cURL Request

```bash
curl -X 'GET' \
  'http://localhost:8080/api/questions/question/a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

### Success Response

```json
{
  "message": "Accepted answer found",
  "acceptedAnswer": {
    "answerId": "a44e0f03-2d17-4da9-bf93-ad5d92badf7e",
    "questionId": "a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44",
    "content": "ansfrom041",
    "isAccepted": true,
    "createdAt": "2026-06-26T12:41:54.925825Z"
  },
  "otherAnswers": []
}
```

---

# Get Question Views

## `GET /api/questions/{questionId}/views`

Retrieve total view count for a question.

### Path Parameter

| Parameter  | Type | Description |
| ---------- | ---- | ----------- |
| questionId | UUID | Question ID |

### cURL Request

```bash
curl -X 'GET' \
  'http://localhost:8080/api/questions/a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44/views' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

### Success Response

```json
2
```

---

# Authentication Rules

* All protected endpoints require a valid JWT Bearer Token.
* Users must register and log in before accessing protected APIs.
* Tokens must be included in the `Authorization` header.

Example:

```http
Authorization: Bearer YOUR_TOKEN
```

---

# Response Status Codes

| Status Code | Description                    |
| ----------- | ------------------------------ |
| 200         | Request completed successfully |
| 400         | Bad request                    |
| 401         | Unauthorized                   |
| 404         | Resource not found             |
| 500         | Internal server error          |

---

```

---

# Answers Endpoints

```http
# Answers API Documentation

## Base URL

```http
http://localhost:8080/api/answers
```

---

# Endpoints

## Create Answer

### `POST /api/answers`

Create a new answer for a question.

### Authorization

Bearer Token Required

### Request Body

```json
{
  "questionId": "a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44",
  "content": "ansfrom041"
}
```

### cURL Request

```bash
curl -X 'POST' \
  'http://localhost:8080/api/answers' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
  "questionId": "a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44",
  "content": "ansfrom041"
}'
```

### Success Response

```json
"a44e0f03-2d17-4da9-bf93-ad5d92badf7e"
```

---

# Get Questions Available For Answer

## `GET /api/answers/available-for-answer`

Retrieve all questions that are available for answering.

### Authorization

Bearer Token Required

### cURL Request

```bash
curl -X 'GET' \
  'http://localhost:8080/api/answers/available-for-answer' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

### Success Response

```json
[
  {
    "questionId": "a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44",
    "title": "string",
    "description": "string",
    "tagName": "string",
    "acceptedAnswer": false,
    "createdAt": "2026-06-25T09:43:52.410205Z"
  }
]
```

---

# Get My Answers

## `GET /api/answers/my`

Retrieve all answers created by the authenticated user.

### Authorization

Bearer Token Required

### cURL Request

```bash
curl -X 'GET' \
  'http://localhost:8080/api/answers/my' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

### Success Response

```json
[
  {
    "answerId": "a44e0f03-2d17-4da9-bf93-ad5d92badf7e",
    "questionId": "a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44",
    "content": "ansfrom041",
    "isAccepted": true,
    "createdAt": "2026-06-26T12:41:54.925825Z"
  }
]
```

---

# Update Answer

## `PUT /api/answers`

Update an existing answer.

### Authorization

Bearer Token Required

### Request Body

```json
{
  "answerId": "a44e0f03-2d17-4da9-bf93-ad5d92badf7e",
  "content": "AnsFrom041"
}
```

### cURL Request

```bash
curl -X 'PUT' \
  'http://localhost:8080/api/answers' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
  "answerId": "a44e0f03-2d17-4da9-bf93-ad5d92badf7e",
  "content": "AnsFrom041"
}'
```

### Success Response

```json
{
  "message": "Answer updated successfully."
}
```

---

# Get Answers By Question ID

## `GET /api/answers/question/{questionId}`

Retrieve accepted and other answers for a specific question.

### Path Parameter

| Parameter  | Type | Description |
| ---------- | ---- | ----------- |
| questionId | UUID | Question ID |

### Authorization

Bearer Token Required

### cURL Request

```bash
curl -X 'GET' \
  'http://localhost:8080/api/answers/question/a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

### Success Response

```json
{
  "message": "Accepted answer found",
  "acceptedAnswer": {
    "answerId": "a44e0f03-2d17-4da9-bf93-ad5d92badf7e",
    "questionId": "a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44",
    "content": "AnsFrom041",
    "isAccepted": true,
    "createdAt": "2026-06-26T12:41:54.925825Z"
  },
  "otherAnswers": []
}
```

---

# Accept Answer

## `PUT /api/answers/accept-answer`

Accept an answer for a question and only Question owner can accept this answer

### Authorization

Bearer Token Required

### Request Body

```json
{
  "answerId": "a44e0f03-2d17-4da9-bf93-ad5d92badf7e"
}
```

### cURL Request

```bash
curl -X 'PUT' \
  'http://localhost:8080/api/answers/accept-answer' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
  "answerId": "a44e0f03-2d17-4da9-bf93-ad5d92badf7e"
}'
```

### Success Response

```json
{
  "message": "Answer accepted successfully."
}
```

---

# Already Accepted Error

### Error Response

```json
[
  {
    "code": "Answer.AlreadyAccepted",
    "description": "This answer is already accepted.",
    "type": 3,
    "numericType": 3,
    "metadata": null
  }
]
```

### Status Code

```http
400 Bad Request
```

---

# Delete Answer

## `DELETE /api/answers/{id}`

Delete an answer by ID by authenticated user

### Path Parameter

| Parameter | Type | Description |
| --------- | ---- | ----------- |
| id        | UUID | Answer ID   |

### Authorization

Bearer Token Required

### cURL Request

```bash
curl -X 'DELETE' \
  'http://localhost:8080/api/answers/{id}' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

### Success Response

```json
{
  "message": "Answer deleted successfully."
}
```

---

# Authentication Rules

* All protected endpoints require a valid JWT Bearer Token.
* Users must register and log in before accessing protected APIs.
* Tokens must be included in the `Authorization` header.

Example:

```http
Authorization: Bearer YOUR_TOKEN
```

---

# Response Status Codes

| Status Code | Description                    |
| ----------- | ------------------------------ |
| 200         | Request completed successfully |
| 400         | Bad request                    |
| 401         | Unauthorized                   |
| 404         | Resource not found             |
| 500         | Internal server error          |

---

# Business Rules

* Users cannot answer their own questions.
* A question can only have one accepted answer.
* Accepted answers cannot be accepted again.
* Only the owner of an answer can update or delete it.
* Only the question owner can accept an answer.

---

```

---

# Votes Endpoints
# Votes API Documentation

## Base URL

```http
http://localhost:8080/api/votes
```

---

# Endpoints

# Get My Questions

## `GET /api/votes/myquestion`

Retrieve all questions created by the authenticated user.

### Authorization

Bearer Token Required

### cURL Request

```bash
curl -X 'GET' \
  'http://localhost:8080/api/votes/myquestion' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

### Success Response

```json
[
  {
    "questionId": "a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44",
    "title": "string",
    "description": "string",
    "tagName": "string",
    "acceptedAnswer": false,
    "createdAt": "2026-06-25T09:43:52.410205Z"
  }
]
```

---

# Get Answers By Question ID

## `GET /api/votes/question/{questionId}`

Retrieve accepted and other answers for a specific question.

### Path Parameter

| Parameter  | Type | Description |
| ---------- | ---- | ----------- |
| questionId | UUID | Question ID |

### Authorization

Bearer Token Required

### cURL Request

```bash
curl -X 'GET' \
  'http://localhost:8080/api/votes/question/a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

### Success Response

```json
{
  "message": "Accepted answer found",
  "acceptedAnswer": {
    "answerId": "a44e0f03-2d17-4da9-bf93-ad5d92badf7e",
    "questionId": "a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44",
    "content": "AnsFrom041",
    "isAccepted": true,
    "createdAt": "2026-06-26T12:41:54.925825Z"
  },
  "otherAnswers": []
}
```

---

# Upvote Answer

## `POST /api/votes/answer/upvote`

Upvote an answer.

### Authorization

Bearer Token Required

### Request Body

```json
{
  "answerId": "a44e0f03-2d17-4da9-bf93-ad5d92badf7e"
}
```

### cURL Request

```bash
curl -X 'POST' \
  'http://localhost:8080/api/votes/answer/upvote' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
  "answerId": "a44e0f03-2d17-4da9-bf93-ad5d92badf7e"
}'
```

### Success Response

```json
{
  "message": "Answer upvoted successfully."
}
```

---

# Downvote Answer

## `POST /api/votes/answer/downvote`

Downvote an answer.

### Authorization

Bearer Token Required

### Request Body

```json
{
  "answerId": "a44e0f03-2d17-4da9-bf93-ad5d92badf7e"
}
```

### cURL Request

```bash
curl -X 'POST' \
  'http://localhost:8080/api/votes/answer/downvote' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
  "answerId": "a44e0f03-2d17-4da9-bf93-ad5d92badf7e"
}'
```

### Success Response

```json
{
  "message": "Answer downvoted successfully."
}
```

---

# Get Questions Available For Vote

## `GET /api/votes/available-answer-for-vote`

Retrieve all questions available for voting.

### Authorization

Bearer Token Required

### cURL Request

```bash
curl -X 'GET' \
  'http://localhost:8080/api/votes/available-answer-for-vote' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

### Success Response

```json
[
  {
    "questionId": "a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44",
    "title": "string",
    "description": "string",
    "tagName": "string",
    "acceptedAnswer": false,
    "createdAt": "2026-06-25T09:43:52.410205Z"
  }
]
```

---

# Downvote Question

## `POST /api/votes/question/downvote`

Downvote a question.

### Authorization

Bearer Token Required

### Request Body

```json
{
  "questionId": "a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44"
}
```

### cURL Request

```bash
curl -X 'POST' \
  'http://localhost:8080/api/votes/question/downvote' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
  "questionId": "a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44"
}'
```

### Success Response

```json
{
  "message": "Question downvoted successfully."
}
```

---

# Upvote Question

## `POST /api/votes/question/upvote`

Upvote a question.

### Authorization

Bearer Token Required

### Request Body

```json
{
  "questionId": "a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44"
}
```

### cURL Request

```bash
curl -X 'POST' \
  'http://localhost:8080/api/votes/question/upvote' \
  -H 'accept: */*' \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
  "questionId": "a0ad97f3-cdec-4a7d-b1b9-7ecea0fdea44"
}'
```

### Success Response

```json
{
  "message": "Question upvoted successfully."
}
```

---

# Authentication Rules

* All protected endpoints require a valid JWT Bearer Token.
* Users must register and log in before accessing protected APIs.
* Tokens must be included in the `Authorization` header.

Example:

```http
Authorization: Bearer YOUR_TOKEN
```

---

# Response Status Codes

| Status Code | Description                    |
| ----------- | ------------------------------ |
| 200         | Request completed successfully |
| 400         | Bad request                    |
| 401         | Unauthorized                   |
| 404         | Resource not found             |
| 500         | Internal server error          |

---

# Business Rules

* Users cannot vote on their own questions.
* Users cannot vote on their own answers.
* A user can upvote or downvote only once per question.
* A user can upvote or downvote only once per answer.
* Users can change their vote type from upvote to downvote and vice versa.
* Voting requires authentication.

---

```

---

# User Profile

## Get User Profile API

### Endpoint

```http
GET /api/userprofile
```

### Success Response

```json
{
  "userId": "d377fb06-43a3-4295-9c8d-962622ac1390",
  "reputationScore": 5
}
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

* API
* PostgreSQL
* Redis

---

# Docker Setup

## Build & Run Containers

```bash
docker compose up --build
```

## Stop Containers

```bash
docker compose down
```

---

# Docker Services

| Service    | Port |
| ---------- | ---- |
| API        | 8080 |
| PostgreSQL | 5432 |
| Redis      | 6379 |

---

# Docker Compose

```yaml
services:

  api:
    build:
      context: .
      dockerfile: Dockerfile

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
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base
WORKDIR /app
EXPOSE 8080

FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

COPY . .

RUN dotnet restore

RUN dotnet publish Stackoverflow.Host/Stackoverflow.Host.csproj \
    -c Release \
    -o /app/publish

FROM base AS final

WORKDIR /app

COPY --from=build /app/publish .

ENTRYPOINT ["dotnet", "Stackoverflow.Host.dll"]
```

---

# Run Locally

## Prerequisites

* .NET 10 SDK
* PostgreSQL
* Redis
* Docker Desktop

## Clone Repository

```bash
git clone <your-repository-url>
```

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

## Repository Pattern

Repositories abstract persistence logic from business logic.

## Unit of Work

Ensures transactional consistency across operations.

## Redis Caching

Improves performance for frequently accessed question data.

---

# Future Improvements

* Refresh Tokens
* Role-Based Authorization
* Pagination
* Search Engine
* Notification System
* Rate Limiting
* Full Redis Distributed Cache
* SignalR Real-Time Notifications
* Elasticsearch Integration
* Unit & Integration Testing
* CI/CD Pipeline

---

# Author

Built as a backend engineering project to demonstrate:

* Clean Architecture
* Scalable API Design
* Redis Integration
* Dockerized Infrastructure
* CQRS + MediatR
* PostgreSQL + EF Core
* JWT Authentication
* Reputation & Voting System
