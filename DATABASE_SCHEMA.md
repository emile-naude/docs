# Database Schema Design

Complete database schema for the Customer Success Management platform with PostgreSQL, TimescaleDB, and supporting databases.

---

## Database Architecture

```
┌─────────────────────────────────────┐
│      PostgreSQL (Primary DB)       │
│  - Organizations & Users            │
│  - Accounts & Contacts              │
│  - Tasks & Projects                 │
│  - Playbooks & Configurations       │
│  - Integrations                     │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│      TimescaleDB (Time-Series)      │
│  - Product usage events             │
│  - Health score history             │
│  - Engagement metrics               │
│  - Activity logs                    │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│         Redis (Cache/Queue)         │
│  - Session cache                    │
│  - Integration credentials cache    │
│  - Job queue (BullMQ)               │
│  - Rate limiting                    │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│    Elasticsearch (Full-text)        │
│  - Account search                   │
│  - Activity search                  │
│  - Contact search                   │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│      Vector DB (AI Embeddings)      │
│  - Customer embeddings              │
│  - Semantic search                  │
│  - Similar customer matching        │
└─────────────────────────────────────┘
```

---

## PostgreSQL Schema

### Core Tables

#### 1. Organizations (Multi-tenant)

```sql
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  slug VARCHAR(100) UNIQUE NOT NULL,

  -- Subscription
  subscription_tier VARCHAR(50) NOT NULL DEFAULT 'starter',
  subscription_status VARCHAR(50) NOT NULL DEFAULT 'active',
  trial_ends_at TIMESTAMP,

  -- Settings
  settings JSONB DEFAULT '{}',

  -- Branding
  logo_url TEXT,
  primary_color VARCHAR(7),

  -- Billing
  stripe_customer_id VARCHAR(255),

  -- Metadata
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMP
);

CREATE INDEX idx_organizations_slug ON organizations(slug);
CREATE INDEX idx_organizations_subscription ON organizations(subscription_tier, subscription_status);
```

#### 2. Users

```sql
CREATE TYPE user_role AS ENUM ('admin', 'csm', 'viewer', 'owner');
CREATE TYPE user_status AS ENUM ('active', 'invited', 'suspended', 'deleted');

CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,

  -- Auth
  email VARCHAR(255) NOT NULL,
  email_verified BOOLEAN DEFAULT FALSE,
  password_hash TEXT, -- NULL for SSO users

  -- Profile
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  avatar_url TEXT,
  timezone VARCHAR(50) DEFAULT 'UTC',

  -- Role & Permissions
  role user_role NOT NULL DEFAULT 'csm',
  permissions JSONB DEFAULT '{}',
  status user_status NOT NULL DEFAULT 'active',

  -- Preferences
  preferences JSONB DEFAULT '{}',

  -- Security
  mfa_enabled BOOLEAN DEFAULT FALSE,
  mfa_secret TEXT,
  last_login_at TIMESTAMP,
  last_login_ip INET,

  -- Metadata
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMP,

  UNIQUE(organization_id, email)
);

CREATE INDEX idx_users_org_id ON users(organization_id);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(organization_id, role);
```

#### 3. Accounts (Customers)

```sql
CREATE TYPE lifecycle_stage AS ENUM (
  'lead',
  'trial',
  'onboarding',
  'active',
  'at_risk',
  'churned',
  'former'
);

CREATE TABLE accounts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,

  -- Basic Info
  name VARCHAR(255) NOT NULL,
  domain VARCHAR(255),
  website TEXT,
  industry VARCHAR(100),
  company_size VARCHAR(50),

  -- Lifecycle
  lifecycle_stage lifecycle_stage NOT NULL DEFAULT 'lead',
  lifecycle_stage_changed_at TIMESTAMP,

  -- Health
  health_score INTEGER CHECK (health_score >= 0 AND health_score <= 100),
  health_score_updated_at TIMESTAMP,
  health_trend VARCHAR(20), -- 'improving', 'stable', 'declining'

  -- Financial
  arr DECIMAL(12, 2),
  mrr DECIMAL(12, 2),
  contract_start_date DATE,
  contract_end_date DATE,
  renewal_date DATE,

  -- Owner
  owner_id UUID REFERENCES users(id) ON DELETE SET NULL,

  -- External IDs
  external_ids JSONB DEFAULT '{}', -- { hubspot: "123", salesforce: "456" }

  -- Custom Fields
  custom_fields JSONB DEFAULT '{}',

  -- Metadata
  tags TEXT[],
  notes TEXT,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMP
);

CREATE INDEX idx_accounts_org_id ON accounts(organization_id);
CREATE INDEX idx_accounts_domain ON accounts(domain);
CREATE INDEX idx_accounts_owner ON accounts(owner_id);
CREATE INDEX idx_accounts_lifecycle ON accounts(lifecycle_stage);
CREATE INDEX idx_accounts_health ON accounts(health_score);
CREATE INDEX idx_accounts_external_ids ON accounts USING GIN(external_ids);
CREATE INDEX idx_accounts_tags ON accounts USING GIN(tags);
```

