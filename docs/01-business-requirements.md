# Business Requirements - Souther Cross IT Services (Fictional)

**Project:** AWS Incident Management Platform

**Document Version:** 0.1

**Status:** Draft

---

## 1. Executive Summary 

Southern Cross IT Services is a fictional Adelaide-based IT company employing approximately 500 people.

The organisation currently manages IT incidents through shared email inboxes and spreadsheets.

## 2. Business  Objectives

The propsed solution should:

- reduce the reisk of incidents being lost.
- Improve Visability for IT Aministrators.
- Minimise infrastructure costs.


## 3. Functional Requirements

Functional requirements define what the application must do.

| ID | Requirement | Priority |
|---|---|---|
| FR-01 | Employees must authenticate before accessing protected application functionality. | Must |
| FR-02 | Employees must be able to submit an incident containing a title, description and severity. | Must |
| FR-03 | Successfully created incidents must receive a unique reference number, returned immediately to the employee. | Must |
| FR-04 | Incidents must be stored persistently and retrievable by authorised users. | Must |
| FR-05 | Background notification processing must operate independently of incident creation. | Must |
| FR-06 | Administrators should be able to view incident processing status. | Should |
| FR-07 | Administrators should receive notifications when background processing repeatedly fails. | Should |
| FR-08 | Employees must only be able to view incidents they have submitted. | Must |
| FR-09 | IT administrators must be able to view incidents submitted by all employees. | Must |
| FR-10 | Employees should receive an email confirmation containing their incident reference after successful creation. | Should |


## 4. Non-Functional Requirements

Non-functional requirements describe how well the system must operate.

| ID | Category | Requirement |
|---|---|---|
| NFR-01 | Security | Only approved users may access ticket data. Employees can access their own incidents, while IT administrators can access all incidents. |
| NFR-02 | Availability | The system should target 99.9% monthly availability, with resilience to individual background-processing failures. |
| NFR-03 | Reliability | Successfully created incidents must not be silently lost and must remain recoverable. |
| NFR-04 | Scalability | The application must accommodate submission spikes without manual server provisioning. |
| NFR-05 | Observability | Administrators must be able to investigate failures through logs, metrics and monitoring. |
| NFR-06 | Cost Optimisation | The solution must minimise fixed infrastructure costs and target total lab spending below US$5. |
| NFR-07 | Data Integrity | HTTP 201 must only be returned after DynamoDB confirms successful incident creation. |
| NFR-08 | Fault Isolation | Email notification failures must not prevent successful incident creation. |







