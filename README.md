# QOK Backend

REST API backend for **QOK**, a two-player online quiz game. Players sign up, get matched with an opponent, pick question categories, answer multiple-choice questions round by round, and earn coins and XP based on their results. Players can also submit their own questions, which go through a review state before they are used in games.

Built with Django and Django REST Framework, using PostgreSQL for storage, Redis for short-lived verification codes, and JWT for authentication.

## Features

- **Email-based authentication** — registration with an email verification link, JWT login, and password reset with a one-time code sent by email.
- **Player accounts** — coins, a coin transaction log, a selectable avatar icon, and a level system driven by XP.
- **Categories** — admin-managed question categories, plus a random pick of three categories for a round.
- **User-submitted questions** — players create questions with one correct and three wrong answers; each question is `pending`, `accept` or `failed`, and only accepted questions are served in games.
- **Question reports** — players can rate a question as good, bad or "no idea".
- **Matchmaking and challenges** — a player is matched with anyone waiting in the queue, or is placed in the queue until someone joins.
- **Rewards** — at the end of a challenge, coins and XP are awarded based on the number of correct answers; giving up costs coins.
- **Statistics** — total games, wins, draws and losses, plus per-category correct-answer counts.
- **Background jobs** — stale challenges are closed and finished ones are archived automatically.

## Tech stack

| Component | Version |
| --- | --- |
| Python | 3 |
| Django | 4.0.1 |
| Django REST Framework | 3.13.1 |
| djangorestframework-simplejwt | 5.0.0 |
| PostgreSQL | latest Docker image |
| Redis | latest Docker image |
| APScheduler | 3.8.1 |
| Pillow | 9.0.1 (image uploads) |

## Project structure

```
QOK-Backend-Master/
├── QOK/              # Project settings, root URLs, WSGI/ASGI entry points
├── user/             # Custom user model (email as login), register, verify, login, password reset
├── account/          # Player account, level, coin log, icons, statistics
├── category/         # Question categories
├── question/         # User-submitted questions and question reports
├── challenge/        # Matchmaking, challenge rounds, answers, results, scheduled jobs
├── media_root/       # Uploaded files (account icons)
├── Dockerfile
├── docker-compose.yml
├── manage.py
└── requirements.txt
```

## Getting started

### Prerequisites

- Docker and Docker Compose

The settings expect the database at host `db` and Redis at host `redis`, which are the service names in `docker-compose.yml`. Running with Docker Compose is the intended setup.

### 1. Create a `.env` file

Create a `.env` file in the project root (it is already listed in `.gitignore`):

```env
# Django
SECRET_KEY=change-me
DEBUG=True

# PostgreSQL
DATABASE_NAME=qok
DATABASE_USER=qok
DATABASE_PASSWORD=change-me
DATABASE_PORT=5432

# Redis
REDIS_PASSWORD=change-me
REDIS_PORT=6379

# Port the API is published on
APP_PORT=8000

# SMTP account used to send verification and reset emails (Gmail)
EMAIL_USEREMAIL=you@gmail.com
EMAIL_PASSWORD=your-app-password
```

| Variable | Description |
| --- | --- |
| `SECRET_KEY` | Django secret key. |
| `DEBUG` | `True` enables debug mode; any other value disables it. |
| `DATABASE_NAME` | PostgreSQL database name. Keep it equal to `DATABASE_USER` (see note below). |
| `DATABASE_USER` | PostgreSQL user. |
| `DATABASE_PASSWORD` | PostgreSQL password. |
| `DATABASE_PORT` | PostgreSQL port inside the Docker network, normally `5432`. |
| `REDIS_PASSWORD` | Password Redis is started with and that Django uses to connect. |
| `REDIS_PORT` | Host port Redis is published on. |
| `APP_PORT` | Host port the API is published on. |
| `EMAIL_USEREMAIL` | Gmail address used to send emails. |
| `EMAIL_PASSWORD` | Password for that account (use a Gmail App Password). |

> **Note:** `docker-compose.yml` passes `POSTGRES_NAME` to the Postgres container, which the official image does not read. The image therefore creates a database named after `POSTGRES_USER`, so set `DATABASE_NAME` to the same value as `DATABASE_USER`.

### 2. Build and start the services

```bash
docker compose up --build
```

This starts three services: `web` (Django on port 8000 inside the container), `db` (PostgreSQL) and `redis`.

### 3. Apply migrations and create an admin user

```bash
docker compose exec web python manage.py migrate
docker compose exec web python manage.py createsuperuser
```