#### 4. Contacts

```sql
CREATE TABLE contacts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,

  -- Basic Info
  email VARCHAR(255) NOT NULL,
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  phone VARCHAR(50),
  title VARCHAR(100),
  department VARCHAR(100),

  -- Flags
  is_primary BOOLEAN DEFAULT FALSE,
  is_decision_maker BOOLEAN DEFAULT FALSE,
  is_champion BOOLEAN DEFAULT FALSE,

  -- External IDs
  external_ids JSONB DEFAULT '{}',

  -- Social
  linkedin_url TEXT,
  twitter_handle VARCHAR(100),

  -- Metadata
  custom_fields JSONB DEFAULT '{}',
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMP,

  UNIQUE(organization_id, email)
);

CREATE INDEX idx_contacts_org_id ON contacts(organization_id);
CREATE INDEX idx_contacts_account_id ON contacts(account_id);
CREATE INDEX idx_contacts_email ON contacts(email);
CREATE INDEX idx_contacts_primary ON contacts(account_id, is_primary);
```

#### 5. Activities (Timeline)

```sql
CREATE TYPE activity_type AS ENUM (
  'email',
  'call',
  'meeting',
  'note',
  'slack_message',
  'teams_message',
  'task_completed',
  'lifecycle_change',
  'health_change',
  'custom'
);

CREATE TYPE activity_direction AS ENUM ('inbound', 'outbound', 'internal');

CREATE TABLE activities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  account_id UUID REFERENCES accounts(id) ON DELETE CASCADE,
  contact_id UUID REFERENCES contacts(id) ON DELETE SET NULL,

  -- Activity Details
  type activity_type NOT NULL,
  source VARCHAR(50), -- 'gmail', 'slack', 'teams', 'manual', 'hubspot'
  direction activity_direction,

  -- Content
  subject TEXT,
  content TEXT,
  summary TEXT, -- AI-generated summary

  -- Metadata
  metadata JSONB DEFAULT '{}', -- message_id, thread_id, etc.
  external_id VARCHAR(255),

  -- Who
  created_by UUID REFERENCES users(id) ON DELETE SET NULL,

  -- When
  occurred_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),

  -- Search
  search_vector tsvector GENERATED ALWAYS AS (
    to_tsvector('english', COALESCE(subject, '') || ' ' || COALESCE(content, ''))
  ) STORED
);

CREATE INDEX idx_activities_org_id ON activities(organization_id);
CREATE INDEX idx_activities_account_id ON activities(account_id);
CREATE INDEX idx_activities_type ON activities(type);
CREATE INDEX idx_activities_occurred ON activities(occurred_at DESC);
CREATE INDEX idx_activities_source ON activities(source, external_id);
CREATE INDEX idx_activities_search ON activities USING GIN(search_vector);
```

#### 6. Health Scores

```sql
CREATE TABLE health_scores (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,

  -- Score
  score INTEGER NOT NULL CHECK (score >= 0 AND score <= 100),

  -- Factor Breakdown
  factors JSONB NOT NULL, -- { product_usage: 80, engagement: 70, support: 90 }

  -- Metadata
  calculated_by VARCHAR(50) DEFAULT 'system', -- 'system', 'manual', 'ai'
  notes TEXT,

  -- When
  calculated_at TIMESTAMP NOT NULL DEFAULT NOW(),
  created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_health_scores_account ON health_scores(account_id, calculated_at DESC);
CREATE INDEX idx_health_scores_score ON health_scores(organization_id, score);
```

#### 7. Tasks

