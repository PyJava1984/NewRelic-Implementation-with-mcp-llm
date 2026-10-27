Supplier Information Management (SIM)
New Relic Observability, LLM and MCP Integration

!Java !Spring Boot !Jetty !Kubernetes !New Relic !AI

1. Overview

This project documents an observability and AI-assisted operations architecture for a Supplier Information Management (SIM) platform.

The platform uses Java, Spring, Jetty, Kubernetes, relational and NoSQL databases, and modern web technologies. New Relic collects telemetry, while Large Language Models (LLMs) and the Model Context Protocol (MCP) enable natural-language investigation of application performance and operational incidents.

The solution aims to help developers, support engineers, DevOps engineers, and platform administrators answer questions such as:

Why is supplier onboarding taking longer than usual?
Which API endpoints have the highest response times?
Are Jetty JVMs experiencing excessive garbage collection?
Which Kubernetes pods are restarting or running out of memory?
Is a database dependency contributing to application latency?
Did a recent deployment introduce additional errors?
What evidence supports a potential root cause?
Which documented troubleshooting steps should the operations team follow?

Design principle: The LLM explains and correlates evidence from approved tools. New Relic remains the source of monitoring telemetry, and the application remains responsible for enforcing business rules and access control.

2. Technology Stack
Backend
Java 8, Java 17, Java 21
Spring MVC
Spring Boot
Spring Security
JPA and Hibernate
Lombok
Jetty
Data and integration
PostgreSQL
Cassandra
REST APIs
MCP servers and clients
Keycloak and JWT authentication
Frontend
Vue.js
FreeMarker (FTL)
jQuery
Observability and AI
New Relic Java agent
New Relic APM
New Relic Kubernetes integration
New Relic Browser monitoring
NRQL (New Relic Query Language)
LLM provider or model gateway
Model Context Protocol (MCP)
Internal runbooks and operational knowledge

Exact framework, agent, driver, and runtime compatibility must be validated for each deployed service.

3. Architecture
3.1 Observability architecture
mermaid
flowchart TD
    U[SIM Users and Administrators]
    F[Vue.js / FreeMarker / jQuery]
    I[Ingress and Kubernetes Service]
    P[Jetty Application Pods]
    J[New Relic Java Agent]
    DB1[(PostgreSQL)]
    DB2[(Cassandra)]
    K[New Relic Kubernetes Integration]
    B[New Relic Browser Agent]
    N[New Relic Platform]

    U --> F
    F --> I
    I --> P
    P --> DB1
    P --> DB2
    P --> J
    F --> B
    K --> N
    J --> N
    B --> N

3.2 LLM and MCP architecture
mermaid
flowchart TD
    E[Developer / Support Engineer]
    UI[Internal AI Assistant]
    L[LLM / AI Agent]
    G[Authorization and Tool Gateway]
    M[MCP Server]
    NR[New Relic MCP / NRQL]
    KU[Read-only Kubernetes Tools]
    KB[Runbooks and Service Metadata]
    OBS[New Relic Telemetry]
    OUT[Evidence-based Incident Summary]

    E --> UI
    UI --> L
    L --> G
    G --> M
    M --> NR
    M --> KU
    M --> KB
    NR --> OBS
    OBS --> M
    KU --> M
    KB --> M
    M --> L
    L --> OUT
    OUT --> UI

3.3 Component responsibilities
Component	Responsibility
New Relic Java agent	Instrument supported JVM, framework, transaction, and dependency activity
New Relic Kubernetes integration	Collect cluster, node, pod, container, and event telemetry
New Relic Browser monitoring	Measure browser performance and JavaScript errors
LLM	Interpret questions, compare evidence, summarize incidents, and propose next steps
MCP server	Expose approved tools and contextual resources to the AI client
AI client	Orchestrate model calls and MCP tool invocations
Tool gateway	Enforce authentication, authorization, input validation, and audit controls
Internal knowledge source	Provide approved runbooks, service ownership, and release metadata

MCP standardizes communication between AI clients and tools. It does not replace application security, monitoring, authorization, or business logic.

