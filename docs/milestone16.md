### **Architect Support for Hosted Redirect and Server-to-Server Integrations**

*   **Problem:** The current system exclusively supports a client-side SDK integration. To broaden our system's appeal and cater to merchants with different technical capabilities and compliance postures, we need to offer simpler (Hosted Redirect) and more direct (Server-to-Server) integration options.

*   **Solution:** We will enhance the `Payment Service` to support two additional, distinct payment flows:
    1.  **Hosted Redirect Flow:** A new endpoint, `POST /v1/payment_sessions`, will be created. The merchant calls this endpoint, and our service communicates with the PSP to generate a secure, one-time-use checkout URL. The merchant then redirects their customer to this URL. The PSP's hosted page handles all sensitive data collection. Upon completion, the PSP notifies our system of the outcome via the existing webhook infrastructure.
    2.  **Server-to-Server Flow:** The `POST /v1/charges` endpoint will be enhanced to accept raw cardholder data (PAN, CVC, etc.). This flow will be strictly firewalled and only accessible to merchants who are explicitly authorized and have demonstrated their own PCI compliance. The `Payment Service` will pass this data directly to the `PSP Gateway`. This significantly increases our system's PCI DSS scope.

*   **Trade-offs:**
    *   **Hosted Redirect:**
        *   **Pro:** Offers the simplest possible integration for merchants and reduces their PCI scope to the minimum (SAQ A).
        *   **Con:** The user is redirected away from the merchant's site, which can negatively impact conversion rates. We have less control over the user experience.
    *   **Server-to-Server:**
        *   **Pro:** Gives merchants who are PCI compliant complete control over their checkout experience.
        *   **Con:** This is a **major security and compliance trade-off**. By accepting raw cardholder data, our system's PCI scope expands dramatically, requiring much stricter security controls, auditing, and potentially dedicated infrastructure. This is a high-risk, high-complexity feature to implement and maintain.

---

#### **Logical View (C4 Component Diagram)**

This diagram shows the two new logical flows alongside the existing SDK flow, highlighting the different paths data takes for each integration method.

```mermaid
graph TD
    subgraph "FinCore Payment System"
        payment_service["Payment Service<br>[Container]<br><br>Handles all three integration flows."]
    end

    api_client["API Client<br>[Merchant's Backend]"]
    user_browser["User's Browser"]
    psp["Payment Service Provider"]

    subgraph "Flow 1: Hosted Redirect"
        direction LR
        api_client -- "1a. Creates Session" --> payment_service
        payment_service -- "1b. Returns Redirect URL" --> api_client
        api_client -- "1c. Redirects User" --> user_browser
        user_browser -- "1d. User pays on PSP page" --> psp
        psp -- "1e. Webhook" --> payment_service
    end

    subgraph "Flow 2: Server-to-Server"
        direction LR
        api_client -- "2a. Sends raw card data" --> payment_service
        payment_service -- "2b. Processes directly" --> psp
    end
    
    subgraph "Flow 3: Client-Side SDK (Existing)"
        direction LR
        user_browser -- "3a. Tokenizes card directly" --> psp
        psp -- "3b. Returns Token" --> user_browser
        user_browser -- "3c. " --> api_client
        api_client -- "3d. Sends Token" --> payment_service
    end
```

---

#### **Physical View (AWS Deployment Diagram)**

The physical architecture does not require new components but demands a significant increase in the security posture of the existing `Payment Service` to handle its expanded PCI scope. This is noted directly on the diagram.

```mermaid
graph TD
    subgraph "AWS Cloud"
        subgraph "VPC (Now a PCI-Compliant Environment)"
            subgraph "Public Subnet"
                alb["Application Load Balancer (ALB)"]
            end

            subgraph "Private Subnet (Application)"
                fargate_service["AWS Fargate Service"]
                task["Fargate Task<br>(Payment Service)<br><br><b>Scope Increased:</b><br>Now handles raw cardholder data.<br>Requires stricter security, logging, and auditing."]
                fargate_service -- "manages" --> task
            end
        end
    end

    api_client["Merchant's Backend"]
    user_browser["User's Browser"]
    psp_api["External PSP API"]

    api_client -- "Server-to-Server Flow (Raw Data)" --> alb
    user_browser -- "Redirect Flow" --> psp_api
    alb -- "forwards traffic" --> task
    task -- "calls" --> psp_api
```

---

#### **Component-to-Resource Mapping Table**

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **Payment Service** | **AWS Fargate Task** | **Role Expanded & Security Hardened:** The service logic is updated to handle all three integration flows. Critically, because it now processes raw cardholder data for the Server-to-Server flow, the entire environment (VPC, Fargate task, logging, etc.) must be brought into a stricter PCI DSS compliant scope, requiring more rigorous security controls, monitoring, and regular auditing. |
| **User's Browser** | **External User Agent** | (No change) Interacts differently depending on the chosen flow (redirected vs. using the SDK). |
| **API Client** | **External System** | (No change) Interacts differently depending on the chosen flow (sending raw data vs. a token). |
| **All other components**| (No change) | The existing databases, queues, and workers continue to function as previously designed. |
