# AGENTS.md - Anvil Make.com Module Development Guide

Guidelines for AI coding agents working on the Anvil Make.com integration module.

## Project Overview

Make.com custom app module for Anvil API (PDF filling, e-signatures, PDF generation, workflows).

```
anvil-make-module/
├── app.json              # Main app configuration
├── app-complete.json     # Bundled app (deployment)
├── modules/actions/      # Action modules
├── modules/search/       # Search/list modules  
├── modules/triggers/     # Webhook triggers
├── modules/universal/    # GraphQL modules
└── webhooks/             # Webhook definitions
```

## Build/Validate

**Validate JSON:** `find anvil-make-module -name "*.json" -exec python3 -m json.tool {} > /dev/null \; -print`

**Deploy:** Copy `app-complete.json` to Make.com editor or use Make Apps SDK VS Code extension.

## Code Style

### JSON Formatting
- 2-space indentation, trailing newlines, no comments

### Module Property Order
```json
{
  "name": "camelCaseName",
  "label": "Human Readable Label", 
  "description": "Description",
  "type": "action|trigger|search|universal",
  "connection": "anvil",
  "communication": {},
  "parameters": [],
  "interface": [],
  "samples": {}  // Triggers only
}
```

### Naming Conventions
- **Modules:** camelCase (`fillPdfTemplate`, `watchEtchComplete`)
- **Parameters:** camelCase (`templateId`, `isDraft`)
- **Labels:** Title Case ("PDF Template ID")

### Parameter Types
`text` | `integer` | `number` | `boolean` | `json` | `email` | `url` | `select` | `array` | `collection` | `color`

### Parameter Structure
```json
{
  "name": "fieldName",
  "label": "Field Label",
  "type": "text",
  "required": true,
  "help": "Help text",
  "mappable": true,
  "advanced": false
}
```

### Select Options
```json
{ "type": "select", "options": [{ "label": "Display", "value": "apiValue" }] }
```

### Array Spec
```json
{
  "name": "items",
  "type": "array",
  "spec": { "type": "collection", "spec": [{ "name": "id", "type": "text" }] }
}
```

## Communication

### REST API
```json
{
  "communication": {
    "url": "/fill/{{parameters.templateId}}.pdf",
    "method": "POST",
    "headers": { "Content-Type": "application/json", "Accept": "application/pdf" },
    "body": { "data": "{{parameters.data}}" },
    "response": { "output": {}, "error": { "message": "..." } }
  }
}
```

### GraphQL
```json
{
  "communication": {
    "url": "https://graphql.useanvil.com",
    "method": "POST",
    "headers": { "Authorization": "Basic {{base64(connection.apiKey + ':')}}" },
    "body": { "query": "...", "variables": {} }
  }
}
```

## Error Handling
```json
{
  "error": {
    "message": "{{if(body.errors; body.errors[0].message; if(body.error; body.error; 'Request failed'))}}"
  }
}
```

## Security

**Sanitize auth headers:** `"log": { "sanitize": ["request.headers.Authorization"] }`

**Auth format:** `Authorization: Basic {{base64(connection.apiKey + ':')}}`

## Interface Output
```json
{
  "interface": [
    { "name": "packetEid", "label": "Packet ID", "type": "text" },
    { "name": "signers", "type": "array", "spec": { "type": "collection", "spec": [...] } }
  ]
}
```

## Webhooks

**Trigger:** `{ "name": "watchEtchComplete", "type": "trigger", "webhook": "etchComplete" }`

**Webhook file:**
```json
{
  "name": "etchComplete",
  "type": "dedicated",
  "communication": {
    "respond": { "type": "json", "status": 200, "body": { "received": true } },
    "iterate": { "container": "{{[body.data]}}" },
    "output": {}
  }
}
```

## Common Patterns

**File output:** `{ "file": "{{body}}", "fileName": "{{parameters.title}}.pdf", "contentType": "application/pdf" }`

**Conditional:** `{{if(parameters.name; parameters.name; 'default')}}`

**Timestamp:** `{{now}}`

## API Endpoints

| Purpose | Endpoint | Method |
|---------|----------|--------|
| PDF Fill | `/fill/{id}.pdf` | POST |
| PDF Gen | `/generate-pdf` | POST |
| GraphQL | `https://graphql.useanvil.com` | POST |

## Anvil Specifics

- **EID Format:** Alphanumeric (`DEtCx3WtJsCIVWGsR1vU`)
- **Test Mode:** Default `isTest: true` for e-signature packets
- **Rate Limits:** 4 req/sec (dev), 40 req/sec (prod)
- **Demo Template:** `05xXsZko33JIO6aq5Pnr`

## Adding Modules

1. Create JSON in `modules/{type}/`
2. Register in `app.json` modules array
3. Define parameters with help, interface for output
4. Add samples for triggers
5. Include error handling, sanitize auth headers
6. Validate JSON syntax
