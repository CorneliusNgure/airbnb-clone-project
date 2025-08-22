# 📘 Airbnb Clone – Guide  Notes


## 🗂️ ER Diagram Deep Dive

The Entity–Relationship (ER) diagram captures the **data model**.  
It defines the core tables (entities), their attributes, and the relationships between them.

---

### 1) Entities (tables) & key fields

#### USER
- **PK:** `id (UUID)`
- **Identity & auth:** `email (unique)`, `password_hash`
- **Profile:** `first_name`, `last_name`, `phone`
- **Role:** `role ∈ {guest, host, admin}` → controls permissions and behavior
- **State & audit:** `is_verified`, `created_at`, `updated_at`

#### PROPERTY
- **PK:** `id (UUID)`
- **Owner:** `host_id (FK → USER.id)` → the host who owns this property
- **Content:** `title`, `description`, `type ∈ {entire, private_room, shared_room}`
- **Location:** `address`, `city`, `country`, `location (PostGIS point)` for geo queries
- **Rich attributes:** `amenities (JSONB)`, `photos (JSONB array)`
- **Commercials:** `price_per_night`, `currency`, `max_guests`
- **Lifecycle:** `status ∈ {active, paused, drafted}`

#### BOOKING
- **PK:** `id (UUID)`
- **What & who:** `property_id (FK → PROPERTY.id)`, `guest_id (FK → USER.id)`
- **Dates & party size:** `start_date`, `end_date`, `guests_count`
- **Money & state:** `total_amount`, `currency`, `status ∈ {pending, confirmed, cancelled}`
- **Audit:** `created_at`

#### PAYMENT
- **PK:** `id (UUID)`
- **Link:** `booking_id (FK → BOOKING.id)`
- **Provider & money:** `provider`, `amount`, `currency`
- **State:** `status ∈ {succeeded, failed, refunded}`
- **Tracking:** `transaction_ref`, `paid_at`

#### REVIEW
- **PK:** `id (UUID)`
- **Links:** `property_id (FK → PROPERTY.id)`, `author_id (FK → USER.id)`, `booking_id (FK → BOOKING.id)`
- **Content:** `rating (1–5)`, `comment`, `is_public`
- **Audit:** `created_at`

---

### 2) Relationships (cardinalities)

- `USER ||--o{ PROPERTY : hosts`  
  One verified host can have **0..N properties**. Each property belongs to exactly one host.

- `USER ||--o{ BOOKING : makes`  
  A guest can make **0..N bookings**. Each booking belongs to exactly one guest.

- `PROPERTY ||--o{ BOOKING : has`  
  A property can have **0..N bookings**. Each booking is for exactly one property.

- `BOOKING ||--|| PAYMENT : includes`  
  Each booking has exactly one payment, and each payment maps to exactly one booking.

- `PROPERTY ||--o{ REVIEW : receives`  
  A property can receive **0..N reviews**. Each review is for exactly one property.

- `USER ||--o{ REVIEW : writes`  
  A user can write **0..N reviews**. Each review has exactly one author.

- `BOOKING ||--o{ REVIEW : relates_to`  
  A booking can be related to **0..N reviews** (commonly zero or one). Each review ties back to exactly one booking.

💡 **Reading tip**:  
- `||` = “exactly one”  
- `o{` = “zero or many”  
So `A ||--o{ B` means: “One A relates to zero or many B”.

---

### 3) Why these links exist (domain story)

- Hosts list properties → `USER (host)` → `PROPERTY`
- Guests book properties → `USER (guest)` + `PROPERTY` → `BOOKING`
- Bookings get paid → `BOOKING` → `PAYMENT`
- After stay completion → `USER (guest)` writes `REVIEW` linked to `BOOKING` + `PROPERTY`

---

### 4) Data types & modeling choices

