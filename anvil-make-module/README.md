# Anvil API Module for Make.com

A comprehensive Make.com custom app module for integrating with the [Anvil API](https://www.useanvil.com/docs/). This module enables you to fill PDF templates, create e-signature packets, manage workflows, and generate PDFs from HTML/CSS.

## Features

- **Fill PDF Templates**: Fill Anvil PDF templates with JSON data
- **E-Signatures**: Create and manage e-signature packets (Etch)
- **Workflows**: Start, monitor, and retrieve data from Anvil Workflows
- **PDF Generation**: Generate PDFs from HTML and CSS content
- **Universal API Access**: Make arbitrary calls to any Anvil API endpoint

## Prerequisites

1. An Anvil account ([sign up free](https://app.useanvil.com/signup))
2. An Anvil API key (find at Organization Settings > API Settings)
3. A Make.com account with custom app development access

## Installation

### Option 1: Using Make.com Web Editor

1. Log in to [Make.com](https://www.make.com)
2. Navigate to **Scenarios** > **Custom apps**
3. Click **Create a new app**
4. Copy the contents of `app-complete.json` and paste it into the editor
5. Save and publish your app

### Option 2: Using VS Code Extension

1. Install the [Make Apps SDK extension](https://marketplace.visualstudio.com/items?itemName=make.make-apps-sdk) for VS Code
2. Clone or download this repository
3. Open the `anvil-make-module` folder in VS Code
4. Use the Make extension to deploy your app

## Configuration

### Setting Up Connection

1. In your Make scenario, add an Anvil module
2. Click **Create a connection**
3. Enter your **API Key** from Anvil (found at Organization Settings > API Settings)
4. Select your **Environment**:
   - **Production**: For live data and legally binding signatures
   - **Development**: For testing (watermarked PDFs, demo signatures)

## Modules

### 1. Fill a PDF Template

Fill an existing PDF template with data and return the filled PDF as a file.

**Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| PDF Template ID | Text | Yes | The template ID from Anvil (e.g., `kA6Da9CuGqUtc6QiBDRR`) |
| Fill Data | JSON | Yes | JSON object with field aliases and values |
| Document Title | Text | No | Title encoded into PDF metadata |
| Output File Name | Text | No | Name for the output PDF file |
| Font Family | Select | No | Font for filled text |
| Font Size | Integer | No | Font size (default: 10) |
| Text Color | Color | No | Hex color code |
| Text Alignment | Select | No | Left, center, or right |
| Use Interactive Fields | Boolean | No | Keep fields editable in PDF readers |

**Example Fill Data:**
```json
{
  "name": "John Doe",
  "email": "john@example.com",
  "date": "2024-01-15",
  "address": {
    "street1": "123 Main St",
    "city": "San Francisco",
    "state": "CA",
    "zip": "94106"
  }
}
```

### 2. List PDF Templates

Retrieve all PDF templates (Casts) available in your organization.

**Output:**
- Template ID (eid)
- Template Name
- Created/Updated dates
- Archive status
- Version number

### 3. Create an E-Signature Packet

Create an e-signature packet with PDFs and signers.

**Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| Packet Name | Text | Yes | Descriptive name for the packet |
| PDF Files | Array | Yes | List of files with template IDs |
| Signers | Array | Yes | List of signers with their info and signature fields |
| Save as Draft | Boolean | No | Save without sending |
| Test Mode | Boolean | No | Demo signatures with watermark |
| Fill Data | JSON | No | Data to fill in PDFs |
| Email Subject | Text | No | Custom email subject |

**Example Signers Configuration:**
```json
[
  {
    "id": "signer1",
    "name": "John Doe",
    "email": "john@example.com",
    "signerType": "email",
    "routingOrder": 1,
    "fields": [
      {
        "fileId": "contract",
        "fieldId": "signature"
      }
    ]
  }
]
```

**Example Files Configuration:**
```json
[
  {
    "id": "contract",
    "castEid": "yourTemplateId",
    "filename": "employment-contract.pdf"
  }
]
```

### 4. Send a Draft E-Signature Packet

Send a previously created draft packet to its signers.

**Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| Packet ID | Text | Yes | EID of the draft packet |

### 5. Download Documents

Get download URLs for all completed documents from a packet or workflow.

**Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| Document Group ID | Text | Yes | EID of the DocumentGroup |

**Output:**
- Status
- ZIP download URL
- List of individual files with download URLs

### 6. List Workflows

Retrieve all Workflows (Welds) available in your organization.

**Output:**
- Workflow ID (eid)
- Title and Name
- Slug
- Creation date
- Associated Webforms (Forges)

### 7. Get Workflow Data

Retrieve data and status from a specific workflow submission.

**Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| Workflow Data ID | Text | Yes | EID of the WeldData |

**Output:**
- Status
- Continue URL
- Display title
- Number of remaining signers
- Submissions with resolved payload
- Document group with signers

### 8. Generate PDF from HTML

Generate a PDF document from HTML and CSS content.

**Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| HTML Content | Text | Yes | HTML for the PDF |
| CSS Styles | Text | No | CSS styles to apply |
| Document Title | Text | No | Title for the PDF |

### 9. Make an API Call

Perform arbitrary API calls to Anvil's REST or GraphQL API.

**Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| Endpoint | Select | Yes | REST or GraphQL |
| HTTP Method | Select | Yes | GET, POST, PUT, PATCH, DELETE |
| URL Path | Text | Yes | Path to append to base URL |
| Query String | Collection | No | Query parameters |
| Request Body | JSON | No | Request body for POST/PUT |

## Common Use Cases

### 1. Fill and Email a PDF

```
[Anvil: Fill PDF Template] → [Email: Send an Email]
```

### 2. Create and Send for E-Signature

```
[Anvil: Create E-Signature Packet] → [Delay: Wait for signatures] → [Anvil: Download Documents]
```

### 3. Monitor Workflow Completion

```
[Anvil: Get Workflow Data] → [Filter: Check status] → [Actions based on status]
```

### 4. Generate PDF from Dynamic Content

```
[Data Source] → [Anvil: Generate PDF from HTML] → [Email/Storage]
```

## Field Types Reference

When filling PDF templates, use these formats:

| Field Type | Format |
|------------|--------|
| shortText | `"Hello World"` |
| email | `"user@example.com"` |
| date | `"2024-01-15"` (YYYY-MM-DD) |
| checkbox | `true` or `false` |
| number | `123.45` |
| dollar | `99.99` |
| phone | `{"num": "5551234567", "region": "US"}` or `"5551234567"` |
| fullName | `{"firstName": "John", "lastName": "Doe"}` or `"John Doe"` |
| usAddress | `{"street1": "123 Main", "city": "SF", "state": "CA", "zip": "94106"}` |
| ssn | `"123121234"` |
| ein | `"921234567"` |

## Error Handling

The module handles errors from the Anvil API and provides meaningful error messages:

- **401 AuthenticationError**: Invalid API key
- **429 RateLimitError**: Rate limit exceeded (wait and retry)
- **400 Validation Error**: Invalid data format
- **404 Not Found**: Template or resource not found

## Rate Limits

| Environment | Rate Limit |
|-------------|------------|
| Development | 4 requests/second |
| Production (Maker) | 4 requests/second |
| Production (Custom) | 40 requests/second |

## Testing

1. Use **Development** environment for testing
2. Enable **Test Mode** when creating e-signature packets
3. Test packets have watermarked PDFs and demo (red) signatures
4. Test packets don't count against your quota

## Support

- **Anvil Documentation**: [https://www.useanvil.com/docs/](https://www.useanvil.com/docs/)
- **Anvil API Reference**: [https://www.useanvil.com/docs/api/graphql/reference/](https://www.useanvil.com/docs/api/graphql/reference/)
- **Anvil Support**: support@useanvil.com
- **Make Documentation**: [https://developers.make.com/custom-apps-documentation](https://developers.make.com/custom-apps-documentation)

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-02-23 | Initial release |

## License

This module is provided as-is for integration with Anvil services. Refer to Anvil's [Terms of Service](https://www.useanvil.com/terms-and-privacy/) for usage guidelines.
