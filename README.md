# Code Meet — Backend

![Java](https://img.shields.io/badge/Java-21-orange?logo=openjdk)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.3.5-brightgreen?logo=springboot)
![Maven](https://img.shields.io/badge/Maven-3.x-red?logo=apachemaven)
![MySQL](https://img.shields.io/badge/MySQL-8.x-blue?logo=mysql)
![WebSocket](https://img.shields.io/badge/WebSocket-STOMP%2FSockJS-yellow)
![Agora](https://img.shields.io/badge/Agora-RTC%2FRTM-blueviolet)
![Cloudinary](https://img.shields.io/badge/Cloudinary-Image%20Hosting-3448C5)

A real-time collaboration platform backend powering video/audio meetings, room-based group chats, peer messaging, friendship management, and live notifications — built with Spring Boot 3 and Java 21.

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Features](#features)
3. [Architecture Overview](#architecture-overview)
4. [Project Structure](#project-structure)
5. [Setup and Installation](#setup-and-installation)
6. [Configuration](#configuration)
7. [Running the Application](#running-the-application)
8. [API Documentation](#api-documentation)
9. [WebSocket Flow](#websocket-flow)
10. [Authentication and Authorization](#authentication-and-authorization)
11. [Error Handling](#error-handling)
12. [External Integrations](#external-integrations)
13. [Future Improvements](#future-improvements)
14. [Contributing](#contributing)

---

## System Overview

**Code Meet** is a backend service for a collaborative video meeting and messaging platform. It exposes a RESTful HTTP API alongside a STOMP-over-WebSocket layer. The application manages users, real-time peer and group chats, video/audio meeting sessions (via Agora), room-based collaboration, friendship networks, and a live notification system.

The system is designed to serve an Angular frontend (default origin: `http://localhost:4200`) and relies on MySQL for persistence, Cloudinary for media storage, and Agora for real-time audio/video token generation.

---

## Features

Only capabilities confirmed in the codebase are listed below.

- **User Management** — Registration, login, profile updates, profile picture upload via Cloudinary, and user search by username or full name.
- **Meeting Management** — Instant and scheduled meetings, participant join-request/accept flow, meeting lifecycle (SCHEDULED → RUNNING → FINISHED), and automatic scheduled-meeting startup via a background job.
- **Room Management** — Create and manage persistent collaboration rooms, room picture uploads, and name-based room search.
- **Peer Messaging** — Bidirectional real-time chat between two users with full message history; chat is created automatically when a friendship is accepted.
- **Room Messaging** — Broadcast messages to all accepted members of a room with full message history per member.
- **Friendship System** — Send, accept, and cancel friend requests; bidirectional state tracking (PENDING → ACCEPTED).
- **Membership System** — Request, accept, and cancel room membership; admin role protection (admins cannot leave their own room).
- **Notification System** — Real-time WebSocket notifications for friendship, membership, meeting, and message events; persistent notification storage with delete support.
- **Agora Token Generation** — Server-side RTM and RTC token generation for Agora SDK integration.
- **Cloudinary Integration** — Profile and room picture upload with automatic overwrite.

---

## Architecture Overview

The application follows a classic three-layer Spring architecture:

```
HTTP / WebSocket Clients
         │
         ▼
┌─────────────────────┐
│    Controller Layer  │  REST controllers + WebSocket message handlers
│  (9 controllers)    │  Request validation, DTO mapping
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│    Service Layer    │  Business logic, transaction management,
│  (11 services)      │  post-commit WebSocket broadcasts
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  Repository Layer   │  Spring Data JPA repositories,
│  (9 repositories)   │  JPQL custom queries
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│    MySQL Database   │  Hibernate ORM (ddl-auto=update)
└─────────────────────┘

Side integrations:
  ├── Cloudinary   (image uploads)
  └── Agora SDK    (RTC/RTM token generation)
```

Key cross-cutting patterns:

- **`@Transactional`** on mutating service methods; `TransactionSynchronizationManager` is used to fire WebSocket broadcasts only *after* the transaction commits.
- **`synchronized`** on `MessageService` methods to prevent race conditions when creating bidirectional chat records.
- **`@EnableAsync` / `@EnableScheduling`** on the main application class to support background job execution for timed meeting startup.

---

## Project Structure

```
backend/
├── src/
│   └── main/
│       ├── java/com/codemeet/
│       │   ├── CodeMeetApplication.java          # Entry point
│       │   ├── configuration/
│       │   │   ├── CorsConfiguration.java         # CORS rules
│       │   │   ├── WebSocketConfiguration.java    # STOMP broker setup
│       │   │   └── CloudinaryConfig.java          # Cloudinary bean
│       │   ├── controller/                        # 9 REST/WS controllers
│       │   │   ├── AuthenticationController.java
│       │   │   ├── UserController.java
│       │   │   ├── MeetingController.java
│       │   │   ├── RoomController.java
│       │   │   ├── ChatController.java
│       │   │   ├── WebSocketController.java
│       │   │   ├── FriendshipController.java
│       │   │   ├── MembershipController.java
│       │   │   ├── NotificationController.java
│       │   │   └── AgoraTokenController.java
│       │   ├── service/                           # 11 service classes
│       │   │   ├── AuthenticationService.java
│       │   │   ├── UserService.java
│       │   │   ├── MeetingService.java
│       │   │   ├── RoomService.java
│       │   │   ├── ChatService.java
│       │   │   ├── MessageService.java
│       │   │   ├── FriendshipService.java
│       │   │   ├── MembershipService.java
│       │   │   ├── NotificationService.java
│       │   │   ├── CloudinaryService.java
│       │   │   └── JobSchedulerService.java
│       │   ├── entity/                            # 18 JPA entities
│       │   │   ├── User.java
│       │   │   ├── Meeting.java
│       │   │   ├── Participant.java
│       │   │   ├── Room.java
│       │   │   ├── Chat.java (abstract, JOINED inheritance)
│       │   │   ├── PeerChat.java
│       │   │   ├── RoomChat.java
│       │   │   ├── Message.java
│       │   │   ├── Friendship.java
│       │   │   ├── Membership.java
│       │   │   ├── Notification.java
│       │   │   └── enums/ (Gender, MeetingStatus, FriendshipStatus,
│       │   │              MembershipStatus, NotificationType)
│       │   ├── repository/                        # 9 JPA repositories
│       │   └── utils/
│       │       ├── dto/                           # Request and response DTOs
│       │       ├── exception/                     # Custom exceptions + GlobalExceptionHandler
│       │       ├── annotation/                    # @FutureTime custom validator
│       │       └── FileUploadUtil.java
│       └── resources/
│           └── application.properties
└── pom.xml
```

---

## Setup and Installation

### Prerequisites

| Tool | Version |
|------|---------|
| Java JDK | 21+ |
| Maven | 3.8+ |
| MySQL | 8.x |

### 1. Clone the repository

```bash
git clone <repository-url>
cd code-meet-backend/backend
```

### 2. Create the MySQL database and user

```sql
CREATE DATABASE CodeMeetDB;
CREATE USER 'CodeMeetUser'@'localhost' IDENTIFIED BY 'Secret-123';
GRANT ALL PRIVILEGES ON CodeMeetDB.* TO 'CodeMeetUser'@'localhost';
FLUSH PRIVILEGES;
```

### 3. Configure application properties

Edit `src/main/resources/application.properties` and update the values described in the [Configuration](#configuration) section.

### 4. Install dependencies

```bash
mvn clean install -DskipTests
```

---

## Configuration

All configuration lives in `src/main/resources/application.properties`.

> **Security note:** API keys and database credentials are currently stored in plain text in this file. Before deploying to any shared or production environment, move all secrets to environment variables or a secrets manager.

| Property | Description |
|----------|-------------|
| `spring.datasource.url` | JDBC URL for MySQL (`jdbc:mysql://localhost:3306/CodeMeetDB`) |
| `spring.datasource.username` | Database username |
| `spring.datasource.password` | Database password |
| `spring.jpa.hibernate.ddl-auto` | Schema management strategy (`update` — auto-migrates schema on startup) |
| `spring.jpa.show-sql` | Logs generated SQL statements to console |
| `agora.app-id` | Agora application ID for token generation |
| `agora.app-certificate` | Agora application certificate for token signing |

Cloudinary credentials are currently hardcoded in `CloudinaryConfig.java` and should be externalised to properties.

---

## Running the Application

```bash
# Using Maven wrapper
mvn spring-boot:run

# Or build a JAR and run it
mvn clean package -DskipTests
java -jar target/code-meet-*.jar
```

The server starts on **`http://localhost:8080`** by default.
The WebSocket endpoint is available at **`ws://localhost:8080/ws`**.

---

## API Documentation

All REST endpoints are prefixed with `/api`. Springdoc OpenAPI is on the classpath; the Swagger UI is available at `http://localhost:8080/swagger-ui.html` once the application is running.

---

### Authentication — `/api/auth`

| Method | Path | Description | Request Body |
|--------|------|-------------|-------------|
| `POST` | `/api/auth/signup` | Register a new user | `UserSignupRequest` |
| `POST` | `/api/auth/login` | Login and retrieve user info | `UserLoginRequest` |

**`UserSignupRequest`** fields: `firstName`, `lastName`, `username`, `email`, `password`, `phoneNumber`, `gender`
**`UserLoginRequest`** fields: `username`, `password`

---

### Users — `/api/user`

| Method | Path | Description | Notes |
|--------|------|-------------|-------|
| `GET` | `/api/user/all` | List all users | — |
| `GET` | `/api/user/{id}` | Get user by ID | — |
| `GET` | `/api/user?username=` | Get user by username | Query param |
| `GET` | `/api/user/search?query=&uno=` | Search users | `uno=true` for username-only search |
| `PUT` | `/api/user/update` | Update user profile | `UserUpdateRequest` body |
| `POST` | `/api/user/update/{userId}/profilePicture` | Upload profile picture | Multipart form-data |

---

### Meetings — `/api/meeting`

| Method | Path | Description | Request Body |
|--------|------|-------------|-------------|
| `GET` | `/api/meeting/{meetingId}` | Get meeting by ID | — |
| `GET` | `/api/meeting/scheduled/{userId}` | Get user's scheduled meetings | — |
| `GET` | `/api/meeting/previous/{userId}` | Get user's finished meetings | — |
| `GET` | `/api/meeting/{meetingId}/participants` | List all participants | — |
| `GET` | `/api/meeting/{meetingId}/user/{userId}` | Get participant info | — |
| `GET` | `/api/meeting/participant/{participantId}` | Get participant by ID | — |
| `POST` | `/api/meeting/participants` | Get multiple participants | `List<UUID>` body |
| `POST` | `/api/meeting/schedule` | Schedule a meeting | `ScheduleMeetingRequest` |
| `POST` | `/api/meeting/instant` | Start an instant meeting | `InstantMeetingRequest` |
| `POST` | `/api/meeting/request-join` | Request to join a meeting | `ParticipantRequest` |
| `POST` | `/api/meeting/accept-join` | Accept a join request | `ParticipantRequest` |
| `PATCH` | `/api/meeting/finish` | Finish a meeting (creator only) | `ParticipantRequest` |

**Meeting statuses:** `SCHEDULED` → `RUNNING` → `FINISHED`

---

### Rooms — `/api/room`

| Method | Path | Description | Request Body |
|--------|------|-------------|-------------|
| `GET` | `/api/room/{roomId}` | Get room by ID | — |
| `GET` | `/api/room/all/{userId}` | Get all rooms created by user | — |
| `GET` | `/api/room/search?query=` | Search rooms by name | — |
| `POST` | `/api/room/create` | Create a new room | `RoomCreationRequest` |
| `PUT` | `/api/room/update` | Update room details | `RoomUpdateRequest` |
| `POST` | `/api/room/update/{roomId}/roomPicture` | Upload room picture | Multipart form-data |

---

### Chats — `/api/chat`

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/chat/peer/{chatId}` | Get peer chat by ID |
| `GET` | `/api/chat/room/{chatId}` | Get room chat by ID |
| `GET` | `/api/chat/peer/{ownerId}/all` | All peer chats for a user |
| `GET` | `/api/chat/room/{ownerId}/all` | All room chats for a user |
| `GET` | `/api/chat/peer/{ownerId}/{peerId}` | Chat between two specific users |
| `GET` | `/api/chat/room/{ownerId}/{roomId}` | User's chat in a specific room |
| `GET` | `/api/chat/peer/{chatId}/messages` | All messages in a peer chat |
| `GET` | `/api/chat/room/{chatId}/messages` | All messages in a room chat |

---

### Friendships — `/api/friendship`

| Method | Path | Description | Body/Params |
|--------|------|-------------|-------------|
| `GET` | `/api/friendship/{userId}` | All friendships for user | — |
| `GET` | `/api/friendship/accepted/{userId}` | Accepted friends | — |
| `GET` | `/api/friendship/pending/{userId}/sent` | Sent pending requests | — |
| `GET` | `/api/friendship/pending/{userId}/received` | Received pending requests | — |
| `GET` | `/api/friendship?fromId=&toId=` | Friendship between two users | Query params |
| `POST` | `/api/friendship/request` | Send a friendship request | `FriendshipRequest` |
| `PATCH` | `/api/friendship/accept` | Accept a friendship request | `FriendshipRequest` |
| `DELETE` | `/api/friendship/cancel/{friendshipId}` | Cancel/remove friendship | — |

Accepting a friendship automatically creates peer chats between both users.

---

### Memberships — `/api/membership`

| Method | Path | Description | Body |
|--------|------|-------------|------|
| `GET` | `/api/membership/room/{roomId}/memberships` | All memberships in a room | — |
| `GET` | `/api/membership/room/{roomId}/memberships/accepted` | Accepted members | — |
| `GET` | `/api/membership/room/{roomId}/memberships/pending` | Pending requests | — |
| `GET` | `/api/membership/user/{userId}/rooms` | All rooms a user belongs to | — |
| `POST` | `/api/membership/request` | Request room membership | `MembershipRequest` |
| `PATCH` | `/api/membership/accept` | Accept membership request | `MembershipRequest` |
| `PATCH` | `/api/membership/accept/{membershipId}` | Accept by membership ID | — |
| `DELETE` | `/api/membership/cancel` | Cancel membership | `MembershipRequest` |
| `DELETE` | `/api/membership/cancel/{membershipId}` | Cancel by membership ID | — |

**Membership statuses:** `PENDING`, `ACCEPTED`, `ADMIN`, `INVITED`
Room admins cannot cancel their own membership.

---

### Notifications — `/api/notification`

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/notification/{receiverId}` | Get all notifications for user |
| `DELETE` | `/api/notification/delete/{notificationId}` | Delete a notification |

---

### Agora Tokens — `/api/agora`

| Method | Path | Description | Query Params |
|--------|------|-------------|-------------|
| `GET` | `/api/agora/rtm-token` | Generate RTM token | `uid`, `channelName`, `tokenExpire` |
| `GET` | `/api/agora/rtc-token` | Generate RTC token | `uid`, `channelName`, `tokenExpire`, `privilegeExpire` |

Tokens are signed with the configured Agora App Certificate. The RTC token grants `ROLE_PUBLISHER` privilege.

---

## WebSocket Flow

The application uses STOMP over SockJS for all real-time features.

### Connection

```
Client connects to:  ws://localhost:8080/ws  (SockJS fallback available)
```

### Broker destinations

| Prefix | Purpose |
|--------|---------|
| `/topic` | General broadcast topics |
| `/notifications` | Per-user notifications |
| `/peer-chat` | Peer-to-peer messages |
| `/room-chat` | Room broadcast messages |
| `/request-join` | Meeting join requests |
| `/accept-join` | Meeting join acceptances |

### Sending messages (client → server)

All client-sent messages use the `/app` application prefix:

| Destination | Payload | Description |
|-------------|---------|-------------|
| `/app/peer-chat` | `PeerMessageRequest` | Send a message to a peer |
| `/app/room-chat` | `RoomMessageRequest` | Send a message to a room |

### Receiving messages (server → client)

| Subscription | Trigger |
|-------------|---------|
| `/peer-chat/{peerId}` | New message received from a peer |
| `/room-chat/{roomId}` | New message in a room |
| `/request-join/{meetingId}` | Join request received (host subscribes) |
| `/accept-join/{userId}/{meetingId}` | Join request accepted |
| `/notifications/{userId}` | Any notification event |

### Message lifecycle

1. Client sends to `/app/peer-chat` or `/app/room-chat`.
2. `MessageService` saves message(s) to the database inside a transaction.
   - For peer messages: two records are created (one per chat direction).
   - For room messages: one record per accepted room member's chat.
3. After the transaction commits (`TransactionSynchronizationManager`), the message is broadcast via `SimpMessagingTemplate`.
4. A notification is also sent to the recipient's `/notifications/{userId}` subscription.

### Notification types

`FRIENDSHIP_REQUEST`, `FRIENDSHIP_ACCEPTED`, `MEMBERSHIP_REQUEST`, `MEMBERSHIP_ACCEPTED`, `MEETING_SCHEDULED`, `MEETING_STARTED`, `PEER_MESSAGE`, `ROOM_MESSAGE`

---

## Authentication and Authorization

> **Current state:** The application does **not** implement Spring Security, JWT, or any session management.

- **Login** is a simple database lookup that compares the submitted password against the stored value.
- **Passwords are stored in plain text.** This is a known gap and must be addressed before any shared deployment.
- All endpoints are publicly accessible; there is no token validation middleware.
- The only security layer currently in place is:
  - **CORS** — requests are restricted to `http://localhost:4200`.
  - **Business-logic guards** — e.g., only the meeting creator may call `/api/meeting/finish`.

---

## Error Handling

A `@RestControllerAdvice` global exception handler maps custom exceptions to HTTP responses:

| Exception | HTTP Status | Scenario |
|-----------|-------------|----------|
| `EntityNotFoundException` | `404 Not Found` | Requested resource does not exist |
| `DuplicateResourceException` | `409 Conflict` | Duplicate username, membership, etc. |
| `AuthenticationException` | `401 Unauthorized` | Invalid login credentials |
| `IllegalActionException` | `400 Bad Request` | Operation not permitted (e.g., non-creator ending a meeting) |
| `MissingServletRequestParameterException` | `400 Bad Request` | Required query parameter absent |
| `MethodArgumentTypeMismatchException` | `400 Bad Request` | Path/query parameter type mismatch |

All error responses are returned as structured JSON objects.

---

## External Integrations

### Agora

The application uses the [Agora Authentication SDK](https://github.com/AgoraIO/Tools/tree/master/DynamicKey) (`io.agora:authentication:2.1.2`) to generate short-lived server-side tokens:

- **RTM tokens** — used by the Agora RTM SDK for real-time messaging in meetings.
- **RTC tokens** — used by the Agora RTC SDK for audio/video streams. All tokens are issued with `ROLE_PUBLISHER`.

Token expiry is caller-controlled via request parameters.

### Cloudinary

Images are managed through the Cloudinary Java SDK (`cloudinary-http44:1.32.2`):

- Profile pictures are stored under `code-meet/profile-picture/{userId}`.
- Room pictures are stored under `code-meet/room-picture/{roomId}`.
- Upload mode is `overwrite`, so re-uploads replace the previous image.
- `FileUploadUtil` enforces a 1 MB file size limit and restricts uploads to `jpg`, `jpeg`, `png`, `svg`, `gif`, and `bmp`.

---

## Future Improvements

Based on gaps identified in the codebase:

- **Security hardening**
  - Implement Spring Security with stateless JWT authentication.
  - Hash passwords with BCrypt before storing them.
  - Add role-based access control (e.g., room admin vs. member).
  - Move all credentials and API keys to environment variables or a vault.
  - Enforce HTTPS.
  - Add rate limiting to auth and public endpoints.

- **Testing**
  - Unit tests for service-layer business logic.
  - Integration tests for controller endpoints and WebSocket handlers.

- **Database**
  - Switch `ddl-auto` from `update` to `validate` (or use Flyway/Liquibase migrations) for production safety.
  - Add database-level indexes where missing.

- **API improvements**
  - Introduce pagination on list endpoints (users, messages, rooms, notifications).
  - Add consistent response envelopes.

- **DevOps**
  - Provide a `docker-compose.yml` for local development (MySQL + backend).
  - Set up a CI pipeline (build, test, lint).

- **Observability**
  - Configure structured logging (Logback + JSON appender).
  - Add Spring Actuator health and metrics endpoints.
  - Integrate with a tracing solution (e.g., Micrometer + Zipkin).

---

## Contributing

1. Fork the repository and create a feature branch from `main`.
2. Follow the existing package and naming conventions (`com.codemeet.<layer>.<Class>`).
3. Write self-contained service methods; keep controllers thin (no business logic).
4. Add `@Transactional` to any service method that mutates state.
5. Map exceptions to the appropriate custom exception class rather than leaking internal errors to HTTP responses.
6. Submit a pull request with a clear description of the change and its motivation.
