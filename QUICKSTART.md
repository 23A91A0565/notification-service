# Quick Start Guide - Notification Service

## TL;DR - Get Running in 5 Minutes

### Prerequisites
- Docker and Docker Compose installed
- 2GB available RAM

### Step 1: Start All Services
```bash
cd notification-service
docker-compose up --build
```

Wait for all services to show "healthy" status (approximately 30-60 seconds).

### Step 2: Test the API

Open a new terminal and create a notification:

```bash
curl -X POST http://localhost:8080/api/notifications \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiI5ZmM4MWZkZC1mYzljLTQ5ZTMtOWRmMS1hMzZkZjdlMWVjNWEiLCJ1c2VybmFtZSI6InRlc3R1c2VyIiwiaWF0IjoxNzAzNjAwMDAwfQ.GyTdZVu2VUh8UZ5qmyY8xR2x8xN9pQ6rT5xK3jL4z8M" \
  -d '{
    "targetUserId": "9fc81fdd-fc9c-49e3-9df1-a36df7e1ec5a",
    "type": "in-app",
    "payload": {
      "title": "Hello World",
      "message": "Welcome to the notification service"
    }
  }'
```

### Step 3: Verify It Works

Check health:
```bash
curl http://localhost:8080/health
```

View logs:
```bash
docker-compose logs -f worker
```

You should see the worker processing the message within 2-3 seconds.

### Step 4: Verify in Database

```bash
docker-compose exec db psql -U notification_user -d notification_db \
  -c "SELECT id, user_id, type, status, created_at, processed_at FROM notifications LIMIT 5;"
```

## What's Running?

| Service | Port | URL |
|---------|------|-----|
| API | 8080 | http://localhost:8080 |
| PostgreSQL | 5432 | localhost:5432 |
| RabbitMQ | 5672 | localhost:5672 |
| RabbitMQ Admin | 15672 | http://localhost:15672 (guest:guest) |

## Common Commands

### Run Tests
```bash
docker-compose exec api npm test
docker-compose exec worker npm test
```

### View Logs
```bash
docker-compose logs -f api
docker-compose logs -f worker
docker-compose logs -f db
```

### Stop Services
```bash
docker-compose down
```

### Clean Everything
```bash
docker-compose down -v  # Removes volumes too
```

## Key Endpoints

### Create Notification
```
POST /api/notifications
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json

Body:
{
  "targetUserId": "<UUID>",
  "type": "email|in-app|sms",
  "payload": { ... any JSON object ... }
}

Response: 202 Accepted
{
  "messageId": "<UUID>",
  "status": "queued",
  "message": "Notification accepted for processing"
}
```

### Health Check
```
GET /health

Response: 200 OK
{
  "status": "healthy",
  "database": "connected",
  "rabbitmq": "connected",
  "timestamp": "<ISO_TIMESTAMP>"
}
```

## Flow Diagram

1. **Client** → Sends POST /api/notifications with JWT token
2. **API** → Validates request, generates messageId, publishes to RabbitMQ
3. **RabbitMQ** → Queues the message durably
4. **Worker** → Consumes message, checks idempotency, stores in DB
5. **Database** → Stores notification with status='processed'
6. **Client** ← Immediately gets 202 Accepted (asynchronous)

## Troubleshooting

### Services won't start
```bash
# Kill any process using port 8080
lsof -ti:8080 | xargs kill -9

# Restart Docker
docker-compose restart
```

### Database not initialized
```bash
docker-compose down -v  # Remove volumes
docker-compose up --build  # Fresh start
```

### Worker not processing
```bash
docker-compose logs worker | tail -50
```

Check the logs for errors. Most common: database connection or RabbitMQ connection issues.

## Next Steps

1. Read the full [README.md](../README.md) for comprehensive documentation
2. Run the test suite: `docker-compose exec api npm test`
3. Explore the [API Documentation](../README.md#api-documentation)
4. Check [Error Handling and Idempotency](../README.md#error-handling-and-idempotency)
5. Review [Database Schema](../README.md#database-schema)

## Test Users

The database comes pre-seeded with test users:

| User | ID | Email |
|------|----|----|
| testuser | 9fc81fdd-fc9c-49e3-9df1-a36df7e1ec5a | test@example.com |
| user1 | (auto-generated) | user1@example.com |
| user2 | (auto-generated) | user2@example.com |

Use any valid user UUID in the `targetUserId` field.

## Project Structure

```
notification-service/
├── api/                 # Express API service
│   ├── src/
│   │   ├── app.js      # Main application
│   │   ├── controllers/
│   │   ├── services/
│   │   ├── routes/
│   │   └── middleware/
│   ├── Dockerfile
│   └── package.json
├── worker/              # Node.js worker service
│   ├── src/
│   │   ├── index.js    # Worker entry point
│   │   ├── consumers/
│   │   ├── services/
│   │   └── rabbitmq.js
│   ├── Dockerfile
│   └── package.json
├── db/
│   └── init.sql        # Database schema
├── tests/              # Test files
├── docker-compose.yml  # Complete stack
├── .env.example        # Environment variables
└── README.md           # Full documentation
```

---

**Enjoy building event-driven systems!** 🚀