4. Implementation Strategy

Implement the solution in phases.

Phase	Deliverable
1	New Relic Java APM in a Jetty application
2	Kubernetes monitoring
3	Database and browser observability
4	Dashboards, NRQL queries, and alert policies
5	Official New Relic MCP integration
6	AI-assisted incident investigation
7	Optional SIM-specific MCP tools
8	Production security, audit, and operational readiness

Start with read-only observability. Introduce operational write actions only after establishing authorization, approval, and audit mechanisms.

5. New Relic Java Agent for Jetty
5.1 Prerequisites
A New Relic account and an appropriately scoped license key.
A compatible Java agent version.
Access to the Jetty application image or JVM startup configuration.
Outbound network access to New Relic.
A development or testing environment.

Documentation:

Java agent installation
Java agent compatibility
5.2 Add the agent to the application image

The following is an illustrative Dockerfile fragment. Adapt it to the existing Jetty image and preserve its startup and security configuration.

dockerfile
FROM your-existing-jetty-image

USER root

RUN mkdir -p /opt/newrelic

COPY monitoring/newrelic.jar /opt/newrelic/newrelic.jar
COPY monitoring/newrelic.yml /opt/newrelic/newrelic.yml

# Configure ownership for the actual Jetty runtime user.
# RUN chown -R jetty:jetty /opt/newrelic

ENV JAVA_TOOL_OPTIONS="-javaagent:/opt/newrelic/newrelic.jar"

# Preserve the existing application deployment and Jetty startup.


Obtain the agent from the official New Relic documentation. Do not copy a real license key into the image or commit one to GitHub.

5.3 Configure Kubernetes

Create a secret in the appropriate namespace:

bash
kubectl create secret generic newrelic-license \
  --namespace sim-dev \
  --from-literal=license-key="$NEW_RELIC_LICENSE_KEY"


Supply the secret through the container environment:

yaml
env:
  - name: NEW_RELIC_APP_NAME
    value: "SIM-API-DEV"

  - name: NEW_RELIC_LICENSE_KEY
    valueFrom:
      secretKeyRef:
        name: newrelic-license
        key: license-key

  - name: NEW_RELIC_LOG_FILE_NAME
    value: "STDOUT"

  - name: JAVA_TOOL_OPTIONS
    value: "-javaagent:/opt/newrelic/newrelic.jar"


Merge these settings into the existing Deployment. Retain its ports, probes, resources, security context, and existing environment variables.

The example assumes the agent exists at /opt/newrelic/newrelic.jar and that the JVM does not already load another APM agent.

For production, use the organization's approved secret manager or external-secrets mechanism rather than creating secrets from credentials stored in shell history.

5.4 Verify APM
bash
kubectl rollout status deployment/sim-api -n sim-dev
kubectl get pods -n sim-dev
kubectl logs deployment/sim-api -n sim-dev


Then:

Generate requests against the application.
Open New Relic and navigate to APM & Services.
Select the configured application.
Inspect transaction response time, errors, JVM health, and traces.
6. Kubernetes Monitoring

Install New Relic Kubernetes monitoring using the official installer or Helm chart.

bash
helm repo add newrelic https://helm-charts.newrelic.com
helm repo update


Use the official Kubernetes installation guide to generate and review the installation configuration for your cluster.

Monitor:

Node and namespace health.
Pod CPU and memory utilization.
Deployment availability and replica counts.
Pod restarts and crash loops.
Kubernetes events.
Application-to-pod correlation.

Verify the integration:

bash
kubectl get pods -n newrelic
kubectl get daemonsets,deployments -n newrelic
helm list -n newrelic


Avoid duplicating an existing Kubernetes monitoring installation without first reviewing the deployed agents and data collection configuration.

7. Database and Frontend Monitoring
PostgreSQL

Monitor JDBC activity through the Java agent and configure a supported PostgreSQL integration for database-level telemetry.

Investigate:

Query latency.
Active connections.
Connection pool saturation.
Locks and deadlocks.
Database availability and resource utilization.

Documentation:

