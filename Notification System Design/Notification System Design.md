# Notification System SaaS

A complete end-to-end design for a scalable, multi-tenant notification platform that supports email, SMS, push, in-app, webhook, and third-party messaging channels.

This document is written as a full system design README. It covers product requirements, architecture, APIs, database schema, event flow, queues, retries, rate limiting, observability, security, deployment, testing, and operations.

---

## Table of Contents

1. [Overview](#overview)
2. [Product Vision](#product-vision)
3. [Core Features](#core-features)
4. [System Requirements](#system-requirements)
5. [High-Level Architecture](#high-level-architecture)
6. [Core Concepts](#core-concepts)
7. [Notification Lifecycle](#notification-lifecycle)
8. [Service Architecture](#service-architecture)
9. [Data Model](#data-model)
10. [Queue and Event Design](#queue-and-event-design)
11. [API Design](#api-design)
12. [Template System](#template-system)
13. [User Preferences and Consent](#user-preferences-and-consent)
14. [Channel Delivery Design](#channel-delivery-design)
15. [Provider Adapter Design](#provider-adapter-design)
16. [Scheduling, Batching, and Digests](#scheduling-batching-and-digests)
17. [Retries, Failures, and Idempotency](#retries-failures-and-idempotency)
18. [Rate Limiting and Throttling](#rate-limiting-and-throttling)
19. [Multitenancy](#multitenancy)
20. [Security](#security)
21. [Observability](#observability)
22. [Scalability](#scalability)
23. [Deployment Architecture](#deployment-architecture)
24. [Local Development](#local-development)
25. [Testing Strategy](#testing-strategy)
26. [Operational Runbooks](#operational-runbooks)
27. [Roadmap](#roadmap)

---

# Overview

The Notification System SaaS is a platform that allows businesses to send notifications to their users across multiple channels.

Example notification types:

* Password reset email
* Order shipped SMS
* Payment failed email
* New message push notification
* Weekly activity digest
* In-app alert
* Webhook callback to customer systems
* Admin broadcast announcement
* Marketing campaign message
* Compliance notification

The system is designed like a real SaaS product. Multiple companies, called tenants, can onboard to the platform. Each tenant can configure templates, channels, providers, rate limits, preferences, and analytics.

---

# Product Vision

The goal is to build a reliable notification infrastructure that product teams can use without worrying about:

* Queue management
* Provider failures
* Retry logic
* Rate limiting
* Template rendering
* User preferences
* Consent compliance
* Channel fallback
* Delivery tracking
* Multitenant isolation
* Notification analytics

Instead of every product team building their own email, SMS, push, and webhook logic, they call one central API.

Example:

```http
POST /v1/notifications/send
```

The platform decides:

1. Whether the user is eligible to receive the message
2. Which channels are enabled
3. Which template version to use
4. Which provider should deliver it
5. Whether the message should be sent now or later
6. How to retry if the provider fails
7. How to track final delivery status

---

# Core Features

## Functional Features

* Send single notification
* Send bulk notifications
* Send scheduled notifications
* Send recurring notifications
* Send transactional notifications
* Send marketing notifications
* Support multiple channels:

  * Email
  * SMS
  * Push
  * In-app
  * Webhook
  * Slack or Microsoft Teams integration
* Template management
* Template versioning
* User preferences
* Unsubscribe handling
* Tenant-level configuration
* Provider configuration
* Delivery status tracking
* Retry handling
* Dead letter queue handling
* Notification logs
* Audit logs
* Analytics dashboard
* Webhook event callbacks

## SaaS Features

* Tenant onboarding
* API keys per tenant
* Role-based access control
* Usage limits
* Billing usage counters
* Tenant-specific branding
* Tenant-specific providers
* Tenant-specific notification categories
* Tenant-specific rate limits
* Tenant isolation
* Admin dashboard

---

# System Requirements

## Functional Requirements

The system must allow tenants to:

1. Create notification templates.
2. Configure delivery channels.
3. Configure providers such as SendGrid, Twilio, Firebase, APNs, or custom webhooks.
4. Send notifications through an API.
5. Track delivery status.
6. Retry failed deliveries.
7. Respect user notification preferences.
8. Support localization.
9. Support notification scheduling.
10. Support high-volume bulk delivery.

## Non-Functional Requirements

| Requirement      | Target                                              |
| ---------------- | --------------------------------------------------- |
| Availability     | 99.9% or higher                                     |
| API latency      | P95 under 200 ms for accepted notification requests |
| Delivery latency | P95 under 5 seconds for transactional notifications |
| Durability       | No accepted notification should be lost             |
| Scalability      | Millions of notifications per day                   |
| Multitenancy     | Strict logical isolation                            |
| Security         | Encrypted secrets, signed webhooks, RBAC            |
| Observability    | Metrics, logs, traces, alerting                     |
| Extensibility    | Easy to add new channels and providers              |
| Compliance       | GDPR, CAN-SPAM, TCPA-aware design                   |

---

# High-Level Architecture

```mermaid
flowchart TD
    Client[Customer Application] --> API[Notification API Gateway]

    API --> Auth[Auth and Tenant Service]
    API --> RequestSvc[Notification Request Service]

    RequestSvc --> PrefSvc[Preference Service]
    RequestSvc --> TemplateSvc[Template Service]
    RequestSvc --> Scheduler[Scheduler Service]
    RequestSvc --> Queue[(Message Broker)]

    Scheduler --> Queue

    Queue --> Orchestrator[Notification Orchestrator]

    Orchestrator --> Renderer[Template Renderer]
    Orchestrator --> Rules[Rules and Policy Engine]
    Orchestrator --> DeliveryQueue[(Channel Delivery Queues)]

    DeliveryQueue --> EmailWorker[Email Worker]
    DeliveryQueue --> SMSWorker[SMS Worker]
    DeliveryQueue --> PushWorker[Push Worker]
    DeliveryQueue --> InAppWorker[In-App Worker]
    DeliveryQueue --> WebhookWorker[Webhook Worker]

    EmailWorker --> EmailProvider[SendGrid / SES / Mailgun]
    SMSWorker --> SMSProvider[Twilio / MessageBird]
    PushWorker --> PushProvider[FCM / APNs]
    WebhookWorker --> CustomerWebhook[Customer Webhook Endpoint]

    EmailProvider --> WebhookReceiver[Provider Webhook Receiver]
    SMSProvider --> WebhookReceiver
    PushProvider --> WebhookReceiver

    WebhookReceiver --> StatusSvc[Delivery Status Service]

    StatusSvc --> DB[(Primary Database)]
    RequestSvc --> DB
    TemplateSvc --> DB
    PrefSvc --> DB
    Orchestrator --> DB

    StatusSvc --> Analytics[(Analytics Store)]
    StatusSvc --> TenantCallback[Tenant Callback Webhooks]
```

---

# Core Concepts

## Tenant

A tenant represents a customer company using the SaaS platform.

Example tenants:

* Acme Inc.
* ShopNow
* FinPay
* HealthTrack

Each tenant has:

* API keys
* Users
* Templates
* Provider settings
* Branding
* Usage limits
* Rate limits
* Notification categories

---

## Recipient

A recipient is the end user who receives notifications.

A recipient may have:

* Email address
* Phone number
* Push tokens
* Time zone
* Locale
* User preferences
* Unsubscribe status
* Device metadata

---

## Notification

A notification is a logical message that a tenant wants to send.

Example:

```json
{
  "tenant_id": "tenant_123",
  "recipient_id": "user_456",
  "event_type": "order.shipped",
  "channels": ["email", "sms"],
  "data": {
    "first_name": "Paras",
    "order_id": "ORD-789",
    "tracking_url": "https://example.com/track/ORD-789"
  }
}
```

---

## Notification Event

A notification event is the incoming request or business event that triggers notification creation.

Example event types:

* `user.created`
* `user.password_reset_requested`
* `order.created`
* `order.shipped`
* `payment.failed`
* `invoice.generated`
* `message.received`
* `security.login_detected`
* `marketing.campaign_started`

---

## Notification Template

A template defines the content of a message.

A single notification template can have different channel-specific versions.

Example:

```text
Template: order.shipped

Email:
Subject: Your order {{order_id}} has shipped
Body: Hi {{first_name}}, your order is on the way.

SMS:
Your order {{order_id}} has shipped. Track it here: {{tracking_url}}

Push:
Title: Order shipped
Body: Your order {{order_id}} is on the way.
```

---

## Delivery

A delivery is a channel-specific attempt to send a notification.

One notification can create multiple deliveries.

Example:

```text
Notification ID: notif_123

Deliveries:
- delivery_1: email
- delivery_2: sms
- delivery_3: in_app
```

Each delivery has its own lifecycle, provider, retry count, and status.

---

# Notification Lifecycle

```mermaid
stateDiagram-v2
    [*] --> RECEIVED
    RECEIVED --> VALIDATED
    VALIDATED --> SCHEDULED
    VALIDATED --> QUEUED
    SCHEDULED --> QUEUED
    QUEUED --> PROCESSING
    PROCESSING --> RENDERED
    RENDERED --> DISPATCHED
    DISPATCHED --> DELIVERED
    DISPATCHED --> FAILED
    FAILED --> RETRYING
    RETRYING --> DISPATCHED
    FAILED --> DEAD_LETTERED
    DELIVERED --> [*]
    DEAD_LETTERED --> [*]
```

## Lifecycle Steps

### 1. Received

The API receives a notification request.

The system validates:

* API key
* Tenant status
* Payload format
* Required fields
* Idempotency key
* Notification category
* Allowed channels

---

### 2. Validated

The request is accepted and persisted.

At this point, the system guarantees that the notification will either be delivered, retried, cancelled, expired, or dead-lettered.

---

### 3. Scheduled or Queued

If the notification should be sent later, it goes to the scheduler.

If it should be sent immediately, it goes to the message broker.

---

### 4. Processing

The orchestrator picks up the message.

It checks:

* Recipient exists
* Recipient preferences
* Consent rules
* Channel availability
* Tenant rate limits
* Quiet hours
* Deduplication rules

---

### 5. Rendered

The template renderer creates channel-specific content.

It handles:

* Variables
* Conditionals
* Localization
* Branding
* Fallback text
* HTML escaping
* Preview text
* Attachments

---

### 6. Dispatched

A channel worker sends the message to a provider.

Example:

* Email worker sends to SendGrid
* SMS worker sends to Twilio
* Push worker sends to FCM
* Webhook worker sends HTTP POST

---

### 7. Delivered, Failed, or Retried

The provider returns a status.

The system updates delivery state.

Failures may be retried based on failure type.

---

# Service Architecture

The system is divided into multiple services. In a small team, these can start as modules inside a modular monolith. As traffic grows, they can become independent microservices.

---

## 1. API Gateway

Responsible for:

* Receiving tenant API requests
* Authenticating API keys
* Validating request shape
* Applying API rate limits
* Returning request acceptance response
* Forwarding internal requests to services

The gateway should not perform expensive delivery work synchronously.

Good behavior:

```text
Client sends notification request
API validates it
API stores request
API returns 202 Accepted
Workers process delivery asynchronously
```

---

## 2. Tenant Service

Responsible for:

* Tenant registration
* Tenant status
* API key management
* Tenant configuration
* Billing plan
* Feature flags
* Allowed channels
* Provider settings

---

## 3. Notification Request Service

Responsible for:

* Creating notification records
* Validating event types
* Storing idempotency keys
* Publishing notification jobs
* Handling scheduled notification requests
* Creating delivery records

---

## 4. Template Service

Responsible for:

* Template CRUD
* Template versioning
* Template validation
* Template publishing
* Template localization
* Preview rendering

Templates should be immutable once published.

New edits create a new version.

---

## 5. Preference Service

Responsible for:

* User preferences
* Category subscriptions
* Channel preferences
* Quiet hours
* Frequency caps
* Unsubscribe links
* Consent status

---

## 6. Scheduler Service

Responsible for:

* Delayed notifications
* Recurring notifications
* Digest windows
* Campaign scheduling
* Time-zone-aware delivery
* Expiration handling

---

## 7. Orchestrator Service

Responsible for converting a logical notification into one or more channel deliveries.

It performs:

* Preference checks
* Channel selection
* Template lookup
* Recipient lookup
* Policy evaluation
* Delivery fanout
* Queue publishing

---

## 8. Renderer Service

Responsible for rendering final message content.

It supports:

* HTML email templates
* Plain text email fallback
* SMS templates
* Push titles and bodies
* In-app cards
* Webhook payloads
* Localization
* Tenant branding

---

## 9. Delivery Workers

Separate workers process each channel.

Recommended worker separation:

* `email-worker`
* `sms-worker`
* `push-worker`
* `inapp-worker`
* `webhook-worker`

Each worker owns channel-specific provider logic.

---

## 10. Delivery Status Service

Responsible for:

* Updating delivery states
* Consuming provider webhooks
* Normalizing provider statuses
* Publishing status events
* Updating analytics
* Triggering tenant callbacks

---

## 11. Analytics Service

Responsible for:

* Delivery metrics
* Open/click metrics
* Bounce metrics
* Failure metrics
* Tenant usage reports
* Billing counters

---

# Data Model

The primary database can be PostgreSQL.

For very high-scale analytics, use a separate OLAP store such as ClickHouse, BigQuery, Snowflake, or Apache Pinot.

---

## Entity Relationship Overview

```mermaid
erDiagram
    TENANTS ||--o{ API_KEYS : has
    TENANTS ||--o{ TEMPLATES : owns
    TENANTS ||--o{ RECIPIENTS : owns
    TENANTS ||--o{ NOTIFICATIONS : sends
    TENANTS ||--o{ PROVIDER_CONFIGS : configures

    RECIPIENTS ||--o{ RECIPIENT_CONTACTS : has
    RECIPIENTS ||--o{ PREFERENCES : has
    RECIPIENTS ||--o{ NOTIFICATIONS : receives

    TEMPLATES ||--o{ TEMPLATE_VERSIONS : has
    NOTIFICATIONS ||--o{ DELIVERIES : creates
    DELIVERIES ||--o{ DELIVERY_ATTEMPTS : has
```

---

## tenants

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY,
    name TEXT NOT NULL,
    slug TEXT UNIQUE NOT NULL,
    status TEXT NOT NULL CHECK (status IN ('active', 'suspended', 'deleted')),
    plan TEXT NOT NULL,
    default_locale TEXT DEFAULT 'en',
    timezone TEXT DEFAULT 'UTC',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## api_keys

```sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    key_hash TEXT NOT NULL,
    name TEXT NOT NULL,
    scopes TEXT[] NOT NULL,
    last_used_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    revoked_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Never store raw API keys. Store only hashes.

---

## recipients

```sql
CREATE TABLE recipients (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    external_id TEXT NOT NULL,
    first_name TEXT,
    last_name TEXT,
    locale TEXT,
    timezone TEXT,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, external_id)
);
```

---

## recipient_contacts

```sql
CREATE TABLE recipient_contacts (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    recipient_id UUID NOT NULL REFERENCES recipients(id),
    type TEXT NOT NULL CHECK (type IN ('email', 'phone', 'push_token', 'webhook')),
    value TEXT NOT NULL,
    is_primary BOOLEAN DEFAULT false,
    is_verified BOOLEAN DEFAULT false,
    provider_metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## notification_categories

```sql
CREATE TABLE notification_categories (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    key TEXT NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    type TEXT NOT NULL CHECK (type IN ('transactional', 'marketing', 'system')),
    is_required BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, key)
);
```

---

## preferences

```sql
CREATE TABLE preferences (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    recipient_id UUID NOT NULL REFERENCES recipients(id),
    category_id UUID NOT NULL REFERENCES notification_categories(id),
    channel TEXT NOT NULL CHECK (channel IN ('email', 'sms', 'push', 'in_app', 'webhook')),
    enabled BOOLEAN NOT NULL DEFAULT true,
    quiet_hours_start TIME,
    quiet_hours_end TIME,
    frequency_limit JSONB DEFAULT '{}',
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (recipient_id, category_id, channel)
);
```

---

## templates

```sql
CREATE TABLE templates (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    key TEXT NOT NULL,
    name TEXT NOT NULL,
    category_id UUID REFERENCES notification_categories(id),
    status TEXT NOT NULL CHECK (status IN ('draft', 'active', 'archived')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, key)
);
```

---

## template_versions

```sql
CREATE TABLE template_versions (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    template_id UUID NOT NULL REFERENCES templates(id),
    version INTEGER NOT NULL,
    locale TEXT NOT NULL DEFAULT 'en',
    channel TEXT NOT NULL CHECK (channel IN ('email', 'sms', 'push', 'in_app', 'webhook')),
    subject TEXT,
    title TEXT,
    body_text TEXT,
    body_html TEXT,
    payload_template JSONB,
    schema JSONB DEFAULT '{}',
    status TEXT NOT NULL CHECK (status IN ('draft', 'published', 'archived')),
    published_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (template_id, version, locale, channel)
);
```

---

## notifications

```sql
CREATE TABLE notifications (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    recipient_id UUID REFERENCES recipients(id),
    external_recipient_id TEXT,
    template_key TEXT NOT NULL,
    category_key TEXT,
    idempotency_key TEXT,
    priority TEXT NOT NULL CHECK (priority IN ('low', 'normal', 'high', 'critical')),
    status TEXT NOT NULL CHECK (
        status IN (
            'received',
            'validated',
            'scheduled',
            'queued',
            'processing',
            'completed',
            'cancelled',
            'failed',
            'expired'
        )
    ),
    channels TEXT[] NOT NULL,
    data JSONB DEFAULT '{}',
    scheduled_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, idempotency_key)
);
```

---

## deliveries

```sql
CREATE TABLE deliveries (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    notification_id UUID NOT NULL REFERENCES notifications(id),
    recipient_id UUID REFERENCES recipients(id),
    channel TEXT NOT NULL CHECK (channel IN ('email', 'sms', 'push', 'in_app', 'webhook')),
    provider TEXT,
    provider_message_id TEXT,
    status TEXT NOT NULL CHECK (
        status IN (
            'pending',
            'queued',
            'rendered',
            'dispatching',
            'sent',
            'delivered',
            'opened',
            'clicked',
            'failed',
            'bounced',
            'rejected',
            'suppressed',
            'cancelled',
            'dead_lettered'
        )
    ),
    rendered_subject TEXT,
    rendered_title TEXT,
    rendered_body_text TEXT,
    rendered_body_html TEXT,
    rendered_payload JSONB,
    attempt_count INTEGER NOT NULL DEFAULT 0,
    next_retry_at TIMESTAMPTZ,
    last_error_code TEXT,
    last_error_message TEXT,
    sent_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## delivery_attempts

```sql
CREATE TABLE delivery_attempts (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    delivery_id UUID NOT NULL REFERENCES deliveries(id),
    attempt_number INTEGER NOT NULL,
    provider TEXT NOT NULL,
    request_payload JSONB,
    response_payload JSONB,
    status TEXT NOT NULL CHECK (status IN ('success', 'failed', 'timeout')),
    error_code TEXT,
    error_message TEXT,
    latency_ms INTEGER,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## provider_configs

```sql
CREATE TABLE provider_configs (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    channel TEXT NOT NULL,
    provider TEXT NOT NULL,
    encrypted_config JSONB NOT NULL,
    priority INTEGER NOT NULL DEFAULT 100,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, channel, provider)
);
```

---

## webhook_endpoints

```sql
CREATE TABLE webhook_endpoints (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    url TEXT NOT NULL,
    secret_hash TEXT NOT NULL,
    events TEXT[] NOT NULL,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## audit_logs

```sql
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY,
    tenant_id UUID REFERENCES tenants(id),
    actor_id UUID,
    action TEXT NOT NULL,
    resource_type TEXT NOT NULL,
    resource_id UUID,
    metadata JSONB DEFAULT '{}',
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

# Queue and Event Design

A message broker is required for asynchronous processing.

Recommended options:

* Kafka
* AWS SQS
* Google Pub/Sub
* RabbitMQ
* NATS
* Redis Streams for smaller systems

---

## Queue Topics

| Topic                      | Purpose                           |
| -------------------------- | --------------------------------- |
| `notification.received`    | New notification accepted         |
| `notification.scheduled`   | Notification scheduled for future |
| `notification.ready`       | Notification ready for processing |
| `delivery.email`           | Email delivery jobs               |
| `delivery.sms`             | SMS delivery jobs                 |
| `delivery.push`            | Push delivery jobs                |
| `delivery.in_app`          | In-app delivery jobs              |
| `delivery.webhook`         | Webhook delivery jobs             |
| `delivery.retry`           | Retryable failed deliveries       |
| `delivery.status.updated`  | Delivery status changes           |
| `notification.dead_letter` | Permanently failed messages       |
| `analytics.events`         | Metrics and analytics events      |

---

## Notification Job Event

```json
{
  "event_id": "evt_01HZY",
  "event_type": "notification.ready",
  "tenant_id": "tenant_123",
  "notification_id": "notif_456",
  "priority": "normal",
  "created_at": "2026-07-08T10:00:00Z",
  "trace_id": "trace_abc"
}
```

---

## Delivery Job Event

```json
{
  "event_id": "evt_01HZK",
  "event_type": "delivery.email",
  "tenant_id": "tenant_123",
  "notification_id": "notif_456",
  "delivery_id": "del_789",
  "channel": "email",
  "provider": "sendgrid",
  "attempt": 1,
  "created_at": "2026-07-08T10:00:01Z",
  "trace_id": "trace_abc"
}
```

---

## Status Update Event

```json
{
  "event_id": "evt_01HZS",
  "event_type": "delivery.status.updated",
  "tenant_id": "tenant_123",
  "notification_id": "notif_456",
  "delivery_id": "del_789",
  "channel": "email",
  "old_status": "sent",
  "new_status": "delivered",
  "provider": "sendgrid",
  "provider_message_id": "sg_msg_123",
  "created_at": "2026-07-08T10:00:10Z"
}
```

---

# API Design

The API should be REST-based initially. GraphQL can be added later for dashboard use cases.

Base URL:

```text
https://api.notification-saas.com
```

Version:

```text
/v1
```

Authentication:

```http
Authorization: Bearer <api_key>
```

Idempotency:

```http
Idempotency-Key: tenant_unique_request_key
```

---

## Send Notification

```http
POST /v1/notifications/send
```

### Request

```json
{
  "recipient": {
    "external_id": "user_123",
    "email": "user@example.com",
    "phone": "+14155552671",
    "locale": "en",
    "timezone": "America/Los_Angeles"
  },
  "template_key": "order.shipped",
  "category_key": "transactional.order_updates",
  "channels": ["email", "sms", "in_app"],
  "priority": "normal",
  "data": {
    "first_name": "Paras",
    "order_id": "ORD-123",
    "tracking_url": "https://example.com/track/ORD-123"
  },
  "scheduled_at": null,
  "expires_at": "2026-07-09T10:00:00Z"
}
```

### Response

```json
{
  "notification_id": "notif_123",
  "status": "queued",
  "accepted_at": "2026-07-08T10:00:00Z"
}
```

Return `202 Accepted`, not `200 OK`, because delivery happens asynchronously.

---

## Send Bulk Notifications

```http
POST /v1/notifications/bulk
```

### Request

```json
{
  "template_key": "weekly.digest",
  "category_key": "product.digest",
  "channels": ["email"],
  "recipients": [
    {
      "external_id": "user_1",
      "email": "user1@example.com",
      "data": {
        "first_name": "Asha",
        "summary_count": 12
      }
    },
    {
      "external_id": "user_2",
      "email": "user2@example.com",
      "data": {
        "first_name": "Ravi",
        "summary_count": 5
      }
    }
  ]
}
```

### Response

```json
{
  "batch_id": "batch_123",
  "accepted_count": 2,
  "status": "processing"
}
```

---

## Get Notification Status

```http
GET /v1/notifications/{notification_id}
```

### Response

```json
{
  "id": "notif_123",
  "status": "completed",
  "template_key": "order.shipped",
  "channels": ["email", "sms"],
  "created_at": "2026-07-08T10:00:00Z",
  "deliveries": [
    {
      "id": "del_email_123",
      "channel": "email",
      "status": "delivered",
      "provider": "sendgrid",
      "sent_at": "2026-07-08T10:00:02Z",
      "delivered_at": "2026-07-08T10:00:05Z"
    },
    {
      "id": "del_sms_123",
      "channel": "sms",
      "status": "sent",
      "provider": "twilio",
      "sent_at": "2026-07-08T10:00:04Z"
    }
  ]
}
```

---

## Cancel Scheduled Notification

```http
POST /v1/notifications/{notification_id}/cancel
```

### Response

```json
{
  "notification_id": "notif_123",
  "status": "cancelled"
}
```

Only scheduled or queued notifications can be cancelled safely.

---

## Create Template

```http
POST /v1/templates
```

### Request

```json
{
  "key": "order.shipped",
  "name": "Order Shipped",
  "category_key": "transactional.order_updates",
  "channels": {
    "email": {
      "locale": "en",
      "subject": "Your order {{order_id}} has shipped",
      "body_html": "<p>Hi {{first_name}}, your order has shipped.</p>",
      "body_text": "Hi {{first_name}}, your order has shipped."
    },
    "sms": {
      "locale": "en",
      "body_text": "Your order {{order_id}} has shipped: {{tracking_url}}"
    },
    "push": {
      "locale": "en",
      "title": "Order shipped",
      "body_text": "Your order {{order_id}} is on the way."
    }
  }
}
```

---

## Publish Template Version

```http
POST /v1/templates/{template_id}/versions/{version}/publish
```

Publishing makes a template version available for production delivery.

---

## Update Recipient Preferences

```http
PUT /v1/recipients/{external_id}/preferences
```

### Request

```json
{
  "preferences": [
    {
      "category_key": "marketing.promotions",
      "channel": "email",
      "enabled": false
    },
    {
      "category_key": "transactional.order_updates",
      "channel": "sms",
      "enabled": true
    }
  ]
}
```

---

## Provider Webhook Receiver

```http
POST /v1/provider-webhooks/{provider}
```

Used by SendGrid, Twilio, Mailgun, SES, Firebase, and other providers to report delivery events.

---

## Tenant Callback Webhooks

Tenants can register webhook endpoints to receive notification events.

Example callback:

```json
{
  "type": "delivery.delivered",
  "tenant_id": "tenant_123",
  "notification_id": "notif_123",
  "delivery_id": "del_456",
  "channel": "email",
  "timestamp": "2026-07-08T10:00:05Z"
}
```

Webhook callbacks must be signed.

Example header:

```http
X-Notification-Signature: sha256=<signature>
X-Notification-Timestamp: 2026-07-08T10:00:05Z
```

---

# Template System

The template system is one of the most important parts of the platform.

It must be:

* Safe
* Versioned
* Previewable
* Localized
* Channel-aware
* Tenant-branded
* Backward compatible

---

## Template Variables

Example template:

```html
<h1>Hello {{first_name}}</h1>
<p>Your order {{order_id}} has shipped.</p>
<a href="{{tracking_url}}">Track order</a>
```

Example data:

```json
{
  "first_name": "Paras",
  "order_id": "ORD-123",
  "tracking_url": "https://example.com/track/ORD-123"
}
```

Rendered output:

```html
<h1>Hello Paras</h1>
<p>Your order ORD-123 has shipped.</p>
<a href="https://example.com/track/ORD-123">Track order</a>
```

---

## Template Validation

Before publishing, validate:

* Required variables
* Missing subject for email
* SMS length
* Invalid HTML
* Unsafe scripts
* Invalid links
* Unsupported syntax
* Missing fallback locale
* Missing plain-text fallback for email

---

## Template Versioning

Templates should be immutable once published.

Example:

```text
order.shipped v1 draft
order.shipped v1 published
order.shipped v2 draft
order.shipped v2 published
order.shipped v1 archived
```

Notifications should store the exact template version used for rendering.

This helps debugging and compliance.

---

## Localization

Templates can support multiple locales.

Example:

```text
order.shipped / email / en
order.shipped / email / hi
order.shipped / email / fr
order.shipped / sms / en
order.shipped / sms / hi
```

Template selection order:

1. Recipient locale
2. Tenant default locale
3. Platform default locale

---

# User Preferences and Consent

The system must respect user preferences before sending notifications.

---

## Preference Rules

Transactional notifications are usually required.

Examples:

* Password reset
* Payment failed
* Security alert
* Account verification

Marketing notifications must be optional.

Examples:

* Promotions
* Newsletters
* Product announcements
* Offers

---

## Preference Evaluation Flow

```mermaid
flowchart TD
    Start[Notification Requested] --> Category{Category Type}
    Category -->|Required Transactional| Allow[Allow]
    Category -->|Marketing| Consent{Has Consent?}
    Consent -->|No| Suppress[Suppress Delivery]
    Consent -->|Yes| ChannelPref{Channel Enabled?}
    ChannelPref -->|No| Suppress
    ChannelPref -->|Yes| QuietHours{Inside Quiet Hours?}
    QuietHours -->|Yes| Delay[Delay or Suppress]
    QuietHours -->|No| Allow
```

---

## Quiet Hours

Recipients may configure quiet hours.

Example:

```json
{
  "quiet_hours": {
    "enabled": true,
    "start": "22:00",
    "end": "08:00",
    "timezone": "Asia/Kolkata"
  }
}
```

Critical notifications may bypass quiet hours.

---

## Frequency Caps

Frequency caps prevent notification spam.

Example:

```json
{
  "max_per_hour": 3,
  "max_per_day": 10,
  "max_marketing_per_week": 2
}
```

---

## Unsubscribe

Every marketing email should include an unsubscribe link.

Example:

```text
https://notifications.example.com/unsubscribe?token=<signed_token>
```

The token should include:

* Tenant ID
* Recipient ID
* Category ID
* Expiration
* Signature

---

# Channel Delivery Design

## Email

Email is used for rich content and transactional communication.

Features:

* HTML body
* Plain text fallback
* Attachments
* Open tracking
* Click tracking
* Bounce handling
* Spam complaint handling
* Unsubscribe headers

Providers:

* SendGrid
* Amazon SES
* Mailgun
* Postmark

Email statuses:

```text
queued -> sent -> delivered -> opened -> clicked
queued -> sent -> bounced
queued -> rejected
queued -> suppressed
```

---

## SMS

SMS is used for urgent short messages.

Features:

* Phone number validation
* Country-specific rules
* Character counting
* Unicode support
* Delivery receipts
* STOP unsubscribe handling

Providers:

* Twilio
* MessageBird
* Vonage

SMS statuses:

```text
queued -> sent -> delivered
queued -> failed
queued -> undelivered
```

---

## Push Notifications

Push is used for mobile and web notifications.

Features:

* Device token management
* Token expiration handling
* Platform-specific payloads
* Deep links
* Badge counts
* Silent notifications

Providers:

* Firebase Cloud Messaging
* Apple Push Notification Service
* Web Push

Push statuses:

```text
queued -> sent
queued -> invalid_token
queued -> failed
```

---

## In-App Notifications

In-app notifications are stored inside the platform database and shown in the product UI.

Features:

* Notification inbox
* Read/unread status
* Real-time updates
* Expiration
* Priority
* Deep links
* User actions

Example in-app payload:

```json
{
  "title": "Payment failed",
  "body": "Please update your payment method.",
  "action_url": "/billing/payment-method",
  "priority": "high"
}
```

---

## Webhooks

Webhooks allow tenants to notify their own systems.

Features:

* Signed requests
* Retry on failure
* Timeout handling
* Secret rotation
* Event filtering
* Delivery logs

Webhook delivery rules:

* Timeout after 5 seconds
* Retry on 408, 429, and 5xx
* Do not retry on 400, 401, 403, or 404 unless configured
* Sign every request
* Store response status and body preview

---

# Provider Adapter Design

Provider adapters isolate third-party provider logic from the rest of the system.

---

## Interface

```ts
interface NotificationProvider {
  send(message: ProviderMessage): Promise<ProviderSendResult>;
  parseWebhook(payload: unknown, headers: Record<string, string>): ProviderWebhookEvent;
  validateConfig(config: ProviderConfig): boolean;
}
```

---

## Email Provider Example

```ts
class SendGridProvider implements NotificationProvider {
  async send(message: ProviderMessage): Promise<ProviderSendResult> {
    // Convert internal message into SendGrid payload
    // Send request to SendGrid
    // Normalize response
    return {
      providerMessageId: "sendgrid_msg_123",
      status: "sent"
    };
  }

  parseWebhook(payload: unknown): ProviderWebhookEvent {
    // Convert SendGrid event into internal event format
    return {
      status: "delivered",
      providerMessageId: "sendgrid_msg_123",
      timestamp: new Date()
    };
  }

  validateConfig(config: ProviderConfig): boolean {
    return Boolean(config.apiKey);
  }
}
```

---

## Why Provider Adapters Matter

Provider adapters make it easy to:

* Add new providers
* Fail over between providers
* Keep internal status models consistent
* Test delivery without real providers
* Support tenant-specific provider configuration

---

# Scheduling, Batching, and Digests

## Scheduled Notifications

A scheduled notification is stored with a `scheduled_at` timestamp.

The scheduler periodically finds due notifications.

```sql
SELECT *
FROM notifications
WHERE status = 'scheduled'
AND scheduled_at <= now()
LIMIT 1000;
```

The scheduler then publishes them to `notification.ready`.

---

## Recurring Notifications

Recurring notifications require a recurrence rule.

Example:

```json
{
  "template_key": "weekly.report",
  "recurrence": {
    "frequency": "weekly",
    "day_of_week": "monday",
    "time": "09:00",
    "timezone": "Asia/Kolkata"
  }
}
```

---

## Digest Notifications

Digest notifications combine multiple events into one message.

Example:

Instead of sending:

```text
New comment
New like
New follower
New mention
```

Send:

```text
You have 4 new updates.
```

Digest flow:

```mermaid
flowchart TD
    Event[User Activity Event] --> DigestBuffer[Digest Buffer]
    DigestBuffer --> Window{Digest Window Closed?}
    Window -->|No| Wait[Wait]
    Window -->|Yes| Build[Build Digest Payload]
    Build --> Template[Render Digest Template]
    Template --> Send[Send Notification]
```

Digest storage can use:

* PostgreSQL for small scale
* Redis sorted sets for medium scale
* Kafka compacted topics for high scale
* ClickHouse or BigQuery for analytics-style digests

---

# Retries, Failures, and Idempotency

## Idempotency

Clients should send an idempotency key.

```http
Idempotency-Key: order_123_shipped_email
```

The system stores:

```text
tenant_id + idempotency_key
```

If the same request is received again, return the original notification response.

This prevents duplicate messages during client retries.

---

## Retryable Failures

Retry when:

* Provider timeout
* Provider rate limit
* Provider 5xx error
* Network failure
* Temporary DNS failure
* Queue processing failure

Do not retry when:

* Invalid email
* Invalid phone number
* Recipient unsubscribed
* Template missing
* Provider authentication invalid
* Permanent provider rejection

---

## Exponential Backoff

Example retry schedule:

| Attempt | Delay      |
| ------- | ---------- |
| 1       | 30 seconds |
| 2       | 2 minutes  |
| 3       | 10 minutes |
| 4       | 30 minutes |
| 5       | 2 hours    |

Add jitter to avoid retry storms.

Example:

```text
delay = base_delay * 2^attempt + random_jitter
```

---

## Dead Letter Queue

After maximum retries, move the delivery to a dead letter queue.

Dead-lettered messages should include:

* Tenant ID
* Notification ID
* Delivery ID
* Channel
* Provider
* Error code
* Error message
* Attempt count
* Last payload preview
* Trace ID

Operators can inspect, replay, or permanently archive dead-lettered messages.

---

# Rate Limiting and Throttling

Rate limiting protects the platform, tenants, providers, and recipients.

---

## API Rate Limits

Example:

| Plan       | Limit              |
| ---------- | ------------------ |
| Free       | 60 requests/min    |
| Startup    | 600 requests/min   |
| Business   | 6,000 requests/min |
| Enterprise | Custom             |

---

## Tenant Delivery Limits

Example:

```json
{
  "email": {
    "max_per_minute": 1000,
    "max_per_day": 500000
  },
  "sms": {
    "max_per_minute": 100,
    "max_per_day": 50000
  }
}
```

---

## Recipient Frequency Limits

Prevent one user from receiving too many notifications.

Example:

```text
Max 5 push notifications per hour
Max 10 marketing emails per week
Max 3 SMS messages per day
```

---

## Provider Rate Limits

Each provider may have its own limit.

Example:

```text
SendGrid: 10,000 emails/minute
Twilio: 100 SMS/second
FCM: project-specific throughput
```

The delivery workers should use token buckets or leaky buckets.

---

# Multitenancy

The platform must isolate tenant data and configuration.

---

## Tenant Isolation Strategy

Recommended approach for most SaaS systems:

```text
Shared database
Shared tables
tenant_id column on every tenant-owned table
Row-level authorization in application code
Optional PostgreSQL Row Level Security
```

For enterprise customers, support:

```text
Dedicated database
Dedicated encryption keys
Dedicated provider accounts
Dedicated rate limits
Dedicated worker pools
```

---

## Every Query Must Include tenant_id

Good:

```sql
SELECT *
FROM notifications
WHERE tenant_id = $1
AND id = $2;
```

Bad:

```sql
SELECT *
FROM notifications
WHERE id = $1;
```

---

## Tenant Configuration

Each tenant can configure:

* Branding
* Sender email
* Sender phone number
* Templates
* Providers
* Rate limits
* Categories
* Preference defaults
* Webhook endpoints
* API keys

---

# Security

## API Authentication

Use API keys for machine-to-machine access.

Rules:

* Show raw key only once
* Store only hash
* Allow key rotation
* Allow key expiration
* Allow scoped keys
* Audit every key usage

Example scopes:

```text
notifications:send
notifications:read
templates:write
templates:read
recipients:write
recipients:read
webhooks:write
```

---

## Dashboard Authentication

Use:

* SSO
* SAML
* OAuth
* MFA
* Role-based access control

Roles:

| Role      | Permissions                        |
| --------- | ---------------------------------- |
| Owner     | Full access                        |
| Admin     | Manage templates, providers, users |
| Developer | API keys, logs, testing            |
| Marketer  | Campaigns and templates            |
| Analyst   | Read-only analytics                |
| Support   | Delivery logs and recipient lookup |

---

## Secrets Management

Provider credentials must be encrypted.

Use:

* AWS KMS
* Google Cloud KMS
* HashiCorp Vault
* Azure Key Vault

Never log:

* API keys
* Provider secrets
* Full message bodies with sensitive content
* OTP codes
* Password reset tokens

---

## Webhook Security

Tenant callback webhooks must be signed.

Signature example:

```text
HMAC_SHA256(secret, timestamp + "." + raw_body)
```

Headers:

```http
X-Notification-Timestamp: 2026-07-08T10:00:00Z
X-Notification-Signature: sha256=<signature>
```

Receiver should reject old timestamps to prevent replay attacks.

---

## Data Privacy

Support:

* Recipient deletion
* Data export
* Consent tracking
* PII encryption
* Retention policies
* Audit logs
* Access logs

---

# Observability

The system must be observable from day one.

---

## Logs

Every log should include:

* `trace_id`
* `tenant_id`
* `notification_id`
* `delivery_id`
* `channel`
* `provider`
* `status`
* `error_code`

Example:

```json
{
  "level": "error",
  "message": "Email delivery failed",
  "tenant_id": "tenant_123",
  "notification_id": "notif_456",
  "delivery_id": "del_789",
  "channel": "email",
  "provider": "sendgrid",
  "error_code": "PROVIDER_TIMEOUT",
  "trace_id": "trace_abc"
}
```

---

## Metrics

Important metrics:

### API Metrics

* Request count
* Request latency
* Error rate
* Authentication failures
* Rate-limit rejections

### Queue Metrics

* Queue depth
* Consumer lag
* Oldest message age
* Retry queue depth
* Dead letter queue count

### Delivery Metrics

* Notifications accepted
* Deliveries attempted
* Deliveries sent
* Deliveries delivered
* Deliveries failed
* Retry count
* Bounce rate
* Open rate
* Click rate

### Provider Metrics

* Provider latency
* Provider error rate
* Provider timeout rate
* Provider rate-limit rate
* Provider failover count

---

## Tracing

Use distributed tracing across:

```text
API Gateway
Notification Request Service
Queue
Orchestrator
Renderer
Delivery Worker
Provider Adapter
Status Service
Webhook Callback
```

Trace example:

```text
POST /notifications/send
  -> validate tenant
  -> persist notification
  -> publish notification.ready
  -> orchestrator consumes
  -> render template
  -> publish delivery.email
  -> email worker consumes
  -> call SendGrid
  -> update status
```

---

## Alerts

Create alerts for:

* API 5xx rate too high
* Queue lag too high
* Dead letter queue growing
* Provider error rate spike
* Delivery latency above SLO
* Database connection saturation
* Webhook callback failures
* SMS cost spike
* Email bounce rate spike

---

# Scalability

## Scaling API Layer

The API layer should be stateless.

Scale horizontally using:

* Kubernetes HPA
* ECS autoscaling
* Load balancers
* API Gateway
* NGINX or Envoy

---

## Scaling Workers

Workers should scale by queue depth.

Example:

```text
If email queue depth > 100,000
Increase email workers from 20 to 100
```

Worker scaling should consider provider rate limits.

More workers do not help if the provider allows only 100 requests per second.

---

## Scaling Database

Start with PostgreSQL.

Scale using:

* Proper indexes
* Partitioning by tenant or created date
* Read replicas
* Connection pooling
* Archiving old deliveries
* Moving analytics to OLAP store

Recommended indexes:

```sql
CREATE INDEX idx_notifications_tenant_created
ON notifications (tenant_id, created_at DESC);

CREATE INDEX idx_deliveries_tenant_status
ON deliveries (tenant_id, status);

CREATE INDEX idx_deliveries_next_retry
ON deliveries (next_retry_at)
WHERE status = 'failed';

CREATE INDEX idx_notifications_scheduled
ON notifications (scheduled_at)
WHERE status = 'scheduled';
```

---

## Partitioning

For high-volume systems, partition large tables.

Good candidates:

* notifications
* deliveries
* delivery_attempts
* analytics_events
* audit_logs

Partition by month:

```text
deliveries_2026_07
deliveries_2026_08
deliveries_2026_09
```

Or partition by tenant for enterprise customers.

---

## Caching

Use Redis for:

* Tenant config cache
* Template cache
* API key lookup cache
* Rate limit buckets
* Recipient preference cache
* Provider config cache

Cache invalidation should happen on config updates.

---

# Deployment Architecture

## Recommended Production Stack

| Layer          | Option                          |
| -------------- | ------------------------------- |
| API            | Node.js, Go, Java, Python       |
| Database       | PostgreSQL                      |
| Cache          | Redis                           |
| Queue          | Kafka, SQS, Pub/Sub, RabbitMQ   |
| Object Storage | S3 or GCS                       |
| Search         | OpenSearch or Elasticsearch     |
| Analytics      | ClickHouse, BigQuery, Snowflake |
| Metrics        | Prometheus                      |
| Logs           | Loki, ELK, Datadog              |
| Tracing        | OpenTelemetry                   |
| Deployment     | Kubernetes                      |
| Secrets        | Vault or Cloud KMS              |

---

## Kubernetes Services

```text
notification-api
tenant-service
template-service
preference-service
scheduler-service
orchestrator-service
email-worker
sms-worker
push-worker
inapp-worker
webhook-worker
provider-webhook-service
analytics-service
dashboard-web
```

---

## Example Environment Variables

```env
APP_ENV=production
DATABASE_URL=postgresql://user:password@host:5432/notifications
REDIS_URL=redis://redis:6379
QUEUE_BROKER_URL=kafka://kafka:9092

API_KEY_PEPPER=change_me
ENCRYPTION_KMS_KEY_ID=notification-saas-prod

DEFAULT_EMAIL_PROVIDER=sendgrid
DEFAULT_SMS_PROVIDER=twilio
DEFAULT_PUSH_PROVIDER=fcm

SENDGRID_API_KEY=encrypted_or_loaded_from_secret_manager
TWILIO_ACCOUNT_SID=encrypted_or_loaded_from_secret_manager
TWILIO_AUTH_TOKEN=encrypted_or_loaded_from_secret_manager

OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
LOG_LEVEL=info
```

---

# Local Development

## Prerequisites

* Docker
* Docker Compose
* Node.js, Go, Java, or preferred backend runtime
* PostgreSQL client
* Redis client

---

## Example Local Services

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: notifications
      POSTGRES_PASSWORD: notifications
      POSTGRES_DB: notifications
    ports:
      - "5432:5432"

  redis:
    image: redis:7
    ports:
      - "6379:6379"

  kafka:
    image: bitnami/kafka:latest
    ports:
      - "9092:9092"

  api:
    build: .
    environment:
      DATABASE_URL: postgresql://notifications:notifications@postgres:5432/notifications
      REDIS_URL: redis://redis:6379
      QUEUE_BROKER_URL: kafka://kafka:9092
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis
      - kafka
```

---

## Suggested Repository Structure

```text
notification-saas/
├── apps/
│   ├── api/
│   ├── dashboard/
│   ├── worker-email/
│   ├── worker-sms/
│   ├── worker-push/
│   ├── worker-webhook/
│   ├── scheduler/
│   └── orchestrator/
├── packages/
│   ├── database/
│   ├── queue/
│   ├── logger/
│   ├── config/
│   ├── auth/
│   ├── templates/
│   ├── providers/
│   ├── rate-limiter/
│   └── observability/
├── migrations/
├── docs/
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── contract/
│   └── load/
├── docker-compose.yml
├── README.md
└── LICENSE
```

---

# Testing Strategy

## Unit Tests

Test:

* Template rendering
* Preference evaluation
* Rate limit calculations
* Provider adapter normalization
* Retry decision logic
* Signature verification
* Idempotency behavior

---

## Integration Tests

Test:

* API to database flow
* API to queue flow
* Queue to worker flow
* Provider mock delivery
* Webhook receiver
* Scheduler
* Retry queue
* Dead letter queue

---

## Contract Tests

Provider integrations should have contract tests.

Example:

* SendGrid response mapping
* Twilio delivery status mapping
* FCM invalid token handling
* Webhook callback format

---

## Load Tests

Test:

* 1,000 notifications/min
* 10,000 notifications/min
* 100,000 notifications/min
* Bulk campaign sending
* Retry storms
* Provider outage scenarios

Important load-test metrics:

* API latency
* Queue lag
* Delivery latency
* DB CPU
* Worker CPU
* Redis memory
* Provider error rates

---

# Operational Runbooks

## Provider Outage

Symptoms:

* Provider 5xx errors increase
* Delivery failures increase
* Queue lag grows

Actions:

1. Confirm provider status.
2. Pause affected provider if needed.
3. Enable fallback provider.
4. Increase retry delay.
5. Notify affected tenants.
6. Monitor delivery recovery.
7. Replay failed messages if safe.

---

## Queue Lag High

Symptoms:

* Queue depth increasing
* Oldest message age increasing
* Delivery latency increasing

Actions:

1. Check worker health.
2. Check provider rate limits.
3. Scale workers if provider capacity allows.
4. Check database latency.
5. Check message poison-pill failures.
6. Move bad messages to DLQ if needed.

---

## Dead Letter Queue Growing

Symptoms:

* DLQ count increasing
* Repeated failures

Actions:

1. Group DLQ messages by error code.
2. Identify tenant, provider, or template issue.
3. Fix root cause.
4. Replay retryable messages.
5. Mark permanent failures as archived.

---

## High Bounce Rate

Symptoms:

* Email bounce rate exceeds threshold

Actions:

1. Identify tenant and campaign.
2. Pause problematic campaign.
3. Check email list quality.
4. Suppress bounced addresses.
5. Review domain reputation.
6. Notify tenant.

---

## SMS Cost Spike

Symptoms:

* SMS usage unexpectedly high

Actions:

1. Identify tenant and notification category.
2. Check bulk sends or loops.
3. Apply emergency rate limit.
4. Pause tenant SMS if needed.
5. Notify tenant owner.
6. Review audit logs.

---

# Reliability Principles

## Accepted Means Durable

Once the API returns `202 Accepted`, the notification must be durably stored.

Never publish to a queue without saving to the database, unless the queue itself is the source of truth.

Recommended pattern:

1. Insert notification into database.
2. Insert outbox event into database.
3. Outbox publisher sends event to broker.
4. Mark outbox event as published.

---

## Outbox Pattern

```sql
CREATE TABLE outbox_events (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    aggregate_type TEXT NOT NULL,
    aggregate_id UUID NOT NULL,
    event_type TEXT NOT NULL,
    payload JSONB NOT NULL,
    status TEXT NOT NULL CHECK (status IN ('pending', 'published', 'failed')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at TIMESTAMPTZ
);
```

This prevents losing events if the API saves to DB but crashes before publishing to the queue.

---

## Exactly-Once vs At-Least-Once

Most real distributed systems provide at-least-once processing.

Therefore the platform must be idempotent.

Workers may receive the same job more than once.

Before sending to a provider, a worker should check:

```text
Has this delivery already been sent successfully?
Has this provider message ID already been recorded?
Is this attempt still valid?
```

---

# Analytics

## Analytics Events

Track events like:

* notification.accepted
* notification.cancelled
* delivery.queued
* delivery.sent
* delivery.delivered
* delivery.failed
* email.opened
* email.clicked
* sms.delivered
* push.sent
* webhook.failed

---

## Dashboard Metrics

Tenant dashboard should show:

* Total notifications sent
* Delivery success rate
* Failure rate by channel
* Provider latency
* Bounce rate
* Open rate
* Click rate
* Top templates
* Usage by category
* Usage by channel
* Cost estimate

---

# Billing and Usage

Billing can be based on:

* Notifications accepted
* Deliveries attempted
* Successful deliveries
* Channel usage
* SMS segments
* Email volume
* Push volume
* Webhook calls
* Seats
* Retention period
* Dedicated infrastructure

Usage records should be append-only.

```sql
CREATE TABLE usage_events (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    event_type TEXT NOT NULL,
    quantity INTEGER NOT NULL,
    unit TEXT NOT NULL,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

# Cost Controls

The platform should support:

* Tenant spending limits
* SMS daily caps
* Campaign approval workflows
* Provider cost tracking
* Alerting on unusual usage
* Per-channel budgets

---

# Example End-to-End Flow

## Scenario

A shopping app wants to notify a user that an order has shipped.

---

## Step 1: Customer App Calls API

```http
POST /v1/notifications/send
Authorization: Bearer tenant_api_key
Idempotency-Key: order_ORD123_shipped
```

```json
{
  "recipient": {
    "external_id": "user_123",
    "email": "paras@example.com",
    "phone": "+919999999999",
    "locale": "en",
    "timezone": "Asia/Kolkata"
  },
  "template_key": "order.shipped",
  "category_key": "transactional.order_updates",
  "channels": ["email", "sms", "in_app"],
  "priority": "normal",
  "data": {
    "first_name": "Paras",
    "order_id": "ORD123",
    "tracking_url": "https://shop.example.com/track/ORD123"
  }
}
```

---

## Step 2: API Validates Request

The API checks:

* API key is valid
* Tenant is active
* Template exists
* Channels are allowed
* Payload is valid
* Idempotency key has not already been used

---

## Step 3: Notification Is Stored

A row is inserted into `notifications`.

Status:

```text
validated
```

---

## Step 4: Outbox Event Is Created

An event is inserted into `outbox_events`.

Event type:

```text
notification.ready
```

---

## Step 5: Queue Event Published

The outbox publisher sends the event to the message broker.

---

## Step 6: Orchestrator Processes Notification

The orchestrator:

1. Loads tenant config
2. Loads recipient
3. Checks preferences
4. Checks quiet hours
5. Loads template
6. Creates delivery records:

   * Email delivery
   * SMS delivery
   * In-app delivery

---

## Step 7: Channel Workers Send Messages

Email worker sends email.

SMS worker sends SMS.

In-app worker stores notification in inbox.

---

## Step 8: Providers Return Status

Email provider says:

```text
accepted
```

SMS provider says:

```text
sent
```

In-app worker says:

```text
created
```

---

## Step 9: Provider Webhooks Update Status

Later, SendGrid sends a webhook:

```text
delivered
```

Twilio sends a webhook:

```text
delivered
```

The status service updates delivery records.

---

## Step 10: Tenant Receives Callback

Tenant callback webhook receives:

```json
{
  "type": "notification.completed",
  "notification_id": "notif_123",
  "status": "completed"
}
```

---

# MVP Scope

For the first version, build:

* Tenant API keys
* Send notification API
* Email channel
* SMS channel
* Basic templates
* Basic preferences
* PostgreSQL persistence
* Redis rate limiting
* Queue-based workers
* Retry logic
* Delivery logs
* Provider webhook receiver
* Basic dashboard

---

# Production Scope

For production, add:

* Push notifications
* In-app notifications
* Webhooks
* Template versioning
* Localization
* Audit logs
* Tenant billing
* Advanced analytics
* Dedicated tenant config
* Provider failover
* Dead letter replay
* Advanced RBAC
* SSO
* Compliance workflows

---

# Roadmap

## Phase 1: Core Transactional System

* Email notifications
* SMS notifications
* Templates
* API keys
* Delivery logs
* Retry queue
* Provider webhooks

## Phase 2: SaaS Dashboard

* Tenant dashboard
* Template editor
* Delivery analytics
* API key management
* User preferences

## Phase 3: Scale and Reliability

* Outbox pattern
* DLQ replay
* Provider failover
* Advanced rate limiting
* Multi-region read replicas

## Phase 4: Advanced Channels

* Push notifications
* In-app notifications
* Webhook notifications
* Slack and Teams notifications

## Phase 5: Enterprise

* SSO
* Audit logs
* Dedicated infrastructure
* Custom retention
* Advanced compliance
* Tenant-level encryption keys

---

# Design Principles

1. Accept fast, process asynchronously.
2. Never lose accepted notifications.
3. Make every worker idempotent.
4. Treat providers as unreliable.
5. Respect user preferences and consent.
6. Separate logical notifications from channel deliveries.
7. Store every important state transition.
8. Use queues for isolation and backpressure.
9. Make tenant isolation non-negotiable.
10. Design for debugging from day one.

---

# License

This project can be licensed under MIT, Apache 2.0, or a private commercial license depending on business requirements.
