# AI Voice Receptionist

An AI-powered phone receptionist that handles incoming calls, answers common practice questions, checks appointment availability, books appointments on Google Calendar, sends SMS confirmations, and records call activity in a Django application.

<img width="993" height="600" alt="image" src="https://github.com/user-attachments/assets/068e0c85-9d3f-407f-a17f-e63d0307c07a" />


## Overview

This project provides a voice-first appointment assistant for practices that need an automated front desk for incoming calls.

The application uses **Vapi as the voice interface**, while the conversational intelligence and business actions are handled by a Django backend. Incoming caller messages are sent to an OpenAI-compatible chat endpoint, where **Claude** interprets the conversation and decides whether to use tools such as calendar availability, appointment booking, SMS confirmation, call logging, or call termination.

The system is currently configured around a dental-practice receptionist scenario, but the workflow and integrations can be adapted for other appointment-based businesses.

---
<img width="1104" height="606" alt="image" src="https://github.com/user-attachments/assets/0411d526-aa45-42c1-bc76-e977a8b5c68d" />

## What It Does

- Handles incoming voice conversations through Vapi
- Converts Vapi's OpenAI-compatible messages into Claude messages
- Uses Claude as the conversational decision engine
- Checks Google Calendar availability
- Books appointments directly on Google Calendar
- Prevents duplicate booking of an occupied slot
- Sends appointment confirmations through Twilio SMS
- Logs call summaries, caller numbers, booking status, and duration
- Provides a Django admin interface for call and appointment records
- Supports appointment types for new patients, returning patients, and emergencies
- Ends calls through an explicit AI tool when the conversation is complete
- Runs with PostgreSQL through Docker Compose

---

## Architecture

```text
                    ┌───────────────────┐
                    │   Incoming Call   │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │       Vapi        │
                    │  Voice / STT/TTS  │
                    └─────────┬─────────┘
                              │
                   OpenAI-compatible API
                              │
                              ▼
                    ┌───────────────────┐
                    │ Django + FastAPI- │
                    │ compatible LLM API│
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │   Claude Agent    │
                    │ Conversation +    │
                    │ Tool Selection    │
                    └─────────┬─────────┘
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
            ▼                 ▼                 ▼
     Google Calendar       Twilio           PostgreSQL
     Availability /       SMS Confirm.      Call Logs /
       Booking                              Appointments
            │                 │                 │
            └─────────────────┼─────────────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │   Response to     │
                    │       Vapi        │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │      Caller       │
                    └───────────────────┘
```

### Responsibility of Each Service

| Component | Responsibility |
|---|---|
| **Vapi** | Voice transport, speech-to-text, text-to-speech, phone-call interface |
| **Django** | Application backend, API endpoints, administration, database integration |
| **Claude** | Conversation handling and tool selection |
| **Google Calendar** | Appointment availability and calendar events |
| **Twilio** | SMS confirmations |
| **PostgreSQL** | Call and appointment persistence |

Vapi does not contain the project's receptionist business logic. The Django application sends the conversation to Claude, executes the selected tools, and returns the generated response.

---

## Conversation Workflow

```text
Caller
  │
  ▼
Vapi receives speech
  │
  ▼
Speech converted to text
  │
  ▼
Django chat endpoint
  │
  ▼
Claude + receptionist instructions
  │
  ├── Need available times?
  │       └── Google Calendar
  │
  ├── Book an appointment?
  │       └── Google Calendar + Database
  │
  ├── Send confirmation?
  │       └── Twilio SMS
  │
  ├── Log call?
  │       └── PostgreSQL
  │
  └── End conversation?
          └── Vapi end-call signal
  │
  ▼
Text response
  │
  ▼
Vapi text-to-speech
  │
  ▼
Caller hears response
```

---

## AI Agent

The receptionist agent is implemented in:

```text
calls/agent/
├── receptionist.py
├── prompts.py
└── tools.py
```

