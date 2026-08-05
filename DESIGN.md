# Embeddable Widget & Lead-Capture Platform

## 1. Problem

This platform allows customers to create embeddable widgets such as signup
forms, contact forms, and call-to-action widgets.

A customer receives a single `<script>` snippet that can be added to any
website. Visitors can then interact with the widget and submit their
information.

The backend is responsible for validating, protecting, enriching, storing,
and exposing these submissions through a dashboard API.

## 2. Goals

- Allow authenticated customers to create and manage widgets.
- Keep each customer's widgets and submissions isolated.
- Generate a one-line embed script for each widget.
- Serve widget configuration through a public cached endpoint.
- Accept submissions from external websites using CORS.
- Validate and reject invalid or oversized requests.
- Protect the public endpoint using rate limiting and spam prevention.
- Enrich submissions with location data using a geo-provider fallback chain.
- Store valid submissions safely.
- Ensure email/webhook failures do not prevent successful submissions.
- Provide dashboard APIs for submissions and basic analytics.

## 3. Data Model

### Users

Stores authenticated widget owners.

- `id` — unique identifier
- `email` — unique login email
- `password_hash` — securely hashed password
- `created_at` — account creation timestamp

### Widgets

Stores widget configurations owned by a user.

- `id` — unique widget identifier
- `user_id` — owner of the widget
- `type` — widget type such as signup, CTA, or popover
- `title` — widget title
- `description` — widget description
- `fields` — JSONB configuration for form fields
- `button_text` — submission button text
- `display_options` — JSONB display configuration
- `created_at` — widget creation timestamp

### Submissions

Stores visitor submissions.

- `id` — unique submission identifier
- `widget_id` — widget that received the submission
- `user_id` — owner of the widget
- `data` — JSONB submitted form data
- `ip_address` — visitor IP address
- `country` — enriched country
- `city` — enriched city
- `created_at` — submission timestamp

### Relationships

- One user can own many widgets.
- One widget can have many submissions.
- Every widget belongs to exactly one user.
- Every submission belongs to exactly one widget and its owner.
- All authenticated queries must enforce user ownership to maintain tenant isolation.

## 4. API Design

### Authentication

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/auth/register` | Create a customer account |
| POST | `/api/auth/login` | Authenticate a customer |

Authenticated endpoints use a JWT token.

### Widget Management API

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/widgets` | Create a widget |
| GET | `/api/widgets` | List the owner's widgets |
| GET | `/api/widgets/:id` | Get one owned widget |
| PUT | `/api/widgets/:id` | Update an owned widget |
| DELETE | `/api/widgets/:id` | Delete an owned widget |
| GET | `/api/widgets/:id/embed` | Generate the embed snippet |

All widget management endpoints require authentication and enforce
tenant isolation.

### Public Widget Delivery

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/widget.js?id=:id` | Serve the widget JavaScript |
| GET | `/api/public/widgets/:id/config` | Return public widget configuration |

The configuration endpoint is public, supports CORS, and uses HTTP caching.

The widget JavaScript is served as a versioned or cache-busted asset.

### Public Submission API

| Method | Endpoint | Purpose |
|---|---|---|
| OPTIONS | `/api/submissions` | Handle CORS preflight |
| POST | `/api/submissions` | Receive a visitor submission |

The submission endpoint:

1. Validates the request.
2. Checks CORS.
3. Applies rate limiting.
4. Checks the spam-prevention mechanism.
5. Enriches the visitor IP using the geo-provider fallback chain.
6. Stores the submission.
7. Triggers the non-critical email/webhook side effect.

A failure in geo enrichment or the side effect must not cause a valid
submission to fail.

### Dashboard API

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/dashboard/submissions` | View owner's submissions |
| GET | `/api/dashboard/stats` | View basic submission statistics |
| GET | `/api/dashboard/widgets/:id/stats` | View statistics for one widget |

Dashboard endpoints require authentication and only return data belonging
to the authenticated owner.

### Error Responses

The API will use appropriate HTTP status codes:

- `200` — successful request
- `201` — resource created
- `400` — invalid request
- `401` — authentication required/invalid
- `403` — access denied
- `404` — resource not found
- `413` — payload too large
- `429` — rate limit exceeded
- `500` — unexpected server error

## 5. Embed Flow

The widget follows this flow:

### Step 1 — Widget Creation

The authenticated customer creates a widget through:

POST /api/widgets

The backend stores the widget configuration and returns a unique widget ID.

### Step 2 — Embed Snippet

The customer requests the embed snippet:

GET /api/widgets/:id/embed

The API returns a script similar to:

<script src="http://localhost:5000/widget.js?id=abc123"></script>

The customer copies this single line into their website.

### Step 3 — Widget Loading

The customer's website loads `widget.js`.

The script:

1. Reads the widget ID from the URL.
2. Requests the widget configuration.
3. Receives the public configuration.
4. Creates the widget UI.
5. Renders the form on the customer's page.

### Step 4 — Visitor Submission

When a visitor submits the widget:

POST /api/submissions

The request contains:

- Widget ID
- Form data
- Honeypot/spam-control field

The backend never trusts the submitted data.

### Step 5 — Submission Processing

The backend processes the request in this order:

Request
→ CORS
→ Payload validation
→ Rate limiting
→ Spam protection
→ Geo enrichment
→ Database storage
→ Non-critical side effect

### Step 6 — Graceful Failure

Geo enrichment uses a fallback chain:

Provider A
→ failure
→ Provider B
→ failure
→ store submission without geo data

The submission must still succeed if both providers are unavailable.

Similarly, if the email/webhook side effect fails, the submission must remain stored
and the API must still return success.

## 6. Security Rules

### Authentication

Authenticated widget-management and dashboard endpoints require a valid JWT.

### Tenant Isolation

Every authenticated database query must be scoped to the authenticated user's ID.

A customer must never be able to read, update, or delete another customer's
widgets or submissions.

### Input Validation

All public input is validated before it reaches business logic.

Invalid or malformed input returns a clean 4xx JSON response.

Oversized payloads are rejected.

### CORS

The public widget endpoints support cross-origin browser requests.

CORS preflight requests using OPTIONS are handled correctly.

Only configured/allowed origins will be accepted where appropriate.

### Rate Limiting

Public submission requests are rate-limited by IP and/or widget.

Requests exceeding the configured limit return:

429 Too Many Requests

### Spam Protection

The widget includes a honeypot field.

Normal visitors leave the field empty.

If the field is filled, the request is treated as spam and rejected or silently
dropped.

### Secrets

Secrets and credentials are stored in environment variables.

No API keys, passwords, tokens, or SMTP credentials are committed to Git.

The `.env` file is ignored by Git, while `.env.example` contains safe placeholders.

## 7. Architecture

The application follows a layered architecture so that HTTP handling,
business logic, and database access remain separate.

```text
Client / Widget
      │
      ▼
   Routes
      │
      ▼
 Controllers
      │
      ▼
   Services
      │
      ▼
 Repositories
      │
      ▼
 PostgreSQL
