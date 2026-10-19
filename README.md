# TRIVO — System Architecture & Tech Specs

A quick, no-nonsense technical breakdown of how TRIVO is built, how the services connect, and the tools used across the stack.

---

## 1. What is TRIVO?

TRIVO is a mobile app (iOS & Android) that connects people through sports and activities:
* **Social Matching:** Swipe on people who want to do the same sport or activity.
* **Club Activities & Events:** Discover, book, and buy tickets for local club events.
* **Club Management:** Tools and analytics for sports clubs to host events and sell passes.

---

## 2. Architecture at a Glance

```
 [Flutter App (iOS & Android)]
      │
      ├── Cache-First (CachedNetworkImage + BlurHash)
      ├── RevenueCat SDK (Subscriptions & Likes)
      └── Stripe SDK (Event Tickets)
      │
      ▼
 [Nginx / Load Balancer (WIP)]
      │
      ▼
 [Python Backend API (Flask + Pydantic v2)]
      │
      ├── In-Memory Cache (TTLCache + Thread Locks)
      │
      ├──► Firebase Firestore (Users, auth, chat, activity feed)
      ├──► Supabase / PostgreSQL (Matching algorithm, swipes, transaction log)
      ├──► Google BigQuery (Club attendance metrics & analytics)
      ├──► RevenueCat API (Server-side receipt verification)
      └──► Stripe Connect (Ticket payments & automatic club payouts)
```

---

## 3. Tech Stack

| Layer | Tool | What it does |
|---|---|---|
| **Mobile App** | **Flutter / Dart** | Single codebase for iOS and Android. Clean layered architecture (UI, Providers, Services, Models). |
| **Mobile UI** | **Apple HIG / Liquid Glass** | iOS-style frosted glass (`BackdropFilter`), haptics, smooth animations, and top toast alerts. |
| **State** | **Provider** | Manages app state and dependency injection. |
| **Backend API** | **Python (Flask)** | Modular REST API split into blueprints (`users`, `matching`, `clubs`, `tickets`, `payments`). |
| **Validation** | **Pydantic v2** | Validates request/response data types and API contracts. |
| **Backend Cache** | **TTLCache (`cachetools`)** | In-memory cache with thread locks (`threading.RLock`) for user profiles, activities, and places. |
| **NoSQL DB** | **Firebase Firestore** | Phone authentication, user profiles, sports catalog, and real-time chat messages. |
| **Relational DB**| **Supabase (PostgreSQL)** | Matching engine, swipe history, daily swipe limits, and purchase transaction records. |
| **Data Warehouse**| **Google BigQuery** | Stores event attendance and engagement data to generate stats and reports for clubs. |
| **In-App Purchases**| **RevenueCat** | Handles Apple/Google subscriptions (TRIVO Premium) and consumable like packs. |
| **Event Payments**| **Stripe Connect** | Handles ticket sales for club events and sends money directly to club accounts. |
| **Places API** | **Google Places** | Search and geocoding for club locations and sports facilities. |
| **AI Assistant** | **Gemini (VS Code)** | AI agent connected to VS Code with live access to Flutter, Supabase, and Firebase code. |

---

## 4. How the App Works (Frontend)

* **Cache-First Loading:** The app loads cached data and images immediately with BlurHash placeholders, then refreshes silently in the background. No jumpy UI.
* **Gradual Like Reveal:** When unlocking a blurred like with a credit, the photo gradually de-blurs over 1.5 seconds (`sigma: 28 → 0`) with synchronized haptic clicks.
* **Match Screen:** Both profile cards slide in from opposite sides, tilt, and meet in the center. The match flame pulses with a continuous heartbeat animation and haptic feedback.
* **Smart Back Buttons:** Navigation checks `Navigator.canPop(context)`. If there is nowhere to go back to (e.g., first screen or tab root), the back button is completely hidden.

---

## 5. How the Backend Works

* **REST Endpoints:** Clean Flask blueprints for each domain (`/api/matching/swipe`, `/api/tickets`, etc.).
* **Safe In-Memory Caching:** 
  * User profiles: cached for 30 minutes.
  * Activity feeds: cached for 2 minutes.
  * Event details: cached for 10 minutes.
  * Google Places: cached for 24 hours.
  * Every cache access is locked with `threading.RLock()` to prevent crashes when multiple requests arrive at once.
* **Retry on DB drops:** Supabase queries automatically retry with exponential backoff if a socket hiccups (`Errno 35` or timeouts).

---

## 6. Databases: Why Three?

Instead of forcing one database to do everything, TRIVO splits data by job:

1. **Firebase Firestore (Real-Time & Documents):**
   * Stores user profiles, club details, and chat messages.
   * Native real-time listeners make chat instant without maintaining custom socket servers.
2. **Supabase / PostgreSQL (Relational & Matching):**
   * Relational data with foreign keys and unique constraints (`UNIQUE(user_from, user_to)`).
   * Fast queries for "who liked who" and checking mutual matches.
   * Row-Level Security (RLS) ensures users only see their own matches.
3. **Google BigQuery (Analytics for Clubs):**
   * High-volume event data (who attended what event, peak hours, club growth).
   * Used to generate business reports for sports clubs without slowing down the production databases.

---

## 7. Payments: RevenueCat vs. Stripe

* **RevenueCat (Mobile IAP):**
  * Used for digital goods: **TRIVO Premium** subscriptions and **Like Packs**.
  * The mobile app purchases through Apple/Google, then the backend validates the receipt server-to-server with RevenueCat.
  * The backend never trusts the client: credits are only added after RevenueCat confirms the transaction, with a unique transaction ID to prevent duplicate crediting.
* **Stripe Connect (Event Ticketing):**
  * Used for real-world tickets: sports tournaments, court bookings, club passes.
  * Uses Stripe Checkout.
  * Club organizers connect their bank account via Stripe Connect, so ticket sales automatically split (club gets their cut, TRIVO gets the platform fee).
  * Webhook confirms the payment, generates a ticket in the database, and creates a QR pass for entry.

---

## 8. AI-Assisted Development

The project was developed with a **Gemini AI agent integrated into VS Code**:
* The agent has live context of the Flutter app, Python backend, Supabase SQL schema, and Firebase rules.
* Used for pair-programming, refactoring code to Apple HIG standards, running `flutter analyze` to keep 0 errors, and generating safe SQL migrations.

---

## 9. Next Steps & Work In Progress (WIP)

1. **Load Balancer (Nginx / Cloud LB):** Put a load balancer in front of the backend to handle SSL termination, rate limiting, and balance requests across multiple backend instances.
2. **Redis Cache:** Move from Python in-memory `TTLCache` to a shared Redis cluster so multiple backend containers can share the same cache.
3. **Container Autoscaling (Docker + Cloud Run / K8s):** Package the backend into Docker containers with Gunicorn workers that scale up automatically during peak hours.
4. **WebSockets for Chat:** Move chat and match alerts to persistent WebSockets (via Supabase Realtime) for sub-50ms message delivery.