- **UUID PKs**: stable across services, scalable for distributed systems.
- **PostGIS point on PROPERTY.location**: enables radius / nearest searches.
- **JSONB (amenities, photos)**: flexible attributes and arrays without needing extra tables.
- **Enums-as-strings** (`role`, `status`, `type`): expressive; can enforce at DB or app layer.

---

### 5) Constraints & performance

- **Uniqueness**: `users.email` must be unique.
- **Composite index**: `(property_id, start_date, end_date)` → faster availability queries.
- **GIST index**: on `properties.location` → efficient geospatial search.
- **Exclusion constraint**: prevents overlapping bookings for the same property and date range.

---

### 6) Lifecycle at a glance

- Host creates **PROPERTY** (status: `drafted → active`)
- Guest creates **BOOKING** (status: `pending`)
- Payment captured → **PAYMENT** created, **BOOKING** confirmed
- Stay completes → guest leaves **REVIEW**

---

### 7) Subtleties to consider

- Current diagram shows: `BOOKING ||--|| PAYMENT`.  
- But your sequence diagram shows booking first, then payment.  
- If you allow unpaid bookings, it could be modeled as:  
  `BOOKING ||--o| PAYMENT` → one booking may have zero or one payment.  
- Keep strict 1–1 only if you always create a payment record (even for failed/refunded).

---

## 🔄 Sequence Diagram Notes – Booking & Payment Flow

The sequence diagram shows **process interactions** between actors and systems.  

**Step breakdown:**
1. Guest searches property availability → API queries DB → results returned.
2. Guest creates booking request → booking stored in DB with `pending` status.
3. Guest proceeds to payment → API contacts Payment Gateway.
4. Payment success → API updates booking (`confirmed`), records payment.
5. Guest receives booking confirmation + receipt.
6. Host receives notification of new confirmed booking.

**Key insight**:  
- Bookings start as `pending` → only confirmed after successful payment.  
- Ties back to ER subtlety (step 7 above).  

---

## 🔐 Flowchart Notes – User Authentication Flow

The flowchart visualizes **sign-up, verification, login, and token refresh**.

**Step breakdown:**
1. User signs up → stored with `is_verified = false`.
2. Verification email sent → user clicks verification link.
3. System updates `is_verified = true`.
4. User logs in → system verifies credentials.
   - Valid → issues JWT (access + refresh tokens).
   - Invalid → reject login.
5. Access token grants API access → refresh token used when access token expires.

**Key insight**:  
- JWT strategy ensures **stateless authentication**.  
- Email verification prevents fake/unverified users from booking/hosting.  

---

## 🛠️ Development Execution Flow (Step by Step)

This is the **high-level plan** for executing the project.

1. **Project Setup**
   - Initialize repository.
   - Setup backend framework (e.g., Express + TypeScript, Django, etc.).
   - Configure environment variables & secrets.

2. **Database Layer**
   - Implement schema from ER diagram.
   - Apply indexes & constraints for performance & safety.
   - Test migrations.

3. **Auth & User Service**
   - Implement signup, email verification, login, JWT issuance, token refresh.
   - Ensure role-based access (guest, host, admin).

4. **Property Service**
   - CRUD for properties.
   - Store amenities & photos (JSONB, file storage/S3).
   - Implement geospatial queries.

5. **Booking Service**
   - Booking creation flow (pending).
   - Availability check (no overlapping bookings).
   - Link to property & guest.

6. **Payment Service**
   - Integrate with gateway (Stripe, PayPal, etc.).
   - Update booking → confirmed on success.
   - Record payment.

7. **Review Service**
   - Enforce: only users with completed booking can review.
   - Store rating, comment, visibility.

8. **Notifications**
   - Notify host of confirmed bookings.
   - Email or push notifications.

9. **Testing**
   - Unit tests per service.
   - Integration tests for booking/payment flow.
   - Authentication & authorization tests.

10. **Deployment**
    - CI/CD pipeline.
    - Deploy to staging → production.
    - Monitor logs, metrics.

---
