# Master-Trip: Business Architecture & Strategy

This document outlines the commercial architecture of Master-Trip. It details our focused provider strategy, payment workflows, and our progressive approach to capturing the travel market.

---

## 1. Focused Provider Strategy (The "Local to Global" Approach)

Based on 20 years of industry experience, Master-Trip rejects the complex "Meta-Aggregator" model (pinging 10+ APIs for the cheapest fare) for our MVP. Using too many aggregators leads to customer service nightmares regarding cancellations, refunds, and ticket changes. 

Instead, our strategy focuses on deep partnerships with tested, trusted, IATA-certified consolidators where we have direct operational relationships. We are building a progressive website—starting with core ticketing—while conquering the Nigerian market first before aggressively expanding globally.

### Core Service Providers

| Vertical | Selected Provider | Rationale |
| :--- | :--- | :--- |
| **Flights** | **247 Travels API** | 247 Travels is an IATA-certified consolidator. They hold direct airline inventory, and we have a direct relationship with their leadership and customer service teams. This guarantees we can support our passengers during cancellations, reissues, or emergencies. |
| **Hotels** | **Booking.com API** | Highly reliable compared to other consolidators. Guarantees that when a passenger arrives at the hotel, the reservation and payment are actually in the hotel's system. |
| **Admissions Abroad** | **Internal Processing** | A core high-margin service we offer alongside flight booking. |
| **Tour Packages** | **Local Nigerian Tours** | Prioritizing local Nigerian tours first to solidify our regional footprint. |

*(Note: Corporate Travels, Travel Insurance, and Car Rentals are deprioritized for the MVP to maintain a lean, focused launch).*

---

## 2. Payment Flow & Transaction Mechanics

Master-Trip operates with lean resources, meaning we **do not** use advance-deposit B2B wallets. We utilize a precise, transaction-by-transaction payment split to ensure zero financial liability on held inventory.

### The Money Flow (Step-by-Step)
1. **The Cart:** The user selects a Flight on Qatar Airways (Base cost: ₦800,000). Master-Trip's dynamic markup adds ₦50,000. Total cart is ₦850,000.
2. **Reservation & Time Limit:** A unique reference ID is generated via the 247 Travels API. The airline provides a strict time limit (e.g., 2 hours). **Prices are never locked without payment.**
3. **User Checkout:** The user pays the full ₦850,000 before the time limit expires. 
4. **Payment Splitting:** 
   * The ₦800,000 base cost is routed directly to the consolidator (247 Travels) to instantly issue the ticket.
   * The ₦50,000 markup is routed to the Master-Trip wallet as pure profit.
5. **Fulfillment:** If the user fails to pay within the airline's time limit, the reservation automatically cancels out, protecting Master-Trip from any airline penalties.

### 2.1 Frictionless B2C Checkout & "Shadow Accounts"
To maximize conversion rates, the platform does not force users to create a password before purchasing.
1. **Guest Checkout:** The user enters their email and details to buy a flight.
2. **Shadow Account Creation:** Upon payment, the backend checks the database. If the email is new, it silently provisions a `User` record to maintain relationship integrity.
3. **Magic Link Auth:** When the user receives their itinerary email, they are provided a Magic Link to access their Next.js dashboard as a registered user, bypassing password creation.

---

## 3. Real-Time Inventory vs. Enterprise Expectations

Unlike static ecommerce, the travel industry relies on Global Distribution Systems (GDS) where inventory is highly volatile. 

### Why We Reject "Holding Deposits" (For Both Individuals and Groups)
Master-Trip does not offer "10% holding deposits" or allow users to lock in prices without full payment, whether they are booking a single ticket or 50 seats for a group. Airline inventory is highly volatile, and reserving seats without immediate full payment exposes Master-Trip to ADMs (Agency Debit Memos) and massive financial penalties.

The flow for all bookings (individuals and enterprise clients) is strictly governed by the airline's real-time pricing:
1. All searches and inquiries are treated as real-time quotes.
2. Users must pay the full amount at the current GDS rate to secure the booking before the airline's strict time limit expires.
3. We absolutely do not manually float, guarantee prices, or hold seats at our own financial risk.

---

## 4. Internal Operations & Data Utilization

To manage the platform securely, Master-Trip utilizes an internal **Role-Based Admin Dashboard**.

### 4.1 Role-Based Access Control (RBAC)
Internal dashboard access is strictly governed:
*   **Support Agents:** View customer inquiries, manage bookings, and respond to support tickets.
*   **Operations / Finance:** Oversee transaction splits and adjust markup rules dynamically based on demand.
*   **Platform Owners:** Full access to view analytics, manage content, and oversee the entire system.

### 4.2 Automated Marketing & Retention
Because the platform automatically captures emails during the checkout flow, the user database naturally scales into a powerful marketing engine.
*   **User Tracking:** We track user search behavior and frequent routes.
*   **Automated Deal Blasts:** If a price drops significantly on a route a user previously searched (or frequently travels), the system triggers a targeted email alert, prompting them to place a booking. This converts one-time users into lifelong customers.
