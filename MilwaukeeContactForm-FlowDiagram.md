# MilwaukeeContactForm — End-to-End Flow Diagram

**Flow API Name:** `MilwaukeeContactForm`  
**LWC Component:** `c:milwaukeeContactForm`  
**Form Name Key:** `MWK_Contact`  
**Process Type:** Screen Flow  
**Run Mode:** `SystemModeWithoutSharing`  
**API Version:** 67.0  
**Status:** Draft

---

## Primary Flow — MilwaukeeContactForm

```mermaid
flowchart TD
    START([🟢 START])

    GET_CONFIG["⚡ Get MWK Form Config\n──────────────────────\nApex Action\nGenericFormConfigController\n─\nIN:  lang, formName\nOUT: labelsJson, picklistDataJson\n     language, formName\n→ MWKFormDataVar"]

    SCREEN["🖥️ Milwaukee Form Screen\n──────────────────────\nLWC: c:milwaukeeContactForm\n─\nIN:  formConfig = MWKFormDataVar\n     language, formName\nOUT: genericFormData → MWKFormDataVar\nshowHeader=false · showFooter=false"]

    GET_CONTACT["🔍 Get Contact By Email\n──────────────────────\nRecord Lookup → Contact\nWHERE Email = MWKFormDataVar.email\ngetFirstRecordOnly = true\nstoreOutputAutomatically = true"]

    CONTACT_DECISION{{"Contact\nExists?"}}

    CREATE_WITH["📝 Create Case With Account\n──────────────────────\nRecord Create → Case\nContactId = lookup.Id\nAccountId  = lookup.AccountId\nOrigin     = 'Web'\nSubject    = CaseSubject\nDescription = CaseDescription\nSuppliedEmail / SuppliedName / SuppliedPhone\n→ MWKFormDataVar (CreatedCaseId)"]

    CREATE_WITHOUT["📝 Create Case Without Account\n──────────────────────\nRecord Create → Case\nOrigin      = 'Web'\nSubject     = CaseSubject\nDescription = CaseDescription\nSuppliedEmail / SuppliedName / SuppliedPhone\n→ MWKFormDataVar (CreatedCaseId)"]

    ATTACH_DECISION{{"Has\nAttachment?"}}

    CREATE_CV["⚡ Create Content Version via Apex\n──────────────────────\nApex Action\nFileUploadDataConfigController\n─\nIN:  base64Data = MWKFormDataVar.attachmentBase64\n     fileName   = MWKFormDataVar.attachmentFileName\nOUT: contentVersionId → FileUploadDataVar.contentVersionId\n     success, errorMessage → FileUploadDataVar"]

    STAMP_CV["✏️ Stamp ContentVersionId on Case\n──────────────────────\nRecord Update → Case\nWHERE Id = CreatedCaseId\nSET ContentVersionId__c = FileUploadDataVar.contentVersionId\n→ Triggers Record-Triggered Flow"]

    THANKYOU["🖥️ Thank You Screen\n──────────────────────\n'Thank you for contacting Milwaukee Tool!'\nallowBack = false · allowFinish = true"]

    ERROR_ASSIGN["📋 ErrorMessage Assignment\n──────────────────────\nvT_Error = \$Flow.FaultMessage"]

    ERROR_SCREEN["🖥️ Error Screen\n──────────────────────\nDisplays: vT_Error\nallowBack = true · allowFinish = true"]

    %% ── Happy path ──────────────────────────────────────────────
    START          --> GET_CONFIG
    GET_CONFIG     --> SCREEN
    SCREEN         --> GET_CONTACT
    GET_CONTACT    --> CONTACT_DECISION
    CONTACT_DECISION -- "✅ Contact Exists\n(IsNull = false)" --> CREATE_WITH
    CONTACT_DECISION -- "❌ Contact Not Found\n(default)" --> CREATE_WITHOUT
    CREATE_WITH    --> ATTACH_DECISION
    CREATE_WITHOUT --> ATTACH_DECISION
    ATTACH_DECISION -- "📎 Has Attachment\n(fileName + base64 not null)" --> CREATE_CV
    ATTACH_DECISION -- "⬜ No Attachment\n(default)" --> THANKYOU
    CREATE_CV      --> STAMP_CV
    STAMP_CV       --> THANKYOU

    %% ── Fault paths ─────────────────────────────────────────────
    GET_CONFIG     -. "⚠️ fault" .-> ERROR_ASSIGN
    GET_CONTACT    -. "⚠️ fault" .-> ERROR_ASSIGN
    CREATE_WITH    -. "⚠️ fault" .-> ERROR_ASSIGN
    CREATE_WITHOUT -. "⚠️ fault" .-> ERROR_ASSIGN
    CREATE_CV      -. "⚠️ fault" .-> ERROR_ASSIGN
    STAMP_CV       -. "⚠️ fault" .-> ERROR_ASSIGN
    ERROR_ASSIGN   --> ERROR_SCREEN

    %% ── Styles ──────────────────────────────────────────────────
    classDef apexAction  fill:#dbeafe,stroke:#3b82d4,color:#1e40af,font-size:12px
    classDef screen      fill:#dcfce7,stroke:#166534,color:#14532d,font-size:12px
    classDef dml         fill:#f7f8fa,stroke:#6b7280,color:#1f2328,font-size:12px
    classDef decision    fill:#fff7ed,stroke:#c2410c,color:#7c2d12,font-size:12px
    classDef fault       fill:#fee2e2,stroke:#c8102e,color:#7f1d1d,font-size:12px
    classDef startend    fill:#1f2328,stroke:#1f2328,color:#ffffff,font-size:13px

    class START startend
    class GET_CONFIG,CREATE_CV apexAction
    class SCREEN,THANKYOU,ERROR_SCREEN screen
    class CREATE_WITH,CREATE_WITHOUT,STAMP_CV,GET_CONTACT dml
    class CONTACT_DECISION,ATTACH_DECISION decision
    class ERROR_ASSIGN,ERROR_SCREEN fault
```

