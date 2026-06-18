# Master-Trip: Business Architecture & Aggregation Strategy

This document outlines the commercial architecture of Master-Trip. It details where the global travel inventory is sourced, how money flows through the platform, and the UX design for enterprise group bookings.

---

## 1. Global Inventory Aggregation Layer

To provide global inventory without negotiating individual contracts with 10,000 different hotels or airlines, Master-Trip acts as a **Meta-Aggregator**. We connect to industry-standard B2B APIs that supply the inventory to our oRPC search layer.

### The Providers (Where the data comes from)

| Vertical | Primary B2B Aggregator(s) | Why We Use Them |
| :--- | :--- | :--- |
| **Flights** | **[Duffel](https://docs.duffel.com/) + [Amadeus](https://developers.amadeus.com/) + [Travelfusion](https://www.travelfusion.com/)** | A multi-provider strategy is required because no single GDS has 100% global coverage. Duffel handles modern NDC carriers, Amadeus covers legacy global routes, and Travelfusion captures Low-Cost Carriers (LCCs). |
| **Hotels** | **[Expedia Partner Solutions (EPS)](https://developer.expediagroup.com/)** or **[Hotelbeds](https://developer.hotelbeds.com/)** | EPS provides immediate access to millions of properties globally with deep wholesale discounts for B2B partners. |
| **Car Rentals** | **[Rentalcars.com Connect](https://www.rentalcars.com/en/affiliate/)** | The largest global aggregator for Hertz, Avis, Enterprise, and local providers. |
| **Tours & Guides** | **[Viator API](https://docs.viator.com/)** or **[GetYourGuide](https://partner.getyourguide.com/en-us/api-documentation)** | Instantly provides millions of bookable excursions, museum tickets, and local private guides. |

**How the System "Sees" Them:**
When a user searches for a trip to "Paris", our Next.js frontend calls our oRPC backend. The backend executes **parallel async requests** to Duffel (Flights), EPS (Hotels), and Viator (Tours). It normalizes the JSON responses and caches the unified result in Upstash Redis, presenting a seamless dashboard to the user in under 500ms.

---

## 2. The "Merchant of Record" & Payment Flow

To achieve "Dynamic Bundling" (hiding our profit margins from the user) and ensure the checkout is seamless, Master-Trip utilizes a **Virtual Credit Card (VCC)** fulfillment model.

### The Money Flow (Step-by-Step)
1. **The Cart:** The user selects a Flight ($500 cost) and a Hotel ($400 cost). Master-Trip adds a $100 markup. The total cart is $1,000.
2. **User Checkout:** The user pays $1,000 via **Stripe**. The user's bank statement reads `MASTER-TRIP TRAVEL`. The funds enter the Master-Trip bank account.
3. **VCC Generation:** Upon payment success, our background worker calls a VCC provider (like **Stripe Issuing** or **ConnexPay**) to instantly generate two single-use virtual credit cards.
   * VCC #1 is loaded with exactly $500.
   * VCC #2 is loaded with exactly $400.
4. **Fulfillment:** 
   * The worker sends VCC #1 to the Flight API (Duffel) to issue the ticket.
   * The worker sends VCC #2 to the Hotel API (EPS) to reserve the room.
5. **Profit Capture:** The remaining $100 sits safely in the Master-Trip Stripe account as pure profit.

### 2.1 B2B Operational Mechanics (Wallets & Environments)
The platform is designed to handle the complex operations required by enterprise API partners like Duffel:
*   **B2B Wallet Balances:** To avoid VCC interchange fees on massive flight bookings, we also utilize native "B2B Wallet Balance" features. Master-Trip maintains a prepaid balance in the Duffel dashboard, and the API deducts funds directly upon order creation.
*   **Strict Environment Parity:** The architecture enforces strict separation of API keys (e.g., `duffel_test_` vs live tokens) injected via Vercel environment variables. This enables our staging environment and QStash workers to simulate end-to-end ticketing, refunds, and KYC verification flows without touching real money.
*   **Automated Post-Booking Management:** If an airline alters a flight time, the GDS webhook pushes the change to our oRPC backend. Our Mastra AI Agent intercepts the event, reads the `Order` history, and automatically emails the user their updated itinerary, eliminating manual dashboard management.

### 2.2 Frictionless B2C Checkout & "Shadow Accounts"
To maximize conversion rates, the platform does not force users to create a password before purchasing. Instead, it utilizes a frictionless background flow:
1. **Guest Checkout:** The user enters their email and credit card to buy a flight.
2. **Shadow Account Creation:** Upon payment, the backend checks Supabase. If the email is new, it silently provisions a `User` record in the background and attaches the new `Trip` to it, ensuring database relational integrity is preserved.
3. **Magic Link Auth:** When the user receives their itinerary email via Resend, they are provided a WorkOS Magic Link. Clicking the link authenticates them instantly into the Next.js dashboard as a registered user, completely bypassing the need for passwords.

---

## 3. Enterprise Group Bookings (The "Invite Link" Design)

To support Schools, Corporations, and massive Excursions without causing administrative nightmares, we use a decentralized roster system. 

### The Enterprise UX Flow

**Phase 1: The Organizer (Teacher/Manager)**
1. The Organizer logs into the Master-Trip Dashboard under their `Organization` profile.
2. They construct a Draft Trip (e.g., *London Study Abroad 2026*).
3. They add inventory to the cart: 50 Flight Seats, 25 Hotel Rooms, 1 Private Bus Tour.
4. They pay a 10% holding deposit via corporate card to lock the rates.

**Phase 2: Roster Delegation (The "Invite Link")**
5. Master-Trip generates a secure link: `master-trip.com/join/london-2026-xyz`.
6. The Organizer emails this link to the 50 students/parents.
7. Each parent clicks the link and is prompted to enter their child's details: Legal Name, Passport ID, Dietary Restrictions (Vegan/Allergies), and Emergency Contacts.

**Phase 3: Automated Assembly & Fulfillment**
8. As parents fill out the forms, the Master-Trip database automatically populates the `Traveler` table linked to the Organizer's `Trip`.
9. The Organizer sees a live dashboard: *"45/50 Travelers Registered"*.
10. Once all 50 are registered, the Organizer clicks **"Finalize Trip"** and pays the remaining invoice.
11. The QStash async workers automatically map the 50 passports to the 50 flight seats and issue all tickets simultaneously via the APIs.

### Why This Design Wins
* **Zero Liability:** Master-Trip and the Organizer never touch or email unencrypted passport PDFs. The users enter PII directly into our secure, encrypted database.
* **Massive Time Savings:** Turns a 3-week manual spreadsheet process into a 2-day automated digital flow.

### 3.1 B2B Scalability: The Adapter Advantage
Because our backend uses the **Adapter Pattern** (an array of multiple API providers searching simultaneously), we automatically solve the two biggest B2B challenges:
1. **Unlocking Corporate Discounts:** Instead of relying on a single API, our search engine pings multiple providers. If Amadeus sees a 50-passenger order, it returns a massive "Group Rate Discount" that other B2C providers might miss.
2. **Splitting Massive Inventory:** If a corporate retreat needs 50 hotel rooms, but Expedia only has 25 left, our system doesn't fail. The aggregator automatically buys 25 rooms from Expedia and 25 from Hotelbeds, fulfilling the massive order seamlessly.

---

## 4. Internal Operations & The Markup Engine

To manage this complex platform securely without bloating the public website, Master-Trip utilizes an internal **Role-Based Admin Dashboard** for staff operations.

### 4.1 The Dynamic Markup Engine (Pricing)
The public Next.js frontend never determines pricing, as this exposes the platform to client-side tampering. Instead, the oRPC backend applies dynamic markups (e.g., adding 10% to the Duffel wholesale cost) *before* the user sees the final cart price. Operations Managers use the Admin Dashboard to dynamically adjust these margins based on seasonal demand without redeploying any code.

### 4.2 Role-Based Access Control (RBAC)
To protect customer data and financial integrity, internal dashboard access is strictly governed by Supabase RLS and WorkOS roles:
*   **Support Agents:** Can monitor live AI chats, trigger manual chat overrides, and issue partial refunds. They cannot see raw Virtual Credit Card numbers (only the last 4 digits).
*   **Operations / Finance:** Can update profit margin rules and view raw Stripe payout reconciliations against VCC spend.
*   **Engineers:** Have exclusive access to monitor the Dead Letter Queue (DLQ) and manually trigger retries for failed API fulfillments.

### 4.3 Automated Marketing & Price Alerts
Because the platform automatically captures emails during the frictionless Guest Checkout flow, the user database naturally scales into a powerful marketing engine. 
*   **Behavioral Targeting:** Using **PostHog**, the system tracks users who searched for specific flights (e.g., "Paris") but abandoned their cart.
*   **Automated Deal Blasts:** A weekly **QStash Cron Job** fetches the latest discounted inventory from the Duffel API. If flights to Paris drop by 30%, the backend triggers a highly targeted marketing email via **Resend**, pushing the discounted flight exclusively to the users who abandoned those carts, drastically increasing long-term conversion rates.
