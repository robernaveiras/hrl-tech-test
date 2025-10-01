# System Design for Custom Domain HTTPS Serving

---

## System Design for Custom Domain HTTPS Serving
We have two core problems, secure the domain with HTTPS using a valid SSL certificate, and route traffic to the customer's career site. Given the <5 minute requirement, the system must be highly automated and leverage Infrastructure as Code (IaC) and cloud-native services.

The chosen design centers on using an Application Load Balancer (ALB) for the public-facing endpoint, coupled with an automated certificate management service, likely Let's Encrypt via a cloud provider's managed service or an orchestrator like cert-manager.

---

## 1. High-Level Architecture Overview

| Step | Component | Action | Time Constraint |
|------|-----------|--------|-----------------|
| 0. Pre-requisite | Customer | Configures a CNAME record in their DNS pointing to our system (e.g., `jobs.mycompany.com → custom-ingress.ourplatform.com`). | External (Customer responsibility) |
| 1. Trigger | UI / API | Submits the custom domain name (e.g., `jobs.mycompany.com`) to the backend. | Seconds |
| 2. Provisioning | Automation Service (e.g./Lambda | Initiates the certificate issuance and domain/routing configuration. | ≈1−3 minutes |
| 3. Validation & Issuance | ACME Client/Cloud Service (e.g., AWS Certificate Manager (ACM) with DNS validation) | Verifies domain ownership (via DNS challenge) and issues the SSL certificate. | ≈1−2 minutes |
| 4. Deployment | Reverse Proxy/Load Balancer (e.g., ALB/NGINX/CDN) | Attaches the new certificate and updates routing rules to serve the customer's content. | Seconds |
| 5. Availability | Global Platform | Customer traffic starts flowing securely to the correct content. | <5 minutes total |

---

## 2. Technical Components & Services

| Component | Technology/Service | Rationale |
|-----------|--------------------|-----------|
| API Backend | Go/Python Service | Receives the customer's domain request, acts as the system's entry point. |
| Infrastructure as Code (IaC) | Terraform | Automates the creation and modification of cloud resources (DNS records, Load Balancers, Target Groups, etc.). Ensures an auditable and repeatable process. |
| Certificate Management | AWS Certificate Manager (ACM), Google Load Balancer/Cloud Run| Cloud-native services simplify and accelerate the Let's Encrypt (or similar) process, especially with DNS validation which is the fastest method. ACM is preferred in AWS for its automatic renewal and integration. |
| Reverse Proxy/Ingress | Cloud Load Balancer (e.g., AWS ALB) | Terminates SSL/TLS traffic, inspects the Server Name Indication (SNI) header, and routes the request to the correct backend service/pod, ensuring multi-tenancy. |
| State Machine/Workflow | simple Message Queue (SQS/Redis) | Orchestrates the multi-step provisioning process (Trigger → IaC → Cert Request → Deployment). |

---

## 3. Detailed Provisioning Workflow

The system must handle two primary technical challenges: **Domain Validation/Certificate Issuance** and **Traffic Routing**.

### A. The Fast Certificate Challenge: DNS Validation
To meet the <5 minute goal, HTTP-01 validation (serving a file) is too slow, as it requires setting up the web server first. DNS-01 validation is much faster.

1. **Request Initiation**: The API receives `jobs.mycompany.com` and validates it (e.g., sanity check, basic syntax).  
2. **DNS Record Creation**: The API triggers an automation service (e.g., a Lambda function) that uses Terraform to interact with the chosen Certificate Manager (e.g., ACM).  
3. **Validation Token**: AWS Certificate Manager generates a CNAME validation record (e.g., `_abcdef.jobs.mycompany.com → _xyz123.acm.aws`).  
4. **Automatic DNS Update**: The automation service uses Terraform to update our internal DNS zone (if we control the customer's CNAME target) or provides the customer with a required CNAME record.  

⟹ **Best Approach**: Assume the customer points their domain's CNAME to a domain we own (`custom-ingress.ourplatform.com`). We then use an ACM Private CA or Let's Encrypt with DNS Validation on our CNAME target's domain to issue the certificate. This is the only way to meet the time constraint without relying on immediate customer action.

### B. The Traffic Routing Update
1. **Certificate Ready**: Once the certificate for `jobs.mycompany.com` is issued, the workflow proceeds.  
2. **IaC Configuration Update**: Terraform updates the configuration of the Reverse Proxy/Ingress.  
   - **ALB/Cloud Load Balancer**: A new Listener Rule is added to the ALB that matches the Host header (`jobs.mycompany.com`) and points to the correct Target Group. The new certificate is attached to the HTTPS Listener.  
3. **Deployment**: IaC changes applied — typically seconds for routing and certificate attachment.  

---

## 4. Post-Provisioning and Maintenance

- **Automatic Renewal**: Using ACM or cert-manager ensures certificates are renewed automatically.  
- **Revocation/Deletion**: When a customer removes a domain, the API triggers IaC to remove the corresponding Listener Rule or Ingress and the certificate.  
- **Monitoring and Alerting**: Critical alerts must be configured for:  
  - Failed Certificate Issuance/Validation (DNS misconfiguration).  
  - High Latency on the custom domain endpoint.  
  - Certificate Expiration (safeguard, though renewal is automatic).  

---

## ✅ Outcome
This design prioritizes **speed, automation, and stability** by leveraging cloud-native managed services and an existing multi-tenant ingress layer — meeting the challenging **<5 minute requirement**.

