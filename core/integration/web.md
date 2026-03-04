# CleverTap Web SDK — Integration Reference

> Self-contained integration guide for the CleverTap Web SDK.
> Covers vanilla JS, React, Next.js, Vue, and Angular.

---

## Prerequisites

| Requirement | Details |
|---|---|
| **npm/yarn installs** | Node.js 14+ |
| **CDN (script tag)** | Any modern browser (Chrome, Firefox, Safari, Edge) |
| **Protocol** | HTTPS required — `file://` will not work |
| **CleverTap account** | You need your **Account ID** and **Region** from the CleverTap dashboard (Settings > Project) |

### Regions

| Region Code | Data Center |
|---|---|
| `us1` | United States |
| `in1` | India |
| `sg1` | Singapore |
| `aps3` | Asia Pacific (Sydney) |
| `mec1` | Middle East |

If your dashboard URL contains `us1.dashboard.clevertap.com`, your region is `us1`. Match accordingly.

---

## Installation

### Option 1: npm (recommended)

```bash
npm install clevertap-web-sdk --save
```

### Option 2: yarn

```bash
yarn add clevertap-web-sdk
```

### Option 3: CDN (legacy, deprecated)

> **Deprecated.** Use npm/yarn for new projects. The CDN approach is documented here only for legacy codebases.

Add this script tag before the closing `</body>` tag:

```html
<script type="text/javascript">
  var clevertap = {event:[], profile:[], account:[], onUserLogin:[], notifications:[], privacy:[]};
  clevertap.account.push({"id": "YOUR_ACCOUNT_ID"});
  clevertap.region = "YOUR_REGION";
  (function () {
    var wzrk = document.createElement('script');
    wzrk.type = 'text/javascript';
    wzrk.async = true;
    wzrk.src = ('https:' == document.location.protocol ? 'https://d2r1yp2w7bby2u.cloudfront.net' : 'http://static.clevertap.com') + '/js/clevertap.min.js';
    var s = document.getElementsByTagName('script')[0];
    s.parentNode.insertBefore(wzrk, s);
  })();
</script>
```

---

## Initialization

### Vanilla JavaScript (npm/yarn)

```javascript
import clevertap from 'clevertap-web-sdk';

clevertap.init('YOUR_ACCOUNT_ID', 'YOUR_REGION');
clevertap.spa = true; // Enable for Single Page Applications
```

### Stub Object Pattern (CDN / advanced control)

The stub object pattern lets you queue calls before the SDK fully loads:

```javascript
var clevertap = {event:[], profile:[], account:[], onUserLogin:[], notifications:[], privacy:[]};

// Required: set your account ID
clevertap.account.push({"id": "YOUR_ACCOUNT_ID"});

// Required: set your region
clevertap.region = "YOUR_REGION"; // us1, in1, sg1, aps3, mec1

// Recommended for SPAs: prevents duplicate page views on route changes
clevertap.spa = true;
```

Calls made to `clevertap.event.push(...)`, `clevertap.profile.push(...)`, etc. are queued in the arrays and replayed once the SDK loads.

---

## Event Tracking

### Basic event (no properties)

```javascript
clevertap.event.push("Product Viewed");
```

### Event with properties

```javascript
clevertap.event.push("Product Viewed", {
  "Product Name": "Wireless Headphones",
  "Category": "Electronics",
  "Price": 79.99,
  "On Sale": true
});
```

### Charged event (transactions)

```javascript
clevertap.event.push("Charged", {
  "Amount": 119.98,
  "Currency": "USD",
  "Charged ID": "order-12345",
  "Items": [
    {
      "Product Name": "Wireless Headphones",
      "Category": "Electronics",
      "Price": 79.99,
      "Quantity": 1
    },
    {
      "Product Name": "USB-C Cable",
      "Category": "Accessories",
      "Price": 39.99,
      "Quantity": 1
    }
  ]
});
```

### Property value rules

- Property names: strings, max 120 characters
- Property values: string, number, boolean, or Date
- Max 256 custom event properties per event
- Event names: strings, max 512 characters
- Avoid special characters in event names; use alphanumeric and spaces

---

## User Identity

### `onUserLogin` — Identify a user on login

Use `onUserLogin` when a user logs in. This creates a new profile or merges with an existing one based on the identity fields.

```javascript
clevertap.onUserLogin.push({
  "Site": {
    "Name": "Jane Doe",
    "Identity": "user-12345",          // Your internal user ID
    "Email": "jane@example.com",
    "Phone": "+14155551234",
    "Gender": "F",                      // M or F
    "DOB": new Date("1990-06-15"),
    // Custom properties
    "Plan": "Premium",
    "Signup Date": new Date()
  }
});
```

**Identity fields** that trigger profile merge:
- `Identity` — your internal user/account ID
- `Email` — email address
- `Phone` — phone number with country code

Include at least one identity field. If a profile with that identity exists, the SDK merges the data. If not, it creates a new profile.

### `profile.push` — Update the current user's profile