```sql
CREATE TYPE task_status AS ENUM ('todo', 'in_progress', 'done', 'cancelled');
CREATE TYPE task_priority AS ENUM ('low', 'medium', 'high', 'urgent');

CREATE TABLE tasks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  account_id UUID REFERENCES accounts(id) ON DELETE CASCADE,
  project_id UUID REFERENCES projects(id) ON DELETE CASCADE,

  -- Task Details
  title VARCHAR(500) NOT NULL,
  description TEXT,
  status task_status NOT NULL DEFAULT 'todo',
  priority task_priority NOT NULL DEFAULT 'medium',

  -- Assignment
  assigned_to UUID REFERENCES users(id) ON DELETE SET NULL,
  created_by UUID REFERENCES users(id) ON DELETE SET NULL,

  -- Dates
  due_date TIMESTAMP,
  completed_at TIMESTAMP,

  -- Metadata
  metadata JSONB DEFAULT '{}',
  tags TEXT[],

  -- Timestamps
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tasks_org_id ON tasks(organization_id);
CREATE INDEX idx_tasks_account_id ON tasks(account_id);
CREATE INDEX idx_tasks_assigned_to ON tasks(assigned_to);
CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_due_date ON tasks(due_date);
```

#### 8. Projects

```sql
CREATE TYPE project_type AS ENUM (
  'onboarding',
  'implementation',
  'renewal',
  'expansion',
  'qbr',
  'custom'
);

CREATE TYPE project_status AS ENUM (
  'not_started',
  'in_progress',
  'on_hold',
  'completed',
  'cancelled'
);

CREATE TABLE projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,

  -- Project Details
  name VARCHAR(255) NOT NULL,
  description TEXT,
  type project_type NOT NULL,
  status project_status NOT NULL DEFAULT 'not_started',

  -- Progress
  progress_percentage INTEGER DEFAULT 0 CHECK (progress_percentage >= 0 AND progress_percentage <= 100),

  -- Dates
  start_date DATE,
  target_end_date DATE,
  actual_end_date DATE,

  -- Owner
  owner_id UUID REFERENCES users(id) ON DELETE SET NULL,

  -- Metadata
  metadata JSONB DEFAULT '{}',

  -- Timestamps
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_projects_org_id ON projects(organization_id);
CREATE INDEX idx_projects_account_id ON projects(account_id);
CREATE INDEX idx_projects_status ON projects(status);
CREATE INDEX idx_projects_owner ON projects(owner_id);
```

#### 9. Playbooks

```sql
CREATE TYPE playbook_trigger AS ENUM (
  'lifecycle_change',
  'health_score_change',
  'time_based',
  'manual',
  'custom_event'
);

CREATE TABLE playbooks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,

  -- Playbook Details
  name VARCHAR(255) NOT NULL,
  description TEXT,

  -- Trigger
  trigger_type playbook_trigger NOT NULL,
  trigger_conditions JSONB NOT NULL, -- { lifecycle_stage: "trial", health_score: { lt: 50 } }

  -- Actions
  actions JSONB NOT NULL, -- [{ type: "create_task", data: {...} }, ...]

  -- Status
  is_active BOOLEAN DEFAULT TRUE,

  -- Stats
  execution_count INTEGER DEFAULT 0,
  success_count INTEGER DEFAULT 0,

  -- Metadata
  created_by UUID REFERENCES users(id) ON DELETE SET NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_playbooks_org_id ON playbooks(organization_id);
CREATE INDEX idx_playbooks_active ON playbooks(organization_id, is_active);
```

#### 10. Playbook Executions

```sql
CREATE TYPE execution_status AS ENUM ('pending', 'running', 'completed', 'failed');

CREATE TABLE playbook_executions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  playbook_id UUID NOT NULL REFERENCES playbooks(id) ON DELETE CASCADE,
  account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,

  -- Execution Details
  status execution_status NOT NULL DEFAULT 'pending',

  -- Results
  actions_completed INTEGER DEFAULT 0,
  actions_failed INTEGER DEFAULT 0,
  results JSONB DEFAULT '{}',
  error_log TEXT,

  -- Timing
  started_at TIMESTAMP,
  completed_at TIMESTAMP,
  created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_playbook_executions_playbook ON playbook_executions(playbook_id);
CREATE INDEX idx_playbook_executions_account ON playbook_executions(account_id);
CREATE INDEX idx_playbook_executions_status ON playbook_executions(status);
```

#### 11. Integrations

