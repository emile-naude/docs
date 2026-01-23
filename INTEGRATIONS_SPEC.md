# Integration Technical Specifications

This document provides detailed technical specifications for all platform integrations: HubSpot, Gmail, Slack, and Microsoft Teams.

---

## Table of Contents

1. [HubSpot Integration](#1-hubspot-integration)
2. [Gmail Integration](#2-gmail-integration)
3. [Slack Integration](#3-slack-integration)
4. [Microsoft Teams Integration](#4-microsoft-teams-integration)
5. [Integration Architecture](#5-integration-architecture)
6. [Data Sync Strategies](#6-data-sync-strategies)
7. [Error Handling](#7-error-handling)

---

## 1. HubSpot Integration

### 1.1 Authentication

**OAuth 2.0 Flow:**

```
1. User clicks "Connect HubSpot"
2. Redirect to HubSpot OAuth URL:
   https://app.hubspot.com/oauth/authorize?
     client_id={CLIENT_ID}&
     redirect_uri={REDIRECT_URI}&
     scope={SCOPES}

3. User authorizes app
4. HubSpot redirects to callback with code
5. Exchange code for access token:
   POST https://api.hubapi.com/oauth/v1/token
   {
     grant_type: "authorization_code",
     client_id: CLIENT_ID,
     client_secret: CLIENT_SECRET,
     redirect_uri: REDIRECT_URI,
     code: CODE
   }

6. Store access_token and refresh_token
7. Set up token refresh (expires in 6 hours)
```

**Required Scopes:**
```
contacts
companies
deals
tickets
timeline
automation
crm.objects.contacts.read
crm.objects.contacts.write
crm.objects.companies.read
crm.objects.companies.write
```

### 1.2 Data Sync Mapping

**Companies → Accounts**

| HubSpot Field | Platform Field | Sync Direction | Transform |
|--------------|----------------|----------------|-----------|
| `hs_object_id` | `external_id` | Both | None |
| `name` | `name` | Both | None |
| `domain` | `domain` | Both | None |
| `industry` | `industry` | Both | None |
| `annualrevenue` | `arr_mrr` | Both | Parse number |
| `lifecyclestage` | `lifecycle_stage` | Both | Map values |
| `createdate` | `created_at` | HubSpot → Platform | Parse date |
| `hs_lastmodifieddate` | `updated_at` | Both | Parse date |
| Custom properties | `custom_fields` | Both | JSONB |

**Contacts → Contacts**

| HubSpot Field | Platform Field | Sync Direction | Transform |
|--------------|----------------|----------------|-----------|
| `hs_object_id` | `external_id` | Both | None |
| `email` | `email` | Both | Lowercase |
| `firstname` | `first_name` | Both | None |
| `lastname` | `last_name` | Both | None |
| `jobtitle` | `role` | Both | None |
| `phone` | `phone` | Both | Format |
| Associated company | `account_id` | Both | Lookup |

**Deals → Opportunities**

| HubSpot Field | Platform Field | Sync Direction | Transform |
|--------------|----------------|----------------|-----------|
| `hs_object_id` | `external_id` | Both | None |
| `dealname` | `name` | Both | None |
| `amount` | `value` | Both | Parse number |
| `dealstage` | `stage` | Both | Map stages |
| `closedate` | `close_date` | Both | Parse date |
| `pipeline` | `pipeline` | Both | None |

**Tickets → Support Tickets**

| HubSpot Field | Platform Field | Sync Direction | Transform |
|--------------|----------------|----------------|-----------|
| `hs_object_id` | `external_id` | Both | None |
| `subject` | `subject` | Both | None |
| `content` | `description` | Both | None |
| `hs_ticket_priority` | `priority` | Both | Map values |
| `hs_pipeline_stage` | `status` | Both | Map stages |

### 1.3 API Endpoints Used

**Read Operations:**
```typescript
// Get all companies (paginated)
GET https://api.hubapi.com/crm/v3/objects/companies
  ?limit=100
  &after={cursor}
  &properties=name,domain,industry,annualrevenue

// Get company by ID
GET https://api.hubapi.com/crm/v3/objects/companies/{companyId}
  ?properties=name,domain,industry

// Search companies
POST https://api.hubapi.com/crm/v3/objects/companies/search
{
  "filterGroups": [...],
  "sorts": [...],
  "properties": [...],
  "limit": 100,
  "after": cursor
}

// Get associations
GET https://api.hubapi.com/crm/v4/objects/companies/{companyId}/associations/contacts
```

**Write Operations:**
```typescript
// Create company
POST https://api.hubapi.com/crm/v3/objects/companies
{
  "properties": {
    "name": "Acme Corp",
    "domain": "acme.com",
    "industry": "Technology"
  }
}

// Update company
PATCH https://api.hubapi.com/crm/v3/objects/companies/{companyId}
{
  "properties": {
    "annualrevenue": "1000000"
  }
}

// Create association
PUT https://api.hubapi.com/crm/v4/objects/companies/{companyId}/associations/contacts/{contactId}
```

### 1.4 Webhook Configuration

**Subscribe to Webhooks:**

```typescript
POST https://api.hubapi.com/webhooks/v3/{appId}/subscriptions
{
  "enabled": true,
  "subscriptionDetails": {
    "subscriptionType": "company.propertyChange",
    "propertyName": "lifecyclestage"
  },
  "webhookUrl": "https://our-app.com/webhooks/hubspot"
}
```

**Webhook Events to Subscribe:**
- `company.creation`
- `company.propertyChange`
- `company.deletion`
- `contact.creation`
- `contact.propertyChange`
- `contact.deletion`
- `deal.creation`
- `deal.propertyChange`
- `deal.deletion`
- `ticket.creation`
- `ticket.propertyChange`

**Webhook Handler:**

```typescript
// POST /webhooks/hubspot
{
  "objectId": 123456,
  "propertyName": "lifecyclestage",
  "propertyValue": "customer",
  "changeSource": "CRM_UI",
  "eventId": 1234567890,
  "subscriptionId": 12345,
  "portalId": 62515,
  "appId": 12345,
  "occurredAt": 1614176862717,
  "subscriptionType": "company.propertyChange",
  "attemptNumber": 0
}
```

### 1.5 Rate Limits

- **Standard**: 100 requests per 10 seconds
- **Burst**: 150 requests (then throttled)
- **Daily**: 250,000 requests per day (Enterprise)

**Rate Limit Handling:**

```typescript
async function hubspotRequest(endpoint, options) {
  const rateLimiter = new RateLimiter({
    tokensPerInterval: 100,
    interval: 10000 // 10 seconds
  });

  await rateLimiter.removeTokens(1);

  try {
    const response = await fetch(endpoint, options);

    if (response.status === 429) {
      const retryAfter = response.headers.get('Retry-After');
      await sleep(retryAfter * 1000);
      return hubspotRequest(endpoint, options); // Retry
    }

    return response;
  } catch (error) {
    // Handle error
  }
}
```

### 1.6 Sync Implementation

**Initial Full Sync:**

```typescript
async function fullSyncHubSpot(organizationId: string) {
  const integration = await getIntegration(organizationId, 'hubspot');
  const hubspot = new HubSpotClient(integration.credentials);

  // Sync companies
  let after = undefined;
  do {
    const response = await hubspot.companies.list({ limit: 100, after });

    for (const company of response.results) {
      await upsertAccount({
        organizationId,
        externalId: company.id,
        source: 'hubspot',
        data: transformCompanyToAccount(company)
      });
    }

    after = response.paging?.next?.after;
  } while (after);

  // Sync contacts, deals, tickets...
}
```

**Incremental Sync (Webhook-based):**

```typescript
async function handleHubSpotWebhook(event: HubSpotWebhookEvent) {
  const integration = await getIntegrationByPortalId(event.portalId);

  if (event.subscriptionType === 'company.propertyChange') {
    const company = await hubspot.companies.get(event.objectId);

    await upsertAccount({
      organizationId: integration.organizationId,
      externalId: event.objectId,
      source: 'hubspot',
      data: transformCompanyToAccount(company),
      syncedAt: new Date(event.occurredAt)
    });
  }
}
```

---

## 2. Gmail Integration

### 2.1 Authentication

**OAuth 2.0 Flow:**

```typescript
const SCOPES = [
  'https://www.googleapis.com/auth/gmail.readonly',
  'https://www.googleapis.com/auth/gmail.send',
  'https://www.googleapis.com/auth/gmail.modify',
  'https://www.googleapis.com/auth/gmail.labels'
];

// Step 1: Generate auth URL
const authUrl = oauth2Client.generateAuthUrl({
  access_type: 'offline',
  scope: SCOPES,
  prompt: 'consent' // Force to get refresh token
});

// Step 2: Exchange code for tokens
const { tokens } = await oauth2Client.getToken(code);
oauth2Client.setCredentials(tokens);

// Store tokens securely
await storeIntegrationCredentials({
  organizationId,
  userId,
  type: 'gmail',
  credentials: {
    accessToken: tokens.access_token,
    refreshToken: tokens.refresh_token,
    expiryDate: tokens.expiry_date
  }
});
```

### 2.2 Email Sync Strategy

**Using Gmail Push Notifications:**

```typescript
// 1. Set up Cloud Pub/Sub topic
// 2. Watch inbox for changes

async function watchGmailInbox(userId: string) {
  const gmail = google.gmail({ version: 'v1', auth: oauth2Client });

  const response = await gmail.users.watch({
    userId: 'me',
    requestBody: {
      topicName: 'projects/YOUR_PROJECT/topics/gmail-notifications',
      labelIds: ['INBOX'],
      labelFilterAction: 'include'
    }
  });

  // Store historyId for incremental sync
  await storeHistoryId(userId, response.data.historyId);
}

// 3. Handle Pub/Sub notifications
async function handleGmailPushNotification(message: PubSubMessage) {
  const data = JSON.parse(Buffer.from(message.data, 'base64').toString());
  const { emailAddress, historyId } = data;

  // Fetch changes since last historyId
  await syncGmailHistory(emailAddress, historyId);
}
```

**Fetch and Store Emails:**

```typescript
async function syncGmailHistory(userId: string, startHistoryId: string) {
  const gmail = google.gmail({ version: 'v1', auth: oauth2Client });

  const response = await gmail.users.history.list({
    userId: 'me',
    startHistoryId,
    historyTypes: ['messageAdded', 'messageDeleted']
  });

  for (const history of response.data.history || []) {
    if (history.messagesAdded) {
      for (const added of history.messagesAdded) {
        await processGmailMessage(userId, added.message.id);
      }
    }
  }
}

async function processGmailMessage(userId: string, messageId: string) {
  const gmail = google.gmail({ version: 'v1', auth: oauth2Client });

  const message = await gmail.users.messages.get({
    userId: 'me',
    id: messageId,
    format: 'full'
  });

  const email = parseGmailMessage(message.data);

  // Find related account by email domain or contact
  const account = await findAccountByEmail(email.from);

  if (account) {
    await createActivity({
      accountId: account.id,
      type: 'email',
      source: 'gmail',
      direction: email.isInbound ? 'inbound' : 'outbound',
      subject: email.subject,
      content: email.body,
      metadata: {
        messageId: email.messageId,
        threadId: email.threadId,
        labels: email.labels,
        from: email.from,
        to: email.to,
        cc: email.cc
      },
      occurredAt: email.date
    });
  }
}
```

### 2.3 Send Email

```typescript
async function sendEmail(userId: string, emailData: EmailData) {
  const gmail = google.gmail({ version: 'v1', auth: oauth2Client });

  // Create email with tracking pixel
  const trackingPixelUrl = `https://our-app.com/track/email/${emailData.trackingId}/pixel.png`;

  const email = [
    'Content-Type: text/html; charset=utf-8',
    'MIME-Version: 1.0',
    `To: ${emailData.to}`,
    `Subject: ${emailData.subject}`,
    '',
    emailData.body,
    `<img src="${trackingPixelUrl}" width="1" height="1" />`
  ].join('\n');

  const encodedMessage = Buffer.from(email)
    .toString('base64')
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=+$/, '');

  const response = await gmail.users.messages.send({
    userId: 'me',
    requestBody: {
      raw: encodedMessage,
      threadId: emailData.threadId // For replies
    }
  });

  // Log sent email
  await createActivity({
    accountId: emailData.accountId,
    type: 'email',
    source: 'gmail',
    direction: 'outbound',
    subject: emailData.subject,
    content: emailData.body,
    metadata: {
      messageId: response.data.id,
      trackingId: emailData.trackingId
    },
    occurredAt: new Date()
  });

  return response.data;
}
```

### 2.4 Email Tracking

**Open Tracking:**

```typescript
// GET /track/email/:trackingId/pixel.png
async function trackEmailOpen(req: Request, res: Response) {
  const { trackingId } = req.params;

  await recordEmailEvent({
    trackingId,
    event: 'opened',
    userAgent: req.headers['user-agent'],
    ipAddress: req.ip,
    timestamp: new Date()
  });

  // Return 1x1 transparent pixel
  const pixel = Buffer.from(
    'iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+M9QDwADhgGAWjR9awAAAABJRU5ErkJggg==',
    'base64'
  );

  res.set('Content-Type', 'image/png');
  res.send(pixel);
}
```

**Click Tracking:**

```typescript
// Replace links in email with tracking URLs
function addClickTracking(html: string, trackingId: string): string {
  return html.replace(
    /href="([^"]+)"/g,
    (match, url) => {
      const trackingUrl = `https://our-app.com/track/email/${trackingId}/click?url=${encodeURIComponent(url)}`;
      return `href="${trackingUrl}"`;
    }
  );
}

// GET /track/email/:trackingId/click
async function trackEmailClick(req: Request, res: Response) {
  const { trackingId } = req.params;
  const { url } = req.query;

  await recordEmailEvent({
    trackingId,
    event: 'clicked',
    url,
    timestamp: new Date()
  });

  res.redirect(url as string);
}
```

---

## 3. Slack Integration

### 3.1 Authentication

**OAuth 2.0 Flow:**

```typescript
const SCOPES = [
  'channels:read',
  'channels:history',
  'groups:read',
  'groups:history',
  'chat:write',
  'users:read',
  'team:read',
  'im:read',
  'im:history'
];

// Step 1: Redirect to Slack OAuth
const authUrl = `https://slack.com/oauth/v2/authorize?
  client_id=${CLIENT_ID}&
  scope=${SCOPES.join(',')}&
  user_scope=&
  redirect_uri=${REDIRECT_URI}`;

// Step 2: Exchange code for tokens
const response = await fetch('https://slack.com/api/oauth.v2.access', {
  method: 'POST',
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
  body: new URLSearchParams({
    client_id: CLIENT_ID,
    client_secret: CLIENT_SECRET,
    code: code,
    redirect_uri: REDIRECT_URI
  })
});

const data = await response.json();

await storeIntegrationCredentials({
  organizationId,
  type: 'slack',
  credentials: {
    accessToken: data.access_token,
    botUserId: data.bot_user_id,
    teamId: data.team.id,
    teamName: data.team.name,
    scope: data.scope
  }
});
```

### 3.2 Slack Events API

**Subscribe to Events:**

```typescript
// Configure Events API in Slack App settings
// Subscribe to: message.channels, message.groups, message.im

// POST /webhooks/slack/events
async function handleSlackEvent(req: Request, res: Response) {
  const { type, challenge, event } = req.body;

  // URL verification
  if (type === 'url_verification') {
    return res.json({ challenge });
  }

  // Event callback
  if (type === 'event_callback') {
    await processSlackEvent(event);
    return res.status(200).send();
  }
}

async function processSlackEvent(event: SlackEvent) {
  if (event.type === 'message' && !event.subtype) {
    const integration = await getIntegrationByTeamId(event.team_id);

    // Check if message mentions a customer
    const customers = await findCustomerMentions(event.text);

    for (const customer of customers) {
      await createActivity({
        accountId: customer.id,
        type: 'slack_message',
        source: 'slack',
        direction: 'internal',
        content: event.text,
        metadata: {
          channel: event.channel,
          user: event.user,
          ts: event.ts,
          thread_ts: event.thread_ts
        },
        occurredAt: new Date(parseFloat(event.ts) * 1000)
      });
    }
  }
}
```

### 3.3 Send Slack Messages

```typescript
async function sendSlackMessage(options: {
  organizationId: string;
  channel: string;
  text: string;
  threadTs?: string;
}) {
  const integration = await getIntegration(options.organizationId, 'slack');
  const client = new WebClient(integration.credentials.accessToken);

  const response = await client.chat.postMessage({
    channel: options.channel,
    text: options.text,
    thread_ts: options.threadTs
  });

  return response;
}
```

### 3.4 Customer Mention Detection

```typescript
async function findCustomerMentions(text: string): Promise<Account[]> {
  // Extract company names from text
  const companyNames = extractCompanyNames(text);

  // Find accounts matching those names
  const accounts = await db.accounts.findMany({
    where: {
      OR: companyNames.map(name => ({
        name: { contains: name, mode: 'insensitive' }
      }))
    }
  });

  return accounts;
}

function extractCompanyNames(text: string): string[] {
  // Simple implementation - can be enhanced with NLP
  const words = text.split(/\s+/);
  const capitalizedWords = words.filter(w => /^[A-Z]/.test(w));

  // Look for sequences of capitalized words
  const names: string[] = [];
  let current = '';

  for (const word of capitalizedWords) {
    if (current) {
      current += ' ' + word;
    } else {
      current = word;
    }

    // Check if this matches a known company
    if (isLikelyCompanyName(current)) {
      names.push(current);
    }
  }

  return names;
}
```

---

## 4. Microsoft Teams Integration

### 4.1 Authentication

**Azure AD OAuth 2.0:**

```typescript
const SCOPES = [
  'https://graph.microsoft.com/Chat.ReadWrite',
  'https://graph.microsoft.com/Team.ReadBasic.All',
  'https://graph.microsoft.com/User.Read',
  'https://graph.microsoft.com/ChannelMessage.Read.All'
];

// Step 1: Redirect to Microsoft OAuth
const authUrl = `https://login.microsoftonline.com/common/oauth2/v2.0/authorize?
  client_id=${CLIENT_ID}&
  response_type=code&
  redirect_uri=${REDIRECT_URI}&
  scope=${SCOPES.join(' ')}&
  response_mode=query`;

// Step 2: Exchange code for tokens
const response = await fetch(
  'https://login.microsoftonline.com/common/oauth2/v2.0/token',
  {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      client_id: CLIENT_ID,
      client_secret: CLIENT_SECRET,
      code: code,
      redirect_uri: REDIRECT_URI,
      grant_type: 'authorization_code',
      scope: SCOPES.join(' ')
    })
  }
);

const data = await response.json();

await storeIntegrationCredentials({
  organizationId,
  type: 'teams',
  credentials: {
    accessToken: data.access_token,
    refreshToken: data.refresh_token,
    expiresIn: data.expires_in
  }
});
```

### 4.2 Microsoft Graph API Usage

**List Chats:**

```typescript
async function syncTeamsChats(organizationId: string) {
  const integration = await getIntegration(organizationId, 'teams');
  const client = Client.init({
    authProvider: (done) => {
      done(null, integration.credentials.accessToken);
    }
  });

  const chats = await client
    .api('/me/chats')
    .expand('members')
    .get();

  for (const chat of chats.value) {
    // Get messages from chat
    const messages = await client
      .api(`/me/chats/${chat.id}/messages`)
      .get();

    for (const message of messages.value) {
      await processTeamsMessage(organizationId, chat, message);
    }
  }
}
```

**Send Message:**

```typescript
async function sendTeamsMessage(options: {
  organizationId: string;
  chatId: string;
  content: string;
}) {
  const integration = await getIntegration(options.organizationId, 'teams');
  const client = Client.init({
    authProvider: (done) => {
      done(null, integration.credentials.accessToken);
    }
  });

  const message = await client
    .api(`/me/chats/${options.chatId}/messages`)
    .post({
      body: {
        content: options.content
      }
    });

  return message;
}
```

### 4.3 Teams Webhook Subscriptions

**Create Subscription:**

```typescript
async function createTeamsSubscription(organizationId: string) {
  const integration = await getIntegration(organizationId, 'teams');
  const client = Client.init({
    authProvider: (done) => {
      done(null, integration.credentials.accessToken);
    }
  });

  const subscription = await client
    .api('/subscriptions')
    .post({
      changeType: 'created,updated',
      notificationUrl: 'https://our-app.com/webhooks/teams',
      resource: '/me/chats/getAllMessages',
      expirationDateTime: new Date(Date.now() + 3600000).toISOString(), // 1 hour
      clientState: organizationId
    });

  // Store subscription ID for renewal
  await storeSubscriptionId(organizationId, subscription.id);
}

// Renew subscription every hour
async function renewTeamsSubscription(subscriptionId: string) {
  const subscription = await client
    .api(`/subscriptions/${subscriptionId}`)
    .patch({
      expirationDateTime: new Date(Date.now() + 3600000).toISOString()
    });
}
```

---

## 5. Integration Architecture

### 5.1 Unified Integration Service

```typescript
interface Integration {
  id: string;
  organizationId: string;
  type: 'hubspot' | 'gmail' | 'slack' | 'teams';
  status: 'active' | 'error' | 'disabled';
  credentials: EncryptedCredentials;
  configuration: IntegrationConfig;
  lastSyncAt: Date;
}

class IntegrationService {
  async connect(type: IntegrationType, authCode: string): Promise<Integration>;
  async disconnect(integrationId: string): Promise<void>;
  async sync(integrationId: string, syncType: 'full' | 'incremental'): Promise<SyncResult>;
  async handleWebhook(type: IntegrationType, payload: any): Promise<void>;
  async refreshCredentials(integrationId: string): Promise<void>;
}
```

### 5.2 Sync Worker Architecture

```mermaid
┌─────────────────────┐
│   Sync Scheduler    │
│   (Cron Jobs)       │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   Sync Queue        │
│   (BullMQ/Redis)    │
└──────────┬──────────┘
           │
    ┌──────┴──────┐
    ▼             ▼
┌────────┐    ┌────────┐
│Worker 1│    │Worker 2│
└────────┘    └────────┘
    │             │
    └──────┬──────┘
           ▼
┌─────────────────────┐
│ Integration Client  │
│  (HubSpot/Gmail/    │
│   Slack/Teams)      │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   Database          │
│   (Upsert Data)     │
└─────────────────────┘
```

**Queue Job Structure:**

```typescript
interface SyncJob {
  integrationId: string;
  syncType: 'full' | 'incremental';
  priority: number;
  retryCount: number;
  metadata?: any;
}

// Add job to queue
await syncQueue.add('sync-hubspot', {
  integrationId: integration.id,
  syncType: 'incremental',
  priority: 5
}, {
  attempts: 3,
  backoff: {
    type: 'exponential',
    delay: 2000
  }
});
```

---

## 6. Data Sync Strategies

### 6.1 Full Sync vs Incremental Sync

**Full Sync:**
- Runs on initial connection
- Runs daily as backup
- Fetches all data from source
- Slower but ensures consistency

**Incremental Sync:**
- Runs every 5-15 minutes
- Uses webhooks + polling fallback
- Only fetches changed data
- Faster and more efficient

### 6.2 Conflict Resolution

**Last-Write-Wins Strategy:**

```typescript
async function upsertWithConflictResolution(data: SyncData) {
  const existing = await db.accounts.findUnique({
    where: { externalId: data.externalId }
  });

  if (existing) {
    // Compare timestamps
    if (data.updatedAt > existing.updatedAt) {
      // External data is newer, update
      await db.accounts.update({
        where: { id: existing.id },
        data: {
          ...data,
          conflictedAt: null
        }
      });
    } else if (data.updatedAt < existing.updatedAt) {
      // Local data is newer, push to external
      await pushToExternalSystem(existing);
    } else {
      // Same timestamp, merge fields
      const merged = mergeData(existing, data);
      await db.accounts.update({
        where: { id: existing.id },
        data: merged
      });
    }
  } else {
    // Create new record
    await db.accounts.create({ data });
  }
}
```

### 6.3 Deduplication

```typescript
async function deduplicateAccount(accountData: AccountData): Promise<string> {
  // Try to find existing account by:
  // 1. External ID
  let existing = await db.accounts.findFirst({
    where: {
      externalId: accountData.externalId,
      source: accountData.source
    }
  });

  if (existing) return existing.id;

  // 2. Domain
  if (accountData.domain) {
    existing = await db.accounts.findFirst({
      where: { domain: accountData.domain }
    });

    if (existing) {
      // Link external ID to existing account
      await db.accounts.update({
        where: { id: existing.id },
        data: {
          externalId: accountData.externalId
        }
      });
      return existing.id;
    }
  }

  // 3. Company name (fuzzy match)
  const similar = await findSimilarAccountsByName(accountData.name);

  if (similar.length > 0) {
    // Present to user for manual merge decision
    await createMergeCandidate({
      newData: accountData,
      candidates: similar
    });
  }

  // Create new account
  const newAccount = await db.accounts.create({ data: accountData });
  return newAccount.id;
}
```

---

## 7. Error Handling

### 7.1 Error Types

```typescript
enum IntegrationErrorType {
  AUTH_EXPIRED = 'auth_expired',
  AUTH_INVALID = 'auth_invalid',
  RATE_LIMIT = 'rate_limit',
  API_ERROR = 'api_error',
  NETWORK_ERROR = 'network_error',
  VALIDATION_ERROR = 'validation_error',
  UNKNOWN = 'unknown'
}

class IntegrationError extends Error {
  constructor(
    public type: IntegrationErrorType,
    public message: string,
    public retryable: boolean,
    public metadata?: any
  ) {
    super(message);
  }
}
```

### 7.2 Retry Logic

```typescript
async function withRetry<T>(
  fn: () => Promise<T>,
  options: {
    maxAttempts: number;
    backoff: 'exponential' | 'linear';
    initialDelay: number;
  }
): Promise<T> {
  let lastError: Error;

  for (let attempt = 0; attempt < options.maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;

      if (!isRetryable(error)) {
        throw error;
      }

      const delay = options.backoff === 'exponential'
        ? options.initialDelay * Math.pow(2, attempt)
        : options.initialDelay * (attempt + 1);

      await sleep(delay);
    }
  }

  throw lastError;
}
```

### 7.3 Error Recovery

```typescript
async function handleIntegrationError(
  integrationId: string,
  error: IntegrationError
) {
  // Log error
  await db.integrationErrors.create({
    data: {
      integrationId,
      type: error.type,
      message: error.message,
      metadata: error.metadata,
      occurredAt: new Date()
    }
  });

  // Take action based on error type
  switch (error.type) {
    case IntegrationErrorType.AUTH_EXPIRED:
      await refreshIntegrationCredentials(integrationId);
      break;

    case IntegrationErrorType.AUTH_INVALID:
      await disableIntegration(integrationId);
      await notifyUser({
        integrationId,
        message: 'Please reconnect your integration'
      });
      break;

    case IntegrationErrorType.RATE_LIMIT:
      // Pause sync temporarily
      await pauseSync(integrationId, 60000); // 1 minute
      break;

    case IntegrationErrorType.API_ERROR:
      if (error.retryable) {
        // Retry in queue
        await requeueSync(integrationId);
      } else {
        await notifyAdmin({ integrationId, error });
      }
      break;
  }
}
```

---

## 8. Performance Optimization

### 8.1 Caching Strategy

```typescript
// Cache integration credentials (5 minutes)
const credentials = await cache.get(`integration:${integrationId}:credentials`);
if (!credentials) {
  credentials = await db.integrations.findUnique({
    where: { id: integrationId },
    select: { credentials: true }
  });
  await cache.set(
    `integration:${integrationId}:credentials`,
    credentials,
    300 // 5 minutes
  );
}

// Cache external ID mappings (1 hour)
const accountId = await cache.get(`mapping:hubspot:${externalId}`);
if (!accountId) {
  const account = await db.accounts.findFirst({
    where: { externalId, source: 'hubspot' }
  });
  if (account) {
    await cache.set(
      `mapping:hubspot:${externalId}`,
      account.id,
      3600 // 1 hour
    );
  }
}
```

### 8.2 Batch Processing

```typescript
async function batchUpsertAccounts(accounts: AccountData[]) {
  const BATCH_SIZE = 100;

  for (let i = 0; i < accounts.length; i += BATCH_SIZE) {
    const batch = accounts.slice(i, i + BATCH_SIZE);

    await db.$transaction(
      batch.map(account =>
        db.accounts.upsert({
          where: { externalId: account.externalId },
          create: account,
          update: account
        })
      )
    );
  }
}
```

---

**Last Updated**: 2026-01-22