PostgreSQL monitoring integration

Cassandra

Use compatible driver instrumentation and an appropriate Cassandra/JMX integration.

Monitor:

Read and write latency.
Node availability.
Connection failures and timeouts.
Pending compactions and dropped messages.
Cassandra JVM health.

Documentation:

Cassandra monitoring integration

Browser monitoring

Install New Relic Browser monitoring for Vue.js, FreeMarker-rendered pages, and jQuery-based interfaces.

Track:

Page load time.
JavaScript exceptions.
Failed API requests.
Slow user interactions.
Frontend-to-backend traces where supported and configured.

Documentation:

Browser monitoring installation

8. NRQL Examples

These queries are starting points. Validate application names, event types, and available attributes against the telemetry collected by your account.

API response time
sql
SELECT average(duration), percentile(duration, 95)
FROM Transaction
WHERE appName = 'SIM-API-PROD'
SINCE 60 minutes ago
TIMESERIES

Slow transactions
sql
SELECT average(duration), percentile(duration, 95)
FROM Transaction
WHERE appName = 'SIM-API-PROD'
FACET name
SINCE 60 minutes ago
LIMIT 20

Transaction errors
sql
SELECT count(*)
FROM TransactionError
WHERE appName = 'SIM-API-PROD'
SINCE 60 minutes ago
TIMESERIES

Database time by transaction
sql
SELECT average(databaseDuration)
FROM Transaction
WHERE appName = 'SIM-API-PROD'
FACET name
SINCE 60 minutes ago
LIMIT 20


Use New Relic's query builder to inspect available attributes if a query returns no results.

Documentation:

NRQL introduction

9. LLM and MCP Integration
9.1 What MCP provides

The Model Context Protocol provides a standardized way for AI clients to discover and invoke tools and retrieve context from connected systems.

For SIM, an AI client can use MCP tools to:

Query approved New Relic telemetry.
Retrieve recent alerts and incidents.
Investigate errors and slow transactions.
Inspect Kubernetes status through a separate, controlled integration.
Retrieve service ownership and release metadata.
Search internal troubleshooting runbooks.

The LLM selects tools based on the user's request and interprets the returned evidence. The MCP server executes the permitted operations.

9.2 Option A — Connect directly to New Relic MCP

This is the recommended first implementation because it does not require building a custom MCP server.

Official resources:

New Relic MCP overview
New Relic MCP setup
New Relic MCP tool reference

For example, in a supported Codex CLI environment, configure the official server:

bash
codex mcp add newrelic --url https://mcp.newrelic.com/mcp/


Then authenticate using the documented OAuth flow for your installed client version.

The configuration is user-specific; it does not automatically deploy MCP inside the SIM application or expose custom SIM business tools.

New Relic supports additional clients, including VS Code and other documented AI development environments. Follow the setup instructions for your selected client.

Use a dedicated identity with appropriate read permissions and enable only the features required by your organization.

9.3 Option B — Build a SIM-specific MCP server

Once the direct integration works, consider creating a separate service such as:

sim-observability-mcp

Potential responsibilities:

Expose approved SIM service metadata.
Provide predefined, parameterized New Relic queries.
Retrieve approved Kubernetes status information.
Search operational runbooks.
Correlate incident data with deployment metadata.
Return structured evidence to the AI client.

Possible technology choices:

Java and Spring Boot.
Spring AI MCP support or another compatible MCP SDK.
A supported LLM API or internal model gateway.
New Relic APIs or the official New Relic MCP server.
Existing identity and authorization infrastructure.

Keep this service separate from the core supplier business API initially. It should not need direct access to supplier bank details, credentials, or unrestricted production data.

9.4 Suggested MCP tool contract

Design a small, explicit tool surface instead of exposing arbitrary infrastructure access.

Tool	Purpose	Initial access
get_service_health	Summarize service health for an allowlisted SIM service	Read-only
get_api_performance	Retrieve latency and throughput for approved endpoints	Read-only
get_recent_errors	Retrieve summarized error information	Read-only
get_kubernetes_status	Inspect approved namespaces and workloads	Read-only
get_deployment_history	Retrieve approved release metadata	Read-only
search_runbooks	Find documented troubleshooting procedures	Read-only
get_supplier_workflow_metrics	Retrieve aggregated onboarding and approval metrics	Read-only

