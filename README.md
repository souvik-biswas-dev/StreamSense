# StreamSense 🎬

A full-stack movie streaming and discovery platform with personalized recommendations, user reviews, and JWT-secured authentication.

Built with a **React + Vite** frontend and a **Go (Gin)** backend, backed by **MongoDB**.

---

## Features

- **Movie Browsing** — explore a curated catalog of movies with posters and metadata
- **Movie Streaming** — watch embedded YouTube trailers via React Player
- **User Authentication** — register, login, and logout with JWT tokens stored in HTTP-only cookies
- **Personalized Recommendations** — genre-based movie suggestions for authenticated users
- **User Reviews** — authenticated users can view and admins can update movie reviews/rankings
- **Protected Routes** — frontend route guards and backend middleware both enforce auth

---

## Tech Stack

| Layer      | Technology                          |
|------------|-------------------------------------|
| Frontend   | React 19, Vite, React Router DOM 7  |
| UI         | React Bootstrap, FontAwesome        |
| HTTP       | Axios (with cookie support)         |
| Video      | React Player (YouTube embed)        |
| Backend    | Go 1.24, Gin Web Framework          |
| Auth       | JWT (golang-jwt v5), bcrypt         |
| Database   | MongoDB (official Go driver v2)     |
| Config     | godotenv                            |

---

## Project Structure

```
StreamSense-main/
├── Client/
│   └── magic-stream-client/   # React + Vite frontend
│       ├── src/
│       │   ├── api/           # Axios instances (public & private)
│       │   ├── context/       # Auth context provider
│       │   ├── hooks/         # useAuth, useAxiosPrivate
│       │   └── components/    # All UI components
│       └── public/
│
├── Server/
│   └── StreamSenseServer/     # Go + Gin backend
│       ├── controllers/       # Request handlers
│       ├── middleware/        # JWT auth middleware
│       ├── models/            # Data structs
│       ├── routes/            # Route registration
│       ├── utils/             # Token generation & validation
│       └── database/          # MongoDB connection
│
└── magic-stream-seed-data/    # JSON seed data for MongoDB
```

---

## Getting Started

### Prerequisites

- Node.js 18+
- Go 1.24+
- MongoDB instance (local or Atlas)

---

### Backend Setup

```bash
cd Server/StreamSenseServer
```

Create a `.env` file:

```env
MONGODB_URI=mongodb://localhost:27017
DATABASE_NAME=magicstream
SECRET_KEY=your_jwt_access_secret
SECRET_REFRESH_KEY=your_jwt_refresh_secret
ALLOWED_ORIGINS=http://localhost:5173
```

Install dependencies and run:

```bash
go mod download
go run main.go
```

Server starts on `http://localhost:8080`.

---

### Frontend Setup

```bash
cd Client/magic-stream-client
```

Create a `.env` file:

```env
VITE_API_BASE_URL=http://localhost:8080
```

Install dependencies and run:

```bash
npm install
npm run dev
```

App starts on `http://localhost:5173`.

---

### Seed the Database

Import the JSON files from `magic-stream-seed-data/` into MongoDB:

```bash
mongoimport --db magicstream --collection movies --file magic-stream-seed-data/movies.json --jsonArray
mongoimport --db magicstream --collection users --file magic-stream-seed-data/users.json --jsonArray
mongoimport --db magicstream --collection genres --file magic-stream-seed-data/genres.json --jsonArray
mongoimport --db magicstream --collection rankings --file magic-stream-seed-data/rankings.json --jsonArray
```

---

## API Reference

### Public Endpoints

| Method | Route      | Description                   |
|--------|------------|-------------------------------|
| GET    | /movies    | Fetch all movies              |
| GET    | /genres    | Fetch all genres              |
| POST   | /register  | Create a new user account     |
| POST   | /login     | Authenticate and receive JWT  |
| POST   | /logout    | Clear session cookies         |
| POST   | /refresh   | Refresh expired access token  |
| GET    | /hello     | Health check                  |

### Protected Endpoints (JWT required)

| Method | Route                      | Description                     |
|--------|----------------------------|---------------------------------|
| GET    | /movie/:imdb_id            | Get single movie details        |
| POST   | /addmovie                  | Add a new movie (admin)         |
| GET    | /recommendedmovies         | Get personalized recommendations|
| PATCH  | /updatereview/:imdb_id     | Update movie review (admin)     |

---

## Authentication Flow

