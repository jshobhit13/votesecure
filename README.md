# VoteSecure

**Concurrent-safe online voting platform built with Spring Boot, MongoDB, and JWT.**

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Java 17 |
| Framework | Spring Boot 3.2.5 |
| Security | Spring Security, JJWT 0.11.5 |
| Database | MongoDB 7.x |
| Build | Maven |
| Testing | JUnit 5, Mockito |

---

## Architecture

```
Client
  └── JwtAuthFilter (validates Bearer token on every request)
        └── Controller Layer
              └── Service Layer (business logic + @Transactional)
                    └── Repository Layer (Spring Data MongoDB)
                          └── MongoDB
                                └── Unique compound index (userId, electionId)
```

**Duplicate vote prevention — three-layer strategy:**

1. Application-level check via `existsByUserIdAndElectionId()` before any write
2. `@Transactional` wrapping the existence check and save atomically
3. MongoDB unique compound index on `(userId, electionId)` as the final database-enforced backstop — catches any concurrent race that bypasses layers 1 and 2

Validated against 100+ simultaneous submissions with zero duplicate entries recorded.

---

## Project Structure

```
src/
├── main/java/com/shobhit/voting_system/
│   ├── config/
│   │   ├── SecurityConfig.java           # Filter chain, CORS, role-based routing
│   │   └── GlobalExceptionHandler.java   # Unified error response format
│   ├── controller/
│   │   ├── AuthController.java           # /auth/register, /auth/login
│   │   ├── VoteController.java           # /api/vote
│   │   ├── ElectionController.java       # /api/elections, /results
│   │   └── AdminController.java          # /admin/** (ROLE_ADMIN only)
│   ├── dto/                              # Request/response contracts
│   ├── entity/                           # MongoDB @Document models
│   ├── repository/                       # MongoRepository interfaces
│   ├── security/
│   │   ├── JwtUtil.java                  # Token generation and validation
│   │   ├── JwtAuthFilter.java            # OncePerRequestFilter implementation
│   │   └── UserDetailsServiceImpl.java
│   └── service/
│       ├── AuthService.java              # Registration and login
│       ├── VoteService.java              # Concurrent-safe vote logic
│       └── ElectionService.java          # Election and result management
└── test/java/com/shobhit/voting_system/
    └── VoteServiceTest.java              # 7 unit tests, Mockito-based
```

---

## Prerequisites

- Java 17+
- Maven 3.8+
- MongoDB running on `localhost:27017`

---

## Running Locally

**Start MongoDB**
```bash
# Windows
net start MongoDB

# macOS / Linux
mongod
```

**Run the application**
```bash
mvn spring-boot:run
```

Server starts at `http://localhost:8080`. MongoDB collections are created automatically on first run.

---

## Configuration

`src/main/resources/application.properties`

```properties
spring.data.mongodb.uri=mongodb://localhost:27017/votesecure
spring.data.mongodb.database=votesecure

app.jwt.secret=<minimum-256-bit-secret>
app.jwt.expiration-ms=86400000

server.port=8080
```

---

## API Reference

### Authentication

| Method | Endpoint | Access | Description |
|---|---|---|---|
| POST | `/auth/register` | Public | Register and receive JWT |
| POST | `/auth/login` | Public | Authenticate and receive JWT |

### Elections

| Method | Endpoint | Access | Description |
|---|---|---|---|
| GET | `/api/elections` | Authenticated | All elections |
| GET | `/api/elections/active` | Authenticated | Active elections only |
| GET | `/api/elections/{id}/candidates` | Authenticated | Candidates for an election |
| GET | `/api/elections/{id}/results` | Authenticated | Vote counts per candidate |

### Voting

| Method | Endpoint | Access | Description |
|---|---|---|---|
| POST | `/api/vote` | Authenticated | Cast a vote |
| GET | `/api/vote/status?electionId=` | Authenticated | Check voting status |

### Admin

| Method | Endpoint | Access | Description |
|---|---|---|---|
| POST | `/admin/elections` | ROLE_ADMIN | Create election |
| PATCH | `/admin/elections/{id}/status` | ROLE_ADMIN | Update election status |
| POST | `/admin/candidates` | ROLE_ADMIN | Add candidate to election |

All protected endpoints require:
```
Authorization: Bearer <token>
```

---

## Running Tests

```bash
mvn test
```

| Test | Coverage |
|---|---|
| `castVote_success` | Valid vote persisted correctly |
| `castVote_electionNotActive_upcoming` | UPCOMING elections reject votes |
| `castVote_electionNotActive_closed` | CLOSED elections reject votes |
| `castVote_candidateNotInElection` | Cross-election candidate injection blocked |
| `castVote_duplicateDetectedInApp` | Application-layer duplicate rejected |
| `castVote_concurrentDuplicateRejectedByDb` | MongoDB index blocks race-condition duplicates |
| `castVote_electionNotFound` | Non-existent election ID handled |

---

## Data Model

**votes** — integrity enforced at the database level:
```
userId        String
electionId    String
candidateId   String
castAt        LocalDateTime

Index: { userId: 1, electionId: 1 }  unique: true
```

**users**
```
email         String  (unique)
password      String  (BCrypt)
role          ROLE_USER | ROLE_ADMIN
```

**elections**
```
title         String
status        UPCOMING | ACTIVE | CLOSED
startTime     LocalDateTime
endTime       LocalDateTime
```

---

## Security

- Passwords hashed with BCrypt
- Stateless authentication via signed JWT (HS256)
- `JwtAuthFilter` validates every request before it reaches any controller
- Role-based access control enforced at the route level via Spring Security
- No stack traces exposed in error responses

---