The agent uses Anthropic's Claude API and supports an agentic tool-calling loop.

When Claude requests a tool, Django executes the corresponding service and sends the result back to Claude. Claude can continue calling tools until it has enough information to respond to the caller.

### Available Tools

#### `check_availability`

Checks Google Calendar for available appointment slots on a requested date.

The current calendar search covers:

```text
08:00 - 18:00
```

with 30-minute slot increments.

#### `book_appointment`

Creates a Google Calendar event and stores the appointment in PostgreSQL.

Appointment duration is currently:

- **New patient:** 60 minutes
- **Returning patient:** 30 minutes
- **Emergency:** handled through the appointment-type field and current booking duration logic

Before creating a new event, the application checks the requested time to avoid creating a duplicate calendar booking.

#### `send_sms`

Sends an SMS confirmation through Twilio.

The AI receptionist can use the caller's phone number by default and ask whether the caller wants the confirmation sent somewhere else.

SMS execution is performed in a background thread during the agentic flow so the voice response does not unnecessarily wait for the Twilio request.

#### `log_call`

Stores call information in the Django database, including:

- Caller phone number
- Call summary
- Whether an appointment was booked
- Call duration

#### `end_call`

Signals that the conversation is complete and allows the Vapi response to indicate that the call should end.

---

## Receptionist Behavior

The current system prompt defines a concise phone-based conversation style.

The receptionist is configured to:

- Keep responses short
- Avoid long explanations during calls
- Confirm appointment date and time before booking
- Suggest nearby available slots when requested availability is unavailable
- Handle caller silence with a follow-up prompt
- Confirm appointments verbally before sending SMS
- End completed conversations explicitly
- Answer configured practice information questions

The current sample practice configuration includes:

- Monday–Friday: 8 AM–6 PM
- Saturday: 9 AM–2 PM
- Sunday: Closed
- General dentistry
- Cleanings
- Whitening
- Emergency care

These details are defined in `calls/agent/prompts.py` and should be customized before using the application for a real business.

---

## API Endpoints

The Django application exposes the following Vapi-facing endpoints.

### Chat Completions

```http
POST /api/vapi/chat/completions/
```

This is the main custom LLM endpoint.

It:

1. Receives OpenAI-compatible messages from Vapi.
2. Extracts caller information when available.
3. Converts the messages into Claude's expected format.
4. Runs the Claude agentic tool loop.
5. Executes calendar, SMS, database, and call-control actions.
6. Returns an OpenAI-compatible chat completion response.

The endpoint also supports the streaming response format expected by Vapi.

### Vapi Webhook

```http
POST /api/vapi/webhook/
```

The webhook receives Vapi call events.

For end-of-call reports, the application extracts information such as:

- Caller phone number
- Call summary
- Transcript fallback
- Duration

and stores the call in the database.

### Webhook Health Check

```http
GET /api/vapi/webhook/
```

Returns a simple service-status response.

---

## Data Models

The Django application currently defines two main models.

### CallLog

Stores:

- Caller phone
- Call summary
- Appointment-booked flag
- Call duration
- Creation timestamp

### Appointment

Stores:

- Patient name
- Patient phone
- Appointment type
- Date
- Time
- Duration
- Google Calendar event ID
- Creation timestamp

Appointment types currently include:

```text
new_patient
returning
emergency
```

---

## Django Admin

The Django admin interface provides access to application records.

Open:

```text
http://localhost:8000/admin/
```

After creating a superuser, administrators can inspect:

- Call logs
- Appointment records
- Caller information
- Booking status
- Appointment dates and times
- Appointment types

The admin registration is implemented in:

```text
calls/admin.py
```

---

## Tech Stack

