# EV Saarthi — End-to-End Implementation Guide

> **Version:** 1.0 | **Date:** February 2026 | **Status:** Production-Ready Blueprint  
> **Audience:** Industry Engineers, Architects, Investors, and Deployment Teams

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Problem Statement & Solution Approach](#2-problem-statement--solution-approach)
3. [Feature Breakdown & Technical Specification](#3-feature-breakdown--technical-specification)
4. [System Architecture](#4-system-architecture)
5. [Technology Stack](#5-technology-stack)
6. [Database Design](#6-database-design)
7. [API Design & Contracts](#7-api-design--contracts)
8. [AI & ML Implementation](#8-ai--ml-implementation)
9. [Third-Party Integrations](#9-third-party-integrations)
10. [Security Architecture](#10-security-architecture)
11. [Scalability & Performance Strategy](#11-scalability--performance-strategy)
12. [Deployment & DevOps Pipeline](#12-deployment--devops-pipeline)
13. [Development Phases & Milestones](#13-development-phases--milestones)
14. [Testing Strategy](#14-testing-strategy)
15. [Monitoring & Observability](#15-monitoring--observability)
16. [Cost Estimation](#16-cost-estimation)
17. [Future Roadmap](#17-future-roadmap)
18. [Green Points & Carbon Footprint System](#18-green-points--carbon-footprint-system)

---

## 1. Executive Summary

**EV Saarthi** is a full-stack, cloud-native Progressive Web Application (PWA) built to serve as the definitive companion for electric vehicle owners in India. It solves the fragmented EV ecosystem problem by unifying charging station discovery, AI-powered trip planning, cost tracking, community intelligence, and multilingual (Hindi + English) AI assistance into a single, scalable platform.

### Why It Matters

| Metric | Value |
|---|---|
| EV owners in India (2026) | ~2.5 Million |
| Projected EV owners by 2030 | ~10 Million |
| Annual EV growth rate | >200% (two-wheelers) |
| Existing unified platforms | **Zero** |

### What We Build

A **community-first, AI-augmented** platform with 9 core modules:
1. Intelligent Charging Station Finder (Map-based)
2. AI-Powered Trip Planner with Range Prediction
3. Cost Tracker & FAME-II Subsidy Calculator
4. Community Hub (Reviews, Photos, Forums, Gamification)
5. EV Knowledge Hub (Guides, Buying Advice)
6. Analytics Dashboard (Usage, CO₂, Savings)
7. AI Chatbot (Hindi + English, 24/7)
8. Authentication & User Management
9. 🌿 **Green Points & Carbon Footprint Tracker** *(new)*

---

## 2. Problem Statement & Solution Approach

### 2.1 Core Problems

| Problem | Impact |
|---|---|
| Fragmented charging station data across 10+ apps | Users arrive at broken stations |
| Manufacturer range specs ignore Indian conditions (heat, terrain, traffic) | Range anxiety blocks adoption |
| No unified cost tracking or subsidy calculator | Users underestimate EV savings |
| No Hindi-language EV assistant | 600M+ Hindi speakers underserved |
| Siloed network apps (Tata Power, Ather, Statiq each have own app) | Poor UX, no cross-network view |

### 2.2 Solution Philosophy

```
Community Intelligence + AI Reasoning + India-First Design = EV Saarthi
```

- **Crowd-sourced truth**: Real-time station status from users beats official APIs
- **AI-calibrated range**: Weather + terrain + traffic + driving style = accurate predictions
- **Bilingual by default**: Hindi and English as first-class citizens, not afterthoughts
- **PWA-first**: No app store friction; works offline for saved trips and station data
- **Open network**: Supports ALL charging networks, not siloed to one

---

## 3. Feature Breakdown & Technical Specification

### 3.1 Feature 1: Intelligent Charging Station Finder

**User Story**: As an EV owner, I want to find nearby working charging stations with real-time status so I never arrive at a broken charger.

**Technical Implementation**:
- Map rendered via **Mapbox GL JS** (or Leaflet.js as fallback)
- Station data seeded from **Open Charge Map API** (free, 50k+ stations globally)
- Community status overlay: users submit status updates stored in PostgreSQL
- **Geospatial queries** using PostGIS extension: `ST_DWithin(location, user_point, radius)`
- Filters: connector type (Bharat AC-001, CCS2, CHAdeMO, Type 2), speed (3kW/22kW/50kW+), network, rating
- Station detail page: pricing, UPI support flag, amenities, safety rating, photo gallery
- **Real-time status**: WebSocket channel per station; users subscribe when viewing details

**Data Flow**:
```
User opens map → Frontend requests /api/stations?lat=&lng=&radius=
→ PostGIS spatial query → Returns GeoJSON → Mapbox renders pins
→ User taps pin → WebSocket joins station-{id} channel
→ Other users submit status → Broadcast to all subscribers
```

**Connector Type Enum**:
```
BHARAT_AC_001 | CCS2 | CHADEMO | TYPE_2 | BHARAT_DC_001 | GB_T
```

---

### 3.2 Feature 2: AI-Powered Trip Planner

**User Story**: As an EV owner planning Mumbai→Pune, I want a realistic range prediction accounting for July heat and highway traffic so I know exactly where to charge.

**Technical Implementation**:

**Range Prediction Formula (Rule-Based Engine v1)**:
```
effective_range = base_range × temp_factor × terrain_factor × traffic_factor × ac_factor × driver_factor

Where:
  base_range       = vehicle.wltp_range_km
  temp_factor      = 0.80 if temp > 35°C | 0.85 if temp < 10°C | 1.0 otherwise
  terrain_factor   = 1 - (elevation_gain_m / 1000) × 0.08
  traffic_factor   = 0.85 if city_ratio > 0.5 | 1.0 if highway
  ac_factor        = 0.90 if temp > 30°C | 1.0 otherwise
  driver_factor    = user.historical_efficiency_ratio (default 1.0)
```

**APIs Used**:
- **OpenRouteService** (free): Route geometry, elevation profile, distance
- **OpenWeatherMap API**: Current + 5-day forecast along route waypoints
- **Google Maps Directions API** (optional upgrade): Traffic-aware routing

**Charging Stop Optimizer**:
- Algorithm: Modified Dijkstra's / greedy interval scheduling
- Inputs: route waypoints, station locations, predicted SOC at each point
- Outputs: optimal stop sequence minimizing total trip time
- Constraints: station reliability score ≥ 3.5/5, charger speed ≥ user preference

**ML Upgrade Path (Phase 2)**:
- Collect actual vs predicted range from completed trips
- Train **XGBoost regression model** on features: vehicle_id, temp, elevation_gain, distance, traffic_ratio, driver_id
- Deploy via **FastAPI** microservice, called by main backend
- Retrain weekly with new trip data using scheduled job

---

### 3.3 Feature 3: Cost Tracker & Savings Calculator

**Technical Implementation**:
- Charging session model: `{user_id, station_id, energy_kwh, cost_inr, type: HOME|PUBLIC, timestamp}`
- Home electricity rate stored per user (state-wise defaults provided)
- **FAME-II Subsidy Calculator**:
  - Vehicle database with FAME-II eligibility flag and subsidy amount
  - State incentive table: `{state, road_tax_waiver_pct, registration_waiver, additional_subsidy}`
- **ICE Comparison Engine**:
  - User inputs: previous vehicle fuel efficiency (km/L), current petrol price (auto-fetched from PPAC API)
  - Formula: `fuel_cost_saved = (distance_km / fuel_efficiency) × petrol_price_per_litre`
- Charts rendered with **Recharts** (React) or **Chart.js**

---

### 3.4 Feature 4: Community Hub

**Technical Implementation**:
- Reviews: star ratings (1-5) across 5 dimensions + text + photo upload
- Photos: stored in **AWS S3** / **Cloudflare R2**, served via CDN
- Real-time status updates: stored with TTL (24h expiry for "charging now" status)
- **Gamification Engine**:
  ```
  Points: review=10, photo=5, status_update=2, helpful_vote_received=1
  Badges: First Review, Road Warrior (50 reviews), Photo Pro (25 photos)
  Leaderboard: weekly/monthly/all-time, queryable by region
  ```
- Forum: threaded discussions, tagged by vehicle model and topic
- **Trust System**: users with 100+ points get "Verified Contributor" badge; their status updates weighted 2× in aggregation

---

### 3.5 Feature 5: EV Knowledge Hub

**Technical Implementation**:
- Content stored as **MDX** (Markdown + JSX) files in `/content` directory
- Rendered server-side via Next.js with static generation (`getStaticProps`)
- Categories: Buying Guide, Subsidies, Home Charging, Best Practices, Service, Finance
- Search powered by **Algolia DocSearch** (free for open content) or **Meilisearch** (self-hosted)
- SEO-optimized: structured data (JSON-LD), Open Graph tags, sitemap.xml auto-generated
- Content update workflow: admin CMS via **Keystatic** or **Contentlayer**

---

### 3.6 Feature 6: Analytics Dashboard

**Technical Implementation**:
- Aggregation queries run on PostgreSQL with materialized views for performance
- Key metrics computed server-side, cached in **Redis** (15-min TTL)
- CO₂ calculation: `co2_avoided_kg = distance_km × 0.12` (avg ICE emission factor for India)
- Efficiency: `efficiency_kmkwh = total_distance / total_energy_consumed`
- Peer comparison: percentile rank among same vehicle model owners
- Charts: interactive time-series (Recharts), donut charts for cost breakdown

---

### 3.7 Feature 7: AI Chatbot (Hindi + English)

*(Detailed in Section 8)*

---

### 3.8 Feature 8: Authentication & User Management

**Methods Supported**:
- Phone + OTP (primary for India) via **MSG91** or **Twilio**
- Email + Password (bcrypt hashed)
- Google OAuth 2.0
- JWT access tokens (15min expiry) + refresh tokens (30 days, stored in httpOnly cookie)

---

### 3.9 Feature 9: 🌿 Green Points & Carbon Footprint Tracker

**User Story**: As an EV owner who has driven 1,000 km this month, I want to see how much CO₂ I've saved and earn Green Points I can redeem for rewards — so my eco-conscious choice is recognized and celebrated.

**Core Concept**:
> Every kilometer driven in an EV instead of a petrol vehicle saves ~120g of CO₂. EV Saarthi quantifies this impact and converts it into **Green Points** — a gamified currency that rewards sustainable driving.

**Points Earning Formula**:
```
green_points_earned = floor(distance_km / 10)

Examples:
  100 km  → 10 Green Points
  500 km  → 50 Green Points
  1000 km → 100 Green Points  🏆 "Green Century" badge unlocked
  5000 km → 500 Green Points  🌳 "Carbon Warrior" badge unlocked

Bonus Multipliers:
  × 1.5  → Trip completed without any fossil-fuel detour
  × 1.25 → Trip during peak pollution hours (6–10 AM, 5–9 PM)
  × 2.0  → First trip of the month (streak bonus)
```

**CO₂ Savings Calculation**:
```
co2_saved_kg = distance_km × 0.120
  (India avg ICE emission: 120g CO₂/km for petrol car)

Equivalents shown to user:
  Trees planted equivalent = co2_saved_kg / 21.77  (1 tree absorbs ~21.77 kg CO₂/year)
  Petrol litres saved      = distance_km / 15       (avg 15 km/L ICE)
  Money saved (₹)          = petrol_litres × current_petrol_price
```

**Badge Tiers**:
| Badge | Threshold | Icon | Reward |
|---|---|---|---|
| 🌱 Eco Starter | 50 km | Seedling | Profile badge |
| 🌿 Green Rider | 500 km | Leaf | Leaderboard highlight |
| 🌳 Carbon Saver | 2,000 km | Tree | Partner discount voucher |
| 🏆 Green Century | 10,000 km | Trophy | Premium feature unlock (1 month) |
| 🌍 Carbon Warrior | 50,000 km | Earth | Hall of Fame listing + certificate |

**Leaderboard**:
- Weekly / Monthly / All-time rankings
- Filter by: city, state, vehicle model
- Top 3 users featured on homepage "Green Heroes" section
- Shareable "My Green Impact" card (PNG) for social media

**Green Points Redemption (Phase 2)**:
- Partner discounts: charging network credits (Tata Power, Statiq)
- EV accessory vouchers (helmets, charger cables)
- Donation to tree-planting NGOs (1 point = ₹1 donated)
- Premium EV Saarthi features (ad-free, advanced analytics)

**Technical Implementation**:

```typescript
// lib/green-points.ts

export function calculateGreenPoints(distanceKm: number, bonuses: BonusFlags): number {
  const base = Math.floor(distanceKm / 10);
  let multiplier = 1.0;
  if (bonuses.noFossilDetour)   multiplier *= 1.5;
  if (bonuses.peakHours)        multiplier *= 1.25;
  if (bonuses.firstTripOfMonth) multiplier *= 2.0;
  return Math.floor(base * multiplier);
}

export function calculateCO2Saved(distanceKm: number) {
  const co2Kg = distanceKm * 0.120;
  return {
    co2_kg:          parseFloat(co2Kg.toFixed(2)),
    trees_equivalent: parseFloat((co2Kg / 21.77).toFixed(2)),
    petrol_litres:   parseFloat((distanceKm / 15).toFixed(2)),
    money_saved_inr: parseFloat(((distanceKm / 15) * CURRENT_PETROL_PRICE).toFixed(2)),
  };
}

// Called after every trip completion
export async function awardGreenPoints(userId: string, trip: Trip) {
  const points = calculateGreenPoints(trip.distanceKm, detectBonuses(trip));
  const co2    = calculateCO2Saved(trip.distanceKm);

  await prisma.$transaction([
    // Append to ledger
    prisma.greenPointsLedger.create({
      data: { userId, tripId: trip.id, points, co2SavedKg: co2.co2_kg,
              distanceKm: trip.distanceKm, reason: 'TRIP_COMPLETED' }
    }),
    // Update user total
    prisma.user.update({
      where: { id: userId },
      data:  { greenPoints: { increment: points }, totalCo2SavedKg: { increment: co2.co2_kg },
               totalGreenKm: { increment: trip.distanceKm } }
    }),
  ]);

  // Check and award badges
  await checkAndAwardBadges(userId);

  // Publish real-time notification
  await ably.channels.get(`user-${userId}`).publish('green_points_earned', {
    points, co2_kg: co2.co2_kg, badge_unlocked: null
  });
}
```

**Data Flow**:
```
User completes trip → POST /api/trips/{id}/complete
  → awardGreenPoints() called server-side
  → GreenPointsLedger row inserted
  → User.greenPoints + totalCo2SavedKg updated
  → Badge check runs (checkAndAwardBadges)
  → Real-time push to user via Ably
  → Frontend shows animated "+X Green Points" toast
  → Dashboard CO₂ card updates live
```

**UI Components**:
- **Green Impact Card**: Animated counter showing total CO₂ saved, trees equivalent, km driven green
- **Points Wallet**: Current balance, recent transactions, redemption options
- **City Leaderboard**: Top green drivers in user's city this month
- **Shareable Card Generator**: Canvas API generates a branded PNG card for Instagram/WhatsApp
- **Streak Tracker**: Consecutive months of green driving, with flame animation

---

## 4. System Architecture

### 4.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
│   Next.js PWA (React 18, TypeScript)                           │
│   Mapbox GL JS | Recharts | TailwindCSS                        │
│   Service Worker (Workbox) for offline caching                 │
└──────────────────────┬──────────────────────────────────────────┘
                       │ HTTPS / WebSocket
┌──────────────────────▼──────────────────────────────────────────┐
│                      API GATEWAY LAYER                          │
│   Next.js API Routes (Edge Runtime for low-latency)            │
│   Rate Limiting (Upstash Redis) | Auth Middleware              │
│   Input Validation (Zod)                                       │
└──────┬───────────────┬──────────────────┬───────────────────────┘
       │               │                  │
┌──────▼──────┐ ┌──────▼──────┐  ┌───────▼──────────┐
│  Core API   │ │  AI Service │  │  Realtime Engine │
│  (Next.js)  │ │  (FastAPI)  │  │  (Ably/Pusher)   │
│  PostgreSQL │ │  ML Models  │  │  WebSocket Rooms │
│  Prisma ORM │ │  Groq/Gemini│  │  Station Status  │
└──────┬──────┘ └─────────────┘  └──────────────────┘
       │
┌──────▼──────────────────────────────────────────────────────────┐
│                       DATA LAYER                                │
│   PostgreSQL + PostGIS (primary DB)                            │
│   Redis (caching, sessions, rate limiting)                     │
│   AWS S3 / Cloudflare R2 (media storage)                       │
│   Meilisearch (full-text search)                               │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Microservices Breakdown

| Service | Technology | Responsibility |
|---|---|---|
| **Web App** | Next.js 14 (App Router) | UI, SSR, API routes |
| **AI Service** | FastAPI (Python) | ML inference, chatbot orchestration |
| **Realtime** | Ably / Pusher | WebSocket station status |
| **Job Queue** | BullMQ + Redis | Async tasks (email, ML retraining) |
| **Search** | Meilisearch | Station and content search |
| **Media** | Cloudflare R2 + CDN | Photo storage and delivery |

### 4.3 Data Flow Diagrams

**Trip Planning Flow**:
```
User submits trip form
  → POST /api/trips/plan
  → Fetch route geometry (OpenRouteService)
  → Fetch elevation profile (OpenRouteService)
  → Fetch weather forecast (OpenWeatherMap)
  → Compute range prediction (rule-based engine or ML service)
  → Query stations along corridor (PostGIS)
  → Run charging stop optimizer
  → Return trip plan with SOC chart data
  → Frontend renders interactive map + timeline
```

**Community Status Update Flow**:
```
User submits station status
  → POST /api/stations/{id}/status
  → Validate user (JWT) + rate limit (1 update/station/hour)
  → Write to PostgreSQL
  → Publish to Ably channel "station-{id}"
  → All subscribers receive real-time update
  → Update station aggregate status (weighted by user trust score)
```

---

## 5. Technology Stack

### 5.1 Complete Stack Table

| Layer | Technology | Justification |
|---|---|---|
| **Frontend Framework** | Next.js 14 (App Router) | SSR/SSG, PWA, API routes in one framework |
| **Language** | TypeScript | Type safety, better DX, fewer runtime bugs |
| **Styling** | TailwindCSS + shadcn/ui | Rapid UI, consistent design system |
| **State Management** | Zustand + React Query (TanStack) | Lightweight global state + server state caching |
| **Maps** | Mapbox GL JS | Best-in-class vector maps, India coverage |
| **Charts** | Recharts | React-native, responsive, customizable |
| **Backend** | Next.js API Routes + FastAPI (AI) | Monorepo simplicity + Python ML ecosystem |
| **ORM** | Prisma | Type-safe DB queries, migrations, schema |
| **Primary DB** | PostgreSQL 15 + PostGIS | Relational + geospatial queries |
| **Cache** | Redis (Upstash serverless) | Sessions, rate limiting, API response cache |
| **Search** | Meilisearch | Fast, typo-tolerant, self-hostable |
| **Realtime** | Ably | WebSocket at scale, free tier generous |
| **Media Storage** | Cloudflare R2 | S3-compatible, zero egress fees |
| **AI/LLM** | Groq API (Llama 3.1) | Fast inference, free tier, multilingual |
| **Auth** | NextAuth.js v5 | OAuth, credentials, JWT, session management |
| **SMS/OTP** | MSG91 | Indian market leader, cheap OTP |
| **Email** | Resend | Modern email API, React Email templates |
| **Hosting** | Vercel (frontend) + Railway/Render (FastAPI) | Zero-config deploy, auto-scaling |
| **CDN** | Cloudflare | Global edge, DDoS protection |
| **CI/CD** | GitHub Actions | Automated test + deploy pipeline |
| **Monitoring** | Sentry + Vercel Analytics | Error tracking + performance |
| **Logging** | Axiom / Logtail | Structured log aggregation |

### 5.2 Why This Stack is Scalable

- **Next.js on Vercel**: Auto-scales to millions of requests; edge functions reduce latency globally
- **PostgreSQL + PostGIS**: Battle-tested for geospatial workloads; handles 10M+ rows efficiently
- **Redis**: Sub-millisecond cache; prevents DB overload at scale
- **Cloudflare R2**: Zero egress cost means photo-heavy community features don't bankrupt the project
- **Groq API**: 500 tokens/sec inference speed; handles concurrent chatbot sessions cheaply

---

## 6. Database Design

### 6.1 Core Schema (PostgreSQL + Prisma)

```prisma
// schema.prisma

model User {
  id                String    @id @default(cuid())
  phone             String?   @unique
  email             String?   @unique
  name              String
  avatarUrl         String?
  preferredLang     Lang      @default(ENGLISH)
  trustScore        Int       @default(0)
  points            Int       @default(0)  // community points
  greenPoints       Int       @default(0)  // 🌿 green points balance
  totalCo2SavedKg   Float     @default(0)  // lifetime CO₂ saved
  totalGreenKm      Float     @default(0)  // lifetime green km driven
  createdAt         DateTime  @default(now())
  
  vehicles          Vehicle[]
  sessions          ChargingSession[]
  reviews           Review[]
  statusUpdates     StationStatus[]
  trips             Trip[]
  greenLedger       GreenPointsLedger[]
  badges            Badge[]
}

model Vehicle {
  id              String    @id @default(cuid())
  userId          String
  make            String    // "Tata"
  model           String    // "Nexon EV"
  year            Int
  batteryCapacity Float     // kWh
  wltpRange       Int       // km
  connectorType   ConnectorType
  efficiencyRatio Float     @default(1.0) // learned from trips
  
  user            User      @relation(fields: [userId], references: [id])
  trips           Trip[]
}

model ChargingStation {
  id              String    @id @default(cuid())
  name            String
  network         String    // "Tata Power", "Ather Grid"
  location        Unsupported("geometry(Point,4326)")
  address         String
  city            String
  state           String
  connectors      Json      // [{type, speed_kw, count}]
  pricing         Json      // {per_kwh, per_min, flat_fee}
  upiSupported    Boolean   @default(false)
  amenities       String[]
  safetyRating    Float?
  overallRating   Float?
  reviewCount     Int       @default(0)
  isActive        Boolean   @default(true)
  source          String    // "OCM", "COMMUNITY", "MANUAL"
  
  reviews         Review[]
  statusUpdates   StationStatus[]
  
  @@index([location], type: Gist)
}

model StationStatus {
  id          String    @id @default(cuid())
  stationId   String
  userId      String
  status      StatusType // OPERATIONAL | BROKEN | OCCUPIED | MAINTENANCE
  note        String?
  createdAt   DateTime  @default(now())
  expiresAt   DateTime  // 24h TTL
  
  station     ChargingStation @relation(fields: [stationId], references: [id])
  user        User            @relation(fields: [userId], references: [id])
}

model Review {
  id              String    @id @default(cuid())
  stationId       String
  userId          String
  overallRating   Int       // 1-5
  speedRating     Int
  cleanlinessRating Int
  safetyRating    Int
  amenitiesRating Int
  text            String
  photoUrls       String[]
  helpfulVotes    Int       @default(0)
  createdAt       DateTime  @default(now())
  
  station         ChargingStation @relation(fields: [stationId], references: [id])
  user            User            @relation(fields: [userId], references: [id])
}

model ChargingSession {
  id          String    @id @default(cuid())
  userId      String
  stationId   String?
  vehicleId   String
  energyKwh   Float
  costInr     Float
  type        SessionType // HOME | PUBLIC
  startedAt   DateTime
  endedAt     DateTime
  
  user        User      @relation(fields: [userId], references: [id])
}

model Trip {
  id              String    @id @default(cuid())
  userId          String
  vehicleId       String
  origin          String
  destination     String
  distanceKm      Float
  plannedStops    Json      // [{stationId, plannedSOC, plannedDuration}]
  predictedRange  Float
  actualRange     Float?    // filled after trip completion
  weatherData     Json
  terrainData     Json
  status          TripStatus // PLANNED | ACTIVE | COMPLETED | CANCELLED
  createdAt       DateTime  @default(now())
  
  user            User      @relation(fields: [userId], references: [id])
  vehicle         Vehicle   @relation(fields: [vehicleId], references: [id])
  greenPoints     GreenPointsLedger?
}

// 🌿 Green Points Ledger — one row per trip/action
model GreenPointsLedger {
  id            String    @id @default(cuid())
  userId        String
  tripId        String?   @unique
  points        Int       // points awarded
  co2SavedKg    Float     // CO₂ saved in kg
  distanceKm    Float
  reason        PointReason // TRIP_COMPLETED | REFERRAL | STREAK_BONUS
  createdAt     DateTime  @default(now())

  user          User      @relation(fields: [userId], references: [id])
  trip          Trip?     @relation(fields: [tripId], references: [id])
}

// 🏆 Badges earned by user
model Badge {
  id          String    @id @default(cuid())
  userId      String
  type        BadgeType
  earnedAt    DateTime  @default(now())

  user        User      @relation(fields: [userId], references: [id])
  @@unique([userId, type])
}

enum Lang { ENGLISH HINDI }
enum ConnectorType { BHARAT_AC_001 CCS2 CHADEMO TYPE_2 BHARAT_DC_001 GB_T }
enum StatusType { OPERATIONAL BROKEN OCCUPIED MAINTENANCE UNKNOWN }
enum SessionType { HOME PUBLIC }
enum TripStatus { PLANNED ACTIVE COMPLETED CANCELLED }
enum PointReason { TRIP_COMPLETED REFERRAL STREAK_BONUS COMMUNITY_CONTRIBUTION }
enum BadgeType { ECO_STARTER GREEN_RIDER CARBON_SAVER GREEN_CENTURY CARBON_WARRIOR }
```

### 6.2 Indexes & Performance

```sql
-- Geospatial index (PostGIS)
CREATE INDEX stations_location_gist ON "ChargingStation" USING GIST (location);

-- Composite index for user dashboard queries
CREATE INDEX sessions_user_date ON "ChargingSession" (user_id, started_at DESC);

-- Partial index for active status updates
CREATE INDEX status_active ON "StationStatus" (station_id, created_at DESC)
  WHERE expires_at > NOW();

-- Full-text search index
CREATE INDEX stations_name_fts ON "ChargingStation" USING GIN (to_tsvector('english', name || ' ' || city));
```

### 6.3 Materialized Views (Analytics)

```sql
-- Refreshed every 15 minutes via cron job
CREATE MATERIALIZED VIEW user_monthly_stats AS
SELECT
  user_id,
  DATE_TRUNC('month', started_at) AS month,
  SUM(energy_kwh) AS total_energy,
  SUM(cost_inr) AS total_cost,
  COUNT(*) AS session_count
FROM "ChargingSession"
GROUP BY user_id, DATE_TRUNC('month', started_at);
```

---

## 7. API Design & Contracts

### 7.1 RESTful API Structure

```
Base URL: https://evsaarthi.in/api/v1

AUTH
  POST   /auth/send-otp          → Send OTP to phone
  POST   /auth/verify-otp        → Verify OTP, return JWT
  POST   /auth/refresh           → Refresh access token
  DELETE /auth/logout            → Invalidate refresh token

STATIONS
  GET    /stations               → List stations (geospatial filter)
  GET    /stations/:id           → Station detail
  POST   /stations               → Add new station (admin/community)
  POST   /stations/:id/status    → Submit real-time status
  GET    /stations/:id/reviews   → Paginated reviews
  POST   /stations/:id/reviews   → Submit review (with photo upload)

TRIPS
  POST   /trips/plan             → Generate trip plan (AI-powered)
  GET    /trips                  → User trip history
  POST   /trips/:id/complete     → Mark trip done, submit actual range

SESSIONS
  GET    /sessions               → User charging sessions
  POST   /sessions               → Log new session
  GET    /sessions/analytics     → Aggregated analytics

VEHICLES
  GET    /vehicles               → User vehicles
  POST   /vehicles               → Add vehicle
  PUT    /vehicles/:id           → Update vehicle

AI
  POST   /ai/chat                → Chatbot message (streaming SSE)
  POST   /ai/route-predict       → Range prediction (ML service)

USERS
  GET    /users/me               → Profile
  PUT    /users/me               → Update profile
  GET    /users/me/dashboard     → Dashboard aggregates

GREEN POINTS
  GET    /green/me               → User's green points balance, CO₂ stats, badges
  GET    /green/leaderboard      → City/state/national leaderboard
  GET    /green/me/ledger        → Points transaction history
  POST   /green/me/share-card    → Generate shareable impact PNG card
  GET    /green/badges           → All badge definitions + user's earned badges
```

### 7.2 Sample API Response: Station List

```json
GET /api/v1/stations?lat=19.0760&lng=72.8777&radius=5000&connector=CCS2

{
  "data": [
    {
      "id": "clx1234",
      "name": "Tata Power EZ Charge - Bandra",
      "network": "Tata Power",
      "distance_m": 1240,
      "location": { "lat": 19.0596, "lng": 72.8295 },
      "connectors": [
        { "type": "CCS2", "speed_kw": 50, "count": 2 }
      ],
      "pricing": { "per_kwh": 18, "currency": "INR" },
      "upi_supported": true,
      "overall_rating": 4.2,
      "review_count": 34,
      "current_status": "OPERATIONAL",
      "status_updated_at": "2026-02-18T08:45:00Z",
      "amenities": ["parking", "restroom", "cafe"]
    }
  ],
  "meta": { "total": 12, "page": 1, "radius_m": 5000 }
}
```

### 7.3 WebSocket Events (Ably)

```
Channel: station-{stationId}

Events Published:
  status_update → { userId, status, note, timestamp, trustScore }
  new_review    → { rating, preview, timestamp }
  charger_free  → { connectorType, timestamp }

Client Subscribes on station detail page open
Client Unsubscribes on page close
```

---

## 8. AI & ML Implementation

### 8.1 AI Chatbot Architecture

**Model**: Groq API with `llama-3.1-70b-versatile` (fast, multilingual, free tier: 14,400 req/day)

**System Prompt Engineering**:
```
You are EV Saarthi, an expert assistant for electric vehicle owners in India.
You answer in the same language the user writes in (Hindi or English).
You have deep knowledge of: Indian EV models (Tata Nexon EV, Ather 450X, Ola S1 Pro, 
MG ZS EV, Mahindra XUV400), FAME-II subsidies, state incentives, charging networks 
(Tata Power, Ather Grid, Statiq, Charge+Zone), home charging installation, 
battery care in Indian climate, and traffic regulations.
Always be helpful, concise, and accurate. If unsure, say so.
User context: {user_vehicle}, {user_city}, {user_language_preference}
```

**Streaming Response** (Server-Sent Events):
```typescript
// /api/ai/chat route
export async function POST(req: Request) {
  const { message, history } = await req.json();
  const user = await getUser(req);
  
  const stream = await groq.chat.completions.create({
    model: "llama-3.1-70b-versatile",
    messages: [
      { role: "system", content: buildSystemPrompt(user) },
      ...history,
      { role: "user", content: message }
    ],
    stream: true,
    max_tokens: 1024,
  });

  return new Response(
    new ReadableStream({
      async start(controller) {
        for await (const chunk of stream) {
          const text = chunk.choices[0]?.delta?.content || "";
          controller.enqueue(new TextEncoder().encode(`data: ${text}\n\n`));
        }
        controller.close();
      }
    }),
    { headers: { "Content-Type": "text/event-stream" } }
  );
}
```

**Knowledge Base Augmentation (RAG)**:
- Static knowledge: Indian EV specs, FAME-II rules, state incentives → stored as embeddings in **pgvector**
- At query time: embed user question → find top-5 relevant chunks → inject into prompt
- Embedding model: `text-embedding-3-small` (OpenAI) or `nomic-embed-text` (free, Ollama)

### 8.2 Range Prediction ML Service (FastAPI)

```python
# ai_service/main.py
from fastapi import FastAPI
from pydantic import BaseModel
import joblib
import numpy as np

app = FastAPI()
model = joblib.load("models/range_predictor_v1.pkl")  # XGBoost

class TripInput(BaseModel):
    vehicle_wltp_range: float
    temperature_celsius: float
    elevation_gain_m: float
    elevation_loss_m: float
    distance_km: float
    city_ratio: float       # 0-1, fraction of route in city
    driver_efficiency: float  # historical ratio, default 1.0
    ac_usage: bool

@app.post("/predict-range")
def predict_range(data: TripInput):
    features = np.array([[
        data.vehicle_wltp_range,
        data.temperature_celsius,
        data.elevation_gain_m,
        data.elevation_loss_m,
        data.distance_km,
        data.city_ratio,
        data.driver_efficiency,
        int(data.ac_usage)
    ]])
    predicted = model.predict(features)[0]
    confidence = calculate_confidence(data)
    return {
        "predicted_range_km": round(predicted, 1),
        "confidence": confidence,  # "HIGH" | "MEDIUM" | "LOW"
        "safety_buffer_km": round(predicted * 0.10, 1)
    }

def calculate_confidence(data: TripInput) -> str:
    uncertainty_factors = 0
    if data.temperature_celsius > 38 or data.temperature_celsius < 8:
        uncertainty_factors += 1
    if data.elevation_gain_m > 500:
        uncertainty_factors += 1
    if data.driver_efficiency == 1.0:  # no historical data
        uncertainty_factors += 1
    return ["HIGH", "MEDIUM", "LOW"][min(uncertainty_factors, 2)]
```

**ML Training Pipeline**:
```
1. Collect trip data: planned vs actual range (after trip completion)
2. Feature engineering: extract weather, terrain, vehicle, driver features
3. Train XGBoost model (scikit-learn pipeline)
4. Evaluate: RMSE < 15km target
5. Deploy updated model via GitHub Actions → Railway
6. Schedule: weekly retraining via BullMQ cron job
```

---

## 9. Third-Party Integrations

| Service | Purpose | Free Tier | Cost at Scale |
|---|---|---|---|
| **Mapbox** | Maps, geocoding, routing | 50k map loads/mo | $0.50/1k loads |
| **OpenRouteService** | Elevation, routing (backup) | 2000 req/day | Free (self-host) |
| **OpenWeatherMap** | Weather forecast | 1M calls/mo | $40/mo for 10M |
| **Open Charge Map** | Seed station data | Free API | Free |
| **Groq API** | LLM inference | 14,400 req/day | $0.27/1M tokens |
| **MSG91** | OTP SMS | 100 free | ₹0.18/SMS |
| **Resend** | Transactional email | 3000/mo | $20/mo for 50k |
| **Ably** | WebSocket realtime | 6M messages/mo | $29/mo for 30M |
| **Cloudflare R2** | Media storage | 10GB free | $0.015/GB |
| **Upstash Redis** | Serverless Redis | 10k req/day | $0.20/100k req |
| **Sentry** | Error monitoring | 5k errors/mo | $26/mo |

### 9.1 Integration Architecture

```typescript
// lib/integrations/weather.ts
export async function getWeatherAlongRoute(waypoints: LatLng[]): Promise<WeatherData[]> {
  // Sample every 50km along route
  const samplePoints = sampleWaypoints(waypoints, 50);
  
  const forecasts = await Promise.all(
    samplePoints.map(point =>
      fetch(`https://api.openweathermap.org/data/2.5/forecast?lat=${point.lat}&lon=${point.lng}&appid=${process.env.OWM_API_KEY}&units=metric`)
        .then(r => r.json())
    )
  );
  
  return forecasts.map(parseWeatherResponse);
}
```

---

## 10. Security Architecture

### 10.1 Authentication Flow

```
1. User enters phone number
2. POST /auth/send-otp → MSG91 sends 6-digit OTP (valid 5 min)
3. User enters OTP
4. POST /auth/verify-otp → Server validates → Issues:
   - Access Token (JWT, 15min, in memory)
   - Refresh Token (opaque, 30 days, httpOnly cookie)
5. API calls include Authorization: Bearer {accessToken}
6. On 401 → Client auto-refreshes via /auth/refresh
```

### 10.2 Security Measures

| Threat | Mitigation |
|---|---|
| SQL Injection | Prisma parameterized queries (no raw SQL) |
| XSS | Next.js auto-escapes JSX; CSP headers |
| CSRF | SameSite=Strict cookies; CSRF tokens for mutations |
| Rate Limiting | Upstash Redis: 100 req/min per IP; 10 OTP/hour per phone |
| Data Encryption | TLS 1.3 in transit; AES-256 at rest (Postgres, R2) |
| Photo Moderation | Cloudflare Images auto-moderation + user reporting |
| Spam Reviews | Rate limit: 1 review/station/user; trust score weighting |
| API Key Exposure | All keys in environment variables; never in client bundle |
| OWASP Top 10 | Security audit before launch; Snyk dependency scanning |

### 10.3 Data Privacy (India DPDP Act 2023 Compliance)

- Location data: collected only with explicit consent; not sold to third parties
- Data deletion: users can delete account + all data within 30 days
- Minimal collection: only data needed for features is stored
- Privacy policy: clear, plain-language, available in Hindi and English

---

## 11. Scalability & Performance Strategy

### 11.1 Caching Strategy

```
L1: Browser Cache (Service Worker)
  - Station list for last searched area: 5 min
  - Saved trips: indefinite (offline access)
  - Knowledge hub articles: 1 hour

L2: CDN Cache (Cloudflare)
  - Static assets (JS, CSS, images): 1 year
  - Knowledge hub pages (SSG): 1 hour
  - Station list API (public): 2 min

L3: Redis Cache (Upstash)
  - User dashboard aggregates: 15 min
  - Station search results: 2 min
  - Chatbot conversation context: 30 min session
  - FAME-II subsidy table: 24 hours

L4: Database (PostgreSQL)
  - Materialized views for analytics: refreshed every 15 min
  - PostGIS spatial indexes for station queries
```

### 11.2 Performance Targets

| Metric | Target | How |
|---|---|---|
| First Contentful Paint | < 1.5s | Next.js SSR + CDN |
| Time to Interactive | < 3s on 3G | Code splitting, lazy loading |
| API P95 latency | < 300ms | Redis cache + DB indexes |
| Map load time | < 2s | Mapbox vector tiles + CDN |
| Chatbot first token | < 500ms | Groq fast inference |
| Uptime | 99.9% | Vercel SLA + health checks |

### 11.3 Horizontal Scaling Plan

```
Users < 10,000:    Single Vercel deployment + 1 PostgreSQL instance (Railway)
Users 10k-100k:    Read replicas for PostgreSQL, Redis cluster
Users 100k-1M:     Database sharding by region, multiple AI service instances
Users > 1M:        Dedicated infrastructure, Kubernetes (GKE/EKS), global DB
```

---

## 12. Deployment & DevOps Pipeline

### 12.1 Infrastructure Overview

```
Production:
  Frontend + API Routes → Vercel (auto-scaling, global edge)
  AI Service (FastAPI)  → Railway (always-on, Python runtime)
  PostgreSQL            → Supabase or Railway (managed, daily backups)
  Redis                 → Upstash (serverless, pay-per-use)
  Media Storage         → Cloudflare R2
  Search                → Meilisearch on Railway
  Realtime              → Ably (managed WebSocket)

Staging:
  Mirror of production with separate DB
  Seeded with anonymized test data

Development:
  Local Docker Compose (Postgres + Redis + Meilisearch)
  Vercel dev server for Next.js
```

### 12.2 Docker Compose (Local Development)

```yaml
# docker-compose.yml
version: '3.8'
services:
  postgres:
    image: postgis/postgis:15-3.3
    environment:
      POSTGRES_DB: evsaarthi_dev
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: devpassword
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  meilisearch:
    image: getmeili/meilisearch:v1.6
    ports: ["7700:7700"]
    environment:
      MEILI_MASTER_KEY: devmasterkey

  ai-service:
    build: ./ai_service
    ports: ["8000:8000"]
    environment:
      GROQ_API_KEY: ${GROQ_API_KEY}

volumes:
  pgdata:
```

### 12.3 CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy EV Saarthi

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm run test
      - run: npm run test:e2e  # Playwright

  deploy-staging:
    needs: test
    if: github.event_name == 'pull_request'
    steps:
      - uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.ORG_ID }}
          vercel-project-id: ${{ secrets.PROJECT_ID }}

  deploy-production:
    needs: test
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-args: '--prod'
      - name: Deploy AI Service
        run: railway up --service ai-service
      - name: Run DB Migrations
        run: npx prisma migrate deploy
      - name: Notify Slack
        uses: rtCamp/action-slack-notify@v2
```

### 12.4 Environment Variables

```bash
# .env.production
# Database
DATABASE_URL="postgresql://user:pass@host:5432/evsaarthi"
DIRECT_URL="postgresql://user:pass@host:5432/evsaarthi"  # Prisma migrations

# Auth
NEXTAUTH_SECRET="<32-char-random-string>"
NEXTAUTH_URL="https://evsaarthi.in"
GOOGLE_CLIENT_ID="..."
GOOGLE_CLIENT_SECRET="..."

# APIs
MAPBOX_TOKEN="pk.eyJ1..."
OPENWEATHER_API_KEY="..."
GROQ_API_KEY="gsk_..."
MSG91_AUTH_KEY="..."
RESEND_API_KEY="re_..."

# Infrastructure
REDIS_URL="rediss://..."
ABLY_API_KEY="..."
R2_ACCESS_KEY_ID="..."
R2_SECRET_ACCESS_KEY="..."
R2_BUCKET_NAME="evsaarthi-media"
MEILISEARCH_HOST="https://search.evsaarthi.in"
MEILISEARCH_KEY="..."

# AI Service
AI_SERVICE_URL="https://ai.evsaarthi.in"
AI_SERVICE_SECRET="..."  # Internal auth between services
```

---

## 13. Development Phases & Milestones

### Phase 1: Foundation (Weeks 1–2)
**Goal**: Working auth + basic map

| Task | Owner | Output |
|---|---|---|
| Initialize Next.js 14 project with TypeScript | Dev | Repo setup |
| Configure Prisma + PostgreSQL + PostGIS | Dev | DB schema v1 |
| Implement NextAuth (phone OTP + Google) | Dev | Auth working |
| Seed charging station data from Open Charge Map | Dev | 5000+ stations in DB |
| Basic Mapbox map with station pins | Dev | Map renders |
| Deploy to Vercel + Railway (staging) | Dev | Staging URL live |

**Milestone**: Users can register and see charging stations on a map ✅

### Phase 2: Core Features (Weeks 3–4)
**Goal**: Station finder + vehicle management

| Task | Output |
|---|---|
| Station search + filters (connector, speed, network) | Filtered map |
| Station detail page (info, photos, status) | Detail page |
| Vehicle management (add/edit vehicle) | Vehicle profile |
| Photo upload to Cloudflare R2 | Media pipeline |
| Basic station status submission | Status updates |

**Milestone**: Full station finder with community status ✅

### Phase 3: Community & Analytics (Weeks 5–6)
**Goal**: Reviews, cost tracking, dashboard

| Task | Output |
|---|---|
| Review + rating system | Reviews live |
| Charging session logging | Session tracker |
| Cost analytics dashboard (Recharts) | Dashboard |
| FAME-II subsidy calculator | Calculator |
| Gamification (points, badges) | Leaderboard |
| Real-time status via Ably WebSocket | Live updates |

**Milestone**: Community hub + personal analytics live ✅

### Phase 4: AI Integration (Weeks 7–8)
**Goal**: Chatbot + trip planner

| Task | Output |
|---|---|
| Groq API chatbot (Hindi + English, streaming) | Chatbot live |
| Rule-based range prediction engine | Range calculator |
| OpenRouteService + OpenWeatherMap integration | Route + weather |
| Trip planner UI (map + SOC timeline) | Trip planner |
| FastAPI AI service deployment on Railway | AI service live |

**Milestone**: AI features functional and tested ✅

### Phase 5: Polish & Launch (Weeks 9–10)
**Goal**: Production-ready launch

| Task | Output |
|---|---|
| Hindi i18n (next-intl) across all pages | Full Hindi support |
| PWA manifest + Service Worker (Workbox) | Offline capability |
| Performance audit (Lighthouse ≥ 90) | Optimized app |
| Security audit (OWASP checklist) | Secure app |
| Knowledge hub content (20+ articles) | Content live |
| SEO: sitemap, meta tags, structured data | SEO ready |
| Monitoring: Sentry + Vercel Analytics | Observability |
| Load testing (k6): 1000 concurrent users | Scalability verified |

**Milestone**: Public launch at evsaarthi.in 🚀

---

## 14. Testing Strategy

### 14.1 Testing Pyramid

```
         ┌─────────────┐
         │  E2E Tests  │  Playwright (10 critical user flows)
         │   (Slow)    │
        ┌┴─────────────┴┐
        │ Integration   │  API route tests (Vitest + Supertest)
        │    Tests      │  DB integration tests
       ┌┴───────────────┴┐
       │   Unit Tests    │  Business logic, utilities, ML functions
       │    (Fast)       │  Vitest + React Testing Library
       └─────────────────┘
```

### 14.2 Key Test Cases

**E2E (Playwright)**:
1. User registers via phone OTP → adds vehicle → sees map
2. User searches stations near location → views detail → submits status
3. User plans Mumbai→Pune trip → sees charging stops → saves trip
4. User logs charging session → views cost dashboard
5. User writes review with photo → sees it on station page
6. Chatbot answers "Nexon EV ki range kitni hai?" in Hindi
7. User calculates FAME-II subsidy for Tata Tiago EV

**Unit Tests**:
- Range prediction formula with edge cases (extreme temp, high elevation)
- Geospatial distance calculations
- Gamification points calculation
- JWT token validation
- OTP expiry logic

### 14.3 Performance Testing

```bash
# k6 load test script
import http from 'k6/http';
export const options = {
  stages: [
    { duration: '2m', target: 100 },   // Ramp up
    { duration: '5m', target: 1000 },  // Sustained load
    { duration: '2m', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<300'],  // 95% under 300ms
    http_req_failed: ['rate<0.01'],    // < 1% errors
  },
};
export default function () {
  http.get('https://staging.evsaarthi.in/api/v1/stations?lat=19.07&lng=72.87&radius=5000');
}
```

---

## 15. Monitoring & Observability

### 15.1 Monitoring Stack

| Tool | Purpose | Alert Threshold |
|---|---|---|
| **Vercel Analytics** | Core Web Vitals, traffic | LCP > 2.5s |
| **Sentry** | Error tracking, performance | Error rate > 1% |
| **Upstash** | Redis metrics | Memory > 80% |
| **Railway** | AI service health | CPU > 80%, restarts |
| **Ably** | WebSocket metrics | Message failures |
| **Cloudflare** | CDN, DDoS, traffic | Anomaly detection |

### 15.2 Key Dashboards

- **Business**: DAU, MAU, new registrations, chatbot sessions, trips planned
- **Technical**: API latency P50/P95/P99, error rates, DB query times
- **AI**: Chatbot satisfaction scores, range prediction accuracy (actual vs predicted)
- **Community**: Reviews/day, status updates/day, photo uploads/day

### 15.3 Alerting

```yaml
# PagerDuty / Slack alerts
- name: High Error Rate
  condition: error_rate > 2% for 5 minutes
  severity: CRITICAL
  
- name: Slow API
  condition: p95_latency > 1000ms for 10 minutes
  severity: WARNING
  
- name: DB Connection Pool Exhausted
  condition: db_connections > 90% of max
  severity: CRITICAL
  
- name: AI Service Down
  condition: ai_service_health_check fails 3 times
  severity: HIGH
```

---

## 16. Cost Estimation

### 16.1 Monthly Infrastructure Costs

| Stage | Users | Monthly Cost (INR) |
|---|---|---|
| **MVP Launch** | 0–1,000 | ₹2,000–₹4,000 |
| **Early Growth** | 1k–10k | ₹5,000–₹15,000 |
| **Scale** | 10k–100k | ₹25,000–₹80,000 |
| **Enterprise** | 100k+ | ₹1,50,000+ |

### 16.2 MVP Cost Breakdown (Month 1)

| Service | Plan | Cost |
|---|---|---|
| Vercel (hosting) | Pro | ₹1,700/mo |
| Railway (PostgreSQL + AI) | Starter | ₹1,200/mo |
| Upstash Redis | Pay-per-use | ₹200/mo |
| Cloudflare R2 | Free tier | ₹0 |
| Mapbox | Free (50k loads) | ₹0 |
| OpenWeatherMap | Free (1M calls) | ₹0 |
| Groq API | Free tier | ₹0 |
| MSG91 OTP | Pay-per-use | ₹500/mo |
| Ably | Free (6M msgs) | ₹0 |
| Domain + SSL | Annual | ₹100/mo |
| **Total** | | **~₹3,700/mo** |

---

## 17. Future Roadmap

### Phase 2 (Months 3–6): Intelligence Upgrade
- [ ] ML-based range prediction (XGBoost, trained on real trip data)
- [ ] Personalized charging station recommendations
- [ ] Battery health trend prediction
- [ ] Fleet management dashboard (B2B)
- [ ] Charging network API partnerships (Tata Power, Statiq)

### Phase 3 (Months 6–12): Platform Expansion
- [ ] Native iOS + Android apps (React Native)
- [ ] Additional regional languages (Tamil, Telugu, Marathi)
- [ ] In-app charging reservation + payment (UPI integration)
- [ ] EV manufacturer API integration (vehicle telemetry)
- [ ] Smart home charging optimization (solar + grid pricing)

### Phase 4 (Months 12–18): Monetization & Scale
- [ ] Premium subscription (₹99–199/mo): advanced analytics, fleet tools
- [ ] Charging network featured listings
- [ ] Affiliate revenue from EV purchases
- [ ] B2B fleet management SaaS
- [ ] International expansion (Bangladesh, Sri Lanka, Nepal)

---

## Appendix A: Project Repository Structure

```
evsaarthi/
├── app/                    # Next.js App Router
│   ├── (auth)/             # Auth pages (login, register)
│   ├── (dashboard)/        # Protected dashboard pages
│   │   ├── map/            # Station finder
│   │   ├── trip/           # Trip planner
│   │   ├── sessions/       # Charging sessions
│   │   ├── analytics/      # Dashboard
│   │   └── community/      # Reviews, forums
│   ├── api/                # API routes
│   │   ├── auth/
│   │   ├── stations/
│   │   ├── trips/
│   │   ├── sessions/
│   │   └── ai/
│   └── knowledge/          # Knowledge hub (SSG)
├── components/             # Reusable React components
│   ├── map/
│   ├── charts/
│   ├── chatbot/
│   └── ui/                 # shadcn/ui components
├── lib/                    # Utilities and integrations
│   ├── db.ts               # Prisma client
│   ├── auth.ts             # NextAuth config
│   ├── redis.ts            # Upstash client
│   ├── mapbox.ts
│   ├── weather.ts
│   └── range-predictor.ts
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── ai_service/             # FastAPI Python service
│   ├── main.py
│   ├── models/
│   └── requirements.txt
├── content/                # MDX knowledge hub articles
├── public/                 # Static assets + PWA manifest
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/               # Playwright tests
├── .github/workflows/      # CI/CD
├── docker-compose.yml
└── .env.example
```

---

## Appendix B: Key Technical Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Framework | Next.js 14 | SSR + API routes + PWA in one; Vercel deployment |
| DB | PostgreSQL + PostGIS | Geospatial queries are core; mature, scalable |
| AI Model | Groq + Llama 3.1 | Fast, cheap, multilingual; no GPU needed |
| Maps | Mapbox | Best India coverage; vector tiles; offline support |
| Realtime | Ably | Managed WebSocket; generous free tier |
| Auth | Phone OTP | Primary method for Indian users (80%+ preference) |
| Storage | Cloudflare R2 | Zero egress fees critical for photo-heavy community |
| Hosting | Vercel | Zero-config, auto-scaling, edge network |

---

*Document maintained by EV Saarthi Engineering Team. Last updated: February 2026.*
*For questions: engineering@evsaarthi.in*