Use `profile.push` to update properties on the already-identified user. This does **not** create or switch profiles.

```javascript
clevertap.profile.push({
  "Site": {
    "Plan": "Enterprise",
    "Company": "Acme Corp",
    "Score": 42
  }
});
```

### When to use which

| Scenario | Method |
|---|---|
| User logs in | `onUserLogin.push(...)` |
| Update a field on current user | `profile.push(...)` |
| User signs up (new account) | `onUserLogin.push(...)` |
| Anonymous user becomes known | `onUserLogin.push(...)` |

---

## Framework Variants

### React

```jsx
// src/App.jsx
import { useEffect } from 'react';
import clevertap from 'clevertap-web-sdk';

function App() {
  useEffect(() => {
    clevertap.init('YOUR_ACCOUNT_ID', 'YOUR_REGION');
    clevertap.spa = true;
  }, []);

  const handlePurchase = () => {
    clevertap.event.push("Purchase Completed", {
      "Amount": 49.99
    });
  };

  return (
    <div>
      <button onClick={handlePurchase}>Buy Now</button>
    </div>
  );
}

export default App;
```

**Key points:**
- Initialize in `useEffect` with an empty dependency array so it runs once on mount
- `clevertap.spa = true` is required for React Router or any client-side routing
- The SDK is a singleton; import it in any component to call `event.push` or `profile.push`

### Next.js

Next.js requires special handling because of server-side rendering — the SDK accesses `window` and `document`, which do not exist on the server.

```jsx
// pages/_app.js
import { useEffect } from 'react';

function MyApp({ Component, pageProps }) {
  useEffect(() => {
    import('clevertap-web-sdk').then((clevertap) => {
      const ct = clevertap.default;
      ct.init('YOUR_ACCOUNT_ID', 'YOUR_REGION');
      ct.spa = true;
    });
  }, []);

  return <Component {...pageProps} />;
}

export default MyApp;
```

**Alternative: dynamic import wrapper**

```jsx
// lib/clevertap.js
import dynamic from 'next/dynamic';

let clevertap = null;

export const initClevertap = async () => {
  if (typeof window !== 'undefined') {
    const ct = (await import('clevertap-web-sdk')).default;
    ct.init('YOUR_ACCOUNT_ID', 'YOUR_REGION');
    ct.spa = true;
    clevertap = ct;
  }
  return clevertap;
};

export const getClevertap = () => clevertap;
```

```jsx
// Usage in any component
import { getClevertap } from '../lib/clevertap';

function TrackButton() {
  const handleClick = () => {
    const ct = getClevertap();
    if (ct) {
      ct.event.push("Button Clicked");
    }
  };

  return <button onClick={handleClick}>Track</button>;
}
```

**Key points:**
- Never import `clevertap-web-sdk` at the top level in Next.js — use dynamic `import()` inside `useEffect` or behind a `typeof window` check
- `next/dynamic` with `ssr: false` also works for wrapping components that use the SDK
- The SDK must only initialize on the client

### Next.js App Router (v13+)

```jsx
// app/providers.jsx
'use client';

import { useEffect } from 'react';

export function ClevertapProvider({ children }) {
  useEffect(() => {
    import('clevertap-web-sdk').then((clevertap) => {
      const ct = clevertap.default;
      ct.init('YOUR_ACCOUNT_ID', 'YOUR_REGION');
      ct.spa = true;
    });
  }, []);

  return <>{children}</>;
}
```

```jsx
// app/layout.jsx
import { ClevertapProvider } from './providers';

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <ClevertapProvider>{children}</ClevertapProvider>
      </body>
    </html>
  );
}
```

### Vue

#### Option A: Init in main.js (simplest)

```javascript
// main.js
import { createApp } from 'vue';
import App from './App.vue';
import clevertap from 'clevertap-web-sdk';

clevertap.init('YOUR_ACCOUNT_ID', 'YOUR_REGION');
clevertap.spa = true;

const app = createApp(App);

// Make clevertap available globally via provide/inject
app.provide('clevertap', clevertap);
app.mount('#app');
```

```vue
<!-- AnyComponent.vue -->
<script setup>
import { inject } from 'vue';

const clevertap = inject('clevertap');

function trackEvent() {
  clevertap.event.push("Feature Used", { "Feature": "Dashboard" });
}
</script>

<template>
  <button @click="trackEvent">Use Feature</button>
</template>
```

#### Option B: Plugin pattern

```javascript
// plugins/clevertap.js
import clevertap from 'clevertap-web-sdk';

export default {
  install(app, options) {
    clevertap.init(options.accountId, options.region);
    clevertap.spa = true;
    app.config.globalProperties.$clevertap = clevertap;
    app.provide('clevertap', clevertap);
  }
};
```

```javascript
// main.js
import { createApp } from 'vue';
import App from './App.vue';
import ClevertapPlugin from './plugins/clevertap';

const app = createApp(App);
app.use(ClevertapPlugin, {
  accountId: 'YOUR_ACCOUNT_ID',
  region: 'YOUR_REGION'
});
app.mount('#app');
```