1. User registers or logs in → server validates credentials and issues JWT tokens
2. Tokens are stored in HTTP-only cookies (`access_token`, `refresh_token`)
3. Frontend persists user info in `localStorage` via `AuthProvider`
4. Protected routes validate JWT via `AuthMiddleware` on the backend
5. Expired access tokens are silently refreshed using the refresh token

---

## Database Schema

### `users`
| Field             | Type     | Description              |
|-------------------|----------|--------------------------|
| user_id           | string   | Unique identifier        |
| first_name        | string   |                          |
| last_name         | string   |                          |
| email             | string   | Unique, validated        |
| password          | string   | bcrypt hashed            |
| role              | string   | `ADMIN` or `USER`        |
| favourite_genres  | []Genre  | Preferred genres         |
| token             | string   | Current access token     |
| refresh_token     | string   | Current refresh token    |

### `movies`
| Field        | Type     | Description                    |
|--------------|----------|--------------------------------|
| imdb_id      | string   | Unique movie identifier        |
| title        | string   |                                |
| poster_path  | string   | URL to poster image            |
| youtube_id   | string   | YouTube video ID for streaming |
| genre        | []string | Associated genres              |
| admin_review | string   | Admin-authored review text     |
| ranking      | Ranking  | Quality ranking object         |

---

## Scripts

### Frontend

```bash
npm run dev       # Start dev server
npm run build     # Production build → dist/
npm run preview   # Preview production build
npm run lint      # Run ESLint
```

### Backend

```bash
go run main.go    # Run server
go build          # Compile binary
```

---

## Low-Level Design (LLD)

### Class Diagrams

#### Backend Data Models

```mermaid
classDiagram
    class User {
        +ObjectID _id
        +string user_id
        +string first_name
        +string last_name
        +string email
        +string password
        +string role
        +time created_at
        +time updated_at
        +string token
        +string refresh_token
        +Genre[] favourite_genres
    }

    class UserLogin {
        +string email
        +string password
    }

    class UserResponse {
        +string user_id
        +string first_name
        +string last_name
        +string email
        +string role
        +Genre[] favourite_genres
    }

    class Movie {
        +ObjectID _id
        +string imdb_id
        +string title
        +string poster_path
        +string youtube_id
        +Genre[] genre
        +string admin_review
        +Ranking ranking
    }

    class Genre {
        +int genre_id
        +string genre_name
    }

    class Ranking {
        +int ranking_value
        +string ranking_name
    }

    class SignedDetails {
        +string email
        +string first_name
        +string last_name
        +string role
        +string user_id
        +string iss
        +int64 iat
        +int64 exp
    }

    User "1" --> "*" Genre : favourite_genres
    Movie "1" --> "*" Genre : genre
    Movie "1" --> "1" Ranking : ranking
    User --> SignedDetails : encoded into JWT
    UserLogin --> User : authenticates
    User --> UserResponse : projected to
```

#### Backend Layer Architecture

```mermaid
classDiagram
    class Router {
        +SetupUnprotectedRoutes(engine)
        +SetupProtectedRoutes(engine)
    }

    class AuthMiddleware {
        +Authenticate(ctx) Handler
        -extractToken(ctx) string
        -validateToken(token) Claims
    }

    class UserController {
        +RegisterUser(ctx)
        +LoginUser(ctx)
        +LogoutHandler(ctx)
        +RefreshTokenHandler(ctx)
        -hashPassword(password) string
        -verifyPassword(hashed, plain) bool
    }

    class MovieController {
        +GetMovies(ctx)
        +GetMovie(ctx)
        +AddMovie(ctx)
        +GetRecommendedMovies(ctx)
        +AdminReviewUpdate(ctx)
        -GetUsersFavouriteGenres(userId) Genre[]
        -GetReviewRanking(review) Ranking
    }

    class TokenUtil {
        +GenerateAllTokens(email, firstName, lastName, role, userId) string, string
        +ValidateToken(token) Claims, error
        +UpdateAllTokens(token, refreshToken, userId)
        +ExtractTokenFromCookie(ctx) string
    }

    class DatabaseConnection {
        +DBInstance() mongo.Client
        +OpenCollection(client, name) mongo.Collection
        -connectDB(uri) mongo.Client
    }

    Router --> AuthMiddleware : applies to protected routes
    Router --> UserController : registers handlers
    Router --> MovieController : registers handlers
    UserController --> TokenUtil : generates and validates tokens
    UserController --> DatabaseConnection : reads and writes users
    MovieController --> DatabaseConnection : reads and writes movies
    AuthMiddleware --> TokenUtil : validates tokens
```

#### Frontend Component Hierarchy

