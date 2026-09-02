# Tech Stack Manual

## 1. Run Questions
a. What config files exist in the system? The config files are already filled out for you - your job is to understand them, not edit them. Find the config files, then pick 3 values from each and explain what each one is for and how it is used by the system.

### 1a. Config Files

| Config File | Location | Config Value | What it's for | How it's used |
| .env| learn-ops-api/.env|LEARN_OPS_DB=learningplatform| Identfying the database's name in settings.py|it's used for django to point to your database|
|.env |learn-ops-client/.env | REACT_APP_ENV="development" | to see if the environment on client side is in dev mode? | Dont see it used anywhere |
| .env| learn-ops-api/.env|GITHUB_TOKEN | The Github toekn used to verify my computer | Its used in authorization in the app |
|docker-compose.yml |service-monarch/docker-compose.yml |networks:
      - learningplatform | For the whole system to be on the same network| IDK? |

### 1b. How to Start It



### 1c. Where to Access It

| Service | Port | URL |
|---|---|---|
| | | | 
| | | |
| | | |

### 1d. Service Dependencies

| Service | Depends On | Why |
|---|---|---|
| | | |
| | | |
| | | |

### 1e. Main Entry Points

| Service | Startup File | Routes / URL Config File |
|---|---|---|
| | | |
| | | |
| | | |

---

## 2. Services

| Service Name | Tech Stack (including version) | Purpose |
|---|---|---|
| | | |

---

## 3. System Overview