**Key points:**
- `clevertap.spa = true` is required for Vue Router
- Use `provide`/`inject` (Composition API) or `globalProperties` (Options API) to access the SDK

### Angular

```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { AppComponent } from './app.component';
import clevertap from 'clevertap-web-sdk';

@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule],
  bootstrap: [AppComponent]
})
export class AppModule {
  constructor() {
    clevertap.init('YOUR_ACCOUNT_ID', 'YOUR_REGION');
    clevertap.spa = true;
  }
}
```

#### Optional: Injectable service

```typescript
// clevertap.service.ts
import { Injectable } from '@angular/core';
import clevertap from 'clevertap-web-sdk';

@Injectable({ providedIn: 'root' })
export class ClevertapService {
  private ct = clevertap;

  init(accountId: string, region: string) {
    this.ct.init(accountId, region);
    this.ct.spa = true;
  }

  trackEvent(name: string, properties?: Record<string, any>) {
    if (properties) {
      this.ct.event.push(name, properties);
    } else {
      this.ct.event.push(name);
    }
  }

  identifyUser(profile: Record<string, any>) {
    this.ct.onUserLogin.push({ "Site": profile });
  }

  updateProfile(properties: Record<string, any>) {
    this.ct.profile.push({ "Site": properties });
  }
}
```

```typescript
// app.component.ts
import { Component, OnInit } from '@angular/core';
import { ClevertapService } from './clevertap.service';

@Component({
  selector: 'app-root',
  template: `<button (click)="track()">Track</button>`
})
export class AppComponent implements OnInit {
  constructor(private ct: ClevertapService) {}

  ngOnInit() {
    this.ct.init('YOUR_ACCOUNT_ID', 'YOUR_REGION');
  }

  track() {
    this.ct.trackEvent('Button Clicked', { location: 'home' });
  }
}
```

**Key points:**
- Initialize in `AppModule` constructor or root component's `ngOnInit`
- Wrap the SDK in a service for testability and consistent API

---

## Common Pitfalls

### 1. `file://` protocol does not work

The SDK requires HTTPS. If you open an HTML file directly in the browser (`file://...`), the SDK will fail silently. Use a local dev server:

```bash
npx serve .
# or
python3 -m http.server 8000
```

### 2. Must initialize before tracking

Calls to `event.push` or `profile.push` before initialization are silently dropped (unless using the stub object pattern with CDN, which queues them).

```javascript
// WRONG: tracking before init
clevertap.event.push("Page View");
clevertap.init('YOUR_ACCOUNT_ID', 'YOUR_REGION');

// CORRECT: init first, then track
clevertap.init('YOUR_ACCOUNT_ID', 'YOUR_REGION');
clevertap.event.push("Page View");
```

### 3. Legacy SDK is deprecated

If you see `clevertap.js` loaded from `static.clevertap.com`, you are using the deprecated CDN version. Migrate to the npm package:

```bash
npm install clevertap-web-sdk --save
```

Then replace the script tag with an `import` statement.

### 4. SPA flag must be set for single-page apps

Without `clevertap.spa = true`, the SDK may fire duplicate page views or miss route changes in React, Vue, Angular, and Next.js apps.

```javascript
clevertap.spa = true; // Set immediately after init
```

### 5. Debug logging

Enable verbose logging during development:

```javascript
// Method 1: setLogLevel (0=off, 1=error, 2=warn, 3=debug)
clevertap.setLogLevel(3);

// Method 2: sessionStorage flag (works even before SDK loads)
sessionStorage['WZRK_D'] = '';
```

Check the browser console for entries prefixed with `[CleverTap]`. Remove debug logging before shipping to production.

### 6. Offline mode

Pause all network requests from the SDK (useful for GDPR consent flows):

```javascript
// Stop sending data
clevertap.setOffline(true);

// Resume sending data (queued events will flush)
clevertap.setOffline(false);
```

### 7. Multiple accounts on one page

If you need to send data to more than one CleverTap account:

```javascript
import clevertap from 'clevertap-web-sdk';

clevertap.init('ACCOUNT_ID_1', 'REGION_1');

// Second instance
const ct2 = clevertap.init('ACCOUNT_ID_2', 'REGION_2', 'secondary');
```

---

## Quick Reference

```javascript
import clevertap from 'clevertap-web-sdk';

// 1. Initialize
clevertap.init('YOUR_ACCOUNT_ID', 'YOUR_REGION');
clevertap.spa = true;

// 2. Identify user on login
clevertap.onUserLogin.push({
  "Site": {
    "Name": "Jane Doe",
    "Identity": "user-12345",
    "Email": "jane@example.com"
  }
});

// 3. Track events
clevertap.event.push("Feature Used", { "Feature": "Search" });

// 4. Update profile
clevertap.profile.push({
  "Site": { "Plan": "Premium" }
});

// 5. Debug (dev only)
clevertap.setLogLevel(3);
```
