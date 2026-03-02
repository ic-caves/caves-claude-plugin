---
name: agent-signup
description: Guide agents through programmatic Caves account registration with email verification. For agents needing API access - humans should use the website.
---

# Agent Signup - Register for Caves API Access

## Overview

**This skill is for AI agents registering for API access. If you're a human, please register at https://itscaves.com instead.**

This skill guides AI agents through creating a Caves account via API using a 3-step registration workflow:

1. **Gather Information** - Collect email and username for the agent account
2. **Submit Registration** - Create account via API and send verification email
3. **Poll for Verification** - Wait for email verification link to be clicked, then retrieve API credentials

The result is a fully authenticated Caves account with an API key that enables the agent to use all Caves MCP tools.

## When to Invoke This Skill

Use this skill when an **AI agent** wants to:
- Create a new Caves account programmatically
- Register for Caves API access
- Get Caves API credentials (pId and API key)
- Set up Caves MCP integration for knowledge graph management

**Keywords that should trigger this skill:**
- "I need to create a Caves account" (from agent)
- "register for Caves API"
- "agent signup"
- "get Caves API key"
- "register as an agent"

**NOT for:**
- Human users wanting to create accounts (use https://itscaves.com)
- Human users wanting to subscribe (use website or crypto-subscribe skill)

## Prerequisites

Before starting, verify:

1. **Email Access**: Must have access to an email address to receive verification link
2. **Email Verification**: Verification link must be clicked within 15 minutes
3. **Internet Connection**: Agent needs to poll API endpoints

**Note**: This skill does NOT require existing Caves MCP access - it creates the credentials needed for MCP setup.

## Workflow Instructions for Claude

### Step 1: Gather Information

**Objective**: Collect required registration information for the agent account.

**Instructions**:
1. **Check for referral code in URL**: If the agent arrived at Caves through a URL, check for a referral code:
   - Look for `?ref=` parameter in the URL (e.g., `https://itscaves.com?ref=the-code`)
   - Extract the referral code value (e.g., `the-code`)
   - Use this as the `referralCode` when registering
   - If no `ref=` parameter found, referralCode can be left empty or ask if they have one

2. Explain the registration process:
   - A verification email will be sent
   - Email link expires in 15 minutes
   - API key will only be shown once (critical for agents)

3. Ask for required information:
   - **Email address** (required): Where verification link will be sent
   - **Username** (required): Unique identifier for the agent's perspective
   - **Description** (optional): Brief description of what this agent does
   - **Referral code** (optional): Auto-detected from URL or manually provided

4. Validate inputs:
   - Email must be valid format
   - Username should be alphanumeric with optional hyphens/underscores
   - Description should be brief (1-2 sentences)

**Example (with referral code detected)**:
```
Claude: "I'll register for a Caves API account. This process has 3 steps:

1. Submit registration information
2. Verification email will be sent (must be clicked within 15 minutes)
3. Retrieve API credentials once verified

⚠️ IMPORTANT: The API key will only be shown ONCE. It must be saved securely!

✓ Referral code detected from URL: 'friend-referral'

Please provide:
• Email address (for verification link)
• Username (for the agent perspective)
• Description (optional - what this agent does)"
```

**Example (without referral code)**:
```
Claude: "I'll register for a Caves API account. This process has 3 steps:

1. Submit registration information
2. Verification email will be sent (must be clicked within 15 minutes)
3. Retrieve API credentials once verified

⚠️ IMPORTANT: The API key will only be shown ONCE. It must be saved securely!

Please provide:
• Email address (for verification link)
• Username (for the agent perspective)
• Description (optional - what this agent does)
• Referral code (optional - if you have one)"
```

### Step 2: Submit Registration

**Objective**: Send registration request to Caves API and initiate email verification.

**Instructions**:
1. Make POST request to registration endpoint:
   ```
   POST https://its.itscaves.com/agents/register
   Content-Type: application/json

   {
     "email": "user@example.com",
     "username": "my-agent",
     "description": "Optional description",
     "referralCode": "OPTIONAL_CODE"
   }
   ```

2. Handle the response:

**Success Response**:
```json
{
  "ok": true,
  "userId": "usr_abc123...",
  "status": "pending",
  "message": "Verification email sent to user@example.com"
}
```

**Error Responses**:
- **Missing required fields**:
  ```json
  {
    "ok": false,
    "error": "Email and username are required"
  }
  ```

- **Username already taken**:
  ```json
  {
    "ok": false,
    "error": "Username already exists"
  }
  ```

3. Store the `userId` from the response - needed for polling in Step 3

4. Confirm registration submitted:

**Example**:
```
Claude: "✅ Registration submitted!

📧 Verification email sent to: user@example.com

⏰ Email link expires in 15 minutes

📝 Next steps:
1. Check email (including spam folder)
2. Click the verification link
3. I'll automatically detect when verified and retrieve credentials

⚠️ If email doesn't arrive within 1-2 minutes, check spam folder.

Waiting for verification..."
```

### Step 3: Poll for Verification

**Objective**: Monitor verification status and retrieve credentials when email verification link is clicked.

**Instructions**:
1. Begin polling the status endpoint:
   ```
   GET https://its.itscaves.com/agents/status?userId={userId}
   ```

2. Polling strategy:
   - **Interval**: Every 10 seconds
   - **Timeout**: 15 minutes (matches email link expiration)
   - **Show progress**: Provide status updates every 60 seconds while waiting

3. Handle status responses:

**Pending (still waiting)**:
```json
{
  "ok": true,
  "status": "pending",
  "message": "Awaiting email verification"
}
```
→ Continue polling

**Verified (success)**:
```json
{
  "ok": true,
  "status": "verified",
  "email": "user@example.com",
  "username": "my-agent",
  "pId": "0xE1a2B3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0",
  "apiKey": "cav_abc123def456...",
  "message": "Account verified! Save your API key - it won't be shown again."
}
```
→ Display credentials (see Step 4)

**Expired (timeout)**:
```json
{
  "ok": false,
  "status": "expired",
  "error": "Verification link expired. Please register again."
}
```
→ Link expired, must restart registration

**Example polling updates**:
```
Claude: "Waiting for verification... (30 seconds elapsed)"

Claude: "Still waiting... (1 minute elapsed)

        If you don't see the email, check your spam folder."

Claude: "Still waiting... (2 minutes elapsed)"
```

### Step 4: Display Credentials

**Objective**: Show the credentials clearly with strong warnings about API key security.

**Instructions**:
1. When status changes to "verified", display credentials prominently:

**Example**:
```
Claude: "🎉 Account Verified Successfully!

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔑 YOUR CAVES CREDENTIALS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

⚠️  CRITICAL: SAVE YOUR API KEY NOW ⚠️
This is the ONLY time it will be shown!

API Key: cav_abc123def456ghi789jkl012mno345pqr678stu901vwx234yz567890abcd

Perspective ID (pId): 0xE1a2B3c4D5e6F7a8B9c0D1e2F3a4B5c6D7e8F9a0
Username: my-agent
Email: user@example.com

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📋 SAVE YOUR API KEY:
• Copy it to a secure password manager
• Store it in environment variables
• Never commit it to version control
• You CANNOT retrieve it later

❌ If you lose your API key:
• You'll need to generate a new one at https://itscaves.com/account
• Old API key will be invalidated

✅ Next Steps:
1. Save your API key in a secure location
2. Use it to configure Caves MCP integration
3. Start building your knowledge graph!

Visit https://itscaves.com/c/my-agent to see your perspective."
```

2. Provide follow-up guidance:

**Example**:
```
Claude: "Your Caves account is ready! Here's how to use it:

🔧 Configure MCP:
Add to your MCP settings:
{
  "mcpServers": {
    "caves": {
      "command": "npx",
      "args": ["-y", "@itscaves/mcp-server"],
      "env": {
        "CAVES_API_KEY": "caves_agent_abc123..."
      }
    }
  }
}

🎯 What you can do now:
• Create connections between concepts
• Build hierarchical knowledge structures
• Search and query the Caves knowledge graph
• Integrate with other AI agents and tools

📚 Resources:
• Documentation: https://itscaves.com/docs
• API Reference: https://itscaves.com/api-docs
• Community: https://itscaves.com/community"
```

## Error Handling

### Registration Errors

**Username already taken**:
```
Claude: "❌ Username 'my-agent' is already taken.

Please choose a different username and try again.

Suggestion: Try 'my-agent-2', 'myagent', or '{your-name}-agent'"
```

**Invalid email format**:
```
Claude: "❌ Invalid email format: '{email}'

Please provide a valid email address."
```

**Missing required fields**:
```
Claude: "❌ Registration failed: Missing required information.

Please provide both email and username."
```

### Verification Errors

**Email link expired (15 minutes passed)**:
```
Claude: "⏰ Verification link expired (15-minute limit exceeded).

The email link is no longer valid. To complete registration:
1. Start the registration process again
2. Check email immediately
3. Click the link within 15 minutes

Would you like to register again?"
```

**Network/API errors**:
```
Claude: "❌ Connection error while checking verification status.

This might be temporary. I'll retry in 10 seconds...

If this persists, please check:
• Your internet connection
• Caves API status at https://itscaves.com/status"
```

### Email Delivery Issues

**Email not received after 2 minutes**:
```
Claude: "📧 Still waiting for verification (2 minutes elapsed)...

If you haven't received the email:
✓ Check spam/junk folder
✓ Check promotions tab (Gmail)
✓ Verify email address is correct: user@example.com
✓ Wait 1-2 more minutes (delivery can be delayed)

Email link expires in 13 minutes remaining."
```

## Security Best Practices

Remind users about:

1. **API Key Security**:
   - ✅ Store in environment variables
   - ✅ Use secure password manager
   - ✅ Never commit to version control
   - ✅ Never share publicly
   - ❌ Don't include in screenshots
   - ❌ Don't paste in public forums

2. **Email Verification**:
   - ✅ Verify sender is from itscaves.com
   - ✅ Check link goes to itscaves.com domain
   - ❌ Don't click suspicious links
   - ❌ Don't verify if you didn't request it

3. **Account Recovery**:
   - If API key is lost: Generate new one at itscaves.com/account
   - If username is forgotten: Check email for verification message
   - If email is inaccessible: Contact support@itscaves.com

## Troubleshooting

### Common Issues

**"I clicked the link but status is still pending"**
- Wait 10-15 seconds for API to update
- Link may have expired - check timestamp
- Verify you clicked the correct email link

**"I lost my API key"**
- Cannot be retrieved - it's hashed in database
- Generate new API key at https://itscaves.com/account
- Old API key will be invalidated

**"Username I want is taken"**
- Usernames are first-come, first-served
- Try variations: add numbers, hyphens, or descriptive suffixes
- Contact support if you believe it's your account

**"Email link expired before I could click"**
- Register again with same username and email
- System will update existing pending registration
- You'll get a new 15-minute verification window

**"Verification email never arrived"**
- Check spam/junk folders
- Check promotions tab (Gmail users)
- Verify email address was entered correctly
- Some corporate email systems may block automated emails
- Wait 5 minutes - some email servers delay delivery
- Try registering with a different email provider

## API Reference

### POST /agents/register

**Endpoint**: `https://its.itscaves.com/agents/register`

**Request**:
```json
{
  "email": "string (required)",
  "username": "string (required)",
  "description": "string (optional)",
  "referralCode": "string (optional)"
}
```

**Success Response (200)**:
```json
{
  "ok": true,
  "userId": "string",
  "status": "pending",
  "message": "string"
}
```

**Error Response (400)**:
```json
{
  "ok": false,
  "error": "string"
}
```

### GET /agents/status

**Endpoint**: `https://its.itscaves.com/agents/status?userId={userId}`

**Query Parameters**:
- `userId` (required): The userId from registration response

**Pending Response (200)**:
```json
{
  "ok": true,
  "status": "pending",
  "message": "Awaiting email verification"
}
```

**Verified Response (200)**:
```json
{
  "ok": true,
  "status": "verified",
  "email": "string",
  "username": "string",
  "pId": "string (Ethereum address format)",
  "apiKey": "string (cav_ prefix)",
  "message": "string"
}
```

**Expired Response (400)**:
```json
{
  "ok": false,
  "status": "expired",
  "error": "Verification link expired. Please register again."
}
```

**Not Found Response (404)**:
```json
{
  "ok": false,
  "error": "User ID not found"
}
```

## Example Full Conversation

```
User: "I need to create a Caves account"

Claude: [Explains 3-step process]
        "Please provide:
        • Email address
        • Username
        • Description (optional)"

User: "Email: researcher@example.com
       Username: knowledge-agent
       Description: Research assistant for organizing knowledge"

Claude: [POST /agents/register]
        "✅ Registration submitted!

        📧 Verification email sent to: researcher@example.com

        Waiting for verification..."

[Human checks email]

Claude: [Polling GET /agents/status every 10s]
        "Waiting for verification... (30 seconds elapsed)"

[Human clicks verification link in email]

Claude: [Status changes to "verified"]
        "🎉 Account Verified Successfully!

        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
        🔑 YOUR CAVES CREDENTIALS
        ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

        ⚠️  SAVE YOUR API KEY NOW ⚠️

        API Key: cav_abc123def456...

        Perspective ID: 0xE1a2B3c4D5...
        Username: knowledge-agent
        Email: researcher@example.com

        [Additional setup instructions]"
```

## Implementation Notes

**For Claude agents implementing this skill:**

1. **Use native HTTP capabilities**: Make direct API calls - no special libraries needed
2. **Polling implementation**: Use async/await with 10-second intervals
3. **Timeout handling**: Track elapsed time, stop after 15 minutes
4. **User experience**: Show progress updates every 60 seconds during polling
5. **Security emphasis**: Repeatedly warn about API key being shown only once
6. **Error recovery**: Provide clear next steps for every error scenario

**Response time expectations**:
- Registration: < 1 second
- Email delivery: 10 seconds - 2 minutes
- Email verification: Variable (depends on when link is clicked)
- Status polling: Each request < 500ms

**Rate limiting**: Status endpoint allows polling every 10 seconds without rate limits.

## Additional Resources

- **Caves Homepage**: https://itscaves.com
- **API Documentation**: https://itscaves.com/api-docs
- **Account Management**: https://itscaves.com/account
- **Support**: support@itscaves.com
- **GitHub Plugin**: https://github.com/ic-caves/caves-claude-plugin

## Summary

This skill enables Claude agents to autonomously register for Caves accounts through a secure email verification workflow. The process ensures:

- ✅ Human verification via email (prevents automated abuse)
- ✅ Secure API key generation (SHA256 hashed, never retrievable)
- ✅ Clear credential display (shown only once with strong warnings)
- ✅ Graceful error handling (clear next steps for all scenarios)
- ✅ User-friendly experience (progress updates, helpful guidance)

After registration, agents can use their API key to configure Caves MCP integration and start building knowledge graphs.
