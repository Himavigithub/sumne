# StockShift — Stock Transfer Management

A web application for managing stock transfers between warehouses: maintain warehouses and their
stock levels, raise transfer requests, move each request through its lifecycle, and have warehouse
inventory updated automatically on completion.

## Features

| # | Feature | Where |
|---|---------|-------|
| 1 | Create warehouses & maintain stock levels | `/warehouses` |
| 2 | Create stock transfer request | `/transfers/new` |
| 3 | Transfer status management | `/transfers/:id` |
| 4 | Update warehouse stock on completion | enforced in the database on status change |
| 5 | Transfer list / history with filters & search | `/` |

### Transfer lifecycle

```text
draft ──▶ approved ──▶ in_transit ──▶ completed
  │           │             │
  └───────────┴─────────────┴──▶ cancelled
```

- Transitions are validated **in the database** (a `BEFORE UPDATE` trigger), so invalid jumps such as
  `draft → completed` are rejected regardless of client.
- Stock is moved **only** when a transfer reaches `completed`: the quantity is deducted from the
  source warehouse and added to the destination in the same transaction.
- Completion is blocked with a clear error if the source warehouse does not hold enough stock
  (`Insufficient stock at source warehouse (available: X, required: Y)`).
- Non-draft transfers cannot have their quantity, product or route edited.

## Tech stack

- **Frontend:** React 19, TanStack Start (SSR) + TanStack Router, TanStack Query, Tailwind CSS v4
- **Backend:** Lovable Cloud (managed Postgres, Data API, row-level security)
- **Validation:** Zod on the client, constraints + triggers in Postgres
- **Language:** TypeScript (strict)

## Data model

| Table | Purpose |
|-------|---------|
| `warehouses` | `code` (unique), `name`, `location` |
| `products` | `sku` (unique), `name`, `unit` |
| `stock_levels` | quantity per `(warehouse_id, product_id)`, non-negative, unique pair |
| `stock_transfers` | `reference` (auto `TR-1001…`), source/destination warehouse, product, quantity, `status`, notes, timestamps |

Database-side rules: quantity > 0, source ≠ destination, stock ≥ 0, status-transition trigger,
stock-application trigger, `updated_at` triggers.

## Setup

```bash
# 1. Install dependencies
npm install          # or: bun install

# 2. Environment variables (.env at project root)
VITE_SUPABASE_URL=<backend url>
VITE_SUPABASE_PUBLISHABLE_KEY=<publishable key>
SUPABASE_URL=<backend url>
SUPABASE_PUBLISHABLE_KEY=<publishable key>

# 3. Run the dev server
npm run dev          # http://localhost:8080

# 4. Production build
npm run build
```

On Lovable the `.env` values are provisioned automatically and the database schema and seed data are
already applied. For a standalone clone, run the SQL in `supabase/migrations/` against a Postgres /
Supabase project.

## Sample test flow

1. **Open `/`** — the dashboard shows warehouse count, total units on hand, open and completed
   transfers, plus the full transfer history. Seed data ships with 4 warehouses, 6 products and
   4 example transfers.
2. **Create a warehouse** — go to *Warehouses & stock*, enter code `WH-CENTRAL`, name
   `Central Distribution Center`, location `Madrid, ES`, then **Create warehouse**. It appears in the
   location list immediately.
3. **Set stock** — select the new warehouse, set `SKU-1001` to `100` and press **Save**.
4. **Create a transfer** — click **New transfer**, choose source `WH-CENTRAL`, destination
   `WH-NORTH`, product `SKU-1001`, quantity `30`, add a note, submit. Note that the form shows the
   live available quantity and warns when the request exceeds it.
5. **Move it through the lifecycle** — on the transfer detail page press **Mark as Approved**, then
   **Mark as In transit**, then **Mark as Completed**.
6. **Verify stock** — return to *Warehouses & stock*: `WH-CENTRAL` now holds `70` and `WH-NORTH`
   increased by `30`.
7. **Negative test** — create a transfer for a quantity larger than the source stock and try to
   complete it. The database rejects it and the UI shows the insufficient-stock error; the transfer
   stays in `in_transit`.
8. **Cancellation** — cancel any non-completed transfer; no stock is moved and the status becomes
   terminal.

## Notes on access control

The app is deployed as an open internal tool: no login is required, and row-level security policies
allow read/write for the app's public API role. Adding authentication would mean enabling email +
Google sign-in and narrowing the policies to authenticated users (and, for approvals, a
role-checking function backed by a separate `user_roles` table).
