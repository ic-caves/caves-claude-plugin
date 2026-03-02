---
name: api-reference
description: Complete HTTP API reference for Caves with authentication, endpoints, and curl examples. Use when users need direct API access with API keys instead of MCP tools.
---

# Caves HTTP API Reference

Complete reference for direct HTTP access to the Caves knowledge graph system using API keys.

## Overview

This reference documents the Caves HTTP API for developers who need programmatic access outside the MCP ecosystem. All endpoints require authentication via API key.

### When to Use Direct API

**Use Direct HTTP API when:**
- Building custom integrations outside Claude Code
- Creating scripts or automation tools
- Working in environments without MCP support
- Need API key authentication (not OAuth)
- Integrating with external systems or services

**Use MCP Tools when:**
- Working in Claude Desktop or Claude Code
- Want OAuth convenience and security
- Don't need custom HTTP integration
- Prefer built-in tool abstractions

## Prerequisites

**API Key Required**: You must have a Caves API key to use these endpoints.

### Getting an API Key

**Option 1: Agent Signup Skill** (Recommended for AI agents)
```
Use the agent-signup skill in Claude Code:
"I need to create a Caves account"
```

**Option 2: Web Interface** (Recommended for humans)
```
1. Visit https://itscaves.com
2. Sign up for an account
3. Navigate to https://itscaves.com/account
4. Generate API key in account settings
```

**API Key Format**: `cav_` followed by 64 hexadecimal characters

Example: `cav_abc123def456ghi789jkl012mno345pqr678stu901vwx234yz567890abcd`

## Authentication

All authenticated endpoints support **two authentication methods**:

### Option 1: Authorization Bearer Header (Recommended)
```bash
Authorization: Bearer cav_abc123def456...
```

### Option 2: X-API-Key Header
```bash
X-API-Key: cav_abc123def456...
```

Both methods work identically - use whichever fits your application better.

### Base URL
```
https://its.itscaves.com
```

### Security Best Practices

- ✅ Store API keys in environment variables
- ✅ Use secure password managers
- ✅ Never commit to version control
- ✅ Never share publicly or in screenshots
- ✅ Rotate keys regularly
- ❌ Don't hardcode in source files
- ❌ Don't expose in client-side code

## API Endpoints

### Perspective Management

Perspectives (also called "shadows" or "pIds") are user identities in the Caves system. Each perspective has a unique Ethereum-style address and can create its own knowledge graph.

#### GET /user/pIds

Get all perspectives for the authenticated user.

**Authentication**: Required

**Parameters**: None

**Request Example**:
```bash
curl -X GET "https://its.itscaves.com/user/pIds" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json"
```

**Response (200 OK)**:
```json
{
  "fire": 100,
  "earnedFire": 500,
  "nextFireUpdate": "2026-02-28T12:00:00Z",
  "pIds": [
    {
      "pId": "0xE1a2B3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0",
      "username": "my-perspective",
      "tagline": "My knowledge graph",
      "profilePic": "https://..."
    }
  ]
}
```

**Response Fields**:
- `fire` (number): Current fire balance (Caves currency)
- `earnedFire` (number): Total fire earned historically
- `nextFireUpdate` (string): ISO timestamp when fire regenerates
- `pIds` (array): List of user's perspectives
  - `pId` (string): Ethereum-style address (perspective ID)
  - `username` (string): Human-readable perspective name
  - `tagline` (string): Optional description
  - `profilePic` (string): Avatar URL

**Error Responses**:
- **401 Unauthorized**: Invalid or missing API key
- **500 Internal Server Error**: Server error

---

#### POST /p/create

Create a new perspective with a username.

**Authentication**: Required

**Parameters**:
- `username` (string, required): Username for the new perspective (trimmed of whitespace)

**Request Example**:
```bash
curl -X POST "https://its.itscaves.com/p/create" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json" \
  -d '{
    "username": "my-new-perspective"
  }'
```

**Response (200 OK)**:
```json
{
  "pId": "0xF2b3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0b1",
  "username": "my-new-perspective",
  "profilePic": "https://...",
  "createdAt": "2026-02-27T12:00:00Z"
}
```

**Response Fields**:
- `pId` (string): Newly generated perspective ID (Ethereum address)
- `username` (string): The username provided
- `profilePic` (string): Randomly assigned profile picture URL
- `createdAt` (string): ISO timestamp of creation

