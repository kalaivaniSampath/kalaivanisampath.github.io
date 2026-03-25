# Product Registration Solution: Technical Design and Business Guide

Version: 1.0
Last Updated: 2026-03-25
Owners: Product/Service Ops, Salesforce Platform Team

## 1) Executive Summary

This document explains the end-to-end Product Registration experience for business stakeholders and technical teams. It covers the user flow, Lightning Web Components (LWCs), Flow orchestration, and Apex service, with security, data model, error handling, accessibility, deployment, and testing guidance.

Scope includes:
- LWCs: `productRegistration`, `productRegistrationPurchaseInfo`, `productRegistrationSummary`, `productRegistrationThankYou`, `webBarcodeScanner`, `addressValidationCustomerCreate`
- Apex: `ProductScannerController`
- Flow: `Asset_Product_Registrtaion_Custom_Design_Final`

Primary objective: customers or agents can register purchased products, validate purchase/address details, and receive confirmation. 

## 2) High-Level Architecture

- UI: Lightning Web Components in Lightning App Pages or Record Pages
- Orchestration: Screen Flow handles record creation and business rules
- Services: Apex used only where Lightning Data Service (LDS) can’t fulfill requirements (e.g., barcode resolution, external address validation)
- Security: Field-Level Security (FLS), org sharing, and LDS-first approach

Key Salesforce Objects:
- Account, Contact, Asset, Product2
- Optional: Order/OrderItem or Opportunity/OpportunityLineItem (purchase source), Case (welcome/support tracking)

## 3) End-to-End User Journeys

### 3.1 Standard Registration
1) productRegistration
- Enter product code/serial or launch web-based scanner
- Validate code format; proceed on success

2) productRegistrationPurchaseInfo
- Provide purchase date, retailer, proof-of-purchase info
- Optionally launch addressValidationCustomerCreate for standardized address

3) productRegistrationSummary
- Review consolidated product, purchase, and (optional) address details
- Accept terms if applicable; Submit → invokes Flow

4) productRegistrationThankYou
- Display success, Asset number, warranty dates, actionable links

### 3.2 Barcode-First
- From productRegistration, launch `webBarcodeScanner`
- Capture code, return value to pre-populate fields
- Follow steps 2–4 as above

## 4) Component Specifications

### 4.1 productRegistration (Entry)
Purpose
- Entry point; capture or scan product identifiers

Key Features
- Serial/Barcode input
- “Scan Barcode” → opens `webBarcodeScanner`
- Basic format validation; next navigation

Data Access
- Prefer LDS GraphQL (`lightning/graphql`) or record adapters to resolve product context
- Fall back to Apex (`ProductScannerController`) only if LDS cannot fulfill

Inputs/Outputs
- Output context (example):
  ```json
  {
    "productCode": "ABC-123",
    "serialNumber": "SN-001",
    "productId": "01t...",
    "userContactId": "003..."
  }
  ```

Errors
- Invalid code → inline guidance
- Permission/network issues → toast + retry

Accessibility
- Labeled inputs, keyboard focus order, descriptive error text

### 4.2 webBarcodeScanner
Purpose
- Capture barcodes with device camera; support desktop and mobile

Key Features
- Permission prompt, live preview, capture/retake
- Parse common symbologies (Code128, EAN-13; QR optional)

Outputs (events)
- `scancomplete`: `{ rawValue, symbology }`
- `scanerror`: `{ message }`

Errors
- No camera: fallback to manual entry
- Denied permission: guidance to enable camera or use manual entry

Security
- No image storage; in-memory processing only

Accessibility
- Clear instructions; ensure sufficient contrast and visible focus

### 4.3 productRegistrationPurchaseInfo
Purpose
- Collect purchase metadata and optionally validate address

Key Fields
- Purchase Date (required; not in future)
- Retailer (picklist or text with suggestions)
- Proof-of-Purchase number or reference

Address Validation
- Embed/launch `addressValidationCustomerCreate`
- Receives normalized address + verification status

Data Access
- LDS for lookups and picklists

Outputs
- Extend context with purchase and address info:
  ```json
  {
    "purchaseDate": "2026-03-01",
    "retailer": "Retailer Name",
    "proofNumber": "INV-123",
    "address": { "...": "..." },
    "addressVerified": true
  }
  ```

Errors/Accessibility
- Field-level messages; SLDS error styling
- Accessible date picker and error summaries

### 4.4 addressValidationCustomerCreate
Purpose
- Validate and normalize customer address for warranty/shipping

Approach
- If purely standard fields: collect and echo back standardized result
- If external provider used: call Apex with Named Credentials

Data Access
- LDS for metadata/picklists
- Apex for external callouts (if any)

Outputs
- Standardized address, verification codes, confidence score

Errors/Security
- If validation fails: allow continuation with warnings
- Secure secrets via Named Credentials; no secrets in client

### 4.5 productRegistrationSummary
Purpose
- Review and confirm submission; editable by section if needed

Features
- Summary of product, purchase, address
- Consent checkboxes if applicable
- Submit → invokes Flow: `Asset_Product_Registrtaion_Custom_Design_Final`

Data Access
- Prefer Flow for DML; LWC serves as presenter/controller

Outputs
- Receives Flow result: `assetId`, `status`, `message`

Errors
- Present detailed create/validation errors with retry options

### 4.6 productRegistrationThankYou
Purpose
- Confirm registration; provide next steps

Content
- Success, Asset Number, Product Name, Warranty Start/End
- Links: View Asset, Register another product, Download confirmation

Behavior
- Optionally show warranty caveats or support resources

## 5) Apex Service: ProductScannerController

Purpose
- Server-side product resolution for barcodes/serials when LDS can’t fulfill
- Optional: duplicate checks and guarded record creation (if Flow not used)