```sql
CREATE TYPE integration_type AS ENUM ('hubspot', 'gmail', 'slack', 'teams');
CREATE TYPE integration_status AS ENUM ('active', 'error', 'disabled', 'expired');

CREATE TABLE integrations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  user_id UUID REFERENCES users(id) ON DELETE SET NULL, -- For user-specific integrations

  -- Integration Details
  type integration_type NOT NULL,
  status integration_status NOT NULL DEFAULT 'active',

  -- Credentials (encrypted)
  credentials BYTEA NOT NULL, -- Encrypted JSON

  -- Configuration
  configuration JSONB DEFAULT '{}',

  -- Sync
  last_sync_at TIMESTAMP,
  last_sync_status VARCHAR(50),
  sync_interval INTEGER DEFAULT 900, -- seconds

  -- Metadata
  external_account_id VARCHAR(255), -- HubSpot portal ID, Slack team ID, etc.
  external_account_name VARCHAR(255),

  -- Timestamps
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW(),

  UNIQUE(organization_id, type, external_account_id)
);

CREATE INDEX idx_integrations_org_id ON integrations(organization_id);
CREATE INDEX idx_integrations_type ON integrations(type);
CREATE INDEX idx_integrations_status ON integrations(status);
```

#### 12. Sync Logs

```sql
CREATE TABLE sync_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  integration_id UUID NOT NULL REFERENCES integrations(id) ON DELETE CASCADE,
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,

  -- Sync Details
  sync_type VARCHAR(50) NOT NULL, -- 'full', 'incremental', 'webhook'
  status VARCHAR(50) NOT NULL, -- 'success', 'partial', 'failed'

  -- Stats
  records_fetched INTEGER DEFAULT 0,
  records_created INTEGER DEFAULT 0,
  records_updated INTEGER DEFAULT 0,
  records_deleted INTEGER DEFAULT 0,
  records_errored INTEGER DEFAULT 0,

  -- Errors
  errors JSONB DEFAULT '[]',

  -- Timing
  started_at TIMESTAMP NOT NULL,
  completed_at TIMESTAMP,
  duration_ms INTEGER,

  created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sync_logs_integration ON sync_logs(integration_id, started_at DESC);
CREATE INDEX idx_sync_logs_status ON sync_logs(status);
```

#### 13. Segments

```sql
CREATE TABLE segments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,

  -- Segment Details
  name VARCHAR(255) NOT NULL,
  description TEXT,

  -- Conditions
  conditions JSONB NOT NULL, -- { AND: [{ field: "health_score", op: "lt", value: 50 }] }

  -- Auto-update
  auto_update BOOLEAN DEFAULT TRUE,

  -- Cache
  account_count INTEGER DEFAULT 0,
  last_calculated_at TIMESTAMP,

  -- Metadata
  created_by UUID REFERENCES users(id) ON DELETE SET NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_segments_org_id ON segments(organization_id);
```

#### 14. Dashboards

```sql
CREATE TABLE dashboards (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,

  -- Dashboard Details
  name VARCHAR(255) NOT NULL,
  description TEXT,

  -- Layout
  layout JSONB NOT NULL, -- Grid layout configuration
  widgets JSONB NOT NULL, -- Widget configurations

  -- Sharing
  is_shared BOOLEAN DEFAULT FALSE,
  shared_with UUID[], -- User IDs

  -- Metadata
  created_by UUID REFERENCES users(id) ON DELETE SET NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_dashboards_org_id ON dashboards(organization_id);
CREATE INDEX idx_dashboards_created_by ON dashboards(created_by);
```

#### 15. Surveys

```sql
CREATE TYPE survey_type AS ENUM ('nps', 'csat', 'ces', 'custom');

CREATE TABLE surveys (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,

  -- Survey Details
  name VARCHAR(255) NOT NULL,
  type survey_type NOT NULL,

  -- Questions
  questions JSONB NOT NULL, -- Array of question objects

  -- Triggering
  trigger_rules JSONB DEFAULT '{}',

  -- Status
  is_active BOOLEAN DEFAULT TRUE,

  -- Metadata
  created_by UUID REFERENCES users(id) ON DELETE SET NULL,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_surveys_org_id ON surveys(organization_id);
```

#### 16. Survey Responses

```sql
CREATE TABLE survey_responses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  survey_id UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
  account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
  contact_id UUID REFERENCES contacts(id) ON DELETE SET NULL,

  -- Response Data
  responses JSONB NOT NULL, -- { question_id: answer }

  -- Score (for NPS/CSAT)
  score INTEGER,

  -- Timing
  submitted_at TIMESTAMP NOT NULL DEFAULT NOW(),

  -- Metadata
  metadata JSONB DEFAULT '{}'
);

CREATE INDEX idx_survey_responses_survey ON survey_responses(survey_id);
CREATE INDEX idx_survey_responses_account ON survey_responses(account_id);
CREATE INDEX idx_survey_responses_score ON survey_responses(score);
```

