# VoltWise Portal — Architecture

## Overview

The **VoltWise Portal** is the primary web interface for the VoltWise EV Charging Platform. It serves two main audiences:

1. **Prospective customers** — through a modern, conversion-optimized public landing page
2. **Hotel administrators** — through a data-rich dashboard to monitor and manage EV charging infrastructure

Built with **Vue 3**, the portal is a single-page application (SPA) that communicates with the VoltWise Cloud API. It is designed to be fast, accessible, mobile-first, and fully internationalized.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Vue 3 (Composition API, `<script setup>`) |
| Build tool | Vite 5 |
| Language | TypeScript |
| Routing | Vue Router 4 |
| State management | Pinia 2 |
| Styling | TailwindCSS 3 |
| i18n | Vue I18n 9 |
| HTTP client | Axios |
| Icons | Heroicons v2 (@heroicons/vue) |

---

## Project Structure

```
voltwise-portal/
├── index.html                  # Entry HTML, loads Inter font
├── vite.config.ts              # Vite config with path alias and API proxy
├── tailwind.config.js          # Brand colors, animation, font
├── src/
│   ├── main.ts                 # App bootstrap (Pinia, Router, i18n)
│   ├── App.vue                 # Root component (RouterView + auth init)
│   ├── styles/
│   │   └── main.css            # Tailwind layers + custom utilities
│   ├── locales/
│   │   ├── pt.json             # Portuguese translations (default)
│   │   └── en.json             # English translations
│   ├── router/
│   │   └── index.ts            # Route definitions + auth guard
│   ├── stores/
│   │   ├── auth.ts             # User state + login/logout/register
│   │   ├── chargers.ts         # Charger list + fetch action
│   │   └── sessions.ts         # Session list + fetch action
│   ├── services/
│   │   └── api.ts              # Axios instance + mock service layer
│   ├── layouts/
│   │   ├── PublicLayout.vue    # Navbar + RouterView + Footer
│   │   └── DashboardLayout.vue # Sidebar + TopBar + RouterView
│   ├── components/
│   │   ├── ui/
│   │   │   ├── Button.vue      # Multi-variant button (primary, secondary, ghost, danger)
│   │   │   ├── Card.vue        # Generic card wrapper
│   │   │   ├── Badge.vue       # Status badge with dot indicator
│   │   │   └── StatCard.vue    # KPI card with icon, trend, subtitle
│   │   ├── landing/
│   │   │   ├── Navbar.vue      # Transparent-on-scroll responsive navbar
│   │   │   └── Footer.vue      # Multi-column dark footer
│   │   └── dashboard/
│   │       ├── Sidebar.vue     # Collapsible sidebar navigation
│   │       └── TopBar.vue      # Top bar with notifications and language switcher
│   └── pages/
│       ├── landing/
│       │   ├── LandingPage.vue            # Composes all landing sections
│       │   └── sections/
│       │       ├── HeroSection.vue        # Full-screen dark hero with mockup
│       │       ├── FeaturesSection.vue    # 6-feature grid
│       │       ├── HowItWorksSection.vue  # 4-step numbered flow
│       │       ├── BenefitsSection.vue    # 4-benefit stat cards
│       │       └── CtaSection.vue         # Email capture with gradient background
│       ├── auth/
│       │   ├── LoginPage.vue              # Email + password login form
│       │   └── RegisterPage.vue           # Full registration form
│       └── dashboard/
│           ├── OverviewPage.vue           # KPI stats + recent activity
│           ├── ChargersPage.vue           # Filterable charger table
│           ├── SessionsPage.vue           # Filterable session table
│           └── SettingsPage.vue           # Tabbed settings (profile, org, notif, security)
```

---

## Landing Page

The landing page (`/`) is composed of five independent section components, mounted by `LandingPage.vue`:

### 1. Hero Section
- Full-viewport dark (`slate-950`) background with animated radial glow effects
- Large gradient headline: "Carregamento Inteligente para **Hotelaria Moderna**"
- Two CTAs: primary "Começar gratuitamente" (→ `/register`) + ghost "Ver demonstração"
- Three stat counters: active hotels, connected chargers, uptime guarantee
- Dashboard preview mockup built with pure CSS/HTML (no external screenshots)

### 2. Features Section
- Six feature cards in a 3-column responsive grid
- Each card has a gradient icon background, hover lift animation, and colored accent bar
- Covers: OCPP Integration, Real-time Monitoring, Multi-site Management, Reports, Payments, Support

### 3. How It Works Section
- Four numbered steps on a light gray (`slate-50`) background
- Desktop: horizontal layout with a decorative connector line
- Mobile: single column
- Step icons use category-appropriate Heroicons with colored rings

### 4. Benefits Section
- Four benefit cards in a 2×2 grid
- Each card highlights a KPI stat ("+40%", "+R$12k", "A+", "98%") alongside descriptive text
- Target audience: hotel managers evaluating ROI

### 5. CTA Section
- Full-width blue gradient background
- Email capture input + submit button (redirects to `/register`)
- Social proof copy and "no credit card" trust note

### Navbar
- Fixed to the top with a transparent state when at top of page
- Transitions to white/blur with shadow on scroll
- Mobile hamburger with animated dropdown menu
- Language toggle (PT ↔ EN) stored in `localStorage`

---

## Authentication

Authentication pages (`/login`, `/register`) follow a minimalist centered card design.

- **LoginPage**: email + password with show/hide toggle, remember me checkbox, demo hint
- **RegisterPage**: name, email, organization, password + confirm fields
- Both connect to `useAuthStore` which calls `authService` (mock in dev, real API in prod)
- On success, JWT is stored in `localStorage` and user is redirected to the dashboard
- Router navigation guard in `router/index.ts` protects all `/dashboard/*` routes

---

## Dashboard

The protected admin dashboard uses `DashboardLayout.vue` which wraps a persistent sidebar and a scrollable content area.

### Overview Page (`/dashboard`)
- Four `StatCard` components: Total Chargers, Active Sessions, Energy Consumed, Monthly Revenue
- Recent activity feed pulled from the sessions store
- Charger status summary (available/charging/offline/error counts)
- Quick action links

### Chargers Page (`/dashboard/chargers`)
- Live search (name, location, ID) + status filter buttons
- Responsive table with status `Badge` components and per-row action menu
- Empty state with prompt to add first charger

### Sessions Page (`/dashboard/sessions`)
- Summary chips (active count, completed count, total energy)
- Live search + status filter
- Table with all session fields including duration, energy, cost
- Loading and empty states

### Settings Page (`/dashboard/settings`)
- Tab navigation: Profile, Organization, Notifications, Security
- Profile tab: name, email, phone with save confirmation
- Organization tab: org name, type (dropdown), address, website
- Notifications tab: toggle switches for each notification type
- Security tab: change password, enable 2FA, delete account

---

## Multi-language (i18n)

Vue I18n 9 is used in **composition mode** (`legacy: false`).

- Default locale: `pt` (Portuguese)
- Fallback locale: `en`
- Locale persisted to `localStorage` on toggle
- Translation files: `src/locales/pt.json` and `src/locales/en.json`
- All UI strings are accessed via `t('key.path')` — no hardcoded text in templates
- Language toggle is available in both the landing Navbar and the dashboard TopBar

### Translation key structure

```
nav.*          — Navigation items
hero.*         — Landing hero section
features.*     — Features section
howItWorks.*   — How it works section
benefits.*     — Benefits section
cta.*          — Call to action section
footer.*       — Footer links and copy
auth.login.*   — Login form
auth.register.*— Register form
dashboard.*    — All dashboard pages
common.*       — Shared UI strings (loading, error, etc.)
language.*     — Language names
```

---

## State Management (Pinia)

Three stores power the application:

| Store | Responsibility |
|---|---|
| `useAuthStore` | User session, token, login, register, logout |
| `useChargersStore` | Charger list, loading state, fetch action |
| `useSessionsStore` | Session list, loading state, fetch action |

