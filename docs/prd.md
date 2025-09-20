### **Payment System**

- Version: v1.0.0
- Date: 2025-09-20

#### **1. Introduction**

**1.1. Project Overview:**
This document specifies the requirements for a Payment System designed to function as an **abstraction layer** between merchants and various external Payment Service Providers (PSPs). The system's core purpose is to decouple the merchant's business logic from the complexities of individual payment gateways. It will provide a single, unified integration point for initiating, tracking, and reconciling payments, while centralizing transaction logging in an immutable, auditable ledger.

**1.2. Key Terminology:**
*   **PSP (Payment Service Provider):** A third-party company that handles payment processing (e.g., Stripe, PayPal).
*   **Merchant:** The business entity receiving payments.
*   **Client/User:** The end-user making a purchase.
*   **Ledger:** An immutable, append-only system for recording financial transactions, based on double-entry bookkeeping principles.
*   **Idempotency Key:** A unique client-generated key (e.g., UUID) used to ensure that API requests can be safely retried without creating duplicate operations.
*   **Payment Integration Methods:** The technical approach used to connect the merchant's system to the payment service (e.g., Server-to-Server, SDK, Hosted Redirect).
*   **Settlement File:** A report provided by a PSP detailing the transactions that have been successfully settled and the funds transferred to the merchant's account.
*   **State Machine:** The defined set of states a payment can be in and the valid transitions between those states.

---

#### **2. System Goals and Objectives**

*   **Goal 1:** Provide a standardized, RESTful API that abstracts away the differences between various PSPs, allowing merchants to integrate once and support multiple payment providers.
*   **Goal 2:** Achieve and maintain PCI DSS compliance by designing the system to minimize its scope, primarily by ensuring raw credit card data never touches the application servers.
*   **Goal 3:** Implement an event-sourced ledger that serves as the immutable source of truth for all transactions, ensuring auditability and correctness.
*   **Goal 4:** Guarantee "exactly-once" processing for all payment and refund operations through robust idempotency and retry mechanisms.

---

#### **3. Functional Requirements**

**FR1: Merchant API Operations**
*   **FR1.1: Initiate Payment Request**
    *   **Endpoint:** `POST /v1/charges`
    *   **Request Body:** `{ "amount": 1000, "currency": "USD", "order_id": "ORD-12345" }`
    *   **Request Headers:** `Idempotency-Key: <unique-key>`
    *   **Success Response:** `{ "charge_id": "chg_abc123", "status": "created", "client_secret": "<for-sdk-use>" }`
*   **FR1.2: Query Payment Status**
    *   **Endpoint:** `GET /v1/charges/{charge_id}`
    *   **Success Response:** Returns the full payment object including its current status, amount, currency, and timestamps.
*   **FR1.3: Initiate Refund**
    *   **Endpoint:** `POST /v1/refunds`
    *   **Request Body:** `{ "charge_id": "chg_abc123", "amount": 500 }` (Amount is optional for full refunds)
    *   **Request Headers:** `Idempotency-Key: <unique-key>`
    *   **Action:** Creates a new refund transaction linked to the original charge.

**FR2: Payment Integration Methods**
The system will support three primary integration methods:
*   **FR2.1: Client-Side SDK (Recommended):** The merchant integrates a provided JS SDK. The SDK collects card details within a secure iframe, sends them directly to the PSP to get a token, and returns the token to the client. The merchant's backend then sends this token to our Payment Service to create the charge. This method offloads the majority of PCI compliance from the merchant.
*   **FR2.2: Hosted Redirect:** For the simplest integration, the merchant initiates a payment session with our service, which returns a secure redirect URL. The client is redirected to the PSP's hosted payment page to complete the transaction. The PSP then notifies our system of the result via a webhook and redirects the user back to the merchant's site.
*   **FR2.3: Server-to-Server:** Reserved for merchants who are fully PCI compliant. The merchant sends raw payment details to our service, which then forwards them to the PSP. This method carries the highest security burden for the merchant.