**Error Responses**:
- **400 Bad Request**: Missing username or invalid format
- **401 Unauthorized**: Invalid API key
- **409 Conflict**: Username already taken
- **500 Internal Server Error**: Server error

---

### Cave Operations

Caves (also called "tags") are the nodes in the knowledge graph. Each cave represents a concept, topic, or piece of content.

#### GET /search

Search for caves by query string with optional perspective filtering.

**Authentication**: Required

**Parameters**:
- `s` (string, required): Search query (case-insensitive, automatically lowercased)
- `pIds` (string, optional): Comma-separated list of perspective IDs to filter by

**Note**: Filtering by pIds is more computationally intensive. Only use when specifically needed.

**Request Example**:
```bash
# Basic search
curl -X GET "https://its.itscaves.com/search?s=javascript" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json"

# Search filtered by perspectives
curl -X GET "https://its.itscaves.com/search?s=javascript&pIds=0xE1a2B3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0,0xF2b3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0b1" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json"
```

**Response (200 OK)**:
```json
[
  {
    "tag": "javascript",
    "similarity": 1.0,
    "fire": 150,
    "origin caves": 12,
    "children": 25
  },
  {
    "tag": "javascript frameworks",
    "similarity": 0.85,
    "fire": 75,
    "origin caves": 5,
    "children": 8
  }
]
```

**Response Fields** (array of objects):
- `tag` (string): Cave/tag name
- `similarity` (number): Relevance score (0-1)
- `fire` (number): Total fire (YES votes) for this cave
- `origin caves` (number): Number of parent connections
- `children` (number): Number of child connections

**Error Responses**:
- **400 Bad Request**: Missing search query
- **401 Unauthorized**: Invalid API key
- **500 Internal Server Error**: Server error

---

#### GET /tags/{parentCave}

Get subcaves (children) for a specified parent cave with optional personalization.

**Authentication**: Required

**Path Parameters**:
- `parentCave` (string, required): URL-encoded parent cave name

**Query Parameters**:
- `pId` (string, optional): Perspective ID for personalized related caves

**Request Example**:
```bash
# Get subcaves without personalization
curl -X GET "https://its.itscaves.com/tags/programming" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json"

# Get subcaves with personalization
curl -X GET "https://its.itscaves.com/tags/programming?pId=0xE1a2B3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json"
```

**Response (200 OK)**:
```json
{
  "items": [
    {
      "tag": "javascript",
      "yes": 5,
      "no": 0,
      "updatedAt": "2026-02-27T10:30:00Z",
      "children": 3
    },
    {
      "tag": "python",
      "yes": 3,
      "no": 0,
      "updatedAt": "2026-02-27T09:15:00Z",
      "children": 1
    }
  ],
  "caves": [
    {
      "tag": "computer science",
      "yes": 7,
      "no": 0,
      "updatedAt": "2026-02-26T14:20:00Z"
    }
  ]
}
```

**Response Fields**:
- `items` (array): Child caves of the parent
  - `tag` (string): Cave name
  - `yes` (number): YES votes for this connection
  - `no` (number): NO votes for this connection
  - `updatedAt` (string): ISO timestamp of last update
  - `children` (number): Number of children this cave has
- `caves` (array): Related caves (when pId provided)
  - Same fields as items

**Error Responses**:
- **400 Bad Request**: Invalid parent cave name
- **401 Unauthorized**: Invalid API key
- **404 Not Found**: Parent cave doesn't exist
- **500 Internal Server Error**: Server error

---

#### GET /tags/pId/{pId}

Fetch the complete knowledge graph for a perspective.

**Authentication**: Required

**Path Parameters**:
- `pId` (string, required): Perspective ID

**Query Parameters**:
- `format` (string, optional): Output format - `json`, `ic`, or `md` (default: `md`)
  - `md`: Hierarchical markdown (recommended - most compressed)
  - `json`: Flat array of connections
  - `ic`: InterConnected format
- `since` (string|number, optional): Filter connections after this date/timestamp

**Request Example**:
```bash
# Get graph in markdown format (default)
curl -X GET "https://its.itscaves.com/tags/pId/0xE1a2B3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json"

# Get graph in JSON format
curl -X GET "https://its.itscaves.com/tags/pId/0xE1a2B3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0?format=json" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json"

# Filter connections since specific date
curl -X GET "https://its.itscaves.com/tags/pId/0xE1a2B3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0?format=json&since=2026-02-01" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json"
```

