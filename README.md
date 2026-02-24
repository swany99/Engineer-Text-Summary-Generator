# Engineer Text Summary Generator (Gemini 2.5 Flash)

A lightweight web app that converts raw engineer notes into a **3‑section professional summary**—**Symptoms**, **Cause**, and **Solution**—with a strict **470‑character** total limit.

- Runs **entirely in the browser** using the **Google Generative Language API (v1)**.
- Uses **Gemini 2.5 Flash** (free tier compatible).
- **No build tools** required (pure HTML + Tailwind CDN).

---

## Features

- **Gemini 2.5 Flash** model.
- **Strict 470‑character** constraint with a client‑side safety guard.
- Clear, repeatable format (**bold headings** + short paragraph per section).
- **Copy to clipboard** button + live character counter.
- Simple, modern UI with Tailwind CSS.

---

## Prerequisites

- A Google Cloud project with **Generative Language API** enabled.
- A **Browser API key** created in **Google Cloud Console → APIs & Services → Credentials**.

**Recommended API key restrictions:**
- Restrict by **HTTP referrers (websites)** to your domain(s), for example:
  - `http://localhost:*` (for local testing if needed)
  - `https://<your-username>.github.io/*` (for GitHub Pages)
- Restrict by **API** to **Generative Language API** only.

> 🔐 **Security Note:** Keys embedded in client code are visible. Restrict by referrer and API. For production/teams, consider a tiny serverless proxy (example included below) so the key stays server-side.

---

## Quick Start (Local)

1. Create a new folder/repo and add the `index.html` from the **Full HTML** section below.
2. Open `index.html` and set your API key:
   ```js
   const USER_API_KEY = "YOUR_RESTRICTED_BROWSER_KEY";