> **Note:** Unlike the Ryobi flow, `Get_MWK_Form_Config` **has** a fault connector — if the Apex action fails, execution routes to `ErrorMessage` → `Error_Screen`.

---

## Supporting Flow — Link File to Case After Guest Upload

```mermaid
flowchart TD
    TRIGGER(["🔄 TRIGGER\n──────────────────────\nCase: RecordAfterSave\n(Create AND Update)\nWHEN ContentVersionId__c\nchanges to NOT NULL"])

    GET_CV["🔍 Get ContentVersion\n──────────────────────\nRecord Lookup → ContentVersion\nWHERE Id = \$Record.ContentVersionId__c\nRetrieves: ContentDocumentId"]

    CHECK_CDL{{"ContentDocument\nFound?"}}

    CREATE_CDL["📎 Create ContentDocumentLink\n──────────────────────\nRecord Create → ContentDocumentLink\nContentDocumentId = Get_ContentVersion.ContentDocumentId\nLinkedEntityId    = \$Record.Id (Case)\nShareType         = 'V'\nVisibility        = 'AllUsers'"]

    CLEAR_CV["✏️ Clear ContentVersionId\n──────────────────────\nRecord Update → Case\nWHERE Id = \$Record.Id\nSET ContentVersionId__c = ''"]

    END_STOP([⛔ STOP — ContentDocument not found])
    END_OK([✅ END])

    TRIGGER       --> GET_CV
    GET_CV        --> CHECK_CDL
    CHECK_CDL -- "✅ ContentDocument Found\n(IsNull = false)" --> CREATE_CDL
    CHECK_CDL -- "❌ Not Found\n(default)" --> END_STOP
    CREATE_CDL    --> CLEAR_CV
    CLEAR_CV      --> END_OK

    classDef dml      fill:#f7f8fa,stroke:#6b7280,color:#1f2328,font-size:12px
    classDef decision fill:#fff7ed,stroke:#c2410c,color:#7c2d12,font-size:12px
    classDef startend fill:#1f2328,stroke:#1f2328,color:#ffffff,font-size:13px
    classDef endpoint fill:#dcfce7,stroke:#166534,color:#14532d,font-size:12px
    classDef stopnode fill:#fee2e2,stroke:#c8102e,color:#7f1d1d,font-size:12px

    class TRIGGER startend
    class GET_CV,CREATE_CDL,CLEAR_CV dml
    class CHECK_CDL decision
    class END_OK endpoint
    class END_STOP stopnode
```

> **Why a separate flow?** Guest users have no `ContentDocument` visibility — they cannot query or link files in the same transaction. The main Screen Flow (running in `SystemModeWithoutSharing`) stamps `ContentVersionId__c` on the Case. This Record-Triggered Flow runs in a fresh **system context** to create the `ContentDocumentLink` and then clears the staging field to prevent re-triggering.

---

## Flow Variables — MilwaukeeContactForm

| Variable | Type | Input | Default | Purpose |
|---|---|---|---|---|
| `lang` | String | ✓ | `en` | Language code (e.g. `en`, `fr`). Also written back from the LWC output. |
| `formName` | String | ✓ | `MWK_Contact` | Identifies which `FormField__mdt` records to load. |
| `MWKFormDataVar` | Apex: `GenericFormConfig` | — | — | Dual-purpose: receives `labelsJson`/`picklistDataJson` from the Apex action; then holds all user-entered values written back by the LWC on screen Next. |
| `FileUploadDataVar` | Apex: `FileUploadDataConfig` | — | — | Receives `contentVersionId`, `success`, and `errorMessage` from the file-upload Apex action. |
| `CreatedCaseId` | String | — | — | Stores the newly created Case Id for the `Stamp_ContentVersionId_on_Case` update. |
| `vT_Error` | String | — | — | Captures `$Flow.FaultMessage` from any fault connector for the Error Screen. |

