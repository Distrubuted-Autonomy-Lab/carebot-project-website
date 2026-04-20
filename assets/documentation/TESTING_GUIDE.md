# Chat Test Cases: Quick Testing Guide

This guide is for the test suite in `chat/testcases.py`, mainly `ChatApiTests`.

## What These Tests Cover

- GET `/chat/` returns HTTP 200.
- POST `/chat/` returns JSON with expected keys.
- POST `/chat/` creates both USER and CHATBOT messages.
- GET `/suggestions/` returns a list of generated follow-up questions.

Most behavior is mocked, so these run as fast unit-style tests.

## Prerequisites

- Docker Desktop running.
- You are in the `root/` directory (same folder as `docker-compose.yml`).

## Run This Test Class

```bash
docker compose -f docker-compose.yml -f docker-compose.local.yml up -d
docker compose -f docker-compose.yml -f docker-compose.local.yml exec -T web python manage.py test --noinput chat.testcases.ChatApiTests
```

## Run A Single Test Method

```bash
docker compose -f docker-compose.yml -f docker-compose.local.yml exec -T web python manage.py test --noinput chat.testcases.ChatApiTests.test_chat_api_post_sends_message_and_returns_response
```

## Notes

- This guide intentionally focuses only on `chat/testcases.py`.
- We are using direct Docker + Django test commands (no wrapper script).
