# Anvil API Module for Make.com - Design Document

## Executive Summary

This document outlines the comprehensive design for integrating Anvil's document automation API with Make.com's automation platform. The module will enable Make users to leverage Anvil's PDF filling, PDF generation, and e-signature capabilities within their automation workflows.

---

## 1. Architecture Overview

### 1.1 Integration Topology

```
┌─────────────────────────────────────────────────────────────────┐
│                        Make.com Platform                         │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    Anvil Module                            │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐    │  │
│  │  │ Connection  │  │   Actions   │  │    Triggers     │    │  │
│  │  │   (API Key) │  │  (Modules)  │  │   (Webhooks)    │    │  │
│  │  └──────┬──────┘  └──────┬──────┘  └────────┬────────┘    │  │
│  │         │                │                   │             │  │
│  └─────────┼────────────────┼───────────────────┼─────────────┘  │
│            │                │                   │                │
└────────────┼────────────────┼───────────────────┼────────────────┘
             │                │                   │
             ▼                ▼                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Anvil API Platform                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │ REST API    │  │ GraphQL API │  │      Webhooks           │  │
│  │ /api/v1/    │  │ /graphql    │  │   (Callbacks)           │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    Anvil Services                           ││
│  │  • PDF Filling    • PDF Generation    • E-Signatures       ││
│  │  • Workflows      • Webforms          • Document AI        ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 API Communication Patterns

| Pattern | Endpoint | Use Case |
|---------|----------|----------|
| REST | `https://app.useanvil.com/api/v1/fill/{templateId}.pdf` | Fill PDF templates |
| REST | `https://app.useanvil.com/api/v1/generate-pdf` | Generate PDFs from HTML/Markdown |
| GraphQL | `https://app.useanvil.com/graphql` | E-signatures, Workflows, Complex operations |
| Webhook | User-defined URL | Event notifications |

### 1.3 Authentication Method

Anvil uses **Basic Authentication** with API keys:
- API Key serves as the username
- Password is empty (append `:` to the API key)
- Base64 encode `API_KEY:` for the Authorization header

---

## 2. Connection Configuration

### 2.1 Connection Parameters

```json
{
  "name": "anvil",
  "label": "Anvil",
  "description": "Connect to Anvil for PDF filling, generation, and e-signatures",
  "connection": {
    "parameters": [
      {
        "name": "apiKey",
        "label": "API Key",
        "type": "password",
        "required": true,
        "help": "Your Anvil API key. Find it in Organization Settings > API Keys. Use production key for live data or development key for testing."
      },
      {
        "name": "environment",
        "label": "Environment",
        "type": "select",
        "required": true,
        "default": "production",
        "options": [
          {"label": "Production", "value": "production"},
          {"label": "Development", "value": "development"}
        ],
        "help": "Development mode adds watermarks to PDFs and has stricter rate limits."
      }
    ],
    "communication": {
      "url": "https://app.useanvil.com/api/v1/fill/05xXsZko33JIO6aq5Pnr.pdf",
      "method": "POST",
      "headers": {
        "Authorization": "Basic {{base64(parameters.apiKey + ':')}}",
        "Content-Type": "application/json"
      },
      "body": {
        "data": {}
      },
      "response": {
        "valid": "{{statusCode === 200 || statusCode === 400}}",
        "error": {
          "message": "{{body.message || body.name || 'Authentication failed'}}"
        }
      },
      "log": {
        "sanitize": [
          "request.headers.Authorization"
        ]
      }
    }
  }
}
```

### 2.2 Base Configuration

```json
{
  "baseUrl": "https://app.useanvil.com",
  "headers": {
    "Authorization": "Basic {{base64(connection.apiKey + ':')}}",
    "Content-Type": "application/json"
  },
  "error": {
    "message": "{{body.message || body.error || 'Anvil API Error'}}"
  },
  "log": {
    "sanitize": [
      "request.headers.Authorization"
    ]
  }
}
```

---

## 3. Recommended API Endpoints as Make Actions

### 3.1 Priority Classification

| Priority | Category | Actions |
|----------|----------|---------|
| **P1 (Essential)** | PDF Operations | Fill PDF, Generate PDF (HTML), Generate PDF (Markdown) |
| **P1 (Essential)** | E-Signatures | Create E-Sign Packet, Get E-Sign Status |
| **P2 (Important)** | E-Signatures | Send E-Sign Packet, Download Documents, Generate Sign URL |
| **P2 (Important)** | Workflows | Create Workflow Submission, Get Workflow Status |
| **P3 (Useful)** | Templates | List PDF Templates, Get Template Fields |
| **P3 (Useful)** | Utilities | Merge PDFs, OCR Document |

---

## 4. Module Definitions

### 4.1 PDF Modules

---