---

## Text Templates — MilwaukeeContactForm

| Name | Used As | Template |
|---|---|---|
| `CaseDescription` | Case `Description` | Plain-text block with all 18 form fields: First/Last Name, Email, Phone (+country code), Culture Code, Describe Yourself, Customer Type, Enquiry About, More Details, Company Name, Job Title, Number of Employees, Trade, City, Business Postcode, Dealer Customer Number, Order Number, Model Name, Milwaukee Contact Name, Attachment filename, form submit timestamp, Message. |
| `CaseSubject` | Case `Subject` | `{enquiryAbout} - {firstName} {lastName}` |
| `SuppliedName` | Case `SuppliedName` | `{firstName} {lastName}` |

---

## Element Summary — MilwaukeeContactForm

| # | Element | Type | Connects To |
|---|---|---|---|
| 1 | `Get_MWK_Form_Config` | Apex Action (`GenericFormConfigController`) | `MilwaukeeFormScreen` |
| 2 | `MilwaukeeFormScreen` | Screen (LWC: `c:milwaukeeContactForm`) | `Get_Contact_By_Email` |
| 3 | `Get_Contact_By_Email` | Record Lookup (Contact) | `Check_Contact_Exist_or_Not` |
| 4 | `Check_Contact_Exist_or_Not` | Decision | `Create_Case_With_Account` / `Create_Case_Without_Account` |
| 5 | `Create_Case_With_Account` | Record Create (Case) | `Check_If_Attachment_Exists` |
| 6 | `Create_Case_Without_Account` | Record Create (Case) | `Check_If_Attachment_Exists` |
| 7 | `Check_If_Attachment_Exists` | Decision | `Create_Content_Version_via_Apex` / `ThankYouScreen` |
| 8 | `Create_Content_Version_via_Apex` | Apex Action (`FileUploadDataConfigController`) | `Stamp_ContentVersionId_on_Case` |
| 9 | `Stamp_ContentVersionId_on_Case` | Record Update (Case) | `ThankYouScreen` |
| 10 | `ThankYouScreen` | Screen | _(end)_ |
| 11 | `ErrorMessage` | Assignment (`vT_Error = $Flow.FaultMessage`) | `Error_Screen` |
| 12 | `Error_Screen` | Screen | _(end)_ |

---

## Element Summary — Link File to Case After Guest Upload

| # | Element | Type | Connects To |
|---|---|---|---|
| 1 | Trigger | RecordAfterSave — Case (Create/Update, `ContentVersionId__c` not null) | `Get_ContentVersion` |
| 2 | `Get_ContentVersion` | Record Lookup (ContentVersion) | `Check_ContentDocument_Found` |
| 3 | `Check_ContentDocument_Found` | Decision | `Create_ContentDocumentLink` / STOP |
| 4 | `Create_ContentDocumentLink` | Record Create (ContentDocumentLink) | `Clear_ContentVersionId` |
| 5 | `Clear_ContentVersionId` | Record Update (Case) | _(end)_ |

---

## How the Two Flows Connect

```mermaid
sequenceDiagram
    actor User
    participant ScreenFlow as MilwaukeeContactForm (Screen Flow)
    participant Apex1 as GenericFormConfigController
    participant LWC as c:milwaukeeContactForm
    participant Apex2 as FileUploadDataConfigController
    participant Case as Case (Salesforce Object)
    participant RTFlow as Link_File_to_Case_After_Guest_Upload (RTF)
    participant CDL as ContentDocumentLink

    User->>ScreenFlow: Open form page
    ScreenFlow->>Apex1: Get MWK Form Config (lang, formName)
    Apex1-->>ScreenFlow: labelsJson, picklistDataJson
    ScreenFlow->>LWC: Render form (formConfig)
    User->>LWC: Fill in form & submit
    LWC-->>ScreenFlow: genericFormData (all field values)
    ScreenFlow->>Case: Create Case (Web origin)
    Case-->>ScreenFlow: CreatedCaseId
    alt File was uploaded
        ScreenFlow->>Apex2: Create ContentVersion (base64, fileName)
        Apex2-->>ScreenFlow: contentVersionId
        ScreenFlow->>Case: Update Case.ContentVersionId__c = contentVersionId
        Case-->>RTFlow: Trigger fires (ContentVersionId__c changed)
        RTFlow->>Case: Lookup ContentVersion → get ContentDocumentId
        RTFlow->>CDL: Create ContentDocumentLink (LinkedEntityId = Case.Id)
        RTFlow->>Case: Clear ContentVersionId__c
    end
    ScreenFlow->>User: Thank You Screen
```