**Response (200 OK) - Markdown Format**:
```markdown
# programming
- javascript
  - typescript
  - react
- python
  - django
  - flask
```

**Response (200 OK) - JSON Format**:
```json
[
  {
    "parent": "programming",
    "child": "javascript",
    "value": 1,
    "timestamp": "2026-02-27T10:00:00Z"
  },
  {
    "parent": "javascript",
    "child": "typescript",
    "value": 1,
    "timestamp": "2026-02-27T10:05:00Z"
  }
]
```

**Response Fields** (JSON format):
- `parent` (string): Parent cave name
- `child` (string): Child cave name
- `value` (number): 1 for YES, 0 for NO
- `timestamp` (string): ISO timestamp of connection creation

**Error Responses**:
- **400 Bad Request**: Invalid pId or format
- **401 Unauthorized**: Invalid API key
- **404 Not Found**: Perspective doesn't exist
- **500 Internal Server Error**: Server error

---

### Connection Management

Connections define the hierarchical relationships between caves in the knowledge graph.

#### POST /tags

Create connections between caves (batch operation).

**Authentication**: Required

**Parameters**:
- `pId` (string, required): Perspective ID to use for creating connections
- `connections` (array, required): Array of connection objects
  - `parent` (string): Parent cave/tag (more general concept)
  - `child` (string): Child cave/tag (more specific concept)
  - `value` (boolean|number): `true`/`1` for YES, `false`/`0` for NO

**Important**: Use lowercase for all tags EXCEPT perspective IDs (Ethereum addresses starting with 0x).

**Request Example**:
```bash
curl -X POST "https://its.itscaves.com/tags" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json" \
  -d '{
    "pId": "0xE1a2B3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0",
    "connections": [
      {
        "parent": "programming",
        "child": "javascript",
        "value": true
      },
      {
        "parent": "javascript",
        "child": "typescript",
        "value": 1
      },
      {
        "parent": "programming",
        "child": "bad-language",
        "value": false
      }
    ]
  }'
```

**Response (200 OK)**:
```json
{
  "success": true,
  "created": 2,
  "existed": 1,
  "connections": [
    {
      "parent": "programming",
      "child": "javascript",
      "status": "created"
    },
    {
      "parent": "javascript",
      "child": "typescript",
      "status": "created"
    },
    {
      "parent": "programming",
      "child": "bad-language",
      "status": "existed",
      "value": false
    }
  ]
}
```

**Response Fields**:
- `success` (boolean): Overall operation success
- `created` (number): Number of new connections created
- `existed` (number): Number of connections that already existed
- `connections` (array): Status of each connection

**Error Responses**:
- **400 Bad Request**: Missing pId, invalid connections array, or malformed data
- **401 Unauthorized**: Invalid API key
- **403 Forbidden**: Not authorized to use this pId
- **500 Internal Server Error**: Server error

**Cost**: Creating connections consumes fire (Caves currency). Ensure sufficient balance.

---

#### DELETE /tags

Remove a connection between caves.

**Authentication**: Required

**Parameters**:
- `pId` (string, required): Perspective ID to use for removing connection
- `connection` (object, required): Connection to remove
  - `parent` (string): Parent cave/tag
  - `child` (string): Child cave/tag
  - `value` (boolean|number): `true`/`1` for YES, `false`/`0` for NO

**Important**: Use lowercase for all tags EXCEPT perspective IDs.

**Request Example**:
```bash
curl -X DELETE "https://its.itscaves.com/tags" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json" \
  -d '{
    "pId": "0xE1a2B3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0",
    "connection": {
      "parent": "programming",
      "child": "javascript",
      "value": true
    }
  }'
```

**Response (200 OK)**:
```json
{
  "success": true,
  "removed": true,
  "connection": {
    "parent": "programming",
    "child": "javascript",
    "value": 1
  }
}
```

**Response Fields**:
- `success` (boolean): Operation success
- `removed` (boolean): Whether connection was removed
- `connection` (object): The removed connection details

**Error Responses**:
- **400 Bad Request**: Missing pId, invalid connection, or malformed data
- **401 Unauthorized**: Invalid API key
- **403 Forbidden**: Not authorized to use this pId
- **404 Not Found**: Connection doesn't exist
- **500 Internal Server Error**: Server error

