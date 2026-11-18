# Technical Architecture & Tech Stack Specification: **Cairn**
**Domain:** Direct-to-Consumer Backcountry & Trail Gear E-Commerce Platform  
**Architecture Style:** Monolithic Full-Stack Web Application (Containerized)  

---

## 1. Executive Summary
**Cairn** is a self-hosted, database-driven e-commerce storefront designed for ultralight backcountry and trail equipment. Rather than relying on third-party headless commerce SaaS platforms, the application implements core commerce primitives—multi-variant SKU management, faceted technical spec filtering, ACID-compliant inventory reservations, and idempotent payment fulfillment—directly within a unified Next.js and PostgreSQL stack.

---

## 2. Core Technology Stack

### **Application & Frontend Layer**
* **Framework:** **Next.js 16 (App Router)**
  * **Server Components (RSC):** Used for catalog browsing (`/gear`, `/packs`, `/shelters`) and product detail pages to maximize initial load speed and SEO without shipping unnecessary client-side JavaScript.
  * **Server Actions:** Handles cart mutations (`addToCart`, `updateQuantity`), inventory reservation locks, and checkout session initialization.
  * **Route Handlers (`/api/*`):** Dedicated HTTP endpoints for asynchronous third-party callbacks (specifically Stripe payment webhooks).
* **Language:** **TypeScript (Strict Mode)**
  * Provides end-to-end type safety from PostgreSQL database schemas through Server Actions to React UI components.
* **Styling:** **Tailwind CSS v4**
  * Configured with CSS-first `@theme` design tokens tailored to an outdoor aesthetic (slate, moss, charcoal, and warm sandstone palettes).

### **Data & Persistence Layer**
* **Primary Database:** **PostgreSQL 18**
  * Serves as the single source of truth for products, variants, carts, inventory holds, and immutable order records.
  * **Key Database Features Used:**
    * Row-level locking (`SELECT ... FOR UPDATE`) to prevent race conditions and overselling during concurrent checkouts.
    * Composite B-Tree indexes on technical specifications (`weight_grams`, `capacity_liters`, `price_cents`) for sub-millisecond faceted filtering.
    * Foreign key constraints and `CHECK` constraints (`stock_quantity >= 0`) to enforce data integrity at the storage level.
* **ORM & Migrations:** **Drizzle ORM & Drizzle Kit**
  * Chosen for its SQL-first query builder, zero runtime overhead, explicit transaction control (`db.transaction()`), and clean TypeScript schema inference.
* **Schema & Input Validation:** **Zod**
  * Validates URL search parameters for catalog filtering (`?category=packs&maxWeight=900&inStock=true`) and sanitizes all Server Action form payloads.

### **Commerce, Auth & Media Infrastructure**
* **Payment Processing:** **Stripe (Checkout Sessions + Webhooks)**
  * Offloads PCI-DSS compliance and card processing to Stripe's hosted checkout while retaining full state-machine control over order fulfillment inside PostgreSQL.
* **Session & Authentication:** **Auth.js (or Custom HTTP-Only Encrypted Cookies)**
  * Supports frictionless guest shopping via an anonymous `cairn_cart_id` cookie, merging guest cart items into persistent user accounts upon sign-in.
* **Asset Storage:** **Local Volume / MinIO (S3-Compatible Object Storage)**
  * Stores high-resolution product photography locally during development via Docker volumes, with an S3-compatible interface ready for production cloud storage.

### **Infrastructure & DevOps**
* **Package Manager:** **pnpm**
* **Containerization:** **Docker & Docker Compose**
  * Orchestrates the entire local and production runtime environment consistently across machines.
* **Testing & Quality Gates:**
  * **Vitest:** Unit and integration testing for cart math, inventory locking transactions, and webhook idempotency logic.
  * **ESLint & TypeScript Compiler (`tsc --noEmit`):** Enforced static code quality checks.

---

## 3. Container Architecture (`docker-compose.yml`)

| Service Name | Image / Build | Port(s) | Responsibility |
| :--- | :--- | :--- | :--- |
| **`web`** | Local `Dockerfile` (Node 22 Alpine) | `3000:3000` | Runs the Next.js full-stack server (SSR, Server Actions, Webhook API). |
| **`db`** | `postgres:18-alpine` | `5432:5432` | Persistent relational database backed by the `cairn_pgdata` named volume. |
| **`stripe-cli`** | `stripe/stripe-cli:latest` | *Internal* | Listens to Stripe test events and forwards webhooks to `http://web:3000/api/webhooks/stripe`. |

---

## 4. Core Architectural Workflows

### **A. Catalog & Faceted Filtering Flow**
1. User selects technical filters in the UI (e.g., *Max Weight: 800g*, *In Stock Only*).
2. Client updates URL query parameters via Next.js router navigation.
3. Next.js Server Component parses and validates `searchParams` using **Zod**.
4. **Drizzle ORM** executes a parameterized `SELECT` query joining `products` and `product_variants`, returning filtered gear directly in the server-rendered HTML payload.

### **B. Checkout & Inventory Reservation Flow (Concurrency Safe)**
1. User clicks **Proceed to Checkout**, triggering a Next.js Server Action.
2. Inside a PostgreSQL transaction (`BEGIN`), the server locks the requested `product_variants` rows using `FOR UPDATE`.
3. System verifies `available_stock >= requested_quantity`. If insufficient, the transaction rolls back and returns an out-of-stock error.
4. System decrements `available_stock`, inserts a `PENDING` order with snapshotted item prices, and creates a `15-minute` reservation expiration timestamp.
5. Server generates a Stripe Checkout Session containing the internal `order_id` in metadata and redirects the user.

### **C. Idempotent Webhook Fulfillment Flow**
1. Upon successful payment, Stripe sends a `checkout.session.completed` POST request to `/api/webhooks/stripe`.
2. Route Handler verifies the cryptographic `Stripe-Signature` header.
3. Handler checks a `processed_webhooks` table for the unique `event.id`. If already present, it immediately returns `200 OK` (preventing duplicate fulfillment).
4. Inside a database transaction, the handler records `event.id`, transitions the order status from `PENDING` to `PAID`, and clears the temporary reservation lock.

---

## 5. Recommended Directory Structure

```text
cairn/
├── docker-compose.yml          # Multi-container orchestration (web, db, stripe-cli)
├── Dockerfile                  # Next.js container build instructions
├── drizzle.config.ts           # Drizzle Kit migration configuration
├── package.json
└── src/
    ├── app/                    # Next.js App Router
    │   ├── (shop)/             # Public storefront routes (catalog, product detail, cart)
    │   ├── (admin)/            # Protected inventory & order management dashboard
    │   ├── api/
    │   │   └── webhooks/
    │   │       └── stripe/     # Stripe webhook route handler
    │   ├── globals.css         # Tailwind v4 imports & Cairn @theme design tokens
    │   └── layout.tsx          # Root layout & navigation shell
    ├── components/             # Reusable UI & domain components (GearCard, FilterSidebar, CartDrawer)
    ├── db/
    │   ├── index.ts            # PostgreSQL connection pool & Drizzle client
    │   ├── schema.ts           # Relational tables, enums, indexes, and relations
    │   └── seed.ts             # Backcountry gear catalog seed script
    ├── actions/                # Next.js Server Actions (cart, checkout, admin mutations)
    └── lib/                    # Shared utilities (Stripe client, Zod validators, currency formatters)
```