| Technology | Purpose |
|---|---|
| **Python** | Application runtime |
| **Django 5.1** | Backend framework and administration |
| **Anthropic Claude** | Conversational AI and tool selection |
| **Vapi** | Voice calling, speech-to-text, and text-to-speech integration |
| **Google Calendar API** | Availability checks and appointment booking |
| **Twilio** | SMS confirmations |
| **PostgreSQL** | Application database |
| **python-dotenv** | Environment configuration |
| **Gunicorn** | Production WSGI server dependency |
| **Docker** | Containerized development/deployment |

---

## Project Structure

```text
ai-voice-agent-main/
│
├── .github/
│   └── FUNDING.yml
│
├── calls/
│   ├── migrations/
│   │   ├── 0001_initial.py
│   │   └── 0002_appointment.py
│   │
│   ├── agent/
│   │   ├── prompts.py
│   │   ├── receptionist.py
│   │   └── tools.py
│   │
│   ├── services/
│   │   ├── calendar.py
│   │   ├── database.py
│   │   └── sms.py
│   │
│   ├── admin.py
│   ├── apps.py
│   ├── models.py
│   ├── urls.py
│   └── views.py
│
├── receptionist_project/
│   ├── asgi.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
│
├── .env.example
├── Dockerfile
├── docker-compose.yml
├── manage.py
├── requirements.txt
└── README.md
```

---

## Requirements

You need:

- Python 3.11 recommended
- Docker and Docker Compose
- Anthropic API access
- Vapi account and configuration
- Google Cloud project with Calendar API enabled
- Google Calendar service-account credentials
- Twilio account and phone number
- A PostgreSQL-compatible environment

The Python dependencies are defined in:

```text
requirements.txt
```

---

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/hammadproject/ai-voice-agent-main.git
cd ai-voice-agent-main
```

### 2. Create the Environment File

```bash
cp .env.example .env
```

On Windows, copy `.env.example` to `.env` manually if the `cp` command is unavailable.

### 3. Generate a Django Secret Key

Generate a secure secret:

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(50))"
```

Add the generated value to:

```env
SECRET_KEY=your-generated-secret-key
```

### 4. Configure Environment Variables

Update `.env` with your own credentials.

Example configuration:

```env
SECRET_KEY=your-generated-secret-key
DEBUG=True
ALLOWED_HOSTS=localhost,127.0.0.1,0.0.0.0

DB_NAME=ai_receptionist
DB_USER=postgres
DB_PASSWORD=postgres
DB_HOST=db
DB_PORT=5432

ANTHROPIC_API_KEY=your-anthropic-api-key

VAPI_API_KEY=your-vapi-api-key

GOOGLE_CALENDAR_CREDENTIALS=google-calendar-credentials.json
GOOGLE_CALENDAR_ID=your-calendar-id

TWILIO_ACCOUNT_SID=your-twilio-account-sid
TWILIO_AUTH_TOKEN=your-twilio-auth-token
TWILIO_PHONE_NUMBER=your-twilio-phone-number
```

Never use the example values as production credentials.

---

## Google Calendar Setup

The application uses a Google service account to access Calendar.

### 1. Create a Google Cloud Project

Create or select a project in Google Cloud.

### 2. Enable Google Calendar API

Enable:

```text
Google Calendar API
```

### 3. Create a Service Account

Create a service account and generate a JSON key.

Store the credential file securely and configure:

```env
GOOGLE_CALENDAR_CREDENTIALS=google-calendar-credentials.json
```

Do not commit this JSON file.

### 4. Find the Calendar ID

Open Google Calendar settings and locate the ID of the calendar that the receptionist should manage.

Configure:

```env
GOOGLE_CALENDAR_ID=your-calendar-id
```

### 5. Share the Calendar

Share the target calendar with the service account email and grant the permission required to create and modify events.

---

## Twilio Setup

Create/configure a Twilio account and obtain:

- Account SID
- Auth token
- SMS-capable phone number

Add them to `.env`:

```env
TWILIO_ACCOUNT_SID=your-account-sid
TWILIO_AUTH_TOKEN=your-auth-token
TWILIO_PHONE_NUMBER=your-twilio-number
```

