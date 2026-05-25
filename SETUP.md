# Local Setup Guide

How to run StreamSense (MagicStream) locally. The `.env` files below are **git-ignored**, so you must create them by hand after a fresh clone.

## Prerequisites

- Docker + Docker Compose (for MongoDB — no local Mongo install needed)
- Go 1.24+
- Node.js 18+

## 1. Start MongoDB

From the project root:

```bash
docker compose up -d
```

This starts:
- **MongoDB** on `localhost:27017` (user `admin`, password `password123`)
- **mongo-express** (web DB admin UI) on http://localhost:8081

## 2. Seed the database

The seed JSON files live in `magic-stream-seed-data/`. `mongoimport` is not installed locally, so run it **inside the Mongo container**:

```bash
docker cp magic-stream-seed-data mongodb_container:/seed
for c in movies users genres rankings; do
  docker exec mongodb_container mongoimport \
    --username admin --password password123 --authenticationDatabase admin \
    --db magicstream --collection "$c" --file "/seed/$c.json" --jsonArray --drop
done
```

## 3. Backend — `Server/MagicStreamServer/.env`

Create this file manually (it is git-ignored):

```env
MONGODB_URI=mongodb://admin:password123@localhost:27017/?authSource=admin
DATABASE_NAME=magicstream
SECRET_KEY=dev_access_secret_change_me
SECRET_REFRESH_KEY=dev_refresh_secret_change_me
ALLOWED_ORIGINS=http://localhost:5173
RECOMMENDED_MOVIE_LIMIT=5

# Only needed for the admin "Update Review" sentiment feature (see section 6)
GEMINI_API_KEY=
GEMINI_MODEL=gemini-2.0-flash
BASE_PROMPT_TEMPLATE=You are a sentiment classifier. Classify the following movie review into exactly ONE of these categories: {rankings}. Respond with only the single category word and nothing else. Review:
```

> The `MONGODB_URI` username/password **must match** `docker-compose.yml`. `?authSource=admin` is required because Mongo runs with root auth.

Run it:

```bash
cd Server/MagicStreamServer
go mod download
go run main.go        # serves on http://localhost:8080
```

## 4. Frontend — `Client/magic-stream-client/.env`

Create this file manually (git-ignored):

```env
VITE_API_BASE_URL=http://localhost:8080
```

Run it:

```bash
cd Client/magic-stream-client
npm install
npm run dev           # serves on http://localhost:5173
```

Open http://localhost:5173.

## 5. Test login

Seeded users have hashed passwords (unknown plaintext), so just **register a new account** in the app, or via API:

```bash
curl -X POST http://localhost:8080/register -H "Content-Type: application/json" \
  -d '{"first_name":"Test","last_name":"User","email":"testuser@example.com","password":"Password1!","role":"USER","favourite_genres":[{"genre_id":2,"genre_name":"Drama"}]}'
```

To test **admin-only** features (Update Review), set a user's `role` to `ADMIN` — either pass `"role":"ADMIN"` when registering via the API above, or edit the user doc in mongo-express (http://localhost:8081).

## 6. Gemini API key (optional)

The admin "Update Review" feature runs sentiment analysis through Google **Gemini** (swapped in from the original OpenAI to stay on a free tier). Everything else works without it.

To enable it:
1. Get a **free** key at https://aistudio.google.com/app/apikey
2. Paste it into `GEMINI_API_KEY` in `Server/MagicStreamServer/.env`
3. Restart the Go server

**Troubleshooting `429 ... limit: 0`:** if the server logs `RESOURCE_EXHAUSTED` with
`generate_content_free_tier_requests, limit: 0`, the key's Google project has **no free-tier
quota** (common when the account's region isn't free-tier eligible, or the key came from a
Cloud project rather than AI Studio). Fixes:
- Create the key from a personal Google account in a [supported region](https://ai.google.dev/gemini-api/docs/available-regions) via AI Studio, **or**
- Enable pay-as-you-go billing on the project (gemini-2.0-flash costs a fraction of a cent per review).

This affects **only** the admin Update-Review feature; the rest of the app runs fine without it.

## Handy commands

```bash
docker compose down          # stop MongoDB (keeps data volume)
docker compose down -v       # stop and WIPE the database
docker compose logs -f       # view Mongo logs
```

## Ports

| Service       | URL                     |
|---------------|-------------------------|
| Frontend      | http://localhost:5173   |
| Backend API   | http://localhost:8080   |
| MongoDB       | localhost:27017         |
| mongo-express | http://localhost:8081   |