### Auth store details
- Reads `auth_token` from `localStorage` on startup via `initialize()`
- Hydrates in-memory user on app mount (`App.vue`)
- Computed property `isAuthenticated` drives the router guard

---

## API Service Layer

`src/services/api.ts` exports:

- A configured **axios instance** with base URL from `VITE_API_URL` env var
- Request interceptor to inject the Bearer token
- Response interceptor to handle 401 (redirect to login)
- **Mock service functions** for local dev: `authService`, `chargersService`, `sessionsService`

All mock functions include a 700ms artificial delay to simulate network latency, making loading states visible during development.

---

## Design System

### Color palette

| Token | Value | Use |
|---|---|---|
| `primary-600` | `#2563EB` | Buttons, links, active nav |
| `primary-50` | `#EFF6FF` | Tinted backgrounds |
| `volt-500` | `#22C55E` | Positive badges, accents |
| `slate-900` | `#0F172A` | Primary text |
| `slate-500` | `#64748B` | Secondary text |
| `slate-100` | `#F1F5F9` | Card borders |

### Typography
- Font: **Inter** (Google Fonts, preconnect optimized)
- Headlines: `font-extrabold tracking-tight`
- Body: `text-sm text-slate-600`

### Component classes (defined in `main.css`)
- `.btn-primary` — Electric blue button
- `.btn-secondary` — White border button
- `.btn-ghost` — Transparent hover button
- `.input-field` — Consistent input with focus ring
- `.card` — White rounded card with soft shadow
- `.container-custom` — Max-width centered layout container
- `.section-padding` — Consistent vertical section padding
- `.gradient-text` — Blue-to-green gradient text clip

---

## Routing

| Route | Name | Protection | Component |
|---|---|---|---|
| `/` | `landing` | Public | `LandingPage` |
| `/login` | `login` | Public (redirect if auth) | `LoginPage` |
| `/register` | `register` | Public (redirect if auth) | `RegisterPage` |
| `/dashboard` | `overview` | Auth required | `OverviewPage` |
| `/dashboard/chargers` | `chargers` | Auth required | `ChargersPage` |
| `/dashboard/sessions` | `sessions` | Auth required | `SessionsPage` |
| `/dashboard/settings` | `settings` | Auth required | `SettingsPage` |

Navigation guard logic:
- Unauthenticated access to `/dashboard/*` → redirect to `/login`
- Authenticated access to `/login` or `/register` → redirect to `/dashboard`

---

## Environment Configuration

```bash
# .env.example
VITE_API_URL=http://localhost:8000   # VoltWise Cloud API base URL
```

In production, set `VITE_API_URL` to the deployed API domain. The Vite dev server proxies `/api` to `localhost:8000` for local development without CORS issues.

---

## Future Improvements

### Real API Integration
- Replace mock service functions with real HTTP calls to `voltwise-cloud`
- Add refresh token rotation and automatic re-authentication
- Handle network errors with user-friendly retry UI

### Charts & Analytics
- Add a chart library (Recharts-Vue or Chart.js) for energy consumption trends
- Revenue breakdown chart on the Overview page
- Per-charger utilization heatmap

### Billing UI
- Subscription plan display and management
- Invoice history table
- Payment method management (Stripe integration)

### Enhanced Features
- Real-time updates via WebSocket for charger status and active sessions
- Push notifications (Web Push API) for critical alerts
- Dark mode toggle (TailwindCSS `dark:` classes are already configured)
- CSV / Excel export for sessions and energy reports
- Multi-organization support (hotel chains)
- Charger onboarding wizard (step-by-step OCPP configuration)

### Testing
- Unit tests for stores and service functions (Vitest)
- Component tests (Vue Test Utils)
- End-to-end tests (Playwright)

### Performance
- Route-level code splitting (already configured via dynamic imports)
- Image optimization and lazy loading
- PWA manifest and service workers for offline resilience
