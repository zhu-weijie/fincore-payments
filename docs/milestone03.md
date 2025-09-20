### **Architect the Secure Client-Side Tokenization Flow**

*   **Problem:** To meet our primary security goal of PCI compliance, our `fincore-payments` service must never store, process, or transmit raw credit card information. We need a secure architectural pattern that allows us to charge a customer's card without the sensitive data ever touching our servers.

*   **Solution:** We will implement a client-side tokenization flow. A lightweight "FinCore JS SDK" will be created and hosted by us. Merchants will embed this SDK on their checkout page. The SDK will render secure iframes provided by the PSP for card input fields. When the user submits their payment details, the SDK sends this information directly from the user's browser to the PSP, completely bypassing our backend. The PSP then returns a secure, one-time-use token to the SDK. This non-sensitive token is the only piece of information that the merchant's backend (the API Client) sends to our `Payment Service` to initiate the charge.

*   **Trade-offs:**
    *   **Integration Experience:**
        *   **Pro:** This model provides a seamless user experience, as the customer never leaves the merchant's website. It allows for significant UI customization to match the merchant's branding.
        *   **Con:** This requires more frontend integration effort from the merchant compared to a simple redirect.
    *   **Security & Compliance:**
        *   **Pro:** This is the industry-standard method for drastically reducing PCI DSS scope (to SAQ A-EP or SAQ A). It provides a very high level of security by ensuring sensitive data is isolated.
        *   **Con:** It introduces a third-party JavaScript dependency onto the merchant's page. We must ensure our SDK is delivered securely (e.g., via a CDN with Sub-resource Integrity) to mitigate supply-chain risks.

---

#### **Logical View (C4 Component Diagram)**

This diagram details the interactions between the new client-side components and the existing backend systems. The key architectural change is the direct communication between the browser and the PSP for tokenization.

```mermaid
graph TD
    subgraph "Client's Web Browser"
        merchant_frontend["Merchant Frontend<br>[JavaScript App]"]
        
        subgraph "FinCore JS SDK"
            direction LR
            sdk_core["SDK Core Logic"]
            psp_iframe["PSP's Iframe<br>(for card fields)"]
        end

        merchant_frontend -- "2. Initializes" --> sdk_core
        sdk_core -- "3. Renders" --> psp_iframe
    end

    subgraph "FinCore Payment System"
        payment_service["Payment Service<br>[Container]"]
    end

    api_client["API Client<br>[Merchant's Backend]"]
    psp["Payment Service Provider<br>[External System]"]

    merchant_frontend -- "1. Serves checkout page" --> User([User])
    User -- "4. Enters card details" --> psp_iframe
    psp_iframe -- "5. Sends details directly to PSP" --> psp
    psp -- "6. Returns payment_token" --> psp_iframe
    psp_iframe -- "7. Passes token" --> sdk_core
    sdk_core -- "8. Passes token" --> merchant_frontend
    merchant_frontend -- "9. Sends token to merchant backend" --> api_client
    api_client -- "10. Makes API call with token" --> payment_service
```

---

#### **Physical View (AWS Deployment Diagram)**

The physical view is updated to include the hosting and delivery of our static JS SDK assets via a Content Delivery Network (CDN). The backend infrastructure remains the same as in the previous issue.

```mermaid
graph TD
    subgraph "Client-Side"
        browser["User's Browser"]
    end
    
    subgraph "AWS Cloud (FinCore)"
        subgraph "Global Content Delivery"
            cdn["Amazon CloudFront (CDN)"]
            s3["Amazon S3 Bucket<br>(Hosts fincore.js)"]
            cdn -- "serves content from" --> s3
        end

        subgraph "VPC"
            alb["Application Load Balancer (ALB)"]
            task["Fargate Task<br>(fincore-payments container)"]
            alb --> task
        end
    end

    subgraph "Merchant Infrastructure"
        merchant_server["Merchant's Web Server"]
    end

    subgraph "PSP Infrastructure"
        psp_api["PSP API & Iframe Host"]
    end

    browser -- "1. GET page" --> merchant_server
    browser -- "2. GET fincore.js" --> cdn
    browser -- "3. GET iframe content" --> psp_api
    browser -- "4. POST card data for token" --> psp_api
    browser -- "5. POST token to merchant" --> merchant_server
    merchant_server -- "6. API Call" --> alb
```

---

#### **Component-to-Resource Mapping Table**

| Logical Component | Physical Resource | Rationale |
| :--- | :--- | :--- |
| **FinCore JS SDK** | **Amazon S3 + Amazon CloudFront (CDN)** | **Performance & Availability:** Hosting the static SDK file on S3 and distributing it globally via CloudFront ensures low-latency, reliable delivery to users anywhere in the world. This is a standard, cost-effective, and highly scalable solution for static asset hosting. |
| **PSP's Iframe** | **External SaaS (Hosted by PSP)** | **Security & Compliance:** This resource is hosted and managed entirely by the Payment Service Provider. We have no control over it. Its purpose is to create a secure, isolated DOM environment for sensitive data entry, which is the cornerstone of our PCI compliance strategy. |
| **Payment Service** | **AWS Fargate Task** | (No change from previous issue) |
| **API Client** | **External System** | (No change from previous issue) |
| **Payment Service Provider**| **External SaaS** | (No change from previous issue) |