The API is now available at `http://localhost:${APP_PORT}/` and the Django admin at `http://localhost:${APP_PORT}/admin/`.

### 4. Seed initial data

Games need data that only an admin can create. Through the Django admin (or the admin-only API endpoints):

1. Create categories and set their `status` to active — inactive categories are hidden from the API.
2. Add questions to each category and set their state to `accept`. A round picks four random questions from a category, so each category needs at least four.
3. Upload account icons (optional).

## Authentication

Protected endpoints use JWT bearer tokens:

```
Authorization: Bearer <access_token>
```

- Access tokens are valid for **5 hours**.
- Refresh tokens are valid for **1 day**.

Get tokens from `POST /user/login/` or `POST /api/token/`, and refresh them with `POST /api/token/refresh/`.

### Registration flow

1. `POST /user/register/` with `email` and `password`. The pending registration is stored in Redis for one hour and a verification link is emailed to the user.
2. The user opens the link (`GET /user/verify?u=<token>`). This creates the user, their account (0 coins) and their level (stage 1).
3. The user logs in with `POST /user/login/`.

### Password reset flow

1. `POST /user/restore-password/` with `email`. A 5-digit code is emailed and kept in Redis for 15 minutes.
2. `POST /user/restore-password/verify/` with `email`, `code` and the new `password`.

## API reference

**Auth** column: `—` public, `User` any authenticated user, `Admin` staff user.

### Tokens

| Method | Endpoint | Auth | Description |
| --- | --- | --- | --- |
| POST | `/api/token/` | — | Get an access/refresh token pair (`email`, `password`). |
| POST | `/api/token/refresh/` | — | Get a new access token (`refresh`). |
| — | `/api/auth/` | — | DRF browsable-API login/logout. |

### User — `/user/`

| Method | Endpoint | Auth | Body / params | Description |
| --- | --- | --- | --- | --- |
| POST | `/user/register/` | — | `email`, `password` | Start registration and send the verification email. |
| GET | `/user/verify?u=<token>` | — | `u` query param | Confirm the email and create the user. |
| POST | `/user/login/` | — | `email`, `password` | Return `access` and `refresh` tokens. |
| POST | `/user/restore-password/` | — | `email` | Email a password reset code. |
| POST | `/user/restore-password/verify/` | — | `email`, `code`, `password` | Set a new password. |

### Account — `/account/`

| Method | Endpoint | Auth | Body / params | Description |
| --- | --- | --- | --- | --- |
| GET | `/account/accounts/` | User | — | List accounts (user info, coins, icon, level). |
| GET | `/account/accounts/<id>/` | User | — | Retrieve an account. |
| POST / PUT / PATCH / DELETE | `/account/accounts/…` | Admin | — | Manage accounts. |
| GET / POST | `/account/coin-logs/` | User | `amount`, `description` | List or add coin transactions. |
| GET / PUT / PATCH / DELETE | `/account/coin-logs/<id>/` | User | — | Manage a coin transaction. |
| POST | `/account/update-account/` | User | `username`, `icon` (icon id) | Update the username and icon. |
| POST | `/account/update-icon/` | User | `icon` (icon id) | Update only the icon. |
| GET | `/account/games-count/` | User | — | Return `total`, `wins`, `equal` and `loss` counts. |
| GET | `/account/summary/` | User | — | Correct and total answers per category for the current user. |

### Category — `/category/`

| Method | Endpoint | Auth | Description |
| --- | --- | --- | --- |
| GET | `/category/` | User | List active categories. |
| GET | `/category/<id>/` | User | Retrieve a category. |
| POST / PUT / PATCH / DELETE | `/category/…` | Admin | Manage categories (`name`, `status`). |
| GET | `/category/random/` | User | Return three random active categories. |

### Question — `/question/`

| Method | Endpoint | Auth | Body / params | Description |
| --- | --- | --- | --- | --- |
| POST | `/question/create/` | User | `category`, `text`, `correct_answer`, `mistake_answer_1`, `mistake_answer_2`, `mistake_answer_3` | Submit a question (starts as `pending`). |
| GET | `/question/question-by-category?category=<name>` | User | `category` query param (category name) | List accepted questions in a category. |
| GET | `/question/question-by-accept-state/` | User | — | Your accepted questions. |
| GET | `/question/question-by-pending-state/` | User | — | Your pending questions. |
| GET | `/question/question-by-failed-state/` | User | — | Your rejected questions. |
| GET / DELETE | `/question/retrieve-delete/<id>` | User | — | Retrieve or delete one of your questions. |
| GET / POST | `/question/report/` | User | `question`, `report` | List reports or report a question. |