The application uses Twilio to send appointment confirmation messages.

---

## Anthropic Setup

The receptionist uses Claude through the Anthropic Python SDK.

Configure:

```env
ANTHROPIC_API_KEY=your-anthropic-api-key
```

The current agent implementation uses the Claude Sonnet model configured in `calls/agent/receptionist.py`.

Keep the API key server-side and never expose it to the client.

---

## Vapi Integration

Vapi is used as the voice layer around the Django custom LLM endpoint.

### Custom LLM URL

Configure Vapi to send conversation requests to:

```text
https://your-domain.example.com/api/vapi/chat/completions/
```

### Server/Webhook URL

Configure Vapi's server URL for call events:

```text
https://your-domain.example.com/api/vapi/webhook/
```

The application expects Vapi to provide conversation messages in an OpenAI-compatible format.

### Local Development with a Tunnel

For local testing, expose port `8000` through a secure public tunnel such as ngrok or another tunneling service.

Example:

```bash
ngrok http 8000
```

Then use the generated HTTPS address in the Vapi configuration.

---

## Running with Docker Compose

The repository includes PostgreSQL and Django services in `docker-compose.yml`.

Start the application with:

```bash
docker compose up --build
```

The configuration provides:

- PostgreSQL on port `5432`
- Django on port `8000`

The web container automatically runs migrations before starting the Django development server.

Open:

```text
http://localhost:8000
```

---

## Creating an Admin User

After the containers are running:

```bash
docker compose exec web python manage.py createsuperuser
```

Follow the prompts to create the administrator account.

Then open:

```text
http://localhost:8000/admin/
```

---

## Running Without Docker

Create and activate a Python virtual environment:

```bash
python -m venv .venv
```

Linux/macOS:

```bash
source .venv/bin/activate
```

Windows:

```powershell
.venv\Scripts\activate
```

Install dependencies:

```bash
pip install -r requirements.txt
```

Configure PostgreSQL and the `.env` file, then run:

```bash
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```

The application will be available at:

```text
http://localhost:8000
```

---

## Configuration & Customization

### Change Practice Information

The receptionist's business information is defined in:

```text
calls/agent/prompts.py
```

Customize:

- Business/practice name
- Receptionist name
- Opening hours
- Address
- Services
- Appointment durations
- Conversation style
- Caller-handling rules

This is the primary place to adapt the sample receptionist to another business.

### Change Appointment Rules

Appointment behavior is implemented in:

```text
calls/services/calendar.py
```

The current implementation uses:

- 30-minute slots
- 8 AM–6 PM calendar search window
- 60-minute duration for new patients
- 30-minute duration for returning patients

Adjust these rules to match the business's scheduling requirements.

### Customize AI Tools

Tool definitions are located in:

```text
calls/agent/tools.py
```

You can add new actions by:

1. Defining the tool schema.
2. Implementing its service logic.
3. Adding the tool to the agent's execution logic.
4. Updating the system prompt if the AI should use it.

### Customize SMS

Twilio functionality is isolated in:

```text
calls/services/sms.py
```

This makes the messaging integration easier to replace or extend.

### Customize Database Behavior

Database logging is handled in:

```text
calls/services/database.py
```

The Django models are defined in:

```text
calls/models.py
```

---

## Security

This application handles phone numbers, appointment information, call summaries, and potentially sensitive healthcare-related information. Production deployments should apply appropriate privacy and security controls.

### Never commit

- `.env`
- Anthropic API keys
- Vapi API keys
- Twilio credentials
- Google Calendar service-account JSON files
- Production database passwords
- Caller information
- Private call transcripts
- Other production secrets

### Recommended practices