#### ACTION: Fill PDF Template

**Name:** `fillPdfTemplate`  
**Label:** Fill a PDF Template  
**Description:** Fill an existing PDF template with data and return the filled PDF.

**Parameters (Mappable):**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| templateId | text | Yes | PDF Template ID (Cast EID) from Anvil dashboard |
| title | text | No | Title encoded into the PDF document |
| data | collection | Yes | Field data to fill the PDF (key-value pairs) |
| fontFamily | text | No | Font family (e.g., "Roboto", "Helvetica") |
| fontSize | number | No | Font size (5-30) |
| textColor | text | No | Text color as hex code (e.g., "#333333") |
| useInteractiveFields | boolean | No | Enable interactive/fillable fields |
| versionNumber | number | No | Template version (-1 for unpublished) |

**Data Parameter Structure:**
```json
{
  "name": "data",
  "label": "Fill Data",
  "type": "collection",
  "required": true,
  "help": "Key-value pairs matching field aliases in your PDF template. Keys are field IDs, values are the data to fill.",
  "spec": [
    {
      "name": "fieldId",
      "label": "Field ID",
      "type": "text",
      "required": true
    },
    {
      "name": "fieldValue",
      "label": "Field Value",
      "type": "any",
      "required": true
    }
  ]
}
```

**Communication:**
```json
{
  "url": "/api/v1/fill/{{parameters.templateId}}.pdf",
  "method": "POST",
  "headers": {
    "Accept": "application/pdf"
  },
  "body": {
    "title": "{{parameters.title}}",
    "fontFamily": "{{parameters.fontFamily}}",
    "fontSize": "{{parameters.fontSize}}",
    "textColor": "{{parameters.textColor}}",
    "useInteractiveFields": "{{parameters.useInteractiveFields}}",
    "data": "{{parameters.data}}"
  },
  "response": {
    "output": {
      "file": "{{body}}",
      "fileName": "{{parameters.title || parameters.templateId}}.pdf",
      "mimeType": "application/pdf"
    }
  }
}
```

**Output:**
```json
{
  "file": "binary PDF data",
  "fileName": "document.pdf",
  "mimeType": "application/pdf",
  "templateId": "abc123",
  "filledAt": "{{timestamp}}"
}
```

---

#### ACTION: Generate PDF from HTML

**Name:** `generatePdfHtml`  
**Label:** Generate PDF from HTML  
**Description:** Generate a new PDF from HTML and CSS content.

**Parameters (Mappable):**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| title | text | No | Title encoded into the PDF |
| html | text | Yes | HTML content for the PDF |
| css | text | No | CSS styles for the PDF |
| pageWidth | text | No | Page width with unit (e.g., "8.5in") |
| pageHeight | text | No | Page height with unit (e.g., "11in") |
| marginTop | text | No | Top margin (e.g., "50px") |
| marginBottom | text | No | Bottom margin (e.g., "50px") |
| marginLeft | text | No | Left margin (e.g., "60px") |
| marginRight | text | No | Right margin (e.g., "60px") |
| pageCount | select | No | Page number position (bottomCenter, etc.) |

**Communication:**
```json
{
  "url": "/api/v1/generate-pdf",
  "method": "POST",
  "headers": {
    "Accept": "application/pdf"
  },
  "body": {
    "type": "html",
    "title": "{{parameters.title}}",
    "data": {
      "html": "{{parameters.html}}",
      "css": "{{parameters.css}}"
    },
    "page": {
      "width": "{{parameters.pageWidth}}",
      "height": "{{parameters.pageHeight}}",
      "marginTop": "{{parameters.marginTop}}",
      "marginBottom": "{{parameters.marginBottom}}",
      "marginLeft": "{{parameters.marginLeft}}",
      "marginRight": "{{parameters.marginRight}}",
      "pageCount": "{{parameters.pageCount}}"
    }
  },
  "response": {
    "output": {
      "file": "{{body}}",
      "fileName": "{{parameters.title || 'generated'}}.pdf",
      "mimeType": "application/pdf"
    }
  }
}
```

**Output:**
```json
{
  "file": "binary PDF data",
  "fileName": "document.pdf",
  "mimeType": "application/pdf",
  "title": "My Document",
  "generatedAt": "{{timestamp}}"
}
```

---

#### ACTION: Generate PDF from Markdown

**Name:** `generatePdfMarkdown`  
**Label:** Generate PDF from Markdown  
**Description:** Generate a PDF from structured Markdown data including tables, labels, and content.

**Parameters (Mappable):**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| title | text | No | Title encoded into the PDF |
| fontFamily | text | No | Font family (e.g., "Roboto") |
| fontSize | number | No | Base font size |
| textColor | text | No | Text color as hex |
| includeTimestamp | boolean | No | Show timestamp at bottom |
| data | array | Yes | Array of content objects (label, content, table, heading) |
| logoUrl | text | No | URL for logo image |
| logoMaxWidth | number | No | Max logo width (default 200) |
| pageSettings | collection | No | Page dimensions and margins |