`report` is one of `None`, `bad_question`, `good_question` or `no_idea`.

### Challenge — `/challenge/`

| Method | Endpoint | Auth | Body | Description |
| --- | --- | --- | --- | --- |
| GET | `/challenge/match/` | User | — | Join an open challenge request, or create one and wait. |
| POST | `/challenge/category/` | User | `challenge`, `category` | Start a round in a category; four random questions are assigned. |
| POST | `/challenge/answer/` | User | `question`, `account`, `challenge_category`, `answer` | Submit an answer to a round question. |
| GET | `/challenge/open-challenge-list/` | User | — | Your challenges still in progress. |
| GET | `/challenge/end-challenge-list/` | User | — | Your finished (non-archived) challenges. |
| GET | `/challenge/challenge-result/` | User | `challenge_id` | Your result (state, coins, XP) for a challenge. |
| POST | `/challenge/give-up/` | User | `challenge` | Forfeit a challenge. |

## Game rules

### Matchmaking

`GET /challenge/match/` looks for a challenge request from another player that nobody has accepted yet.

- **One found:** the request is accepted and a new challenge is created between the two players. The response is the challenge.
- **None found:** a new challenge request is created for the caller. The response is that request; call the endpoint again later to check for a match.

### Rounds and answers

- In each round a player picks a category (`POST /challenge/category/`), and four random questions from that category are assigned to the round.
- Each correct answer adds one win to that player's score in the challenge.
- The challenge ends automatically when **48 answers** have been submitted — for example 6 rounds × 4 questions × 2 players.

### Rewards

When a challenge ends, each player's reward is `int(correct_answers × 1.5)`.

- **XP:** both players gain their reward as XP.
- **Coins:** the winner gains their reward in coins and the loser gains none. On a draw, both players gain their reward.
- Every coin change is written to the coin log, and a result (`win`, `equal` or `loss`) is stored for each player.

### Giving up

The player who gives up loses the challenge and is charged 50 coins. The opponent wins and gains their reward in coins. Both players still gain XP for their correct answers.

### Levels

Each account has a level with a `stage` and an XP range (`minimum` to `maximum`). New accounts start at stage 1 with a range of 0–10. When the XP reaches `maximum`, the player moves to the next stage, the extra XP carries over, and `maximum` grows by 10.

### Scheduled jobs

An APScheduler background scheduler starts with the `challenge` app and runs every minute:

- Challenges still in progress more than **1 day** after creation are marked as ended.
- Ended challenges are archived **2 days** after they ended, which hides them from the challenge lists.

## Data model

| App | Model | Purpose |
| --- | --- | --- |
| `user` | `CustomUser` | User with `email` as the login field and an optional `username`. |
| `account` | `Account` | One per user: coins and icon. |
| `account` | `Level` | One per account: stage and XP range. |
| `account` | `CoinLog` | Coin transactions with amount and description. |
| `account` | `AccountIcon` | Uploadable avatar images (stored in `media_root/icons/`). |
| `category` | `Category` | Question category with an active/inactive `status`. |
| `question` | `Question` | Question text, one correct answer, three wrong answers and a review `state`. |
| `question` | `ReportQuestion` | A player's rating of a question. |
| `challenge` | `ChallengeRequest` | A player waiting for an opponent. |
| `challenge` | `Challenge` | A match between two accounts with each player's win count, state and archive flag. |
| `challenge` | `ChallengeCategory` | One round: the chosen category and its four questions. |
| `challenge` | `ChallengeAnswer` | A player's answer to a round question. |
| `challenge` | `ChallengeResult` | The final outcome, coins and XP for each player. |

## Useful commands

```bash
# Create migrations after changing models
docker compose exec web python manage.py makemigrations

# Apply migrations
docker compose exec web python manage.py migrate

# Run the test suite
docker compose exec web python manage.py test

# Open a Django shell
docker compose exec web python manage.py shell
```

## Production notes

The current configuration is set up for development. Before deploying:

- Set `DEBUG=False` and use a strong, unique `SECRET_KEY`.
- Restrict `ALLOWED_HOSTS` in `QOK/settings.py` (it is currently `['*']`).
- Run the app with a production server such as Gunicorn instead of `runserver`.
- Serve static and media files from a web server or object storage.
- The verification link is built as `http://<host>/user/verify?...`; serve the API over HTTPS.
