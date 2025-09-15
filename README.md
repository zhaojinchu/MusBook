# MusBook

An end-to-end scheduling and communications platform that helps teachers and students coordinate lessons, share resources, and stay informed. MusBook combines Django, Channels, Celery, and Tailwind CSS to deliver real-time messaging, scheduling automation, and a role-aware dashboard experience.

## Table of Contents
- [Overview](#overview)
- [Feature Highlights](#feature-highlights)
  - [Role-aware accounts & security](#role-aware-accounts--security)
  - [Scheduling & calendar management](#scheduling--calendar-management)
  - [Communications & task management](#communications--task-management)
  - [Real-time & background processing](#real-time--background-processing)
  - [User interface & developer experience](#user-interface--developer-experience)
- [Architecture](#architecture)
- [Directory structure](#directory-structure)
- [Prerequisites](#prerequisites)
- [Configuration](#configuration)
- [Local development](#local-development)
  - [Run the Django application](#run-the-django-application)
  - [Start background workers](#start-background-workers)
  - [Tailwind CSS tooling](#tailwind-css-tooling)
- [Testing](#testing)
- [Production operations](#production-operations)
  - [Logs](#logs)
  - [Restart services](#restart-services)
  - [Reload systemd units](#reload-systemd-units)
  - [Services in use](#services-in-use)
- [Troubleshooting & tips](#troubleshooting--tips)

## Overview
MusBook provides a connected space for tuition centres or small teaching teams. Students can discover and manage lessons with teachers they are connected to, while teachers coordinate schedules, issue assignments, and track attendance from a unified dashboard. The platform embraces automation with background reminders, timezone-aware scheduling, and real-time notifications so both parties always have the latest information.

## Feature Highlights

### Role-aware accounts & security
- Custom `User` model with explicit student/teacher roles, timezone preferences, phone numbers, and profile metadata for each persona.
- Email-based two-factor authentication, password-reset tokens, and session-aware settings updates keep accounts secure while remaining accessible.
- Invitation flow allows teachers to connect with students before sharing calendars, messaging, or assignments, with dedicated student and teacher management views.

### Scheduling & calendar management
- Students and teachers can raise lesson requests, optionally specify recurring series, and approve or decline them with automatic lesson creation.
- Full rescheduling workflow records proposed changes, routes approvals to the other party, and tracks historical lessons that were moved.
- Attendance tracking tools show recent lessons for teachers and allow updating student presence inline.
- Calendar hub (FullCalendar) merges confirmed lessons with personal events (`CalendarEvent`) so users can block off other commitments without leaving the app.
- Timezone conversion middleware ensures every form submission and calendar view respects the viewer’s locale when creating or reviewing events.

### Communications & task management
- Real-time direct messaging between connected users runs over WebSockets with message persistence and optional read-only locks per relationship.
- Central notification centre streams in-app alerts for lesson requests, reschedules, attendance, assignments, and other important updates.
- Teachers can create assignments for specific students; students track progress and mark tasks complete when finished.
- Teacher notes provide a lightweight CRM to capture context about each student while staying attached to the corresponding invite relationship.

### Real-time & background processing
- Django Channels powers WebSocket routes for the messaging inbox and notification feed, delivered through the ASGI stack and served by Daphne or another ASGI server in production.
- Celery workers deliver scheduled reminder notifications (1 hour, 24 hours, or 30 minutes before lessons) and can be extended for additional asynchronous jobs.
- Redis (default broker/result backend) coordinates Celery tasks; Django’s in-memory channel layer is used for development, with Redis recommended for production deployments.

### User interface & developer experience
- Tailwind CSS (via `django-tailwind`) and Alpine.js drive a responsive dashboard shell with collapsible navigation, reusable components, and live message banners.
- Real-time notification dropdown fetches unread notifications over JSON endpoints and pushes new alerts instantly through a WebSocket consumer.
- `django-browser-reload` accelerates local development with automatic page refreshes during template or static file edits.

## Architecture
- **Backend:** Django 5.x project with dedicated apps for users, scheduling, communications, and dashboard presentation.
- **Real-time layer:** Django Channels and Daphne (ASGI) expose WebSocket consumers; Gunicorn handles synchronous HTTP traffic in production.
- **Task queue:** Celery with Redis powers reminder scheduling (`communications.tasks.check_scheduled_notifications`).
- **Database:** SQLite is bundled for development; switch to PostgreSQL or another Django-compatible database for production.
- **Frontend tooling:** Tailwind CSS compiled through the `theme` app, plus Alpine.js for interactive UI elements.

## Directory structure
```
cs_nea/
├── cs_nea/                 # Project configuration (settings, URLs, ASGI/WSGI, Celery, middleware)
├── communications/         # Notifications, messaging, assignments, Celery tasks, and WebSocket consumers
├── scheduling/             # Lesson requests, calendar views, rescheduling, attendance, and calendar event APIs
├── users/                  # Custom user model, authentication, signup flows, settings, invites, and notes
├── dashboard/              # Role-aware dashboard entry points and shared context helpers
├── theme/                  # Tailwind app with static_src (package.json, tailwind.config.js) and compiled assets
├── static/                 # Global static assets collected for deployment
├── staticfiles/            # Collected static files (created by `collectstatic`)
├── templates/              # Project-wide templates used by all apps
├── requirements.txt        # Python dependencies
└── manage.py
```

## Prerequisites
- Python 3.11+ (Django 5 requires Python 3.10 or later).
- Node.js 18+ (for Tailwind CSS compilation via `django-tailwind`).
- Redis server (recommended for Celery broker/result backend and Channels in production).
- A relational database. SQLite ships with the repository; install PostgreSQL/MySQL for multi-user production deployments.
- SMTP credentials for sending two-factor codes and password reset emails. Development defaults target Gmail but should be replaced with environment variables.

## Configuration
1. Create a virtual environment and install Python dependencies:
   ```bash
   python -m venv .venv
   source .venv/bin/activate
   pip install --upgrade pip
   pip install -r requirements.txt
   ```
2. Provide configuration through environment variables (recommended) or update `cs_nea/cs_nea/settings.py`. Common settings include:
   ```bash
   export DJANGO_SECRET_KEY="replace-me"
   export DJANGO_DEBUG=1
   export DJANGO_ALLOWED_HOSTS="127.0.0.1,localhost"
   export EMAIL_HOST="smtp.example.com"
   export EMAIL_HOST_USER="no-reply@example.com"
   export EMAIL_HOST_PASSWORD="app-password"
   export EMAIL_PORT=587
   export EMAIL_USE_TLS=1
   export CELERY_BROKER_URL="redis://localhost:6379/0"
   export CELERY_RESULT_BACKEND="redis://localhost:6379/0"
   ```
   > ⚠️ Never commit real secrets. The sample Gmail credentials in `settings.py` should be replaced before deploying.
3. Adjust `ALLOWED_HOSTS`, database connection, and channel layer configuration for your target environment.

## Local development
1. Apply database migrations and create an admin user:
   ```bash
   python manage.py migrate
   python manage.py createsuperuser
   ```
2. Optionally load any fixtures or seed data relevant to your deployment.

### Run the Django application
Start the HTTP + WebSocket development server (Channels runs under `runserver`):
```bash
python manage.py runserver
```
Access the site at <http://127.0.0.1:8000/>.

### Start background workers
Celery powers automated notifications. Run both a worker and beat scheduler in separate terminals after Redis is available:
```bash
# Terminal 1: Celery worker
celery -A cs_nea worker -l info

# Terminal 2: Celery beat (scheduled jobs)
celery -A cs_nea beat -l info
```
`check_scheduled_notifications` polls upcoming lessons every five seconds and dispatches reminders based on each user’s notification preferences.

### Tailwind CSS tooling
The `theme` app contains Tailwind sources.
```bash
# Install JS dependencies once
python manage.py tailwind install

# Watch CSS changes during development
python manage.py tailwind start

# Build production CSS (before collectstatic/deploy)
python manage.py tailwind build
```
To ship static assets, run `python manage.py collectstatic`.

## Testing
Run the Django test suite:
```bash
python manage.py test
```
Add additional unit tests for new functionality inside each app’s `tests.py` or by creating dedicated test modules.

## Production operations
The project has been deployed with systemd-managed services. The following commands assume you have corresponding unit files for Daphne, Gunicorn, Celery, and Nginx.

### Logs
```bash
journalctl -u daphne
```
Adjust the unit name to inspect other services (e.g., `gunicorn`, `celery`, `celerybeat`, `nginx`).

### Restart services
```bash
sudo systemctl restart daphne
sudo systemctl restart nginx
sudo systemctl restart gunicorn
sudo systemctl restart celery
sudo systemctl restart celerybeat
```
Restart the relevant service after deploying new code or configuration.

### Reload systemd units
```bash
sudo systemctl daemon-reload
```
Run this command whenever you change systemd unit files.

### Services in use
- **Redis** – Celery broker/result store and recommended channel layer backend.
- **Gunicorn** – WSGI HTTP server.
- **Nginx** – Reverse proxy/static file server.
- **Celery** – Background task processor (paired with Celery Beat for schedules).
- **Daphne** – ASGI server for Channels/WebSockets.

## Troubleshooting & tips
- Ensure Redis is running before starting Celery; otherwise workers exit immediately.
- Use `python manage.py check --deploy` to surface configuration warnings before going live.
- For production WebSockets, configure `CHANNEL_LAYERS` to use Redis instead of the in-memory layer shipped for development.
- Keep user timezones accurate—lesson forms convert submitted datetimes into UTC based on the client timezone hidden field.
- If notification emails do not arrive, verify SMTP credentials and confirm that less secure app access (or app passwords) are enabled for the provider.