#### 17. AI Insights

```sql
CREATE TYPE insight_type AS ENUM (
  'summary',
  'sentiment',
  'churn_risk',
  'expansion_opportunity',
  'next_best_action'
);

CREATE TABLE ai_insights (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  account_id UUID REFERENCES accounts(id) ON DELETE CASCADE,

  -- Insight Details
  type insight_type NOT NULL,
  title VARCHAR(500),
  content TEXT NOT NULL,

  -- Confidence
  confidence_score DECIMAL(3, 2), -- 0.00 - 1.00

  -- Metadata
  metadata JSONB DEFAULT '{}',
  model_used VARCHAR(100),

  -- When
  generated_at TIMESTAMP NOT NULL DEFAULT NOW(),
  expires_at TIMESTAMP,

  -- Feedback
  feedback VARCHAR(50), -- 'helpful', 'not_helpful', 'incorrect'
  feedback_comment TEXT
);

CREATE INDEX idx_ai_insights_account ON ai_insights(account_id, generated_at DESC);
CREATE INDEX idx_ai_insights_type ON ai_insights(type);
```

---

## TimescaleDB Schema (Time-Series Data)

```sql
-- Enable TimescaleDB extension
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Product Usage Events
CREATE TABLE product_usage_events (
  time TIMESTAMPTZ NOT NULL,
  organization_id UUID NOT NULL,
  account_id UUID NOT NULL,
  user_id UUID,

  -- Event Details
  event_type VARCHAR(100) NOT NULL,
  feature_name VARCHAR(200),

  -- Metadata
  properties JSONB DEFAULT '{}',

  -- Session
  session_id VARCHAR(100)
);

-- Convert to hypertable
SELECT create_hypertable('product_usage_events', 'time');

-- Create indexes
CREATE INDEX idx_usage_events_account ON product_usage_events (account_id, time DESC);
CREATE INDEX idx_usage_events_type ON product_usage_events (event_type, time DESC);

-- Health Score Time-Series
CREATE TABLE health_score_timeseries (
  time TIMESTAMPTZ NOT NULL,
  organization_id UUID NOT NULL,
  account_id UUID NOT NULL,

  -- Score
  score INTEGER NOT NULL,

  -- Factor Breakdown
  product_usage_score INTEGER,
  engagement_score INTEGER,
  support_score INTEGER,
  financial_score INTEGER
);

SELECT create_hypertable('health_score_timeseries', 'time');
CREATE INDEX idx_health_ts_account ON health_score_timeseries (account_id, time DESC);

-- Engagement Metrics
CREATE TABLE engagement_metrics (
  time TIMESTAMPTZ NOT NULL,
  organization_id UUID NOT NULL,
  account_id UUID NOT NULL,

  -- Metric
  metric_type VARCHAR(100) NOT NULL, -- 'email_sent', 'email_opened', 'meeting_held', etc.
  value NUMERIC,

  -- Metadata
  metadata JSONB DEFAULT '{}'
);

SELECT create_hypertable('engagement_metrics', 'time');
CREATE INDEX idx_engagement_account ON engagement_metrics (account_id, time DESC);
CREATE INDEX idx_engagement_type ON engagement_metrics (metric_type, time DESC);

-- Continuous Aggregates (pre-computed rollups)
CREATE MATERIALIZED VIEW daily_account_health
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('1 day', time) AS day,
  account_id,
  AVG(score) as avg_score,
  MIN(score) as min_score,
  MAX(score) as max_score
FROM health_score_timeseries
GROUP BY day, account_id;

SELECT add_continuous_aggregate_policy('daily_account_health',
  start_offset => INTERVAL '1 month',
  end_offset => INTERVAL '1 day',
  schedule_interval => INTERVAL '1 hour');
```

---

## Entity Relationship Diagram

```
organizations
    │
    ├──< users
    │
    ├──< accounts
    │       │
    │       ├──< contacts
    │       ├──< activities
    │       ├──< health_scores
    │       ├──< tasks
    │       ├──< projects
    │       ├──< survey_responses
    │       └──< ai_insights
    │
    ├──< integrations
    │       └──< sync_logs
    │
    ├──< playbooks
    │       └──< playbook_executions
    │
    ├──< segments
    ├──< dashboards
    └──< surveys
```

---

## Sample Queries

### Get Account 360° View