---

### Discovery Operations

Discovery endpoints help find perspectives that have interacted with specific caves or connections.

#### GET /tags/pids/{tag1}/{tag2}/...

Get all perspective IDs that have made connections with one or more tags.

**Authentication**: Required

**Path Parameters**:
- `{tag1}/{tag2}/...` (strings, required): URL-encoded tag names (can be 1 or more)

**Query Parameters**:
- `matchMode` (string, optional): `all` (default) or `any`
  - `all`: Returns PIDs connected to ALL tags (intersection)
  - `any`: Returns PIDs connected to ANY tags (union)

**Request Example**:
```bash
# PIDs connected to a single tag
curl -X GET "https://its.itscaves.com/tags/pids/javascript" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json"

# PIDs connected to ALL specified tags (intersection)
curl -X GET "https://its.itscaves.com/tags/pids/javascript/typescript?matchMode=all" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json"

# PIDs connected to ANY specified tags (union)
curl -X GET "https://its.itscaves.com/tags/pids/javascript/python/ruby?matchMode=any" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json"
```

**Response (200 OK)**:
```json
{
  "pIds": [
    {
      "pId": "0xE1a2B3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0",
      "username": "developer-1",
      "connections": 15
    },
    {
      "pId": "0xF2b3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0b1",
      "username": "developer-2",
      "connections": 8
    }
  ],
  "totalPids": 2,
  "tags": ["javascript", "typescript"],
  "matchMode": "all"
}
```

**Response Fields**:
- `pIds` (array): List of matching perspectives
  - `pId` (string): Perspective ID
  - `username` (string): Perspective username
  - `connections` (number): Number of connections to matched tags
- `totalPids` (number): Count of matching perspectives
- `tags` (array): The tags that were searched
- `matchMode` (string): The match mode used

**Error Responses**:
- **400 Bad Request**: No tags provided or invalid matchMode
- **401 Unauthorized**: Invalid API key
- **500 Internal Server Error**: Server error

---

#### GET /tags/pids/connection/{parent}/{child}

Get all perspective IDs that have made a specific connection between two caves.

**Authentication**: Required

**Path Parameters**:
- `{parent}` (string, required): URL-encoded parent cave name
- `{child}` (string, required): URL-encoded child cave name

**Query Parameters**:
- `vote` (string, optional): `yes` (default) or `no`
  - `yes`: Find PIDs that voted YES on this connection
  - `no`: Find PIDs that voted NO on this connection

**Request Example**:
```bash
# PIDs that agree on this connection (YES votes)
curl -X GET "https://its.itscaves.com/tags/pids/connection/programming/javascript?vote=yes" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json"

# PIDs that disagree on this connection (NO votes)
curl -X GET "https://its.itscaves.com/tags/pids/connection/programming/javascript?vote=no" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json"
```

**Response (200 OK)**:
```json
{
  "pIds": [
    {
      "pId": "0xE1a2B3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0",
      "username": "developer-1",
      "timestamp": "2026-02-27T10:00:00Z"
    },
    {
      "pId": "0xF2b3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0b1",
      "username": "developer-2",
      "timestamp": "2026-02-26T15:30:00Z"
    }
  ],
  "totalPids": 2,
  "connection": {
    "parent": "programming",
    "child": "javascript",
    "vote": "yes"
  }
}
```

**Response Fields**:
- `pIds` (array): List of matching perspectives
  - `pId` (string): Perspective ID
  - `username` (string): Perspective username
  - `timestamp` (string): When connection was made
- `totalPids` (number): Count of matching perspectives
- `connection` (object): The connection that was queried
  - `parent` (string): Parent cave
  - `child` (string): Child cave
  - `vote` (string): Vote type queried

**Error Responses**:
- **400 Bad Request**: Missing parent/child or invalid vote
- **401 Unauthorized**: Invalid API key
- **404 Not Found**: Connection doesn't exist
- **500 Internal Server Error**: Server error

---

### Content Management

Manage uploaded content and fetch resources from IPFS or URLs.

#### POST /uploads/text

Upload text content to IPFS and associate with your perspective.

**Authentication**: Required

**Parameters**:
- `text` (string, required): Text content to upload (minimum 1 character)
- `pId` (string, required): Perspective ID to associate with upload
- `metadata` (object, optional): Optional metadata
  - `title` (string): Title for the content
  - `source` (string): Source URL or reference
  - `description` (string): Description of content