Each tool should have a documented input schema, output schema, access policy, and audit trail.

Do not allow the model to construct unrestricted shell commands, execute arbitrary Kubernetes commands, access arbitrary database records, or run unrestricted production queries.

10. LLM-Assisted Incident Investigation
Example: Slow supplier onboarding

User request

Investigate why supplier onboarding has become slower in production over the last 30 minutes.

Suggested workflow

Identify the correct production service and time window.
Query approved New Relic metrics for throughput, latency, and errors.
Identify the slowest relevant transactions.
Inspect relevant traces and database spans.
Retrieve Kubernetes pod health and recent deployment metadata.
Compare the observed symptoms with internal runbooks.
Summarize the evidence, probable causes, confidence, and recommended next steps.

Example output structure

text
Incident: Increased supplier onboarding latency

Environment: Production
Time window: Last 30 minutes

Observed symptoms:
- API p95 latency increased.
- Supplier onboarding throughput changed.
- Database call duration increased for the affected transaction.

Evidence:
- Relevant transaction metrics and time window.
- Trace or error references.
- Kubernetes events, if relevant.
- Recent deployment information, if available.

Potential causes:
- Database query regression.
- Connection pool saturation.
- Resource contention in application pods.
- A downstream dependency slowdown.

Recommended next steps:
1. Verify the relevant transaction traces.
2. Inspect connection pool and database metrics.
3. Compare performance before and after deployment.
4. Follow the approved remediation runbook.

Confidence: Based on the available evidence.


This is an output template, not a claim that these conditions currently exist. The assistant must populate it only with observations returned by connected tools and clearly label hypotheses.

Example: Kubernetes pod instability

Question:

Which SIM pods have restarted frequently, and do their restart times correlate with API errors?

The AI agent should retrieve authorized pod status and event data, compare the relevant time windows with New Relic error telemetry, and provide evidence-backed findings. It should not restart or delete pods automatically.

11. LLM Integration Patterns

There are two reasonable ways to introduce an LLM.

Pattern A — Engineer-facing assistant

An engineer uses a supported AI client connected to New Relic MCP and, optionally, an approved Kubernetes or runbook MCP server.

Advantages:

Fastest implementation.
Minimal changes to the existing Java application.
Useful for debugging, on-call support, and incident analysis.
Pattern B — Embedded SIM operations assistant

Build an internal web interface and a backend orchestration service.

text
Vue.js Operations UI
        |
        v
Spring Boot AI Orchestrator
        |
        +--> LLM Provider / Model Gateway
        |
        +--> New Relic MCP Client
        |
        +--> SIM Runbook MCP Client
        |
        +--> Read-only Kubernetes Tools
        |
        v
Evidence-based Response


Advantages:

Integrates with existing Keycloak authentication.
Supports SIM-specific terminology and workflows.
Enables controlled access by team, service, and environment.
Provides a foundation for approved incident workflows.

The model provider and MCP transport are separate architectural choices. A model must be used through a client or orchestration layer that supports the selected tools and security requirements.

12. Security Requirements

Security is essential because the SIM platform may process confidential supplier information.

Identity and authorization
Use Keycloak or the approved enterprise identity provider for the internal assistant.
Enforce authorization in the backend and MCP server, not only in the LLM prompt.
Use least-privilege New Relic identities.
Restrict Kubernetes access to specific namespaces and read-only resources.
Keep development, testing, and production access separate.
Sensitive data

Never send the following to the LLM unless there is a formally approved, specifically justified process:

Passwords and credentials.
Keycloak access tokens and refresh tokens.
JWTs and authorization headers.
Supplier bank account information.
Sensitive supplier contact or personal data.
Unredacted request bodies or database records containing confidential data.
Tool safety
Start with read-only tools.
Validate all inputs and constrain time windows, query scope, and result sizes.
Treat telemetry, logs, traces, and runbook content as untrusted data.
Prevent prompt injection in retrieved content from changing tool permissions.
Require human approval for any future write action.
Log tool invocation metadata and authorization decisions without logging secrets.
Review LLM provider retention, processing, and data-handling policies.

The LLM should not be the final authority for supplier approval, compliance decisions, or production infrastructure changes.

13. Dashboards and Alerts

Recommended New Relic dashboard sections:

Section	Metrics
API performance	Throughput, average latency, p95 latency
Application reliability	Error rate and failed transactions
JVM health	Heap usage, garbage collection, threads
Kubernetes	Pod restarts, CPU, memory, replica health
PostgreSQL	Query latency and connection health
Cassandra	Read/write latency and node health
Browser	Page load time and JavaScript errors
Business operations	Onboarding duration and approval failure rate

Example initial alert objectives:

API p95 latency above the agreed service threshold.
Sustained increase in application error rate.
Pod restart or availability anomalies.
Database connectivity failures.
Supplier onboarding failures above the agreed baseline.

Tune thresholds against actual traffic, service-level objectives, and operational requirements.

The AI assistant can summarize alerts and suggest investigation steps, but existing alerting and incident-management controls must remain operational independently of the LLM.

14. Suggested Repository Structure
text
sim-observability/
├── README.md
├── monitoring/
│   ├── newrelic/
│   │   └── README.md
│   ├── dashboards/
│   │   └── README.md
│   └── alerts/
│       └── README.md
├── k8s/
│   ├── sim-api-deployment.yaml
│   └── README.md
├── ai/
│   ├── architecture/
│   │   └── README.md
│   ├── mcp/
│   │   └── README.md
│   ├── prompts/
│   │   └── incident-investigation.md
│   └── security/
│       └── README.md
└── runbooks/
    ├── high-api-latency.md
    ├── pod-restarts.md
    └── database-connectivity.md


This is a suggested layout. Add configuration files and code as the implementation evolves, and do not commit credentials, production data, or unredacted logs.

15. Implementation Checklist
Observability
Install and verify the New Relic Java agent.
Confirm Jetty transaction instrumentation.
Install Kubernetes monitoring.
Configure PostgreSQL and Cassandra integrations as appropriate.
Configure Browser monitoring.
Create NRQL queries and dashboards.
Configure alerts and notifications.
MCP and AI
Select a supported AI client and LLM.
Connect the official New Relic MCP server.
Verify authentication and read permissions.
Test natural-language telemetry queries.
Define the incident-investigation workflow.
Add runbook retrieval.
Evaluate whether custom SIM-specific MCP tools are needed.
Add audit logging and access controls.
Test incorrect assumptions, missing telemetry, and prompt-injection scenarios.
Production readiness
Review data retention and LLM provider policies.
Validate sensitive-data filtering.
Verify namespace and service-level authorization.
Test failure behavior when the LLM or MCP server is unavailable.
Require approval for any operational write action.
Document rollback and incident response procedures.
16. Official Documentation
New Relic
Java agent installation
Kubernetes monitoring
New Relic MCP overview
New Relic MCP setup
New Relic MCP tool reference
NRQL documentation
MCP and AI
Model Context Protocol
MCP security guidance
OpenAI Agents SDK MCP documentation
17. Final Recommendation

Implement the platform in this order:

Establish reliable New Relic telemetry for Jetty and Kubernetes.
Configure dashboards, NRQL queries, and alerts.
Connect an engineer-facing AI client to the official New Relic MCP server.
Validate evidence-based incident investigation with real operational scenarios.
Add custom SIM MCP tools only when business context or data sources are not already available through existing integrations.
Introduce an embedded operations assistant only after security, authorization, and audit requirements are established.

The expected outcome is a SIM platform where engineers can investigate operational issues using natural language, backed by actual telemetry, controlled tool access, and documented troubleshooting procedures.

Maintainer note: Update service names, environment names, deployed versions, authentication configuration, and operational procedures to match the actual implementation.