```sql
WITH account_data AS (
  SELECT a.*,
    u.first_name || ' ' || u.last_name as owner_name,
    (SELECT COUNT(*) FROM contacts WHERE account_id = a.id) as contact_count,
    (SELECT COUNT(*) FROM tasks WHERE account_id = a.id AND status != 'done') as open_tasks,
    (SELECT score FROM health_scores WHERE account_id = a.id ORDER BY calculated_at DESC LIMIT 1) as latest_health_score
  FROM accounts a
  LEFT JOIN users u ON a.owner_id = u.id
  WHERE a.id = $1
),
recent_activities AS (
  SELECT * FROM activities
  WHERE account_id = $1
  ORDER BY occurred_at DESC
  LIMIT 10
),
active_projects AS (
  SELECT * FROM projects
  WHERE account_id = $1
    AND status IN ('in_progress', 'not_started')
)
SELECT
  json_build_object(
    'account', row_to_json(account_data),
    'recent_activities', (SELECT json_agg(row_to_json(recent_activities)) FROM recent_activities),
    'active_projects', (SELECT json_agg(row_to_json(active_projects)) FROM active_projects)
  ) as account_360;
```

### Health Score Distribution

```sql
SELECT
  CASE
    WHEN health_score >= 80 THEN 'Healthy'
    WHEN health_score >= 60 THEN 'Moderate'
    WHEN health_score >= 40 THEN 'At Risk'
    ELSE 'Critical'
  END as health_category,
  COUNT(*) as account_count,
  ROUND(AVG(arr), 2) as avg_arr
FROM accounts
WHERE organization_id = $1
  AND deleted_at IS NULL
GROUP BY health_category
ORDER BY MIN(health_score) DESC;
```

### CSM Performance

```sql
SELECT
  u.id,
  u.first_name || ' ' || u.last_name as csm_name,
  COUNT(DISTINCT a.id) as accounts_managed,
  SUM(a.arr) as total_arr,
  ROUND(AVG(a.health_score), 1) as avg_health_score,
  COUNT(DISTINCT t.id) FILTER (WHERE t.status = 'done') as tasks_completed,
  COUNT(DISTINCT t.id) FILTER (WHERE t.status = 'done' AND t.completed_at > NOW() - INTERVAL '30 days') as tasks_completed_30d
FROM users u
LEFT JOIN accounts a ON a.owner_id = u.id
LEFT JOIN tasks t ON t.assigned_to = u.id
WHERE u.organization_id = $1
  AND u.role = 'csm'
  AND u.status = 'active'
GROUP BY u.id, u.first_name, u.last_name
ORDER BY total_arr DESC;
```

### Churn Risk Analysis

```sql
SELECT
  a.id,
  a.name,
  a.health_score,
  a.mrr,
  a.renewal_date,
  DATE_PART('day', a.renewal_date - NOW()) as days_until_renewal,
  (SELECT COUNT(*) FROM activities WHERE account_id = a.id AND occurred_at > NOW() - INTERVAL '30 days') as activities_30d,
  (SELECT COUNT(*) FROM tasks WHERE account_id = a.id AND status != 'done') as open_tasks
FROM accounts a
WHERE a.organization_id = $1
  AND a.lifecycle_stage = 'active'
  AND (
    a.health_score < 50
    OR (a.renewal_date < NOW() + INTERVAL '90 days' AND a.health_score < 70)
  )
ORDER BY a.health_score ASC, a.renewal_date ASC;
```

### Product Usage Trends (TimescaleDB)

```sql
SELECT
  time_bucket('1 day', time) AS day,
  account_id,
  COUNT(*) as event_count,
  COUNT(DISTINCT feature_name) as features_used,
  COUNT(DISTINCT user_id) as active_users
FROM product_usage_events
WHERE organization_id = $1
  AND time > NOW() - INTERVAL '30 days'
GROUP BY day, account_id
ORDER BY day DESC, event_count DESC;
```

---

## Migrations Strategy

### Using Prisma

```prisma
// schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model Organization {
  id                  String   @id @default(uuid())
  name                String
  slug                String   @unique
  subscriptionTier    String   @default("starter")
  subscriptionStatus  String   @default("active")
  settings            Json     @default("{}")
  createdAt           DateTime @default(now())
  updatedAt           DateTime @updatedAt

  users               User[]
  accounts            Account[]
  integrations        Integration[]
  playbooks           Playbook[]

  @@map("organizations")
}

// ... rest of models
```

---

**Last Updated**: 2026-01-22
