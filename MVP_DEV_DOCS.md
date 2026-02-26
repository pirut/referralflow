# ReferralFlow - MVP Technical Specification

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Frontend | Next.js + Tailwind + shadcn/ui |
| Backend | Next.js API Routes |
| Database | PostgreSQL (Supabase or Neon) |
| Auth | Clerk or Supabase Auth |
| Payments | Stripe |
| Email | Resend |
| SMS (optional) | Twilio |

---

## Core Features (MVP)

### F1: Business Dashboard
- View total referrals this month
- See top referrers
- See referral conversion rate
- Simple stats cards

### F2: Customer Management
- Add customers manually
- Import CSV
- View customer referral history
- Assign referral codes

### F3: Referral Codes
- Auto-generate unique codes per customer
- Manual code creation
- Code format: [BUSINESS]-[CODE] e.g., SALON-ABC123
- Trackable links (optional)

### F4: Reward Configuration
- Set reward for referrer (discount %, $ credit)
- Set reward for new customer (first visit discount)
- Enable/disable rewards
- Set max monthly rewards

### F5: Referral Tracking
- New customer enters referral code
- System tracks: referrer, referred, date, reward status
- Mark rewards as "given" or "expired"

---

## Data Models

### Business
```sql
CREATE TABLE businesses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  slug VARCHAR(100) UNIQUE NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  phone VARCHAR(50),
  address TEXT,
  tier VARCHAR(20) DEFAULT 'starter',
  reward_referrer_type VARCHAR(20) DEFAULT 'percentage',
  reward_referrer_value DECIMAL(10,2) DEFAULT 10,
  reward_new_type VARCHAR(20) DEFAULT 'percentage',
  reward_new_value DECIMAL(10,2) DEFAULT 10,
  max_monthly_rewards INTEGER DEFAULT 100,
  stripe_customer_id VARCHAR(255),
  created_at TIMESTAMP DEFAULT NOW()
);
```

### Customer
```sql
CREATE TABLE customers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255),
  phone VARCHAR(50),
  referral_code VARCHAR(50) UNIQUE NOT NULL,
  total_referrals INTEGER DEFAULT 0,
  total_rewards_earned DECIMAL(10,2) DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW()
);
```

### Referral
```sql
CREATE TABLE referrals (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_id UUID REFERENCES businesses(id) ON DELETE CASCADE,
  referrer_id UUID REFERENCES customers(id) ON DELETE CASCADE,
  referred_id UUID REFERENCES customers(id) ON DELETE CASCADE,
  reward_given BOOLEAN DEFAULT FALSE,
  reward_given_date TIMESTAMP,
  reward_type VARCHAR(20),
  created_at TIMESTAMP DEFAULT NOW()
);
```

### Indexes
```sql
CREATE INDEX idx_customers_business ON customers(business_id);
CREATE INDEX idx_customers_code ON customers(referral_code);
CREATE INDEX idx_referrals_business ON referrals(business_id);
CREATE INDEX idx_referrals_referrer ON referrals(referrer_id);
CREATE INDEX idx_referrals_referred ON referrals(referred_id);
```

---

## API Endpoints

### Auth
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | /api/auth/signup | Register new business |
| POST | /api/auth/login | Login |
| GET | /api/auth/me | Get current user |

### Businesses
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | /api/businesses | Create business |
| GET | /api/businesses/:id | Get business |
| PATCH | /api/businesses/:id | Update business |
| GET | /api/businesses/:id/stats | Get dashboard stats |

### Customers
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | /api/customers | Create customer |
| GET | /api/customers | List customers (filter by business_id) |
| GET | /api/customers/:id | Get customer |
| POST | /api/customers/import | Import CSV |
| POST | /api/customers/:id/refer | Record a referral |

### Referrals
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /api/referrals | List referrals |
| GET | /api/referrals/stats | Get referral stats |

---

## API Response Examples

### Get Dashboard Stats
```json
{
  "total_referrals": 156,
  "total_referrals_this_month": 23,
  "top_referrers": [
    { "name": "John Smith", "referral_count": 12, "rewards_earned": 120 },
    { "name": "Jane Doe", "referral_count": 8, "rewards_earned": 80 }
  ],
  "referral_conversion_rate": 0.34,
  "rewards_given_this_month": 230.50,
  "new_customers_this_month": 45
}
```

### Create Referral
```json
// POST /api/customers/:id/refer
// Body: { referred_code: "SALON-ABC123" }

{
  "success": true,
  "referral": {
    "id": "uuid",
    "referrer": "John Smith",
    "referred": "New Customer",
    "reward_given": false,
    "created_at": "2024-01-15T10:30:00Z"
  },
  "rewards_earned": {
    "referrer": 10.00,
    "new_customer": 15.00
  }
}
```

---

## User Flows

### Flow 1: Business Onboarding
1. Sign up → Email verification
2. Enter business name, industry
3. Configure rewards (referrer gets X%, new customer gets Y%)
4. Get referral code template
5. Redirect to dashboard

### Flow 2: Customer Gets Referred
1. New customer visits business
2. Staff asks "who referred you?"
3. New customer gives referral code
4. Staff enters code in system (or QR scan)
5. System validates code (exists + belongs to business)
6. System tracks referral
7. Both get reward notification

### Flow 3: Reward Redemption
1. Referrer returns → shows code
2. Staff applies discount
3. Marks reward as "given" in system

---

## UI Screens

### 1. Landing Page
- Hero: "Track referrals. Reward customers. Grow."
- Features: 3-4 key benefits
- Pricing: 3 tiers
- CTA: "Start Free Trial"

### 2. Sign Up
- Email + password
- Business name
- Industry selection

### 3. Dashboard
- Stats cards (total referrals, this month, conversion rate)
- Top referrers list
- Recent referrals table
- Quick actions: Add customer, Record referral

### 4. Customers Page
- Table of customers
- Columns: Name, Email, Code, Referrals, Rewards
- Search/filter
- Add customer button
- Import CSV button

### 5. Customer Detail
- Customer info
- Referral history
- Rewards earned
- Their unique referral code (for sharing)

### 6. Settings
- Business info
- Reward configuration
- Billing (Stripe)

---

## Security Considerations

- All API routes require authentication
- Business can only see their own customers/referrals
- Rate limiting on auth endpoints
- Input validation on all forms
- SQL injection prevention (use parameterized queries)

---

## MVP Milestones & Hours

| Milestone | Description | Hours |
|-----------|-------------|-------|
| M1 | Auth + Business CRUD | 4 |
| M2 | Customer management | 6 |
| M3 | Referral code generation | 4 |
| M4 | Recording referrals | 6 |
| M5 | Reward configuration | 4 |
| M6 | Dashboard + analytics | 6 |
| M7 | Landing page | 4 |
| M8 | Stripe integration | 4 |
| M9 | Testing + bug fixes | 6 |
| **Total** | | **44** |

---

## Future Features (Post-MVP)

- [ ] QR codes for easy referral entry
- [ ] SMS notifications for rewards
- [ ] Email marketing integration
- [ ] Zapier integration
- [ ] Webhook to POS systems
- [ ] Multi-location support
- [ ] API for developers
- [ ] White-label for agencies