- Generate a strong Django `SECRET_KEY`.
- Set `DEBUG=False` in production.
- Restrict `ALLOWED_HOSTS`.
- Use secure production database credentials.
- Keep Google credentials outside source control.
- Restrict API access where appropriate.
- Use HTTPS for public Vapi endpoints.
- Review webhook authentication and request validation before exposing endpoints publicly.
- Protect Django admin with strong credentials and appropriate access controls.
- Avoid logging sensitive personal or healthcare information unnecessarily.
- Apply appropriate data-retention and privacy policies for call and appointment records.

The current development configuration contains default database values and a development secret fallback. These must be replaced for production.

---

## Production Considerations

Before deploying publicly:

1. Set `DEBUG=False`.
2. Configure production `ALLOWED_HOSTS`.
3. Use a managed or properly secured PostgreSQL database.
4. Store secrets in a secure secrets manager or deployment environment.
5. Protect the Vapi webhook and custom LLM endpoints.
6. Configure HTTPS.
7. Review Google Calendar permissions.
8. Configure Twilio production credentials.
9. Review logging to avoid exposing sensitive information.
10. Test appointment conflict handling.
11. Test SMS failures and API failures.
12. Review privacy and regulatory requirements applicable to your business.

The included `Dockerfile` uses Django's development server for its container command. For a production deployment, use a production-grade WSGI deployment configuration rather than relying on `runserver`.

---

## Troubleshooting

### Claude API Errors

Check:

```env
ANTHROPIC_API_KEY=
```

and verify the API key is valid and available to the Django process.

### Calendar Errors

Check:

```env
GOOGLE_CALENDAR_CREDENTIALS=
GOOGLE_CALENDAR_ID=
```

Also verify that the calendar has been shared with the service account.

### SMS Errors

Check:

```env
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_PHONE_NUMBER=
```

Make sure the Twilio number can send messages to the destination number.

### Database Connection Errors

When using Docker Compose, verify:

```env
DB_HOST=db
DB_PORT=5432
DB_NAME=ai_receptionist
DB_USER=postgres
DB_PASSWORD=postgres
```

Also check the PostgreSQL container health:

```bash
docker compose ps
```

### Vapi Cannot Reach the API

For local development, the Django server must be publicly reachable by Vapi. Use an HTTPS tunnel or deploy the API to a public server.

Verify the configured URLs:

```text
/api/vapi/chat/completions/
/api/vapi/webhook/
```

---

## Future Improvements

Potential enhancements include:

- Add stronger authentication for public Vapi endpoints.
- Add webhook signature verification.
- Add configurable business profiles instead of hard-coded practice information.
- Add support for multiple calendars or locations.
- Add rescheduling and appointment cancellation.
- Add automated appointment reminders.
- Add multilingual voice conversations.
- Add richer call analytics.
- Add persistent conversation/session tracking.
- Add retry and timeout handling for external services.
- Replace background threads with a production task queue for reliable asynchronous work.
- Add comprehensive integration tests for calendar, SMS, and Vapi flows.
- Add production-grade observability and alerting.
- Add configurable business hours and appointment durations through the database or admin interface.

---

## Contributing

Contributions and improvements are welcome.

Before submitting changes:

1. Keep changes focused.
2. Test affected functionality locally.
3. Update migrations when models change.
4. Update documentation when configuration changes.
5. Never commit credentials or private caller data.

---

## License

This repository contains an **MIT License**.

The repository's license file includes a copyright notice that is legally part of the MIT license text. If you redistribute or modify this project, retain the required license notice according to the license terms.

See the `LICENSE` file for the complete license text.

---

## Summary

This project provides a practical foundation for an AI-powered phone receptionist.

It combines voice transport, conversational AI, calendar automation, SMS communication, and persistent call records into one Django-based application:

```text
Voice Call
   ↓
Vapi
   ↓
Django
   ↓
Claude
   ↓
Tools
 ┌─ Google Calendar
 ├─ Twilio
 ├─ PostgreSQL
 └─ Call Control
```

The architecture is intentionally modular so the receptionist behavior, scheduling rules, business information, and external integrations can be adapted to different appointment-based workflows.