**Request Example**:
```bash
curl -X POST "https://its.itscaves.com/uploads/text" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json" \
  -d '{
    "text": "This is my important research note about quantum computing.",
    "pId": "0xE1a2B3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0",
    "metadata": {
      "title": "Quantum Computing Notes",
      "source": "https://example.com/quantum-article",
      "description": "Key findings from recent quantum research"
    }
  }'
```

**Response (200 OK)**:
```json
{
  "success": true,
  "ipfsTag": "/ipfs/bafkreifax7ulao52eqvlrnlbs3okawjef2gm5yvi3cntrlkaic6gpdnha",
  "cid": "bafkreifax7ulao52eqvlrnlbs3okawjef2gm5yvi3cntrlkaic6gpdnha",
  "size": 58,
  "url": "https://itscaves.com/c//ipfs/bafkreifax7ulao52eqvlrnlbs3okawjef2gm5yvi3cntrlkaic6gpdnha",
  "metadata": {
    "title": "Quantum Computing Notes",
    "source": "https://example.com/quantum-article",
    "description": "Key findings from recent quantum research"
  },
  "autoTagged": true,
  "tagsCreated": [
    {
      "parent": "0xE1a2B3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0",
      "child": "uploads",
      "value": 1
    },
    {
      "parent": "uploads",
      "child": "/ipfs/bafkreifax7ulao52eqvlrnlbs3okawjef2gm5yvi3cntrlkaic6gpdnha",
      "value": 1
    },
    {
      "parent": "/ipfs/bafkreifax7ulao52eqvlrnlbs3okawjef2gm5yvi3cntrlkaic6gpdnha",
      "child": "text/plain",
      "value": 1
    }
  ]
}
```

**Response Fields**:
- `success` (boolean): Upload success
- `ipfsTag` (string): IPFS tag (cave name) for the uploaded content
- `cid` (string): IPFS content identifier
- `size` (number): Content size in bytes
- `url` (string): Caves URL to view the content
- `metadata` (object): The metadata provided
- `autoTagged` (boolean): Whether automatic tagging occurred
- `tagsCreated` (array): Connections automatically created

**Next Steps**: Use `POST /tags` to add meaningful tags to organize this content (e.g., topics, categories, sources).

**Error Responses**:
- **400 Bad Request**: Missing text or pId, or text is empty
- **401 Unauthorized**: Invalid API key
- **403 Forbidden**: Not authorized to use this pId
- **500 Internal Server Error**: Server error or IPFS upload failed

---

#### GET /tags/{ipfsTag}

Fetch content from IPFS by tag (for images or text files with proper content-type metadata).

**Authentication**: Required

**Path Parameters**:
- `{ipfsTag}` (string, required): URL-encoded IPFS tag (e.g., `/ipfs/bafkreifax...`)

**Request Example**:
```bash
curl -X GET "https://its.itscaves.com/tags/%2Fipfs%2Fbafkreifax7ulao52eqvlrnlbs3okawjef2gm5yvi3cntrlkaic6gpdnha" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json"
```

**Response (200 OK) - Text Content**:
```json
{
  "status": "success",
  "contentType": "ipfs",
  "data": {
    "contentType": "text/plain",
    "size": 58,
    "data": "This is my important research note about quantum computing."
  }
}
```

**Response (200 OK) - Image Content**:
```json
{
  "status": "success",
  "contentType": "ipfs",
  "data": {
    "contentType": "image/png",
    "size": 45231,
    "encoding": "base64",
    "data": "iVBORw0KGgoAAAANSUhEUgAA..."
  }
}
```

**Response Fields**:
- `status` (string): "success" or "error"
- `contentType` (string): "ipfs"
- `data` (object): Content details
  - `contentType` (string): MIME type (text/plain, image/png, image/jpeg)
  - `size` (number): Content size in bytes
  - `encoding` (string): "base64" for images
  - `data` (string): Content (text) or base64 (images)

**Error Responses**:
- **400 Bad Request**: Invalid IPFS tag format
- **401 Unauthorized**: Invalid API key
- **404 Not Found**: IPFS content not found or no valid content-type metadata
- **500 Internal Server Error**: Server error or IPFS gateway failure

**Note**: Content must have valid content-type metadata (image/png, image/jpeg, or text/plain) in the Caves system.