Contract (business-level)
- `resolveProductByCode(code: String) → Product2 summary`
- `checkDuplicateRegistration(key: String, contactId?: Id) → { isDuplicate: Boolean, assetId?: Id }`
- Optional: `createOrUpdateAsset(context: RegistrationContext) → Asset Id` (use Flow preferred)

Security & Limits
- `with sharing`; user-mode or `WITH SECURITY_ENFORCED` queries
- Check FLS before read/write
- Bulk-safe; no SOQL/DML in loops

Errors
- Throw custom exceptions mapped to friendly messages in UI/Flow

## 6) Flow: Asset_Product_Registrtaion_Custom_Design_Final

Type
- Screen Flow (primary) with optional subflows

Responsibilities
- Ensure/locate Contact and Account (if needed)
- Create Asset with Product2 link, Account, Contact, Serial, Purchase Details
- Optional: Case creation for welcome/support
- Branch for duplicates, warranty eligibility

Inputs (from LWCs)
- `varProductId` or `varProductCode`
- `varSerialNumber`
- `varPurchaseDate`
- `varRetailer`
- `varContactId` / `varAccountId` or raw customer data
- `varAddressJson` (normalized address)

Outputs (to LWCs)
- `varAssetId`
- `varStatus`
- `varMessage`

Best Practices
- Subflows: EnsureCustomer, CheckDuplicate, CreateAsset
- Fault screens with recovery instructions
- Decision elements with clear outcomes

## 7) Data Model and Mapping

Asset (core fields)
- Name: Product + Serial or Serial
- Product2Id: Resolved product
- AccountId, ContactId: Owner/contact
- SerialNumber (standard)
- PurchaseDate__c (custom)
- Retailer__c (custom)
- ProofOfPurchase__c (custom)
- Shipping/Mailing Address or related Address object
- WarrantyStartDate__c, WarrantyEndDate__c (custom if not using Entitlements)

Duplicate Strategy
- Key: (SerialNumber + Product2Id + ContactId/AccountId)
- On duplicate, present existing Asset link instead of creating new

## 8) Security, Compliance, Governance

- Enforce FLS/sharing via LDS; Apex with `with sharing` and `WITH SECURITY_ENFORCED`
- No hardcoded IDs or URLs
- Minimize PII; protect sensitive fields
- Audit: enable Field History on critical Asset fields
- Permission Set: “Product Registration User” with necessary CRUD/FLS

## 9) Error Handling and Messaging

Common Errors and Guidance
- Camera not available → “Enter the product code manually.”
- Product not found → “We couldn’t find that product. Check the code.”
- Duplicate registration → “Already registered. Asset: {link}”
- Address validation failed → “Continue or try again.”
- Record creation failed → “Please try again or contact support.”

Operational Logging (optional)
- Platform Events or custom logging object for telemetry

## 10) Accessibility and UX Standards

- SLDS base components and utilities
- Proper labels, ARIA attributes, keyboard navigation
- Focus management for modals/scanner
- Contrast and readable error summaries

## 11) Non-Functional Requirements

- Performance: Sub-2s typical server actions; smooth scanning with graceful fallback
- Scalability: Bulk-safe Apex; resilient Flow
- Observability: Clear errors; admin-friendly diagnostics

## 12) Deployment and Configuration

Metadata to Deploy
- LWC bundles: all components with .html/.js/.css/.js-meta.xml
- Apex: `ProductScannerController.cls` and -meta.xml
- Flow: `Asset_Product_Registrtaion_Custom_Design_Final` (activate post-deploy)
- Permission Set: Product Registration User
- Optional Named Credentials for address validation

Post-Deploy Steps
- Assign permission set to users
- Place `productRegistration` on target Lightning pages
- Verify Flow input/output variables and activation
- Test barcode scanning on supported browsers/devices

## 13) Reporting and KPIs

Recommended Reports/Dashboards
- Registrations by Product and Month
- Duplicate Attempt Rate
- Address Validation Success Rate
- Registration Source (scan vs manual)

Ensure needed fields captured in Section 7 are reportable.

## 14) Test Strategy

LWC Jest
- Validate input flows, scanner events, summary/submit, error states

Apex Unit Tests
- `resolveProductByCode` happy/negative
- Duplicate detection
- Asset creation (if implemented in Apex)

Flow Tests
- Validate decisions, subflow outcomes, and faults

Coverage
- ≥75% across Apex; meaningful assertions everywhere

## 15) Open Questions & Assumptions

- Authoritative purchase source (Order vs Opportunity)?
- Address validation provider and response schema?
- Duplicate policy scope (per Account/Contact vs org-wide)?
- Warranty policy: static duration per Product vs Entitlement process?

## Appendix A: Proposed Flow Variables

Inputs
- `varProductId` (Text/Id)
- `varProductCode` (Text)
- `varSerialNumber` (Text)
- `varPurchaseDate` (Date)
- `varRetailer` (Text)
- `varContactId` (Id)
- `varAccountId` (Id)
- `varAddressJson` (Text; JSON)

Outputs
- `varAssetId` (Id)
- `varStatus` (Text)
- `varMessage` (Text)

## Appendix B: Inter-Component Events

- productRegistration → productRegistrationPurchaseInfo: navigate with context payload
- productRegistrationPurchaseInfo → addressValidationCustomerCreate: open modal; resolve returns normalized address
- productRegistrationSummary → Flow: invoke with assembled inputs; handle output events

## Appendix C: Governance Checklist

- LWCs use LDS-first; Apex only when necessary
- Apex with sharing; `WITH SECURITY_ENFORCED` for queries under user context
- No SOQL/DML in loops; bulk-safe
- No hardcoded IDs or URLs
- Accessibility and SLDS adherence verified
- Unit tests and Flow tests in place; coverage ≥75%