**Data Object Structure:**
```json
{
  "name": "data",
  "label": "Content Sections",
  "type": "array",
  "required": true,
  "spec": [
    {
      "name": "type",
      "label": "Section Type",
      "type": "select",
      "options": [
        {"label": "Label", "value": "label"},
        {"label": "Content (Markdown)", "value": "content"},
        {"label": "Heading", "value": "heading"},
        {"label": "Table", "value": "table"}
      ]
    },
    {
      "name": "label",
      "label": "Label Text",
      "type": "text",
      "condition": "{{item.type === 'label'}}"
    },
    {
      "name": "content",
      "label": "Markdown Content",
      "type": "text",
      "condition": "{{item.type === 'content'}}"
    },
    {
      "name": "heading",
      "label": "Heading Text",
      "type": "text",
      "condition": "{{item.type === 'heading'}}"
    },
    {
      "name": "tableRows",
      "label": "Table Data",
      "type": "array",
      "spec": {
        "type": "array",
        "spec": {"type": "text"}
      },
      "condition": "{{item.type === 'table'}}"
    }
  ]
}
```

**Communication:**
```json
{
  "url": "/api/v1/generate-pdf",
  "method": "POST",
  "headers": {
    "Accept": "application/pdf"
  },
  "body": {
    "type": "markdown",
    "title": "{{parameters.title}}",
    "fontFamily": "{{parameters.fontFamily}}",
    "fontSize": "{{parameters.fontSize}}",
    "textColor": "{{parameters.textColor}}",
    "includeTimestamp": "{{parameters.includeTimestamp}}",
    "logo": {
      "src": "{{parameters.logoUrl}}",
      "maxWidth": "{{parameters.logoMaxWidth}}"
    },
    "data": "{{parameters.data}}",
    "page": "{{parameters.pageSettings}}"
  },
  "response": {
    "output": {
      "file": "{{body}}",
      "fileName": "{{parameters.title || 'generated'}}.pdf",
      "mimeType": "application/pdf"
    }
  }
}
```

---

### 4.2 E-Signature Modules

---

#### ACTION: Create E-Signature Packet

**Name:** `createEtchPacket`  
**Label:** Create an E-Signature Packet  
**Description:** Create a new e-signature packet with PDFs and signers for electronic signature collection.