```mermaid
classDiagram
    class App {
        +routes JSX
        +render() JSX
    }

    class AuthProvider {
        +auth object
        +setAuth function
        +loading boolean
        +render() JSX
    }

    class RequiredAuth {
        +allowedRoles string[]
        -checkAuth() boolean
        +render() JSX
    }

    class Header {
        +auth object
        +handleLogout() void
        +render() JSX
    }

    class Home {
        +movies Movie[]
        +loading boolean
        +fetchMovies() void
        +render() JSX
    }

    class Movies {
        +movies Movie[]
        +render() JSX
    }

    class Movie {
        +movie object
        +render() JSX
    }

    class Login {
        +email string
        +password string
        +handleSubmit() void
        +render() JSX
    }

    class Register {
        +formData object
        +genres Genre[]
        +handleSubmit() void
        +render() JSX
    }

    class StreamMovie {
        +ytId string
        +render() JSX
    }

    class Recommended {
        +movies Movie[]
        +loading boolean
        +fetchRecommended() void
        +render() JSX
    }

    class Review {
        +movie object
        +reviewText string
        +handleSubmit() void
        +render() JSX
    }

    class useAuth {
        +auth object
        +setAuth function
    }

    class useAxiosPrivate {
        +axiosPrivate AxiosInstance
        -attachInterceptors() void
        -refreshToken() Promise
    }

    App --> AuthProvider : wraps app
    App --> Header : always rendered
    App --> RequiredAuth : guards protected routes
    RequiredAuth --> Recommended : protected
    RequiredAuth --> Review : protected (admin only)
    RequiredAuth --> StreamMovie : protected
    Home --> Movies : renders grid
    Movies --> Movie : renders each card
    Recommended --> Movies : renders recommendations
    Login --> useAuth : reads and sets auth
    Login --> useAxiosPrivate : API calls
    Review --> useAxiosPrivate : API calls
    useAxiosPrivate --> useAuth : reads auth context
```

---

### Sequence Diagrams — Authentication Flows

#### User Registration

```mermaid
sequenceDiagram
    actor User
    participant Register as Register Component
    participant Axios as Axios (Public)
    participant Server as Go Server
    participant DB as MongoDB

    User->>Register: Fill form (name, email, password, genres)
    Register->>Axios: POST /register {user data}
    Axios->>Server: HTTP POST /register
    Server->>Server: Validate input fields
    Server->>DB: Find user by email
    DB-->>Server: No existing user found
    Server->>Server: Hash password (bcrypt)
    Server->>DB: Insert new User document
    DB-->>Server: Inserted OK
    Server-->>Axios: 201 Created
    Axios-->>Register: Success response
    Register->>User: Redirect to /login
```

#### User Login

```mermaid
sequenceDiagram
    actor User
    participant Login as Login Component
    participant Axios as Axios (Public)
    participant Server as Go Server
    participant TokenUtil as Token Util
    participant DB as MongoDB
    participant Auth as AuthProvider

    User->>Login: Enter email + password
    Login->>Axios: POST /login {email, password}
    Axios->>Server: HTTP POST /login
    Server->>DB: Find user by email
    DB-->>Server: User document
    Server->>Server: Verify password (bcrypt.Compare)
    Server->>TokenUtil: GenerateAllTokens(email, name, role, userId)
    TokenUtil-->>Server: access_token (24h) + refresh_token (7d)
    Server->>DB: UpdateAllTokens(userId, tokens)
    DB-->>Server: Updated OK
    Server-->>Axios: 200 OK + Set-Cookie (access_token, refresh_token)
    Axios-->>Login: UserResponse payload
    Login->>Auth: setAuth(userResponse)
    Auth->>Auth: Persist to localStorage
    Login->>User: Redirect to original location or /
```

#### Silent Token Refresh

```mermaid
sequenceDiagram
    participant Component as React Component
    participant Interceptor as Axios Interceptor
    participant Server as Go Server
    participant TokenUtil as Token Util
    participant DB as MongoDB

    Component->>Interceptor: API call with expired access_token
    Interceptor->>Server: Original request → 401 Unauthorized
    Server-->>Interceptor: 401 response
    Interceptor->>Interceptor: Queue subsequent requests
    Interceptor->>Server: POST /refresh (refresh_token cookie)
    Server->>Server: Read refresh_token from cookie
    Server->>TokenUtil: ValidateToken(refresh_token)
    TokenUtil-->>Server: Valid claims
    Server->>TokenUtil: GenerateAllTokens(...) → new access_token
    TokenUtil-->>Server: New access_token
    Server->>DB: UpdateAllTokens(userId, new tokens)
    DB-->>Server: Updated OK
    Server-->>Interceptor: 200 OK + Set-Cookie (new access_token)
    Interceptor->>Interceptor: Flush queued requests
    Interceptor->>Server: Retry original request with new token
    Server-->>Component: Original response data
```

