# React-Quantum-SaaS

A SaaS starter built with React, Firebase and Stripe. It gives you user sign-in, multi-user subscription accounts, Stripe billing and a set of ready-made React components, so you can start from a working base instead of wiring all of this up yourself.

Built on top of the `@fireactjs/core` package for authentication.

## What you get

- Stripe subscriptions: pricing table, checkout, plan changes, cancellation, invoices and saved payment methods
- Subscription accounts with their own users. The owner can invite people by email and manage them
- Permission control at the subscription level (for example `access` and `admin`)
- Firebase Auth (Google, Facebook, Microsoft, GitHub, Twitter, email/password) through `@fireactjs/core`
- Firebase Cloud Functions that handle the server side: Stripe webhooks, invites, billing updates
- Optional invite emails through Mailgun
- Material UI components that you can swap out or restyle

## Tech stack

| Part | Used for |
| --- | --- |
| React 18 + React Router 6 | Front end |
| Material UI 5 | UI components |
| Firebase (Auth, Firestore, Functions, Hosting) | Back end and hosting |
| Stripe | Payments and subscriptions |
| Mailgun (optional) | Invite emails |
| Babel | Builds `src/lib` into `dist/esm` and `dist/cjs` |

## Project structure

```
React-Quantum-SaaS/
├── src/
│   ├── App.js            # Demo app that wires everything together
│   ├── lib/              # The SaaS components (this is what gets built to dist/)
│   │   ├── components/   # Pricing, billing, users, settings, etc.
│   │   ├── pathnames.json
│   │   └── index.js
│   └── authMethods.json  # Turn sign-in providers on or off
├── functions/            # Firebase Cloud Functions
├── dist/                 # Built output (esm + cjs)
├── public/
├── firestore.rules
└── package.json
```

## Getting started

### 1. Requirements

- Node.js 18 or newer
- A Firebase project (Auth, Firestore and Functions enabled)
- A Stripe account
- Firebase CLI: `npm i -g firebase-tools`

### 2. Install

```bash
git clone <your-repo-url>
cd React-Quantum-SaaS
npm install

cd functions
npm install
cd ..
```

### 3. Create the config files

These files hold keys and are not in the repo, so you need to create them yourself. Do not commit real keys.

**`src/firebaseConfig.json`** - the web app config from your Firebase project settings.

**`src/config.json`** - front-end settings:

```json
{
    "stripe": {
        "public_api_key": "pk_test_..."
    },
    "plans": [
        {
            "id": "free",
            "title": "Free",
            "popular": false,
            "priceIds": ["price_1..."],
            "currency": "$",
            "price": 0,
            "frequency": "week",
            "description": [
                "10 users included",
                "2 GB of storage",
                "Help center access",
                "Email support"
            ],
            "free": true,
            "legacy": false
        },
        {
            "id": "pro",
            "title": "Pro",
            "popular": true,
            "priceIds": ["price_2..."],
            "currency": "$",
            "price": 10,
            "frequency": "week",
            "description": [
                "20 users included",
                "10 GB of storage",
                "Help center access",
                "Priority email support"
            ],
            "free": false,
            "legacy": false
        }
    ],
    "permissions": {
        "access": { "default": true, "admin": false },
        "admin": { "default": false, "admin": true }
    },
    "subscription": {
        "singular": "project",
        "plural": "projects"
    }
}
```

**`functions/config.json`** - server-side settings. It uses the same `plans` and `permissions` as above, plus your secret keys:

```json
{
    "brand": "My Brand",
    "site_name": "My SaaS App",
    "site_url": "your-site-url",
    "sign_in_url": "your-sign-in-page-url",
    "sign_up_url": "your-sign-up-page-url",
    "stripe": {
        "secret_api_key": "sk_test_...",
        "end_point_secret": "whsec_..."
    },
    "plans": [],
    "permissions": {
        "access": { "default": true, "admin": false },
        "admin": { "default": false, "admin": true }
    },
    "mailgun": {
        "api_key": "...",
        "domain": "your-mailgun-domain",
        "from": "No Reply <no-reply@your-mailgun-domain>",
        "templates": { "invite_email": "invite" }
    }
}
```

Fill `plans` with the same plan list you used in `src/config.json`. The `mailgun` block is optional.

### 4. Set up plans in Stripe

- Create a product in Stripe for each plan. A product can have more than one price, but all prices in one plan must have the same billing period.
- Put each Stripe `price_...` ID in the plan's `priceIds`.
- A plan can hold prices from different products (for example a license plus a support price).
- Set `free: true` for plans that should not ask for a card. Make sure every price in that plan is 0 in Stripe.
- Set `legacy: true` on a plan you no longer sell. It is hidden from the pricing table but existing customers keep it.