**Parameters (Mappable):**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| name | text | Yes | Name of the signature packet |
| isDraft | boolean | No | Create as draft (don't send yet) |
| isTest | boolean | No | Create as test packet (watermarked) |
| files | array | Yes | PDF files/templates to sign |
| signers | array | Yes | People who will sign |
| data | collection | No | Data to fill PDF fields before signing |
| signaturePageOptions | collection | No | Customization for signing page |
| webhookUrl | text | No | URL for completion webhook |
| mergePdfs | boolean | No | Merge all PDFs into single file |
| signatureEmailSubject | text | No | Custom email subject |
| signatureEmailBody | text | No | Custom email body |

**Files Array Structure:**
```json
{
  "name": "files",
  "label": "Documents",
  "type": "array",
  "required": true,
  "spec": [
    {
      "name": "fileId",
      "label": "File Reference ID",
      "type": "text",
      "required": true,
      "help": "Unique ID to reference this file (e.g., 'contract1')"
    },
    {
      "name": "castEid",
      "label": "PDF Template ID",
      "type": "text",
      "required": true,
      "help": "The EID of your PDF template from Anvil"
    },
    {
      "name": "filename",
      "label": "Output Filename",
      "type": "text"
    },
    {
      "name": "title",
      "label": "Document Title",
      "type": "text"
    }
  ]
}
```

**Signers Array Structure:**
```json
{
  "name": "signers",
  "label": "Signers",
  "type": "array",
  "required": true,
  "spec": [
    {
      "name": "id",
      "label": "Signer ID",
      "type": "text",
      "required": true,
      "help": "Unique ID for this signer (e.g., 'signer1')"
    },
    {
      "name": "name",
      "label": "Full Name",
      "type": "text",
      "required": true
    },
    {
      "name": "email",
      "label": "Email Address",
      "type": "email",
      "required": true
    },
    {
      "name": "signerType",
      "label": "Signer Type",
      "type": "select",
      "default": "email",
      "options": [
        {"label": "Email (Anvil sends email)", "value": "email"},
        {"label": "Embedded (You control the flow)", "value": "embedded"}
      ]
    },
    {
      "name": "routingOrder",
      "label": "Routing Order",
      "type": "number",
      "default": 1,
      "help": "Order in which signers sign (1 = first)"
    },
    {
      "name": "signatureFields",
      "label": "Signature Fields",
      "type": "array",
      "spec": [
        {
          "name": "fileId",
          "label": "File Reference",
          "type": "text"
        },
        {
          "name": "fieldId",
          "label": "Field ID",
          "type": "text"
        }
      ]
    }
  ]
}
```

**Communication (GraphQL):**
```json
{
  "url": "/graphql",
  "method": "POST",
  "body": {
    "query": "mutation CreateEtchPacket($variables: CreateEtchPacketInput!) { createEtchPacket(variables: $variables) { eid name status documentGroup { eid status } signers { eid name email status } } }",
    "variables": {
      "name": "{{parameters.name}}",
      "isDraft": "{{parameters.isDraft}}",
      "isTest": "{{parameters.isTest}}",
      "files": "{{parameters.files}}",
      "signers": "{{parameters.signers}}",
      "data": "{{parameters.data}}",
      "webhookURL": "{{parameters.webhookUrl}}",
      "mergePDFs": "{{parameters.mergePdfs}}"
    }
  },
  "response": {
    "output": {
      "etchPacketId": "{{body.data.createEtchPacket.eid}}",
      "name": "{{body.data.createEtchPacket.name}}",
      "status": "{{body.data.createEtchPacket.status}}",
      "documentGroupId": "{{body.data.createEtchPacket.documentGroup.eid}}",
      "documentGroupStatus": "{{body.data.createEtchPacket.documentGroup.status}}",
      "signers": "{{body.data.createEtchPacket.signers}}",
      "detailsUrl": "https://app.useanvil.com/etch/{{body.data.createEtchPacket.eid}}"
    }
  }
}
```

**Output:**
```json
{
  "etchPacketId": "DEtCx3WtJsCIVWGsR1vU",
  "name": "Employment Agreement",
  "status": "sent",
  "documentGroupId": "FhrKzFgrN5vZ4mwzy",
  "documentGroupStatus": "partial",
  "signers": [
    {
      "eid": "wAoklfcF67rI6A95Q7Yh",
      "name": "John Doe",
      "email": "john@example.com",
      "status": "sent"
    }
  ],
  "detailsUrl": "https://app.useanvil.com/etch/DEtCx3WtJsCIVWGsR1vU",
  "createdAt": "{{timestamp}}"
}
```

---

#### ACTION: Get E-Signature Packet Status

**Name:** `getEtchPacket`  
**Label:** Get E-Signature Status  
**Description:** Retrieve the current status of an e-signature packet.

**Parameters (Mappable):**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| etchPacketId | text | Yes | The EID of the e-signature packet |

**Communication (GraphQL):**
```json
{
  "url": "/graphql",
  "method": "POST",
  "body": {
    "query": "query GetEtchPacket($eid: String!) { etchPacket(eid: $eid) { eid name status isTest createdAt updatedAt documentGroup { eid status } signers { eid name email status routingOrder completedAt } } }",
    "variables": {
      "eid": "{{parameters.etchPacketId}}"
    }
  },
  "response": {
    "output": {
      "etchPacketId": "{{body.data.etchPacket.eid}}",
      "name": "{{body.data.etchPacket.name}}",
      "status": "{{body.data.etchPacket.status}}",
      "isTest": "{{body.data.etchPacket.isTest}}",
      "createdAt": "{{body.data.etchPacket.createdAt}}",
      "updatedAt": "{{body.data.etchPacket.updatedAt}}",
      "documentGroup": "{{body.data.etchPacket.documentGroup}}",
      "signers": "{{body.data.etchPacket.signers}}",
      "isComplete": "{{body.data.etchPacket.status === 'completed'}}"
    }
  }
}
```

**Output:**
```json
{
  "etchPacketId": "DEtCx3WtJsCIVWGsR1vU",
  "name": "Employment Agreement",
  "status": "completed",
  "isTest": false,
  "createdAt": "2026-02-20T10:00:00Z",
  "updatedAt": "2026-02-21T14:30:00Z",
  "isComplete": true,
  "documentGroup": {
    "eid": "FhrKzFgrN5vZ4mwzy",
    "status": "completed"
  },
  "signers": [
    {
      "eid": "wAoklfcF67rI6A95Q7Yh",
      "name": "John Doe",
      "email": "john@example.com",
      "status": "completed",
      "routingOrder": 1,
      "completedAt": "2026-02-21T14:30:00Z"
    }
  ]
}
```

---

#### ACTION: Send E-Signature Packet

**Name:** `sendEtchPacket`  
**Label:** Send E-Signature Packet  
**Description:** Send a draft e-signature packet to begin the signing process.

**Parameters (Mappable):**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| etchPacketId | text | Yes | The EID of the draft e-signature packet |

**Communication (GraphQL):**
```json
{
  "url": "/graphql",
  "method": "POST",
  "body": {
    "query": "mutation SendEtchPacket($eid: String!) { sendEtchPacket(eid: $eid) { eid name status signers { eid name email status } } }",
    "variables": {
      "eid": "{{parameters.etchPacketId}}"
    }
  },
  "response": {
    "output": {
      "etchPacketId": "{{body.data.sendEtchPacket.eid}}",
      "name": "{{body.data.sendEtchPacket.name}}",
      "status": "{{body.data.sendEtchPacket.status}}",
      "signers": "{{body.data.sendEtchPacket.signers}}",
      "sentAt": "{{timestamp}}"
    }
  }
}
```

---

#### ACTION: Generate Signer URL

**Name:** `generateSignerUrl`  
**Label:** Generate Signer URL  
**Description:** Generate a signing URL for an embedded signer.

**Parameters (Mappable):**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| signerEid | text | Yes | The EID of the signer |
| clientUserId | text | Yes | Your identifier for this user session |

**Communication (GraphQL):**
```json
{
  "url": "/graphql",
  "method": "POST",
  "body": {
    "query": "mutation GenerateEtchSignUrl($signerEid: String!, $clientUserId: String!) { generateEtchSignUrl(signerEid: $signerEid, clientUserId: $clientUserId) { url signerToken } }",
    "variables": {
      "signerEid": "{{parameters.signerEid}}",
      "clientUserId": "{{parameters.clientUserId}}"
    }
  },
  "response": {
    "output": {
      "signUrl": "{{body.data.generateEtchSignUrl.url}}",
      "signerToken": "{{body.data.generateEtchSignUrl.signerToken}}",
      "signerEid": "{{parameters.signerEid}}"
    }
  }
}
```

**Output:**
```json
{
  "signUrl": "https://app.useanvil.com/sign/...",
  "signerToken": "abc123...",
  "signerEid": "wAoklfcF67rI6A95Q7Yh"
}
```

---

#### ACTION: Download Signed Documents

**Name:** `downloadEtchDocuments`  
**Label:** Download Signed Documents  
**Description:** Download completed signed documents as a ZIP file or individual PDFs.

**Parameters (Mappable):**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| documentGroupId | text | Yes | The document group EID |
| format | select | No | Download format (zip or individual) |

**Communication:**
```json
{
  "url": "/api/document-group/{{parameters.documentGroupId}}.zip",
  "method": "GET",
  "headers": {
    "Accept": "application/zip"
  },
  "response": {
    "output": {
      "file": "{{body}}",
      "fileName": "signed_documents_{{parameters.documentGroupId}}.zip",
      "mimeType": "application/zip",
      "documentGroupId": "{{parameters.documentGroupId}}"
    }
  }
}
```

---

### 4.3 Workflow Modules

---

#### ACTION: Create Workflow Submission

**Name:** `createWorkflowSubmission`  
**Label:** Create Workflow Submission  
**Description:** Start a new workflow submission with data to fill forms and PDFs.

**Parameters (Mappable):**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| weldEid | text | Yes | The workflow (Weld) EID |
| data | collection | Yes | Data to fill the workflow forms |
| isTest | boolean | No | Create as test submission |
| webhookUrl | text | No | URL for completion webhook |

**Communication (GraphQL):**
```json
{
  "url": "/graphql",
  "method": "POST",
  "body": {
    "query": "mutation CreateWeldData($weldEid: String!, $webhookURL: String, $isTest: Boolean) { createWeldData(weldEid: $weldEid, webhookURL: $webhookURL, isTest: $isTest) { eid status weld { eid name } } }",
    "variables": {
      "weldEid": "{{parameters.weldEid}}",
      "webhookURL": "{{parameters.webhookUrl}}",
      "isTest": "{{parameters.isTest}}"
    }
  },
  "response": {
    "output": {
      "weldDataId": "{{body.data.createWeldData.eid}}",
      "status": "{{body.data.createWeldData.status}}",
      "weld": "{{body.data.createWeldData.weld}}"
    }
  }
}
```

---

### 4.4 Search/List Modules

---

#### SEARCH: List PDF Templates

**Name:** `listPdfTemplates`  
**Label:** List PDF Templates  
**Description:** Retrieve a list of all PDF templates in your organization.

**Parameters (Mappable):**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| limit | number | No | Maximum number of results (default 50) |

**Communication (GraphQL):**
```json
{
  "url": "/graphql",
  "method": "POST",
  "body": {
    "query": "query GetOrganizationCasts($limit: Int) { organization { casts(limit: $limit) { eid name createdAt isArchived } } }",
    "variables": {
      "limit": "{{parameters.limit}}"
    }
  },
  "response": {
    "output": "{{body.data.organization.casts}}"
  }
}
```

**Output (Array):**
```json
[
  {
    "eid": "kA6Da9CuGqUtc6QiBDRR",
    "name": "Employment Agreement",
    "createdAt": "2026-01-15T10:00:00Z",
    "isArchived": false
  }
]
```

---

#### SEARCH: Get Template Fields

**Name:** `getTemplateFields`  
**Label:** Get Template Fields  
**Description:** Retrieve all fillable fields from a PDF template.

**Parameters (Mappable):**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| templateId | text | Yes | PDF Template ID (Cast EID) |

**Communication (GraphQL):**
```json
{
  "url": "/graphql",
  "method": "POST",
  "body": {
    "query": "query GetCast($eid: String!) { cast(eid: $eid) { eid name fields { id label type aliasId format } } }",
    "variables": {
      "eid": "{{parameters.templateId}}"
    }
  },
  "response": {
    "output": {
      "templateId": "{{body.data.cast.eid}}",
      "name": "{{body.data.cast.name}}",
      "fields": "{{body.data.cast.fields}}"
    }
  }
}
```

**Output:**
```json
{
  "templateId": "kA6Da9CuGqUtc6QiBDRR",
  "name": "Employment Agreement",
  "fields": [
    {
      "id": "field1",
      "label": "Employee Name",
      "type": "fullName",
      "aliasId": "employeeName",
      "format": null
    },
    {
      "id": "field2",
      "label": "Start Date",
      "type": "date",
      "aliasId": "startDate",
      "format": "MM/DD/YYYY"
    }
  ]
}
```

---

### 4.5 Universal Module

---

#### UNIVERSAL: Make a GraphQL Query

**Name:** `makeGraphQLQuery`  
**Label:** Execute GraphQL Query  
**Description:** Execute a custom GraphQL query or mutation against the Anvil API.

**Parameters (Mappable):**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| query | text | Yes | GraphQL query or mutation string |
| variables | collection | No | Variables for the GraphQL operation |

**Communication:**
```json
{
  "url": "/graphql",
  "method": "POST",
  "body": {
    "query": "{{parameters.query}}",
    "variables": "{{parameters.variables}}"
  },
  "response": {
    "output": "{{body.data}}"
  }
}
```

---

## 5. Trigger Definitions (Webhooks)

### 5.1 Supported Webhook Events

| Event | Trigger Name | Description |
|-------|--------------|-------------|
| `etchPacketComplete` | Watch E-Sign Complete | Fires when all signers complete signing |
| `signerComplete` | Watch Signer Complete | Fires when a single signer completes |
| `signerUpdateStatus` | Watch Signer Status | Fires on signer status changes |
| `weldComplete` | Watch Workflow Complete | Fires when a workflow completes |
| `forgeComplete` | Watch Form Complete | Fires when a webform is submitted |

---

### TRIGGER: Watch E-Signature Complete (Instant)

**Name:** `watchEtchComplete`  
**Label:** Watch E-Signature Complete  
**Description:** Triggers when all signers have completed signing an e-signature packet.  
**Type:** Instant Trigger (Webhook)

**Webhook Payload Structure:**
```json
{
  "action": "etchPacketComplete",
  "token": "webhook_verification_token",
  "data": {
    "name": "NDA Packet",
    "eid": "DEtCx3WtJsCIVWGsR1vU",
    "isTest": false,
    "status": "completed",
    "downloadZipURL": "https://app.useanvil.com/api/document-group/xxx.zip",
    "documentGroup": {
      "eid": "FhrKzFgrN5vZ4mwzy",
      "status": "completed"
    },
    "signers": [
      {
        "name": "John Doe",
        "email": "john@example.com",
        "eid": "wAoklfcF67rI6A95Q7Yh",
        "status": "completed",
        "routingOrder": 1
      }
    ]
  }
}
```

**Output:**
```json
{
  "etchPacketId": "DEtCx3WtJsCIVWGsR1vU",
  "name": "NDA Packet",
  "status": "completed",
  "isTest": false,
  "downloadZipURL": "https://app.useanvil.com/api/document-group/xxx.zip",
  "documentGroupId": "FhrKzFgrN5vZ4mwzy",
  "signers": "{{body.data.signers}}",
  "completedAt": "{{timestamp}}"
}
```

---

### TRIGGER: Watch Signer Complete (Instant)

**Name:** `watchSignerComplete`  
**Label:** Watch Signer Complete  
**Description:** Triggers when a signer finishes signing their portion.  
**Type:** Instant Trigger (Webhook)

**Output:**
```json
{
  "signerEid": "0ZTRyrIlfcYsAo6A95Qk",
  "signerName": "John Doe",
  "signerEmail": "john@example.com",
  "signerStatus": "completed",
  "routingOrder": 1,
  "etchPacketId": "DEtCx3WtJsCIVWGsR1vU",
  "documentGroupId": "8jJ9yrIlfcYsAo6A95Qk",
  "documentGroupStatus": "partial",
  "allSigners": "{{body.data.signers}}",
  "completedAt": "{{timestamp}}"
}
```

---

### TRIGGER: Watch Workflow Complete (Instant)

**Name:** `watchWorkflowComplete`  
**Label:** Watch Workflow Complete  
**Description:** Triggers when an entire workflow is completed (all forms + signatures done).  
**Type:** Instant Trigger (Webhook)

**Output:**
```json
{
  "weldDataId": "GHEJsCVWsR1vtCx3WtUI",
  "isComplete": true,
  "isTest": false,
  "weldEid": "KtHa4IhKyoZO6hbQaQJK",
  "weldSlug": "employee-onboarding",
  "documents": [
    {
      "type": "application/zip",
      "url": "https://app.useanvil.com/download/xxx.zip"
    }
  ],
  "submissions": "{{body.data.submissions}}",
  "completedAt": "{{timestamp}}"
}
```

---

## 6. Complete Module JSON Structure

```json
{
  "name": "anvil",
  "label": "Anvil",
  "version": "1.0.0",
  "description": "PDF filling, generation, and e-signature automation",
  "icon": "data:image/svg+xml;base64,...",
  "baseUrl": "https://app.useanvil.com",
  
  "connection": {
    "label": "Anvil API Key",
    "type": "apiKey",
    "parameters": [
      {
        "name": "apiKey",
        "label": "API Key",
        "type": "password",
        "required": true,
        "help": "Your Anvil API key from Organization Settings > API Keys"
      },
      {
        "name": "environment",
        "label": "Environment",
        "type": "select",
        "default": "production",
        "options": [
          {"label": "Production", "value": "production"},
          {"label": "Development", "value": "development"}
        ]
      }
    ],
    "communication": {
      "url": "/graphql",
      "method": "POST",
      "headers": {
        "Authorization": "Basic {{base64(parameters.apiKey + ':')}}",
        "Content-Type": "application/json"
      },
      "body": {
        "query": "{ organization { name } }"
      },
      "response": {
        "valid": "{{statusCode === 200}}",
        "metadata": {
          "organizationName": "{{body.data.organization.name}}"
        }
      },
      "log": {
        "sanitize": ["request.headers.Authorization"]
      }
    }
  },

  "base": {
    "headers": {
      "Authorization": "Basic {{base64(connection.apiKey + ':')}}",
      "Content-Type": "application/json"
    },
    "error": {
      "message": "{{body.message || body.errors[0].message || 'Anvil API Error'}}"
    },
    "log": {
      "sanitize": ["request.headers.Authorization"]
    }
  },

  "modules": {
    "actions": [
      {
        "name": "fillPdfTemplate",
        "label": "Fill a PDF Template",
        "description": "Fill a PDF template with data"
      },
      {
        "name": "generatePdfHtml",
        "label": "Generate PDF from HTML",
        "description": "Generate a PDF from HTML and CSS"
      },
      {
        "name": "generatePdfMarkdown",
        "label": "Generate PDF from Markdown",
        "description": "Generate a PDF from Markdown content"
      },
      {
        "name": "createEtchPacket",
        "label": "Create E-Signature Packet",
        "description": "Create a new e-signature packet"
      },
      {
        "name": "getEtchPacket",
        "label": "Get E-Signature Status",
        "description": "Get status of an e-signature packet"
      },
      {
        "name": "sendEtchPacket",
        "label": "Send E-Signature Packet",
        "description": "Send a draft packet to signers"
      },
      {
        "name": "generateSignerUrl",
        "label": "Generate Signer URL",
        "description": "Generate embedded signing URL"
      },
      {
        "name": "downloadEtchDocuments",
        "label": "Download Signed Documents",
        "description": "Download completed signed documents"
      },
      {
        "name": "createWorkflowSubmission",
        "label": "Create Workflow Submission",
        "description": "Start a new workflow"
      }
    ],
    "search": [
      {
        "name": "listPdfTemplates",
        "label": "List PDF Templates",
        "description": "Get all PDF templates"
      },
      {
        "name": "getTemplateFields",
        "label": "Get Template Fields",
        "description": "Get fillable fields from a template"
      }
    ],
    "triggers": [
      {
        "name": "watchEtchComplete",
        "label": "Watch E-Signature Complete",
        "description": "Triggers when all signers complete",
        "type": "instant"
      },
      {
        "name": "watchSignerComplete",
        "label": "Watch Signer Complete",
        "description": "Triggers when a signer completes",
        "type": "instant"
      },
      {
        "name": "watchWorkflowComplete",
        "label": "Watch Workflow Complete",
        "description": "Triggers when a workflow completes",
        "type": "instant"
      }
    ],
    "universal": [
      {
        "name": "makeGraphQLQuery",
        "label": "Execute GraphQL Query",
        "description": "Run custom GraphQL operations"
      }
    ]
  }
}
```

---

## 7. Error Handling

### 7.1 Common Error Codes

| Status Code | Error Type | Handling |
|-------------|------------|----------|
| 400 | ValidationError | Display field-specific validation errors |
| 401 | AuthenticationError | Re-prompt for API key |
| 403 | AuthorizationError | Check API key permissions |
| 404 | NotFoundError | Resource doesn't exist |
| 413 | RequestTooLarge | Payload exceeds 1MB limit |
| 429 | RateLimitError | Wait and retry with exponential backoff |
| 500 | ServerError | Retry with backoff |

### 7.2 Error Response Structure

```json
{
  "error": {
    "message": "{{body.message || body.errors[0].message || 'Unknown error'}}",
    "code": "{{statusCode}}",
    "details": "{{body.fields || body.errors}}"
  }
}
```

---

## 8. Rate Limiting

| Environment | Rate Limit |
|-------------|------------|
| Development | 4 requests/second |
| Production (Maker) | 4 requests/second |
| Production (Custom) | 40 requests/second |

**Implementation Notes:**
- Monitor `X-RateLimit-Remaining` header
- Implement exponential backoff on 429 responses
- Use `Retry-After` header value for wait time

---

## 9. Implementation Phases

### Phase 1: Core PDF Operations (Week 1)
1. Connection setup with API key authentication
2. Fill PDF Template action
3. Generate PDF from HTML action
4. Generate PDF from Markdown action

### Phase 2: E-Signature Operations (Week 2)
1. Create E-Signature Packet action
2. Get E-Signature Status action
3. Send E-Signature Packet action
4. Generate Signer URL action
5. Download Signed Documents action

### Phase 3: Webhooks & Triggers (Week 3)
1. Watch E-Signature Complete trigger
2. Watch Signer Complete trigger
3. Watch Workflow Complete trigger
4. Webhook URL configuration

### Phase 4: Search & Universal (Week 4)
1. List PDF Templates search
2. Get Template Fields search
3. Create Workflow Submission action
4. Execute GraphQL Query universal module
5. Testing and documentation

---

## 10. Testing Strategy

### 10.1 Unit Tests
- Connection validation with valid/invalid API keys
- Each action with valid parameters
- Each action with missing required parameters
- Error response handling

### 10.2 Integration Tests
- End-to-end PDF filling workflow
- Complete e-signature flow (create → send → download)
- Webhook trigger reception and parsing

### 10.3 Test Data
- Use Anvil's demo PDF template: `05xXsZko33JIO6aq5Pnr`
- Create test packets with `isTest: true`
- Use development API key for testing

---

## 11. Security Considerations

1. **API Key Storage**: Keys stored encrypted in Make.com's credential store
2. **Log Sanitization**: Authorization headers excluded from logs
3. **HTTPS Only**: All API calls use TLS encryption
4. **Webhook Verification**: Validate webhook token against configured value
5. **Data Encryption**: Support RSA encryption for sensitive payloads (optional)

---

## Appendix A: Field Type Reference

| Field Type | Input Format | Example |
|------------|--------------|---------|
| shortText | String | "Hello World" |
| longText | String | "Multi\nline\ntext" |
| email | String | "user@example.com" |
| date | String | "2026-02-23" (YYYY-MM-DD) |
| phone | Object/String | {"num": "5551234567", "region": "US"} |
| dollar | Number | 1234.56 |
| number | Number | 42 |
| checkbox | Boolean | true |
| fullName | Object | {"firstName": "John", "lastName": "Doe"} |
| usAddress | Object | {"street1": "123 Main", "city": "SF", "state": "CA", "zip": "94102"} |
| imageFile | String (URL) | "https://example.com/image.png" |
| signature | (Signer assignment) | Assigned to signer in packet |

---

## Appendix B: GraphQL Query Examples

### Create Etch Packet
```graphql
mutation CreateEtchPacket($variables: CreateEtchPacketInput!) {
  createEtchPacket(variables: $variables) {
    eid
    name
    status
    documentGroup {
      eid
      status
    }
    signers {
      eid
      name
      email
      status
    }
  }
}
```

### Get Template Fields
```graphql
query GetCast($eid: String!) {
  cast(eid: $eid) {
    eid
    name
    fields {
      id
      label
      type
      aliasId
      format
    }
  }
}
```

---

*Document Version: 1.0.0*  
*Last Updated: February 23, 2026*  
*Author: Systems Integration Engineer*