#### User Logout

```mermaid
sequenceDiagram
    actor User
    participant Header as Header Component
    participant Axios as Axios (Public)
    participant Server as Go Server
    participant DB as MongoDB
    participant Auth as AuthProvider

    User->>Header: Click Logout
    Header->>Axios: POST /logout {user_id}
    Axios->>Server: HTTP POST /logout
    Server->>DB: Clear tokens (token = "", refresh_token = "")
    DB-->>Server: Updated OK
    Server-->>Axios: 200 OK + Set-Cookie (MaxAge=-1 clears cookies)
    Axios-->>Header: Success
    Header->>Auth: setAuth({})
    Auth->>Auth: Clear localStorage
    Header->>User: Redirect to /login
```

---

### Sequence Diagrams — Feature Flows

#### Movie Streaming

```mermaid
sequenceDiagram
    actor User
    participant Movie as Movie Card
    participant RequiredAuth as RequiredAuth Guard
    participant Stream as StreamMovie Component
    participant YouTube as YouTube (Embedded)

    User->>Movie: Click movie card
    Movie->>RequiredAuth: Navigate to /stream/:yt_id
    RequiredAuth->>RequiredAuth: Check auth context
    alt Not authenticated
        RequiredAuth->>User: Redirect to /login
    else Authenticated
        RequiredAuth->>Stream: Render StreamMovie
        Stream->>YouTube: Embed player (youtube.com/watch?v=yt_id)
        YouTube-->>Stream: Video player ready
        Stream->>User: Display embedded video player
        User->>Stream: Play / pause / seek
    end
```

#### Personalized Recommendations

```mermaid
sequenceDiagram
    actor User
    participant Recommended as Recommended Component
    participant Interceptor as useAxiosPrivate
    participant Middleware as AuthMiddleware
    participant Controller as MovieController
    participant DB as MongoDB

    User->>Recommended: Navigate to /recommended
    Recommended->>Interceptor: GET /recommendedmovies
    Interceptor->>Middleware: Request with access_token cookie
    Middleware->>Middleware: ValidateToken(access_token)
    Middleware->>Middleware: Extract userId and role → set in Gin context
    Middleware->>Controller: Forward to handler
    Controller->>DB: Find user by userId → get favourite_genres
    DB-->>Controller: Genre[] list
    Controller->>DB: Find movies matching genres, sort by ranking ASC, limit 5
    DB-->>Controller: Movie[] results
    Controller-->>Interceptor: 200 OK + Movie[]
    Interceptor-->>Recommended: Movie data
    Recommended->>User: Render personalized movies grid
```

#### Admin Review and Sentiment Analysis

```mermaid
sequenceDiagram
    actor Admin
    participant Review as Review Component
    participant Interceptor as useAxiosPrivate
    participant Middleware as AuthMiddleware
    participant Controller as MovieController
    participant LangChain as LangChain (OpenAI)
    participant DB as MongoDB

    Admin->>Review: Navigate to /review/:imdb_id
    Review->>Interceptor: GET /movie/:imdb_id
    Interceptor->>Controller: Authenticated request
    Controller->>DB: Find movie by imdb_id
    DB-->>Controller: Movie document
    Controller-->>Review: Movie with existing review and ranking
    Review->>Admin: Display movie info + review textarea (admin role)

    Admin->>Review: Write review text and click Submit
    Review->>Interceptor: PATCH /updatereview/:imdb_id {admin_review}
    Interceptor->>Middleware: Request with access_token
    Middleware->>Middleware: ValidateToken → verify role = ADMIN
    Middleware->>Controller: Forward to handler
    Controller->>DB: Fetch all rankings (Excellent, Good, Okay, Bad, Terrible)
    DB-->>Controller: Ranking[] list
    Controller->>LangChain: Analyze sentiment of review against ranking options
    LangChain->>LangChain: Call OpenAI API with prompt + review text
    LangChain-->>Controller: Sentiment string (e.g. "Excellent")
    Controller->>Controller: Map sentiment to ranking_value (1–5)
    Controller->>DB: Update movie (admin_review, ranking)
    DB-->>Controller: Updated OK
    Controller-->>Review: 200 OK + updated Movie
    Review->>Admin: Display updated review and ranking badge
```

---

## License

MIT