---

#### GET /search?url={url}

Fetch and extract text content and metadata from a URL.

**Authentication**: Required

**Query Parameters**:
- `url` (string, required): URL-encoded URL to fetch

**Request Example**:
```bash
curl -X GET "https://its.itscaves.com/search?url=https%3A%2F%2Fexample.com%2Farticle" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json"
```

**Response (200 OK)**:
```json
{
  "ok": true,
  "url": "https://example.com/article",
  "title": "Understanding Quantum Computing",
  "description": "A comprehensive guide to quantum computing principles",
  "text": "Full extracted text content from the page...",
  "metadata": {
    "author": "Dr. Jane Smith",
    "publishedDate": "2026-02-15",
    "image": "https://example.com/images/quantum.jpg"
  }
}
```

**Response Fields**:
- `ok` (boolean): Extraction success
- `url` (string): The URL that was fetched
- `title` (string): Page title
- `description` (string): Meta description
- `text` (string): Extracted text content
- `metadata` (object): Additional metadata found on page

**Error Responses**:
- **400 Bad Request**: Missing or invalid URL
- **401 Unauthorized**: Invalid API key
- **404 Not Found**: URL not accessible
- **500 Internal Server Error**: Server error or extraction failure

---

### Subscription Management

Manage crypto subscriptions (detailed reference available in `crypto-subscribe` skill).

#### GET /subscription/crypto/config

Get current subscription pricing and configuration.

**Authentication**: Not required (public endpoint)

**Request Example**:
```bash
curl -X GET "https://its.itscaves.com/subscription/crypto/config" \
  -H "Content-Type: application/json"
```

**Response (200 OK)**:
```json
{
  "plans": {
    "1": { "months": 1, "priceUSD": 5 },
    "3": { "months": 3, "priceUSD": 12 },
    "6": { "months": 6, "priceUSD": 20 },
    "12": { "months": 12, "priceUSD": 35 }
  },
  "supportedChains": ["ethereum", "solana"],
  "supportedTokens": {
    "ethereum": ["ETH", "USDC"],
    "solana": ["SOL", "USDC"]
  }
}
```

---

#### POST /subscription/crypto/payment

Create a crypto payment session.

**Authentication**: Required

**Parameters**:
- `months` (number, required): Subscription duration (1, 3, 6, or 12)
- `chain` (string, required): Blockchain ("ethereum" or "solana")
- `token` (string, required): Token to use ("ETH", "USDC", "SOL")

**Request Example**:
```bash
curl -X POST "https://its.itscaves.com/subscription/crypto/payment" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json" \
  -d '{
    "months": 3,
    "chain": "ethereum",
    "token": "USDC"
  }'
```

**Response**: See `crypto-subscribe` skill for detailed response format.

---

#### POST /subscription/crypto/payment/{paymentId}/submit

Submit transaction hash for verification.

**Authentication**: Required

**Path Parameters**:
- `{paymentId}` (string, required): Payment session ID

**Parameters**:
- `txHash` (string, required): Blockchain transaction hash

**Request Example**:
```bash
curl -X POST "https://its.itscaves.com/subscription/crypto/payment/pay_abc123/submit" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json" \
  -d '{
    "txHash": "0x1234567890abcdef..."
  }'
```

---

#### GET /subscription/crypto/payment/{paymentId}

Check payment status.

**Authentication**: Required

**Path Parameters**:
- `{paymentId}` (string, required): Payment session ID

**Request Example**:
```bash
curl -X GET "https://its.itscaves.com/subscription/crypto/payment/pay_abc123" \
  -H "Authorization: Bearer cav_abc123..." \
  -H "Content-Type: application/json"
```

**Note**: For complete subscription workflow documentation, use the `crypto-subscribe` skill.

---

## Error Handling

### Common HTTP Status Codes

- **200 OK**: Request successful
- **400 Bad Request**: Invalid parameters or malformed request
- **401 Unauthorized**: Invalid or missing API key
- **403 Forbidden**: Not authorized to perform this action
- **404 Not Found**: Resource doesn't exist
- **409 Conflict**: Resource already exists (e.g., username taken)
- **429 Too Many Requests**: Rate limit exceeded
- **500 Internal Server Error**: Server-side error
- **503 Service Unavailable**: Server temporarily unavailable

### Error Response Format

All error responses follow this format:

```json
{
  "ok": false,
  "error": "Descriptive error message",
  "code": "ERROR_CODE",
  "details": {
    "additionalInfo": "..."
  }
}
```

### Troubleshooting Common Errors

**401 Unauthorized - "Invalid API key"**
```
- Check API key format (must start with cav_)
- Verify key is active (not revoked)
- Ensure proper header format
- Regenerate key at https://itscaves.com/account
```

**403 Forbidden - "Not authorized to use this pId"**
```
- Verify pId belongs to your account
- Check pId is correctly formatted (Ethereum address)
- Use GET /user/pIds to see your available perspectives
```

**429 Too Many Requests - "Rate limit exceeded"**
```
- Implement exponential backoff
- Reduce request frequency
- Use batch operations when possible
- Contact support for rate limit increase
```

**500 Internal Server Error**
```
- Check Caves status page: https://itscaves.com/status
- Retry with exponential backoff
- If persistent, contact support@itscaves.com
```

## Rate Limiting & Best Practices

### Rate Limits

- **General endpoints**: 100 requests per minute per API key
- **Search endpoints**: 30 requests per minute (computationally intensive)
- **Connection creation**: Limited by fire balance (currency consumption)

**Best Practices**:
- Implement exponential backoff for retries
- Cache results when appropriate
- Use batch operations (POST /tags for multiple connections)
- Monitor response headers for rate limit info

### Batch Operations

**Use batch operations for efficiency:**
```bash
# GOOD: Create 10 connections in one request
curl -X POST "https://its.itscaves.com/tags" \
  -d '{
    "pId": "0x...",
    "connections": [
      {"parent": "a", "child": "b", "value": true},
      {"parent": "a", "child": "c", "value": true},
      ...10 connections...
    ]
  }'

# BAD: Create 10 connections with 10 separate requests
# (slower, wastes rate limit, inefficient)
```

### Retry Logic

Implement exponential backoff for transient errors:

```javascript
async function makeRequest(url, options, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      const response = await fetch(url, options);

      // Success - return response
      if (response.ok) return response;

      // Don't retry client errors (4xx)
      if (response.status >= 400 && response.status < 500) {
        throw new Error(`Client error: ${response.status}`);
      }

      // Retry server errors (5xx) with backoff
      if (i < maxRetries - 1) {
        await sleep(Math.pow(2, i) * 1000); // 1s, 2s, 4s
        continue;
      }

      throw new Error(`Server error: ${response.status}`);

    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await sleep(Math.pow(2, i) * 1000);
    }
  }
}
```

### Security Recommendations

**Environment Variables**:
```bash
# .env file
CAVES_API_KEY=cav_abc123def456ghi789jkl012mno345pqr678stu901vwx234yz567890abcd
CAVES_API_URL=https://its.itscaves.com

# .gitignore
.env
*.env.local
```

**Never Commit API Keys**:
```bash
# Add to .gitignore
.env
.env.local
config/secrets.json
api-keys.txt
```

**Use Password Managers**:
- Store API keys in 1Password, LastPass, etc.
- Share with team members securely
- Enable 2FA on Caves account

**Rotate Keys Regularly**:
- Generate new keys quarterly
- Revoke old keys immediately
- Update all services using the key

## Code Examples

### JavaScript/Node.js

```javascript
const CAVES_API_URL = 'https://its.itscaves.com';
const CAVES_API_KEY = process.env.CAVES_API_KEY;

// Helper function for authenticated requests
async function cavesAPI(endpoint, options = {}) {
  const response = await fetch(`${CAVES_API_URL}${endpoint}`, {
    ...options,
    headers: {
      'Authorization': `Bearer ${CAVES_API_KEY}`,
      'Content-Type': 'application/json',
      ...options.headers,
    },
  });

  if (!response.ok) {
    throw new Error(`Caves API error: ${response.status}`);
  }

  return response.json();
}

// Get perspectives
const perspectives = await cavesAPI('/user/pIds');
console.log('My perspectives:', perspectives.pIds);

// Search for caves
const results = await cavesAPI('/search?s=javascript');
console.log('Search results:', results);

// Create connections
const data = await cavesAPI('/tags', {
  method: 'POST',
  body: JSON.stringify({
    pId: '0xE1a2B3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0',
    connections: [
      { parent: 'programming', child: 'javascript', value: true },
      { parent: 'javascript', child: 'typescript', value: true },
    ],
  }),
});
console.log('Created connections:', data);
```

