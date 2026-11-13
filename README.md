# Security Assessment & Penetration Testing: nopCommerce

[![Target](https://img.shields.io/badge/Target-nopCommerce%20v4.70+-blue.svg)](https://github.com/nopSolutions/nopCommerce)
[![Framework](https://img.shields.io/badge/Framework-ASP.NET%20Core%20%7C%20.NET%2010-purple.svg)](https://dotnet.microsoft.com/)
[![Methodology](https://img.shields.io/badge/Methodology-OWASP%20WSTG%20v4.2-orange.svg)](https://owasp.org/www-project-web-security-testing-guide/)
[![Status](https://img.shields.io/badge/Status-Completed-success.svg)](#)

> **Acknowledgement & Credit:**  
> This security evaluation was conducted on [nopCommerce](https://github.com/nopSolutions/nopCommerce), a leading open-source e-commerce platform built on ASP.NET Core by [nopSolutions](https://www.nopcommerce.com). We express our appreciation to the nopCommerce team and community for providing a robust, enterprise-grade architecture for educational research, security auditing, and quality assurance practice.

---

## 📌 Executive Summary

This repository contains the deliverables, execution logs, and formal documentation for a **Grey-Box Web Application Penetration Test** conducted against a staging deployment of nopCommerce. 

The audit evaluated core e-commerce business flows, customer data isolation, authentication resilience, and input handling against the **OWASP Top 10 (2021)** framework and e-commerce business logic standards.

### Key Scorecard
* **Total Security Scenarios Tested:** 12
* **Passed Controls (Secure):** 10 (83.3%)
* **Identified Vulnerabilities:** 2 (16.7%)
  * 🔴 **1 High Severity:** Concurrency Race Condition on Single-Use Coupons (CVSS 7.5)
  * 🟢 **1 Low Severity:** Missing Rate Limiting on Customer Authentication (CVSS 3.7)

---

## 🔍 Highlight Finding: VULN-01 (High Severity)

### **Coupon Limit Overrun via Checkout Race Condition (TOCTOU)**
* **Vulnerability Type:** CWE-367: Time-of-Check to Time-of-Use (TOCTOU) Race Condition
* **CVSS v3.1 Score:** **7.5 (High)** — `CVSS:3.1/AV:N/AC:H/PR:L/UI:N/S:U/C:N/I:H/A:N`
* **Affected Endpoint:** `POST /checkout/OpcConfirmOrder/`
* **Target Components:** `OrderProcessingService.cs` & `DiscountService.cs`

#### **Summary & Root Cause:**
When a promotional discount is configured for single-use (`N times only = 1`), the application validates prior usage count before saving the order. However, the record of usage (`DiscountUsageHistory`) is only inserted into the database after order creation. 

By synchronizing parallel checkout confirmation requests using **Burp Suite Parallel Groups (Single-Packet Attack)**, two separate authenticated accounts simultaneously evaluated `Count = 0`, successfully placing two distinct orders (**Order #13 and Order #14**) at the exact same timestamp (`1:46:11 PM`) while redeeming the single-use voucher twice.

```text
Request A (Customer 10)               Request B (Customer 11)
      │                                     │
      ├─► 1. Check coupon count? (0)       │
      │      Valid!                         ├─► 2. Check coupon count? (0)
      │                                     │      Valid!
      ├─► 3. Save Order #13 (Discounted)    │
      │      Insert usage record (Count=1)  ├─► 4. Save Order #14 (Discounted)
      │                                     │      Insert usage record (Count=2)
```

#### **Remediation:**
Wrap the checkout validation and discount insertion sequence in an atomic transaction or acquire an explicit distributed lock (e.g., Redis distributed lock or database row lock on `DiscountId`) inside `OrderProcessingService.PlaceOrderAsync` before invoking `ValidateDiscountAsync`.

---

## 🛡️ Positive Security Findings (Defense-in-Depth)

The assessment verified that nopCommerce implements strong baseline protections across its core architecture:
1. **Broken Access Control (IDOR):** Direct object references on order details (`/orderdetails/{id}`) and printable invoices (`/orderdetails/pdf/{id}`) strictly validate customer ownership. Unauthorized requests are rejected with access denial notices.
2. **Context-Aware Output Encoding (XSS):** Dynamic user input across product reviews, customer profile attributes, order notes, and search queries are properly HTML-entity-encoded by the ASP.NET Core Razor view engine, preventing stored and reflected script execution.
3. **Price Integrity:** The shopping cart service recalculates order totals strictly from server-side database catalog entities, ignoring client-side parameter injection (`OrderTotal=0.01` or `CustomerEnteredPrice`).
4. **Checkout State Machine:** Direct attempts to invoke `OpcConfirmOrder` while bypassing payment steps are rejected by the server workflow state validator.

---

## 📂 Deliverables in this Repository

| File | Description | Format |
| :--- | :--- | :---: |
| [**nopCommerce_Penetration_Testing_Report_v1.1.docx**](./nopCommerce_Penetration_Testing_Report_v1.1.docx) | Formal executive penetration test report covering scope, methodology, full vulnerability write-ups with PoC, and code remediations. | Word / PDF |
| [**nopCommerce_Security_Test_Matrix_v1.1.xlsx**](./nopCommerce_Security_Test_Matrix_v1.1.xlsx) | Comprehensive test case management workbook featuring 9 tabs, test execution status, metrics, and defect registers. | Excel |

---

## 🛠️ Testing Tools & Environment

* **Target Application:** [nopCommerce v4.70+](https://github.com/nopSolutions/nopCommerce) running on ASP.NET Core MVC (.NET 10 Kestrel server)
* **Proxy & Fuzzing:** Burp Suite Community Edition (Proxy, Repeater Parallel Engine, Intruder)
* **DOM & Protocol Inspection:** Google Chrome DevTools
* **Test Tracking:** Microsoft Excel

---

## 👤 Author
* **Tester:** Phan Do Gia Tue
* **Assessment Date:** November 2026