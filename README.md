Based on your requirements, ami tomake ekta complete SRS, System Design, and Schema Design provide korchi. This is a multi-tenant SaaS platform with serverless architecture. [mongodb](https://www.mongodb.com/docs/atlas/build-multi-tenant-arch/)

## Software Requirements Specification (SRS)

### 1. Introduction

#### 1.1 Purpose
This document specifies the requirements for a Multi-Tenant SaaS Platform that provides various software applications (POS, Hospital Management, Construction Management, etc.) through a unified backend with separate frontends. [github](https://github.com/jam01/SRS-Template)

#### 1.2 Scope
- Product Name: Universal SaaS Platform
- Backend: Serverless (AWS Lambda + API Gateway)
- Database: MongoDB with multi-tenant architecture
- Features: Multi-level user management, subscription management, app provisioning, demo/real app separation

#### 1.3 Definitions
- **Platform User**: Main system-level user who creates and manages apps
- **Software User**: Application-level user (e.g., cashier in POS system)
- **Tenant**: An organization/user instance with dedicated app(s)
- **App Instance**: A deployed software application (Demo or Real)

### 2. System Overview

#### 2.1 Product Perspective
A serverless multi-tenant SaaS platform enabling users to create and manage multiple software applications with flexible subscription models. [aws.amazon](https://aws.amazon.com/blogs/compute/building-multi-tenant-saas-applications-with-aws-lambdas-new-tenant-isolation-mode/)

#### 2.2 User Classes
1. **Platform Users**: Create accounts, manage apps, invite admins
2. **App Admins**: Manage specific app instances
3. **Software Users**: End-users within each software application
4. **Super Admin**: Platform administrators

### 3. Functional Requirements

#### 3.1 User Management (Platform Level)

**FR-1.1: User Registration**
- Users can register with email/phone/username
- Email/phone verification required
- Unique identifier across platform

**FR-1.2: Authentication**
- Dynamic authentication (email/phone/username based on user choice)
- JWT-based token authentication
- Refresh token mechanism

**FR-1.3: User Roles**
- Primary Owner: Creates apps
- Co-Admin: Invited by owner, can access/manage specific apps

#### 3.2 App Management

**FR-2.1: App Creation**
- Users can create multiple apps
- Two types: Demo and Real
- Each app type selection during creation

**FR-2.2: Demo Apps**
- Validity: 7 days from creation
- Auto-suspend after expiry
- No data retention after suspension
- Cannot be converted to Real

**FR-2.3: Real Apps**
- Unlimited validity with active subscription
- Cannot be directly deleted
- Deletion requires request → confirmation call → permanent deletion
- No data recovery after deletion

**FR-2.4: App Access Management**
- Owner can invite multiple admins per app
- Admins receive email/SMS invitation
- Role-based access control per app

#### 3.3 Subscription Management

**FR-3.1: Free Plan**
- Duration: 1-3 months (configurable)
- App Limit: 1-2 apps
- Auto-suspend after period expiry
- All features included

**FR-3.2: Paid Plans**
- Unlimited duration
- Includes: 2 instances per app type
- All features included
- Auto-renewal

**FR-3.3: Add-ons**
- Additional app instances beyond plan limit
- Per-instance pricing
- Immediate activation

#### 3.4 Software-Level User Management

**FR-4.1: Dynamic Authentication**
- Each software can choose: Username, Email, or Phone
- Independent from platform authentication
- Scoped to specific app instance

**FR-4.2: User Creation**
- Platform admins create software users
- Role assignment within software
- Permissions managed at software level

#### 3.5 Feature Distribution

**FR-5.1: Universal Feature Access**
- All updates available to all users
- No feature gating by subscription tier
- Only instance count limited by plan

### 4. Non-Functional Requirements

#### 4.1 Performance
- API response time: < 200ms (P95)
- Lambda cold start: < 1s
- Database query time: < 100ms

#### 4.2 Scalability
- Support 10,000+ tenants
- Auto-scaling with AWS Lambda
- Horizontal scaling for MongoDB

#### 4.3 Security
- Tenant data isolation [geeksforgeeks](https://www.geeksforgeeks.org/dbms/build-a-multi-tenant-architecture-in-mongodb/)
- Encrypted data at rest and in transit
- Rate limiting per tenant
- JWT with expiration

#### 4.4 Availability
- 99.9% uptime SLA
- Multi-region deployment
- Automated backups

## System Design

### Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Frontend Layer                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │   POS    │  │ Hospital │  │Construction│  ...        │
│  │  React   │  │  Next.js │  │   Vue.js   │             │
│  └──────────┘  └──────────┘  └──────────┘              │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                   API Gateway                            │
│            (Authentication & Routing)                    │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│              AWS Lambda Functions                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Auth       │  │    App       │  │  Subscription│  │
│  │  Service     │  │  Management  │  │  Service     │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   User       │  │   Software   │  │   Billing    │  │
│  │  Service     │  │   Services   │  │   Service    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                MongoDB (Multi-Tenant)                    │
│     Shared Collections with Tenant Isolation            │
└─────────────────────────────────────────────────────────┘
```

### Lambda Functions Structure

```
src/
├── functions/
│   ├── auth/
│   │   ├── register.js
│   │   ├── login.js
│   │   └── refresh-token.js
│   ├── apps/
│   │   ├── create-app.js
│   │   ├── list-apps.js
│   │   ├── invite-admin.js
│   │   └── request-deletion.js
│   ├── subscriptions/
│   │   ├── create-subscription.js
│   │   ├── upgrade-plan.js
│   │   └── add-addon.js
│   ├── software-users/
│   │   ├── create-user.js
│   │   ├── authenticate-user.js
│   │   └── manage-roles.js
│   └── cron/
│       ├── suspend-expired-demos.js
│       └── suspend-expired-free-plans.js
├── layers/
│   ├── database/
│   │   └── mongodb-connection.js
│   ├── middleware/
│   │   ├── tenant-isolation.js
│   │   └── auth-middleware.js
│   └── utils/
│       └── helpers.js
└── serverless.yml
```

## Database Schema Design

### Multi-Tenant Strategy
Using **Shared Collections with Tenant Field** approach for optimal balance between isolation and scalability. [digitalocean](https://www.digitalocean.com/community/questions/i-am-using-mongodb-with-nodejs-application-how-to-setup-database-for-multi-tenancy-saas-application)

### Collections

#### 1. **users** (Platform Level)
```javascript
{
  _id: ObjectId,
  email: String,  // unique, optional
  phone: String,  // unique, optional
  username: String,  // unique, optional
  password: String,  // hashed
  fullName: String,
  authMethod: String,  // 'email', 'phone', 'username'
  isVerified: Boolean,
  role: String,  // 'user', 'super_admin'
  status: String,  // 'active', 'suspended', 'deleted'
  createdAt: Date,
  updatedAt: Date,
  lastLogin: Date
}

// Indexes
db.users.createIndex({ email: 1 }, { unique: true, sparse: true })
db.users.createIndex({ phone: 1 }, { unique: true, sparse: true })
db.users.createIndex({ username: 1 }, { unique: true, sparse: true })
```

#### 2. **apps** (Tenant/App Instances)
```javascript
{
  _id: ObjectId,
  tenantId: ObjectId,  // references users._id (owner)
  appType: String,  // 'pos', 'hospital', 'construction', etc.
  instanceType: String,  // 'demo', 'real'
  name: String,
  subdomain: String,  // unique subdomain
  status: String,  // 'active', 'suspended', 'pending_deletion'
  
  // Demo specific
  demoExpiryDate: Date,  // 7 days from creation
  
  // Real app specific
  deletionRequest: {
    requestedAt: Date,
    requestedBy: ObjectId,
    status: String,  // 'pending', 'confirmed', 'cancelled'
    confirmedBy: ObjectId,
    confirmationCallDate: Date
  },
  
  // Admins (multi-level access)
  admins: [{
    userId: ObjectId,  // references users._id
    role: String,  // 'owner', 'admin'
    addedAt: Date,
    addedBy: ObjectId
  }],
  
  // Software level auth config
  softwareAuthConfig: {
    method: String,  // 'username', 'email', 'phone'
    customFields: [String]  // additional required fields
  },
  
  subscriptionId: ObjectId,  // references subscriptions._id
  createdAt: Date,
  updatedAt: Date
}

// Indexes
db.apps.createIndex({ tenantId: 1, status: 1 })
db.apps.createIndex({ subdomain: 1 }, { unique: true })
db.apps.createIndex({ "admins.userId": 1 })
db.apps.createIndex({ demoExpiryDate: 1 }, { 
  expireAfterSeconds: 0,
  partialFilterExpression: { instanceType: 'demo', status: 'active' }
})
```

#### 3. **subscriptions**
```javascript
{
  _id: ObjectId,
  userId: ObjectId,  // references users._id
  planType: String,  // 'free', 'basic', 'pro', 'enterprise'
  status: String,  // 'active', 'expired', 'cancelled'
  
  // Plan limits
  limits: {
    appCount: Number,  // 1-2 for free
    instancesPerApp: Number,  // 2 for paid
    duration: Number  // months (null for unlimited)
  },
  
  // Current usage
  usage: {
    activeApps: Number,
    appInstances: [{
      appId: ObjectId,
      appType: String
    }]
  },
  
  // Add-ons
  addons: [{
    type: String,  // 'additional_instance'
    appType: String,
    quantity: Number,
    price: Number,
    addedAt: Date
  }],
  
  startDate: Date,
  expiryDate: Date,  // null for paid unlimited
  autoRenew: Boolean,
  
  // Payment info
  paymentMethod: String,
  lastPaymentDate: Date,
  nextBillingDate: Date,
  
  createdAt: Date,
  updatedAt: Date
}

// Indexes
db.subscriptions.createIndex({ userId: 1, status: 1 })
db.subscriptions.createIndex({ expiryDate: 1 })
```

#### 4. **software_users** (App-level users)
```javascript
{
  _id: ObjectId,
  appId: ObjectId,  // references apps._id
  tenantId: ObjectId,  // for isolation
  
  // Dynamic auth fields
  email: String,  // optional based on softwareAuthConfig
  phone: String,  // optional based on softwareAuthConfig
  username: String,  // optional based on softwareAuthConfig
  password: String,  // hashed
  
  // User details
  fullName: String,
  role: String,  // app-specific roles (cashier, manager, doctor, etc.)
  permissions: [String],  // app-specific permissions
  
  status: String,  // 'active', 'inactive'
  metadata: Object,  // app-specific data
  
  createdBy: ObjectId,  // platform user who created this
  createdAt: Date,
  updatedAt: Date,
  lastLogin: Date
}

// Indexes
db.software_users.createIndex({ appId: 1, email: 1 }, { 
  unique: true, 
  sparse: true,
  partialFilterExpression: { email: { $exists: true } }
})
db.software_users.createIndex({ appId: 1, phone: 1 }, { 
  unique: true, 
  sparse: true,
  partialFilterExpression: { phone: { $exists: true } }
})
db.software_users.createIndex({ appId: 1, username: 1 }, { 
  unique: true, 
  sparse: true 
})
db.software_users.createIndex({ tenantId: 1, appId: 1 })
```

#### 5. **app_data_{appType}** (Dynamic per app type)
```javascript
// Example: app_data_pos
{
  _id: ObjectId,
  appId: ObjectId,
  tenantId: ObjectId,  // tenant isolation
  
  // POS specific data
  products: [{...}],
  sales: [{...}],
  inventory: [{...}],
  
  createdAt: Date,
  updatedAt: Date
}

// Each app type has its own collection structure
// app_data_hospital, app_data_construction, etc.

// Indexes
db.app_data_pos.createIndex({ appId: 1, tenantId: 1 })
db.app_data_pos.createIndex({ tenantId: 1 })
```

#### 6. **deletion_requests**
```javascript
{
  _id: ObjectId,
  appId: ObjectId,
  requestedBy: ObjectId,
  tenantId: ObjectId,
  
  status: String,  // 'pending', 'approved', 'rejected'
  reason: String,
  
  callLog: {
    attemptedAt: [Date],
    connectedAt: Date,
    confirmedBy: ObjectId,  // admin who confirmed
    notes: String
  },
  
  scheduledDeletionDate: Date,
  deletedAt: Date,
  
  createdAt: Date,
  updatedAt: Date
}

// Indexes
db.deletion_requests.createIndex({ appId: 1, status: 1 })
db.deletion_requests.createIndex({ status: 1, scheduledDeletionDate: 1 })
```

#### 7. **feature_flags** (For universal feature distribution)
```javascript
{
  _id: ObjectId,
  featureName: String,
  description: String,
  appType: String,  // 'all', 'pos', 'hospital', etc.
  enabled: Boolean,
  rolloutPercentage: Number,  // 0-100
  createdAt: Date,
  updatedAt: Date
}

// Indexes
db.feature_flags.createIndex({ appType: 1, enabled: 1 })
```

### Data Isolation Strategy

```javascript
// Middleware for tenant isolation
const tenantIsolation = async (event, context) => {
  const userId = event.requestContext.authorizer.userId;
  const appId = event.pathParameters?.appId;
  
  // Verify user has access to this app
  const app = await db.apps.findOne({
    _id: appId,
    $or: [
      { tenantId: userId },
      { "admins.userId": userId }
    ]
  });
  
  if (!app) {
    throw new Error('Unauthorized');
  }
  
  // Attach to context for downstream use
  context.tenantId = app.tenantId;
  context.appId = appId;
  
  return { tenantId: app.tenantId, appId };
};
```

### Key Design Decisions

1. **Single Backend**: All Lambda functions in one Serverless Framework project, using tenant isolation middleware [aws.amazon](https://aws.amazon.com/blogs/compute/building-multi-tenant-saas-applications-with-aws-lambdas-new-tenant-isolation-mode/)

2. **Shared Collections with tenantId**: Optimal for your scale, easier management, better cost efficiency [geeksforgeeks](https://www.geeksforgeeks.org/dbms/build-a-multi-tenant-architecture-in-mongodb/)

3. **Dynamic Authentication**: Each app can choose auth method through `softwareAuthConfig`

4. **Feature Distribution**: All features enabled for all users via `feature_flags` collection

5. **Demo Auto-Expiry**: Using MongoDB TTL indexes for automatic demo suspension

6. **Soft Delete**: Deletion requests go through workflow before permanent deletion

Ei design MongoDB er multi-tenant best practices follow kore  and AWS Lambda er serverless architecture utilize kore. Tomar project requirements onujayi customize kora hoyeche jate scalability and maintainability ensure hoy. [mongodb](https://www.mongodb.com/docs/atlas/build-multi-tenant-arch/)
