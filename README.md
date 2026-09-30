# 🌾 AgriMind AI – Crop Advisory Assistant

> **Production-ready, AI-powered precision crop advisory system built for Replit and modern cloud environments.**  
> Powered by **Gemini 1.5 Flash**, **React 18**, **Node.js/Express**, and **PostgreSQL with Row Level Security (RLS)**.

---

## 🌟 Overview & Core Capabilities

AgriMind AI empowers farmers, agronomists, and agricultural researchers with scientific, location-specific crop health intelligence. By grounding crop diagnoses in specific plot characteristics (soil pH, organic matter, irrigation infrastructure) and dynamic environmental telemetry (weather, growth stage, foliar observations), AgriMind generates actionable, structured agronomic guidance.

### ✨ Key Features
1. **User Authentication & Session Security**: Secure bcrypt password hashing, `express-session` with `connect-pg-simple`, and cookie protection.
2. **Multi-Farm Plot Management**: Register, monitor, edit, and soft-delete multiple agricultural holdings with soil pH, organic matter %, and irrigation methods.
3. **5-Step Guided Advisory Wizard**:
   - `Step 1`: Farm Holding Selection (auto-injects soil & irrigation context).
   - `Step 2`: Crop Details & Phenological Stage (variety, planting date, target yield).
   - `Step 3`: Environment & Weather (temperature, humidity, rainfall, soil moisture).
   - `Step 4`: Visual Observations & Symptoms (chlorosis, blight, pests, weed pressure).
   - `Step 5`: Farmer Goals & Constraints (IPM vs Organic, budget limits, urgency).
4. **Server-Side Gemini 1.5 Flash AI**: Strictly validated structured JSON generation with rate-limiting (max 10 calls/hour/user).
5. **Differential Advisory Comparison**: Side-by-side comparison of baseline vs current advisories to track risk trajectory, NPK shifts, and threat resolution over time.
6. **Professional PDF Field Reports**: One-click branded PDF export (`jspdf` + `jspdf-autotable`) with executive summary, priority action checklist, fertilizer charts, and pre-harvest intervals.
7. **PostgreSQL Row Level Security (RLS)**: Enforces tenant isolation on `users`, `farms`, and `advisories` (`SET app.current_user_id = user.id`).
8. **Responsive Dark/Light Mode**: Aesthetic Tailwind design system with automatic preference detection and persistent local storage.

---

## 🛠️ Technology Stack

| Layer | Technology |
|---|---|
| **Frontend** | React 18, TypeScript, Vite, Tailwind CSS, React Router v6, React Hook Form, Zod, TanStack Query v5, Lucide React |
| **Backend** | Node.js, Express, TypeScript, TSX |
| **Database** | PostgreSQL with Row Level Security (RLS) & `connect-pg-simple` session store |
| **Artificial Intelligence** | `@google/generative-ai` & `@google/genai` (Model: `gemini-1.5-flash`), server-side only |
| **Document Export** | `jspdf`, `jspdf-autotable` |
| **Security & Hardening** | `helmet`, `cors`, `express-rate-limit`, `bcrypt`, Parameterized SQL Queries |

---

## 🗄️ Database Architecture & Row Level Security (RLS)

All database tables are defined in [`server/schema.sql`](server/schema.sql) with strict Row Level Security (RLS) policies:

### 1. `users` Table
- `id` (UUID PRIMARY KEY DEFAULT `gen_random_uuid()`)
- `email` (VARCHAR(255) UNIQUE NOT NULL)
- `password_hash` (VARCHAR(255) NOT NULL)
- `full_name` (VARCHAR(255) NOT NULL)
- `role` (VARCHAR(50) NOT NULL DEFAULT `'farmer'`)
- `created_at`, `updated_at` (TIMESTAMPTZ)

### 2. `farms` Table
- `id` (UUID PRIMARY KEY DEFAULT `gen_random_uuid()`)
- `user_id` (UUID REFERENCES `users(id)` ON DELETE CASCADE)
- `name` (VARCHAR(255) NOT NULL)
- `location_text` (TEXT NOT NULL)
- `latitude`, `longitude` (NUMERIC)
- `area_hectares` (NUMERIC(10, 2) NOT NULL)
- `soil_type` (VARCHAR(100) NOT NULL)
- `soil_ph` (NUMERIC(4, 2))
- `organic_matter_percent` (NUMERIC(5, 2))
- `irrigation_type` (VARCHAR(100) NOT NULL)
- `primary_crops` (TEXT[] NOT NULL DEFAULT `'{}'`)
- `is_deleted` (BOOLEAN DEFAULT `FALSE`) — Enables non-destructive soft-delete
- `created_at`, `updated_at` (TIMESTAMPTZ)

### 3. `advisories` Table
- `id` (UUID PRIMARY KEY DEFAULT `gen_random_uuid()`)
- `user_id` (UUID REFERENCES `users(id)` ON DELETE CASCADE)
- `farm_id` (UUID REFERENCES `farms(id)` ON DELETE CASCADE)
- `crop_name`, `variety`, `growth_stage` (VARCHAR)
- `input_payload` (JSONB NOT NULL)
- `ai_response` (JSONB NOT NULL)
- `confidence_score` (NUMERIC(5, 2) NOT NULL)
- `risk_level` (VARCHAR(50) NOT NULL)
- `model_used` (VARCHAR(100) DEFAULT `'gemini-1.5-flash'`)
- `tokens_used` (INTEGER DEFAULT `0`)
- `created_at` (TIMESTAMPTZ)

### 4. `session` Table
- Backs `connect-pg-simple` with persistent user logins.