### 5. Stripe webhook

Create a webhook endpoint in Stripe that points to your deployed Cloud Function:

```
<your-cloud-functions-base-url>/fireactjsSaas-stripeWebHook
```

Copy the signing secret into `end_point_secret` in `functions/config.json`.

Send these events to the endpoint:

```
customer.subscription.created
customer.subscription.updated
customer.subscription.deleted
customer.subscription.trial_will_end
customer.subscription.pending_update_applied
customer.subscription.pending_update_expired
invoice.created
invoice.updated
invoice.deleted
invoice.finalized
invoice.paid
invoice.sent
invoice.voided
invoice.marked_uncollectible
invoice.payment_action_required
invoice.payment_failed
invoice.payment_succeeded
```

### 6. Firestore rules

The rules in `firestore.rules` secure the subscription data. Add them to your Firestore rules, or deploy with `firebase deploy --only firestore:rules`. The key rules are:

```
match /users/{userId}/paymentMethods/{paymentMethodId} {
    allow read, update, create, delete: if request.auth.uid == userId;
}
match /subscriptions/{subscriptionId} {
    allow read: if request.auth.uid != null && request.auth.uid in resource.data.permissions.access;
    allow update: if request.auth.uid != null && request.auth.uid in resource.data.permissions.admin && (!request.resource.data.diff(resource.data).affectedKeys()
        .hasAny(['currency', 'ownerId', 'paymentCycle', 'paymentMethod', 'plan', 'price', 'stripeItems', 'stripeSubscriptionId', 'subscriptionCreated', 'subscriptionCurrentPeriodEnd', 'subscriptionCurrentPeriodStart', 'subscriptionEnded', 'subscriptionStatus']));
}
match /subscriptions/{subscriptionId}/invoices/{invoiceId} {
    allow read: if request.auth.uid != null && request.auth.uid in get(/databases/$(database)/documents/subscriptions/$(subscriptionId)).data.permissions.admin;
}
match /invites/{inviteId} {
    allow read, delete: if request.auth.uid != null && request.auth.token.email == resource.data.email;
}
```

### 7. Mailgun invite emails (optional)

Invites work without Mailgun, but then you have to tell new users yourself where to sign up. To send invite emails:

1. Add your Mailgun details to the `mailgun` block in `functions/config.json`.
2. In Mailgun, create a template named `invite` with the subject `{{sender}} invited you to {{site_name}}` and this body:

```
Hi {{name}},

You received this invite because {{sender}} invited you to join {{site_name}}. Please sign in ({{sign_in_url}}) to accept the invite.

If you don't have a user account yet, please sign up ({{sign_up_url}}) here.

Best regards,

The {{site_name}} team
```

Template variables: `{{sender}}`, `{{site_name}}`, `{{name}}`, `{{sign_in_url}}`, `{{sign_up_url}}`.

## Running locally

```bash
npm start
```

The app opens in your browser on port 3000. To test the Cloud Functions locally, run `npm run serve` inside `functions/`.

## Scripts

| Command | What it does |
| --- | --- |
| `npm start` | Start the dev server |
| `npm run build` | Build `src/lib` into `dist/esm` and `dist/cjs` |
| `npm test` | Run tests |

## Deploying to Firebase

```bash
firebase login
firebase deploy --only functions
firebase deploy
```

If you see a blank page on your live URL, check that the Firebase Hosting public folder is set to `build`.

## How the pieces connect

`src/App.js` is the reference setup. It wraps the app in `FireactProvider` and `AuthProvider` from `@fireactjs/core`, then adds the subscription routes from `src/lib`. Routes under `/sub/:subscriptionId` are protected by `PermissionRouter`, so you can restrict a page to `access` or `admin` users. The routes themselves are listed in `src/lib/pathnames.json`.

Note: the Cloud Functions are exported as `fireactjsSaas` in `functions/index.js`, and the front end calls them by that name (for example `fireactjsSaas-createSubscription`). If you rename the export, update the calls in `src/lib/components` too.

## Customizing

- Brand name and logo: edit `Brand`, `brand` and the `Logo` component in `src/App.js`
- Sign-in providers: edit `src/authMethods.json`
- Plans, permissions and the "project" wording: edit `src/config.json` (and `functions/config.json` to match)
- Components: everything in `src/lib/components` can be edited or replaced

## Author

Sikandar

- GitHub: RanaSikandar0033

## Credits and license

This project is based on the open-source fireactjs/saas package by Chaoming Li and uses `@fireactjs/core`. Released under the ISC license.