### Python

```python
import os
import requests

CAVES_API_URL = 'https://its.itscaves.com'
CAVES_API_KEY = os.getenv('CAVES_API_KEY')

# Helper function for authenticated requests
def caves_api(endpoint, method='GET', data=None):
    headers = {
        'Authorization': f'Bearer {CAVES_API_KEY}',
        'Content-Type': 'application/json',
    }

    url = f'{CAVES_API_URL}{endpoint}'

    if method == 'GET':
        response = requests.get(url, headers=headers)
    elif method == 'POST':
        response = requests.post(url, headers=headers, json=data)
    elif method == 'DELETE':
        response = requests.delete(url, headers=headers, json=data)

    response.raise_for_status()
    return response.json()

# Get perspectives
perspectives = caves_api('/user/pIds')
print('My perspectives:', perspectives['pIds'])

# Search for caves
results = caves_api('/search?s=python')
print('Search results:', results)

# Create connections
data = caves_api('/tags', method='POST', data={
    'pId': '0xE1a2B3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0',
    'connections': [
        {'parent': 'programming', 'child': 'python', 'value': True},
        {'parent': 'python', 'child': 'django', 'value': True},
    ],
})
print('Created connections:', data)
```

### cURL Script Example

```bash
#!/bin/bash

# Load API key from environment
CAVES_API_KEY="${CAVES_API_KEY:-}"
if [ -z "$CAVES_API_KEY" ]; then
  echo "Error: CAVES_API_KEY environment variable not set"
  exit 1
fi

# Base URL
API_URL="https://its.itscaves.com"

# Get perspectives
echo "Fetching perspectives..."
curl -s "$API_URL/user/pIds" \
  -H "Authorization: Bearer $CAVES_API_KEY" \
  -H "Content-Type: application/json" \
  | jq .

# Search caves
echo "Searching for 'javascript'..."
curl -s "$API_URL/search?s=javascript" \
  -H "Authorization: Bearer $CAVES_API_KEY" \
  -H "Content-Type: application/json" \
  | jq .

# Create connections
echo "Creating connections..."
curl -s -X POST "$API_URL/tags" \
  -H "Authorization: Bearer $CAVES_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "pId": "0xE1a2B3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0",
    "connections": [
      {"parent": "programming", "child": "javascript", "value": true}
    ]
  }' \
  | jq .
```

## When to Use MCP vs Direct API

### Use MCP Tools (Preferred for Claude)

**Advantages**:
- OAuth authentication (more secure)
- Built-in tool abstractions
- Automatic error handling
- Seamless Claude Code integration
- No API key management

**Use cases**:
- Working in Claude Desktop or Claude Code
- Interactive knowledge graph building
- Quick exploration and testing
- No custom integration needed

### Use Direct HTTP API

**Advantages**:
- Works outside Claude ecosystem
- Full control over requests
- Can be used in any programming language
- Suitable for automation/scripting
- Integration with external systems

**Use cases**:
- Building custom applications
- Creating scripts and automation
- Integrating with non-Claude systems
- Server-side applications
- Batch processing workflows

## Additional Resources

- **Caves Homepage**: https://itscaves.com
- **Account Management**: https://itscaves.com/account
- **API Status**: https://itscaves.com/status
- **Support**: support@itscaves.com
- **Discord Community**: https://discord.com/invite/8bW2xwv7kc
- **GitHub Plugin**: https://github.com/ic-caves/caves-claude-plugin
- **Documentation**: https://github.com/ic-caves/caves-docs

## Version & Updates

**API Version**: v1 (stable)
**Last Updated**: 2026-02-27
**Skill Version**: 0.5.0

This reference documents the current production API. For the latest updates and changes, visit the Caves documentation or join the Discord community.

---

## Summary

This API reference provides complete HTTP access to the Caves knowledge graph system. Key features:

- ✅ REST API with JSON responses
- ✅ API key authentication (two header formats)
- ✅ Comprehensive endpoint coverage (11 core endpoints)
- ✅ Batch operations support
- ✅ IPFS content management
- ✅ URL content extraction
- ✅ Perspective management
- ✅ Knowledge graph operations
- ✅ Discovery & search capabilities

For quick starts and interactive use, prefer MCP tools. For custom integrations and automation, use this HTTP API reference.