### 🛡️ RLS Policy Enforcement
When executing queries for an authenticated user, the database connection sets:
```sql
SET app.current_user_id = '<user.id>';
```
Policies ensure that users cannot inspect, update, or delete any record outside their own session:
```sql
CREATE POLICY farm_isolation_policy ON farms
    FOR ALL
    USING (user_id::text = current_setting('app.current_user_id', true));
```

---

## 📡 API Specification

### Authentication (`/api/auth`)
- `POST /api/auth/register` — Create account (`email`, `password`, `full_name`, `role`).
- `POST /api/auth/login` — Sign in and establish session cookie.
- `POST /api/auth/logout` — Invalidate session and clear cookies.
- `GET /api/auth/me` — Retrieve currently logged-in user profile.

### Farms (`/api/farms`)
- `GET /api/farms` — List all active farms for the authenticated user.
- `POST /api/farms` — Create new farm holding with soil & irrigation specifications.
- `GET /api/farms/:id` — Get detailed metadata for a specific farm.
- `PUT /api/farms/:id` — Update farm details.
- `DELETE /api/farms/:id` — Soft-delete farm (`is_deleted = TRUE`).

### Advisories (`/api/advisories`)
- `POST /api/advisories` — Run Gemini 1.5 Flash diagnosis on farm/crop payload. Rate-limited to max 10/hour/user.
- `GET /api/advisories` — List user's past advisories (supports `?farm_id=...` filter).
- `GET /api/advisories/:id` — Retrieve full advisory record.
- `GET /api/advisories/compare?id1=...&id2=...` — Calculate differential metrics between two advisories.

---

## 🤖 Gemini 1.5 Flash AI Schema

AgriMind prompts Gemini 1.5 Flash with strict JSON output constraints. Every response is verified with Zod before saving:

```json
{
  "summary": "Detailed executive agronomic assessment",
  "confidence_score": 92,
  "risk_level": "low" | "medium" | "high" | "critical",
  "immediate_actions": [
    { "action": "Foliar Zinc spray", "priority": "high", "timeline": "Within 24 hours" }
  ],
  "fertilizer_plan": {
    "npk_ratio": "120:60:40 kg/ha N-P-K",
    "products": [
      { "name": "Urea", "dose": "50", "unit": "kg/ha" }
    ],
    "application_schedule": [
      { "stage": "Vegetative Growth", "recommendation": "Secondary top dressing" }
    ]
  },
  "irrigation_schedule": {
    "method": "Drip Irrigation",
    "frequency": "Every 3 days",
    "quantity_mm": "25 mm",
    "notes": "Maintain 70% field capacity"
  },
  "pest_disease_management": [
    {
      "threat": "Aphids / Thrips",
      "severity": "Moderate",
      "cultural": "Yellow sticky traps",
      "chemical": "Targeted active ingredient",
      "organic": "Cold-pressed Neem Oil"
    }
  ],
  "growth_stage_advice": "Detailed physiological guidance",
  "harvest_recommendation": {
    "window_start": "Stage estimate start",
    "window_end": "Stage estimate end",
    "indicators": ["75% canopy golden coloration"]
  },
  "yield_optimization_tips": ["Morning foliar application timing"],
  "soil_health_actions": ["Post-harvest green manuring"],
  "warnings": ["Check 48-hour rainfall forecast before spray"],
  "references": ["FAO Crop Evapotranspiration Guidelines"],
  "generated_at": "2026-09-30T14:00:00.000Z"
}
```

---

## 🚀 Deployment & Local Setup

### 1. Running on Replit
1. Import this repository into a new Repl.
2. In Replit's **Secrets (Environment Variables)** panel, add:
   - `DATABASE_URL`: Your Replit PostgreSQL connection string.
   - `SESSION_SECRET`: A secure random secret string.
   - `GEMINI_API_KEY`: Your Gemini API key from [Google AI Studio](https://aistudio.google.com/).
3. Click **Run**. Replit will run `npm run dev` and launch the application.

### 2. Running Locally
```bash
# 1. Clone and enter directory
cd agrimind-ai

# 2. Install dependencies
npm install

# 3. Configure environment variables
cp .env.example .env
# Edit .env and supply your GEMINI_API_KEY and DATABASE_URL

# 4. Initialize Database (Optional if using live Postgres)
npm run db:init

# 5. Start development servers
# Starts Express API server with TSX
npm run dev

# In a second terminal (for Vite hot reload during frontend work):
npm run dev:client
```

### 3. Production Build
```bash
npm run build
npm start
```
The server will automatically serve the compiled React SPA from `dist/client` alongside the API endpoints at `http://localhost:5000`.

---

## 🧪 Testing & Verification

AgriMind includes an automated end-to-end verification script testing all 13 core flows:
```bash
npx tsx server/testEndToEnd.ts
```
Expected output:
```text
Starting End-to-End API and RLS Verification Suite...
1. Testing GET /api/health -> 200 OK
2. Testing POST /api/auth/register -> 201 Created
3. Testing GET /api/auth/me -> 200 OK
4. Testing POST /api/farms -> 201 Created
5. Testing GET /api/farms -> 200 OK
6. Testing POST /api/advisories (Baseline) -> 201 Created
7. Testing POST /api/advisories (Follow-up) -> 201 Created
8. Testing GET /api/advisories/:id -> 200 OK
9. Testing GET /api/advisories/compare -> 200 OK
10. Testing PUT /api/farms/:id -> 200 OK
11. Testing DELETE /api/farms/:id -> 200 OK (Soft-delete verified)
12. Testing POST /api/auth/logout -> 200 OK (401 on protected route)
✅ ALL 13 TEST CASES PASSED WITH 100% SUCCESS!
```

---

## 📄 License
MIT License. Built for precision agriculture and sustainable crop productivity.
