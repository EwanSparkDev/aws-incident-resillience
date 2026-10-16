
# Architecture Decision Records (ADRs)

**Project:** AWS Incident Management Platform

**Client:** Southern Cross IT Services

**Version:** 0.1

**Status:** Proposed — Architecture Design

**Date:** 16 October 2026

---

## ADR-001: DynamoDB-First Incident Creation

### Context

The customer requires employees to receive a unique
incident reference immediately after successfully
submitting a ticket.

Successfully created incidents must be persistently
stored and immediately retrievable by authorised users.

### Decision

Use a synchronous incident creation workflow:

Browser -> API Gateway -> Lambda -> DynamoDB

The Lambda function will:

1. Receive and validate the incident details.
2. Generate a unique incident reference.
3. Write the incident record to DynamoDB.
4. Wait for confirmation that the write succeeded.
5. Return HTTP 201 Created and the incident reference.

### Justification

This approach ensures that an incident has been stored
before the employee receives successful confirmation.

It also avoids depending on background processing
to make an incident available for retrieval.

### Alternatives Considered

**SQS-First Architecture**

API Gateway -> Lambda -> SQS -> Lambda -> DynamoDB

This would allow ticket submissions to be queued
during temporary database outages.

However, employees could receive a reference before
the incident becomes available in the database.

### Trade-offs

- Ticket creation depends on DynamoDB availability.
- Database latency affects submission response time.
- Ambiguous write timeouts require idempotent retry handling.

### Status

Proposed — Pending implementation and testing.

---

## ADR-002: Authentication and Authorisation

### Context

The application contains internal company incident
records that must not be accessible to unauthorised users.

Employees should only be able to access their own
incidents, while IT administrators require visibility
across the organisation.

### Decision

Use Amazon Cognito for authentication and API Gateway
JWT authorisation.

Application-level permissions will be enforced by Lambda.

Two Cognito groups will be configured:

- Employees
- ITAdmins

Approved employees will be able to create tickets and
retrieve their own incidents.

IT administrators will be able to retrieve incidents
submitted by all employees.

### Justification

Cognito provides managed identity services.

API Gateway can validate access tokens before
requests reach backend Lambda functions.

Lambda will check approved group membership and
verify ticket ownership before returning records.

### Alternatives Considered

**Custom Authentication System**

Developing our own authentication system would introduce
unnecessary development and security responsibilities.

**Google Federation**

Google sign-in could be introduced later, but successful
Google authentication alone must not grant application
access. Approved enrolment would still be required.

### Trade-offs

- Cognito configuration adds some complexity.
- Application-level authorisation must still be enforced.
- User provisioning and offboarding require controls.

### Status

Proposed — Pending implementation and testing.

---

## ADR-003: Asynchronous Email Notifications

### Context

Employees should receive an email confirming that
their incident has been created.

The email should include their unique ticket reference.

A failure in email delivery must not prevent successful
incident creation.

### Decision

Use DynamoDB Streams to initiate an asynchronous
notification workflow after incident creation.

Proposed architecture:

DynamoDB -> DynamoDB Streams -> Lambda -> Amazon SES

The notification Lambda will:

1. Receive a new incident INSERT event.
2. Extract the incident reference and employee details.
3. Construct a confirmation email.
4. Call Amazon SES to send the email.

### Justification

The email workflow operates independently of the
incident creation process.

Employees can receive their reference number immediately
after the database write succeeds without waiting
for the email notification.

### Alternatives Considered

**Amazon SNS**

SNS can notify predefined subscribers such as an
IT support mailbox.

However, SNS email subscriptions are not ideal for
sending individual confirmation emails to dynamic
employee addresses.

**EventBridge Pipes**

EventBridge Pipes can filter, transform and route
DynamoDB Streams events.

It was not selected for the initial design because
Lambda can consume DynamoDB Streams directly.

**Amazon SQS**

SQS could provide additional buffering and independent
retry management.

It may be introduced if notification reliability
requirements justify the extra complexity.

### Trade-offs

- Notification processing may be delayed.
- DynamoDB Streams records expire after 24 hours.
- Retries may deliver duplicate events.
- Failed notifications require monitoring and recovery.
- SES delivery and sender verification require configuration.

### Outstanding Considerations

- Configure retries and an on-failure destination.
- Handle duplicate notification events.
- Define a process for recovering failed notifications.
- Test SES failure scenarios.

### Status

Proposed — Pending implementation and testing.

---

## Decision Summary

| ID | Decision | Status |
|---|---|---|
| ADR-001 | DynamoDB-first incident creation | Proposed |
| ADR-002 | Cognito authentication and authorisation | Proposed |
| ADR-003 | Asynchronous email notifications | Proposed |

## Future Decisions

- ADR-004: Incident Retrieval and DynamoDB Access Patterns
- Notification failure recovery and monitoring
- Cost optimisation and scaling configuration
