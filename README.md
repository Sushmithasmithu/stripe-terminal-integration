# 🚀 Stripe Terminal Integration Guide

This guide walks you through integrating **Stripe Terminal** with both **backend (Node.js + Express)** and **frontend (React/JS client)** to accept in-person payments.

---

## 🏷️ What is Stripe Terminal?

**Stripe Terminal** is Stripe’s solution for **in-person payments**.  
It provides **pre-certified card readers** (like **WisePOS E**, **Verifone P400**, etc.) and SDKs to build custom **point-of-sale (POS)** experiences.

With Stripe Terminal, you can:  

- 💳 **Accept card-present payments** (chip, swipe, contactless, Apple Pay, Google Pay).  
- 📡 **Connect readers** to your web or mobile app using the **Stripe Terminal SDKs**.  
- 🔑 **Stay PCI compliant** — card data is encrypted & handled by Stripe.  
- 🏦 **Support platforms (Stripe Connect)** → split payments or charge on behalf of connected accounts.  
- 🔗 **Unify online & offline payments** in one Stripe dashboard.  

👉 Best suited for businesses or platforms needing **seamless online + offline payments**.

---

## 📊 Stripe Terminal vs Stripe Payments (Online)

| Feature | **Stripe Terminal** (In-Person) | **Stripe Payments** (Online) |
|---------|---------------------------------|------------------------------|
| Payment Type | Card-present (chip, swipe, tap, wallets) | Card-not-present (web, mobile, subscriptions) |
| Hardware | Requires physical reader (WisePOS E, P400, etc.) | No hardware needed |
| SDKs | Terminal SDK (Web, iOS, Android) | Stripe.js / Stripe API |
| PCI Compliance | Managed by Stripe (reader handles sensitive data) | Managed by Stripe Elements & APIs |
| Use Case | Retail, restaurants, SaaS POS systems | E-commerce, SaaS billing, marketplaces |

---

## 📦 1. Install Dependencies

### Backend (Node.js + Express)
```bash
npm install express stripe cors dotenv
```

### Frontend (React or any client app)
```bash
npm install @stripe/terminal-js
```

---

## ⚙️ 2. Backend Setup

Set up an **Express server** to create connection tokens and PaymentIntents.

### `server.js`
```javascript
import express from "express";
import Stripe from "stripe";
import cors from "cors";
import dotenv from "dotenv";

dotenv.config();

const app = express();
app.use(cors());
app.use(express.json());

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY, {
  apiVersion: "2024-06-20",
});

// 1️⃣ Create a Connection Token (used by readers to connect securely)
app.post("/connection_token", async (req, res) => {
  const token = await stripe.terminal.connectionTokens.create();
  res.json({ secret: token.secret });
});

// 2️⃣ Create a PaymentIntent
app.post("/create_payment_intent", async (req, res) => {
  const { amount, currency, metadata, transferData, applicationFee } = req.body;

  const paymentIntent = await stripe.paymentIntents.create({
    amount,
    currency,
    payment_method_types: ["card_present"], // Required for Terminal
    capture_method: "automatic",
    metadata,
    ...(transferData && {
      application_fee_amount: applicationFee,
      transfer_data: { destination: transferData },
    }),
  });

  res.json(paymentIntent);
});

app.listen(3000, () =>
  console.log("✅ Server running on http://localhost:3000")
);
```

---

## 💻 3. Frontend Setup

Use the **Stripe Terminal JavaScript SDK** to discover readers and process payments.

### `terminal.js`
```javascript
import { loadStripeTerminal } from "@stripe/terminal-js";

let terminal;

async function initTerminal() {
  terminal = await loadStripeTerminal();

  // Fetch connection token from backend
  terminal.setFetchConnectionToken(async () => {
    const response = await fetch("http://localhost:3000/connection_token", {
      method: "POST",
    });
    const data = await response.json();
    return data.secret;
  });

  terminal.setFetchConnectionTokenFailed(() => {
    console.error("❌ Failed to fetch connection token");
  });
}

async function discoverReaders() {
  const result = await terminal.discoverReaders();
  if (result.error) {
    console.error(result.error);
  } else if (result.discoveredReaders.length === 0) {
    console.log("⚠️ No available readers.");
  } else {
    const selectedReader = result.discoveredReaders[0];
    await terminal.connectReader(selectedReader);
    console.log("📡 Connected to reader:", selectedReader.label);
  }
}

async function collectPayment(amount) {
  // Create PaymentIntent on backend
  const response = await fetch("http://localhost:3000/create_payment_intent", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ amount, currency: "usd" }),
  });
  const paymentIntent = await response.json();

  // Collect payment method via reader
  const result = await terminal.collectPaymentMethod(paymentIntent.client_secret);
  if (result.error) {
    console.error(result.error);
    return;
  }

  // Process the payment
  const confirmResult = await terminal.processPayment(result.paymentIntent);
  if (confirmResult.error) {
    console.error(confirmResult.error);
  } else {
    console.log("✅ Payment successful:", confirmResult.paymentIntent);
  }
}
```

---

## 📝 4. Key Notes on Stripe Terminal

- **🔑 Connection Token** → Temporary key issued by backend, used by reader to connect to Stripe. Expires after ~90 seconds.  
- **💳 PaymentIntent** → Represents an in-person payment. Must be created with `"payment_method_types": ["card_present"]`.  
- **📡 Readers** → Hardware devices like **BBPOS WisePOS E** & **Verifone P400** that securely handle card data.  
- **🏦 Platform Payments (Connect)** → Use `application_fee_amount` & `transfer_data.destination` for marketplace payouts.  
- **⚡ Typical Flow**:  
  1. Backend → Create **connection token**  
  2. Frontend → Discover & connect to reader  
  3. Backend → Create **PaymentIntent**  
  4. Reader → Collects card  
  5. Stripe → Processes payment  

---

## 📚 5. References & Further Reading

- 🔗 [Stripe Terminal Overview](https://stripe.com/docs/terminal)  
- 🔗 [Stripe Terminal API Reference](https://stripe.com/docs/api/terminal)  
- 🔗 [Readers Supported by Stripe](https://stripe.com/docs/terminal/readers/overview)  
- 🔗 [Accept In-Person Payments](https://stripe.com/docs/terminal/payments)  
- 🔗 [Connect Platforms with Terminal](https://stripe.com/docs/terminal/connect)  
- 🔗 [Security & Connection Tokens](https://stripe.com/docs/terminal/fleet/connection-tokens)  
