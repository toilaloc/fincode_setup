# Fincode Project Setup Guide

## Table of Contents
1. [System Requirements](#system-requirements)
2. [Clone Repository](#1-clone-repository)
3. [Environment Configuration](#2-environment-configuration)
4. [Docker Build & Up](#3-docker-build--up)
5. [Database Seeding](#4-database-seeding)
6. [Run Application](#5-run-application)
7. [Frontend Application](#6-frontend-application)
8. [Email Login](#7-email-login)
9. [Check Email](#8-check-email)
10. [Troubleshooting](#troubleshooting)

---

## System Requirements

Before starting, ensure your machine has the following installed:

- **Docker** (version 20.10 or higher)
- **Docker Compose** (version 2.0 or higher)
- **Git**

Check versions:
```bash
docker --version
docker-compose --version
git --version
```

---

## 1. Clone Repository

This setup requires three repositories in the same parent directory:

```bash
# Clone backend repository
git clone <backend-repo-url> fincode_app

# Clone frontend repository
git clone <frontend-repo-url> fincode_frontend

# Clone docker orchestration repository
git clone <docker-repo-url> fincode_setup
```

Expected directory structure:
```
.
├── fincode_app/              # Backend Rails application
├── fincode_frontend/         # Frontend React application
└── fincode_setup/            # Docker orchestration scripts
    └── bin/
        ├── setup_application  # Build and start all services
        ├── stop_application   # Stop all services
        ├── logs              # View logs from all services
        └── status            # Check status of all services
```

---

## 2. Environment Configuration

### 2.1. Backend Environment Configuration

Navigate to the backend directory and create `.env` file:

```bash
cd fincode_app
cp .env.example .env
```

### 2.2. Configure Backend Environment Variables

Open the `.env` file and update the necessary information:

```bash
# Rails Environment
RAILS_ENV=development

# Database Configuration
DATABASE_HOST=mysql
DATABASE_PORT=3306
DATABASE_USERNAME=fincode_user
DATABASE_PASSWORD=fincode_password
DATABASE_NAME=fincode_development

# Mailer Configuration
MAILER_HOST=localhost
MAILER_PORT=3005

# Fincode API Configuration
FINCODE_API_URL=https://api.test.fincode.jp
FINCODE_PUBLIC_KEY=p_test_xxxxx
FINCODE_SECRET_KEY=s_test_xxxxx
```

> **⚠️ Important - Fincode API Keys**: 
> 
> You **MUST** update `FINCODE_PUBLIC_KEY` and `FINCODE_SECRET_KEY` with actual API keys from Fincode.
> 
> **How to get Fincode API Keys:**
> 1. Sign up for a Fincode account at [https://www.fincode.jp/](https://www.fincode.jp/)
> 2. Log in to the Fincode Dashboard
> 3. Navigate to **Settings** → **API Keys**
> 4. Copy your **Public Key** (starts with `p_test_` for test mode)
> 5. Copy your **Secret Key** (starts with `s_test_` for test mode)
> 6. Paste them into your `.env` file
> 
> **Key Formats:**
> - Public Key: `p_test_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
> - Secret Key: `s_test_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
> 
> **Note:** For production, use keys starting with `p_live_` and `s_live_`.

### 2.3. Frontend Environment Configuration (Optional)

If the frontend requires environment configuration:

```bash
cd ../fincode_frontend
cp .env.example .env
```

Update frontend `.env` with backend API URL:
```bash
VITE_API_URL=http://localhost:3005
VITE_FINCODE_PUBLIC_KEY=p_test_xxxxx
```

---

## 3. Docker Build & Up

### 3.1. Build and Start All Services

From the `fincode_setup` directory:

```bash
cd fincode_setup
bin/setup_application up -d
```

This will:
- Build the Rails application image
- Pull MySQL 8.0 image
- Pull Redis 7 Alpine image
- Pull Mailcatcher image
- Start all services in detached mode
- Run database migrations and seeds automatically

The `-d` flag runs containers in detached mode (background).

### 3.2. Check Container Status

Verify that all containers are running successfully:

```bash
bin/status
```

Or use Docker directly:
```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

Expected output:
```
NAME                          STATUS                    PORTS
fincode_rails                 Up (healthy)              0.0.0.0:3005->3000/tcp
fincode_frontend-react-dev-1  Up                        0.0.0.0:3006->3006/tcp
fincode_mysql                 Up (healthy)              0.0.0.0:33060->3306/tcp
fincode_redis                 Up (healthy)              0.0.0.0:63790->6379/tcp
fincode_mailcatcher           Up                        0.0.0.0:10800->1080/tcp, 0.0.0.0:10250->1025/tcp
```

> **Note**: Wait until MySQL and Redis show `healthy` status before proceeding.

### 3.3. View Logs

To view logs from all services:

```bash
bin/logs
```

To view logs from a specific service:

```bash
bin/logs rails
bin/logs frontend
```

Or follow logs in real-time:

```bash
bin/logs -f rails
```

---

## 4. Database Seeding

The database will be automatically created and migrated when the Rails container starts for the first time (as configured in `docker-compose.yml`).

### 4.1. Check Logs

View the Rails container logs to ensure seeding completed successfully:

```bash
bin/logs rails
```

You should see output similar to:
```
Cleaning up database...
Seeding Users...
Seeding Categories...
Seeding Products...
Seeding done.
```

### 4.2. Re-run Seeding (If Needed)

If you need to re-run seeding:

```bash
docker exec -it fincode_rails bundle exec rails db:seed
```

Or reset the entire database:

```bash
docker exec -it fincode_rails bundle exec rails db:reset
```

---

## 5. Run Application

### 5.1. Access Application

After all containers are running successfully, open your browser and navigate to:

```
http://localhost:3005
```

### 5.2. Health Check

Check the API health endpoint:

```bash
curl http://localhost:3005/api/v1/health
```

Expected response:
```json
{
  "status": "ok",
  "timestamp": "2026-01-20T08:19:31Z"
}
```

---

## 6. Frontend Application

### 6.1. Access Frontend

The frontend application runs separately from the backend API. After setting up the frontend project, access it at:

```
http://localhost:3006
```

### 6.2. Frontend Configuration

The frontend is configured to communicate with the backend API at `http://localhost:3005`. Ensure both services are running:

- **Backend API**: `http://localhost:3005`
- **Frontend**: `http://localhost:3006`

### 6.3. Development Workflow

1. Start all services using `bin/setup_application up -d`
2. Access the frontend at `http://localhost:3006`
3. The frontend will make API calls to `http://localhost:3005`
4. Code changes will auto-reload:
   - **Backend**: Rails auto-reload enabled
   - **Frontend**: Vite HMR (Hot Module Replacement)

---

## 7. Email Login

The application uses **Magic Link Authentication** (passwordless login via email).

### 7.1. Available Test Accounts

After seeding, the following accounts are available:

| Email                    | Display Name    |
|--------------------------|-----------------|
| test@example.com         | Test Customer   |
| john@example.com         | johndoe         |
| johnwick@example.com     | johndoe         |
| janewick@example.com     | janesmith       |

### 7.2. Login Process

1. Navigate to the login page at `http://localhost:3006`
2. Enter an email address (e.g., `test@example.com`)
3. Click the "Send Magic Link" button
4. The system will send an email containing a magic link
5. Check the email in Mailcatcher (see next section)
6. Click the magic link in the email to log in

---

## 8. Check Email

### 8.1. Access Mailcatcher

Mailcatcher is a mock SMTP server that captures all emails in the development environment.

Open your browser and navigate to:

```
http://localhost:10800/
```

Or:

```
http://0.0.0.0:10800/
```

### 8.2. View Magic Link Email

1. In the Mailcatcher interface, you'll see a list of sent emails
2. Click on the latest email (Magic Link Email)
3. The email content will contain the login link
4. Click the link or copy it to log in

### 8.3. SMTP Configuration

Mailcatcher is running with the following configuration:
- **Web Interface**: `http://localhost:10800/` (port 1080 → 10800)
- **SMTP Server**: `localhost:10250` (port 1025 → 10250)

---

## Troubleshooting

### Issue 1: Container Fails to Start

**Symptoms**: One or more containers fail during startup.

**Solution**:
```bash
# View detailed logs
bin/logs [service-name]

# Example: view Rails logs
bin/logs rails

# Restart all services
bin/setup_application restart

# Or stop and start again
bin/stop_application
bin/setup_application up -d
```

### Issue 2: Database Connection Error

**Symptoms**: Rails cannot connect to MySQL.

**Solution**:
```bash
# Check if MySQL is healthy
bin/status

# Wait a few more seconds for MySQL to fully start
# Then restart the Rails container
cd ../fincode_app
docker compose restart rails
```

### Issue 3: Port Already in Use

**Symptoms**: Error "port is already allocated".

**Solution**:
```bash
# Check which ports are in use
sudo lsof -i :3005
sudo lsof -i :3006
sudo lsof -i :33060
sudo lsof -i :63790
sudo lsof -i :10800

# Kill the process using the port (if needed)
sudo kill -9 <PID>

# Or change ports in docker-compose.yml files
```

### Issue 4: Mailcatcher Not Receiving Emails

**Symptoms**: Emails don't appear in Mailcatcher.

**Solution**:
```bash
# Check Mailcatcher container
bin/status

# Check logs
bin/logs mailcatcher

# Restart Mailcatcher
cd ../fincode_app
docker compose restart mailcatcher

# Check Rails mailer configuration
docker exec -it fincode_rails bundle exec rails console
# In console:
ActionMailer::Base.delivery_method
ActionMailer::Base.smtp_settings
```

### Issue 5: Seeding Failed

**Symptoms**: Errors when running seeds.

**Solution**:
```bash
# Drop and recreate database
docker exec -it fincode_rails bundle exec rails db:drop db:create db:migrate db:seed

# Or reset database
docker exec -it fincode_rails bundle exec rails db:reset
```

### Issue 6: Bundle Install Errors

**Symptoms**: Errors when installing gems.

**Solution**:
```bash
# Rebuild Rails container
cd fincode_setup
bin/setup_application up -d --build

# Or exec into container and install manually
docker exec -it fincode_rails bundle install
```

### Issue 7: Fincode API Errors

**Symptoms**: Payment operations fail with API errors.

**Solution**:
```bash
# 1. Verify API keys are set correctly
docker exec -it fincode_rails bundle exec rails console
# In console:
ENV['FINCODE_PUBLIC_KEY']
ENV['FINCODE_SECRET_KEY']

# 2. Check if keys are valid (should start with p_test_ and s_test_)
# 3. Verify API URL is correct
ENV['FINCODE_API_URL']

# 4. Test API connection
# In Rails console:
require 'net/http'
uri = URI("#{ENV['FINCODE_API_URL']}/v1/payments")
response = Net::HTTP.get_response(uri)
puts response.code
```

### Issue 8: Frontend Cannot Connect to Backend

**Symptoms**: Frontend shows API connection errors.

**Solution**:
```bash
# 1. Verify backend is running
curl http://localhost:3005/api/v1/health

# 2. Check CORS configuration in backend
docker exec -it fincode_rails bundle exec rails console
# In console:
Rails.application.config.action_dispatch.cors_origins

# 3. Check frontend environment variables
docker exec -it fincode_frontend-react-dev-1 sh
# In container:
env | grep VITE
```

---

## Useful Commands

### Docker Orchestration Commands

```bash
# Start all services
bin/setup_application up -d

# Stop all services
bin/stop_application

# View logs from all services
bin/logs

# View logs from specific service
bin/logs rails
bin/logs frontend

# Follow logs in real-time
bin/logs -f rails

# Check status of all services
bin/status

# Rebuild and restart all services
bin/setup_application up -d --build

# Stop and remove volumes (caution: will lose data!)
bin/stop_application -v
```

### Rails Commands

```bash
# Exec into Rails container
docker exec -it fincode_rails bash

# Run Rails console
docker exec -it fincode_rails bundle exec rails console

# Run tests
docker exec -it fincode_rails bundle exec rspec

# Run migrations
docker exec -it fincode_rails bundle exec rails db:migrate

# Rollback migration
docker exec -it fincode_rails bundle exec rails db:rollback

# Reset database
docker exec -it fincode_rails bundle exec rails db:reset

# Seed database
docker exec -it fincode_rails bundle exec rails db:seed
```

### Frontend Commands

```bash
# Exec into Frontend container
docker exec -it fincode_frontend-react-dev-1 sh

# Install npm packages
docker exec -it fincode_frontend-react-dev-1 npm install

# Run build
docker exec -it fincode_frontend-react-dev-1 npm run build
```

### MySQL Direct Access

```bash
# Connect to MySQL container
docker exec -it fincode_mysql mysql -u fincode_user -pfincode_password fincode_development

# Or from host machine
mysql -h 127.0.0.1 -P 33060 -u fincode_user -pfincode_password fincode_development
```

### Redis Commands

```bash
# Connect to Redis
docker exec -it fincode_redis redis-cli -a redis_password

# Check Redis
docker exec -it fincode_redis redis-cli -a redis_password ping
```

---

## Project Structure

```
.
├── fincode_app/                # Backend Rails application
│   ├── app/                    # Rails application code
│   │   ├── controllers/        # Controllers
│   │   ├── models/            # Models
│   │   ├── mailers/           # Mailers (Magic Link)
│   │   ├── services/          # Business logic services
│   │   └── views/             # Views
│   ├── config/                # Configuration files
│   ├── db/                    # Database files
│   │   ├── migrate/           # Migrations
│   │   └── seeds.rb           # Seed data
│   ├── docker/                # Docker configuration
│   ├── spec/                  # RSpec tests
│   ├── docker-compose.yml     # Docker Compose configuration
│   ├── Dockerfile             # Rails Docker image
│   └── .env                   # Environment variables
│
├── fincode_frontend/          # Frontend React application
│   ├── src/                   # React source code
│   ├── public/                # Static assets
│   ├── docker-compose.dev.yml # Frontend Docker Compose
│   ├── Dockerfile.dev         # Frontend Docker image
│   └── .env                   # Frontend environment variables
│
└── fincode_setup/             # Docker orchestration
    ├── bin/
    │   ├── setup_application  # Start all services
    │   ├── stop_application   # Stop all services
    │   ├── logs              # View logs
    │   └── status            # Check status
    └── README.md             # This file
```

---

## Services & Ports

| Service      | Container Name               | Internal Port | External Port | URL                          |
|--------------|------------------------------|---------------|---------------|------------------------------|
| Rails (API)  | fincode_rails                | 3000          | 3005          | http://localhost:3005        |
| Frontend     | fincode_frontend-react-dev-1 | 3006          | 3006          | http://localhost:3006        |
| MySQL        | fincode_mysql                | 3306          | 33060         | localhost:33060              |
| Redis        | fincode_redis                | 6379          | 63790         | localhost:63790              |
| Mailcatcher  | fincode_mailcatcher          | 1080, 1025    | 10800, 10250  | http://localhost:10800       |

---

## Database Credentials

- **Database**: fincode_development
- **User**: fincode_user
- **Password**: fincode_password
- **Root Password**: root_password

## Redis Credentials

- **Password**: redis_password

---

## Development Workflow

### Initial Setup (First Time)

```bash
# 1. Clone repositories
git clone <backend-repo> fincode_app
git clone <frontend-repo> fincode_frontend
git clone <docker-repo> fincode_setup

# 2. Configure environment
cd fincode_app
cp .env.example .env
# Edit .env and add Fincode API keys

# 3. Start all services
cd ../fincode_setup
bin/setup_application up -d

# 4. Wait for services to be healthy
bin/status

# 5. Access application
# Frontend: http://localhost:3006
# Backend: http://localhost:3005
# Mailcatcher: http://localhost:10800
```

### Daily Development

```bash
# Start services
cd fincode_setup
bin/setup_application up -d

# Code changes will auto-reload
# - Backend: Rails auto-reload enabled
# - Frontend: Vite HMR (Hot Module Replacement)

# View logs if needed
bin/logs -f rails

# End of day - stop services
bin/stop_application
```

### Making Changes

```bash
# After changing Gemfile
docker exec -it fincode_rails bundle install
bin/setup_application restart

# After changing package.json
docker exec -it fincode_frontend-react-dev-1 npm install
cd ../fincode_frontend
docker compose -f docker-compose.dev.yml restart

# After changing database schema
docker exec -it fincode_rails bundle exec rails db:migrate

# After changing Docker configuration
bin/setup_application up -d --build
```

---

## Architecture Notes

- Scripts use `docker compose -f` to keep each project's configuration independent
- Each project can still be run independently if needed
- Volumes are used to persist data (MySQL, Redis, node_modules)
- The setup assumes all three repositories are siblings in the same parent directory
- Hot reload is enabled for both frontend and backend for faster development

---

## Contact & Support

If you encounter issues during setup, please:
1. Check the [Troubleshooting](#troubleshooting) section
2. View detailed logs: `bin/logs [service-name]`
3. Check container status: `bin/status`
4. Contact the team for support

---

**Happy coding! 🚀**