**FR3: Payment Lifecycle and State Machine**
*   The system will model payments as a state machine. The primary states are: **`CREATED`**, **`PROCESSING`**, **`SUCCEEDED`**, **`FAILED`**.
*   **Transitions:**
    *   `[API Request]` -> **`CREATED`**: A new payment object is created.
    *   `CREATED` -> **`PROCESSING`**: After the payment is submitted to the PSP.
    *   `PROCESSING` -> **`SUCCEEDED`**: On receipt of a success webhook from the PSP.
    *   `PROCESSING` -> **`FAILED`**: On receipt of a failure webhook from the PSP.

**FR4: Reconciliation Process**
*   The system will include a scheduled job to automatically fetch (e.g., via SFTP) and parse daily settlement files from each integrated PSP.
*   The reconciliation service will compare the transactions in the settlement file against the `SUCCEEDED` transactions in the ledger for that period.
*   It will generate a daily reconciliation report detailing:
    *   Successfully matched transactions.
    *   Discrepancies (e.g., amount mismatch).
    *   Missing transactions (in ledger but not in settlement, or vice versa).

**FR5: Fraud Detection Engine**
*   The Risk Engine must be invoked synchronously before a payment request is sent to the PSP.
*   Initial rules will include:
    *   Blacklisting of known fraudulent IP addresses, email domains, and card BINs.
    *   Velocity checks (e.g., flagging >3 transactions from the same IP address in 1 hour).
*   A transaction flagged with a high fraud score will be immediately moved to the `FAILED` state with a reason of `fraud_risk`.

---

#### **4. Non-Functional Requirements**

**NFR1: Security**
*   The system is designed to operate within a PCI DSS Level 1 compliant environment.
*   The architecture's primary goal is to minimize PCI scope by never storing, processing, or transmitting raw PAN data on our servers. All sensitive cardholder data will be handled via PSP-provided tokens, iframes, or hosted fields.

**NFR2: Reliability**
*   **Availability:** The core payment processing APIs (`/charges`, `/refunds`) must maintain **99.95% uptime**.
*   **Failure Handling:** The system must be able to withstand and recover from temporary PSP outages. The retry mechanism will attempt to re-process failed requests for up to 48 hours before moving them to a Dead Letter Queue (DLQ) and triggering an alert.

**NFR3: Consistency & Correctness**
*   **Idempotency:** The `Idempotency-Key` provided by the merchant will be stored and checked for 24 hours. If a request with a duplicate key is received within this window, the system will return the original response without re-processing the request.
*   **Ledger Integrity:** The ledger must be append-only. All operations (charges, refunds, chargebacks) must be recorded as new, immutable entries.

**NFR4: Scalability & Performance**
*   **Latency:** The p99 latency for the `POST /v1/charges` API endpoint must be **under 200ms**.
*   **Throughput:** The system must be architected to handle an initial load of **500 Transactions Per Second (TPS)**, with the ability to scale horizontally.

---

#### **5. High-Level Architecture**

The system will be built on a service-oriented, event-driven architecture:
*   **Core Components:**
    *   **Client:** Integrates via SDK or is redirected.
    *   **Payment Service:** A stateless service that orchestrates the payment flow.
    *   **Risk Engine:** A dedicated service for real-time fraud analysis.
    *   **PSP Gateways:** Modules for integrating with different PSPs (e.g., StripeGateway, PayPalGateway).
    *   **Reconciliation Service:** A batch-processing service for handling settlement files.
*   **Data Layer:**
    *   **Event Store (Source of Truth):** An immutable log (e.g., Kafka, AWS QLDB) that captures every state change as a distinct event.
    *   **Read Models / Materialized Views:** Optimized data stores (e.g., PostgreSQL, DynamoDB) built by consuming events from the Event Store. These provide fast, queryable access for the Payment Service (`PaymentDB`) and for financial reporting (`Ledger`). This CQRS (Command Query Responsibility Segregation) pattern ensures that the write path is optimized for throughput and the read path is optimized for queries.
