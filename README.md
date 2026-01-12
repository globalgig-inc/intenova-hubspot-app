# Intenova HubSpot App

A multi-tenant web application that enables users to connect their HubSpot accounts for AI-powered CRM access via the HubSpot MCP (Model Context Protocol) server.

## Architecture Overview

```
┌─────────────────┐                      ┌─────────────────────┐
│     User        │                      │  Google OAuth       │
│   (Browser)     │──── Google SSO ─────►│  (accounts.google   │
└────────┬────────┘                      │   .com)             │
         │                               └─────────────────────┘
         │ Sign in complete
         ▼
┌─────────────────┐
│  Intenova Web   │◄── redirect ─────────
│  App (Firebase  │
│  Hosting/GCP)   │
└────────┬────────┘
         │ User clicks "Connect HubSpot"
         ▼
┌─────────────────┐     OAuth Flow       ┌─────────────────────┐
│  HubSpot OAuth  │◄────────────────────►│  User's HubSpot     │
│  Callback       │                      │  Account            │
│  (test.intenova │                      └─────────────────────┘
│   .ai/callback) │
└────────┬────────┘
         │ Store tokens in Firestore
         ▼
┌─────────────────┐     MCP Protocol     ┌─────────────────────┐
│  MCP Client     │◄────────────────────►│  mcp.hubspot.com    │
│  (uses stored   │                      │  (HubSpot MCP       │
│   tokens)       │                      │   Server)           │
└─────────────────┘                      └─────────────────────┘
```

**Key Components:**
- **Google SSO**: Authenticates users to the Intenova app
- **HubSpot OAuth**: Connects each user's HubSpot account (separate flow)
- **Firebase Hosting**: Hosts the web app at `test.intenova.ai`
- **Firestore**: Stores HubSpot OAuth tokens per user
- **HubSpot MCP Server**: Provides AI-ready read access to CRM data

## Prerequisites

### 1. HubSpot Developer Account

1. Go to [developers.hubspot.com](https://developers.hubspot.com)
2. Click "Get Started" or "Create a developer account"
3. Sign in with your existing HubSpot credentials OR create a new account
4. Accept the Developer Terms of Service
5. Create a "Developer Test Account" for testing (free sandbox environment)

### 2. Required Tools

- **Node.js v18+**: [Download](https://nodejs.org/)
- **HubSpot CLI v7.60.0+**:
  ```bash
  npm install -g @hubspot/cli@latest
  hs --version  # Verify version
  ```
- **Firebase CLI**:
  ```bash
  npm install -g firebase-tools
  firebase --version  # Verify installation
  ```

### 3. GCP/Firebase Project

This project uses GCP project `agentic-platform-482920` with existing Google OAuth configuration.

## Setup Guide

### Step 1: Authenticate with HubSpot CLI

```bash
hs auth
```

Follow the prompts to authenticate with your HubSpot developer account.

### Step 2: Deploy HubSpot App

```bash
# From project root
hs project upload
```

This uploads the app configuration to HubSpot. After upload:

1. Go to [HubSpot Developer Portal](https://app.hubspot.com/developer)
2. Find "Intenova HubSpot MCP Connector" in your apps
3. Copy the **Client ID** and **Client Secret**

### Step 3: Configure Firebase

```bash
# Login to Firebase
firebase login

# Verify project access
firebase projects:list

# Use the existing project
firebase use agentic-platform-482920
```

### Step 4: Get Firebase Configuration

1. Go to [Firebase Console](https://console.firebase.google.com/)
2. Select project `agentic-platform-482920`
3. Go to Project Settings > General
4. Under "Your apps", find or add a Web app
5. Copy the Firebase configuration (apiKey, appId, etc.)

### Step 5: Update Configuration Files

Edit `public/index.html` and `public/hubspot-callback.html`:

```javascript
// Firebase configuration
const firebaseConfig = {
    apiKey: "YOUR_FIREBASE_API_KEY",        // From Firebase Console
    authDomain: "agentic-platform-482920.firebaseapp.com",
    projectId: "agentic-platform-482920",
    storageBucket: "agentic-platform-482920.appspot.com",
    messagingSenderId: "973127896330",
    appId: "YOUR_FIREBASE_APP_ID"           // From Firebase Console
};

// HubSpot OAuth configuration
const hubspotConfig = {
    clientId: 'YOUR_HUBSPOT_CLIENT_ID',     // From HubSpot Developer Portal
    clientSecret: 'YOUR_HUBSPOT_CLIENT_SECRET', // From HubSpot Developer Portal
    // ...
};
```

### Step 6: Deploy to Firebase

```bash
# Deploy hosting and Firestore rules
firebase deploy
```

### Step 7: Configure Custom Domain (Optional)

To use `test.intenova.ai`:

1. Go to Firebase Console > Hosting
2. Click "Add custom domain"
3. Enter `test.intenova.ai`
4. Follow DNS configuration instructions

## Usage

### For Users

1. Visit `test.intenova.ai` (or your deployed URL)
2. Click "Sign in with Google"
3. Click "Connect HubSpot"
4. Authorize the app in HubSpot
5. You're connected! Tokens are stored securely.

### For MCP Clients (Claude Code, etc.)

After users connect their HubSpot accounts, AI clients can access their CRM data via the MCP server.

Add to your MCP client configuration:

```json
{
  "mcpServers": {
    "hubspot": {
      "url": "https://mcp.hubspot.com",
      "auth": {
        "type": "oauth",
        "clientId": "<YOUR_CLIENT_ID>",
        "clientSecret": "<YOUR_CLIENT_SECRET>"
      }
    }
  }
}
```

## Available CRM Scopes

### Required Scopes (Always Requested)
- `oauth` - Base OAuth scope
- `crm.objects.contacts.read` - Read contacts
- `crm.objects.companies.read` - Read companies
- `crm.objects.deals.read` - Read deals

### Optional Scopes (User Can Grant)
- `crm.objects.tickets.read` - Read tickets
- `crm.objects.users.read` - Read users
- `crm.objects.owners.read` - Read owners
- `crm.objects.products.read` - Read products
- `crm.objects.orders.read` - Read orders
- `crm.objects.invoices.read` - Read invoices
- `crm.objects.quotes.read` - Read quotes
- `crm.objects.line_items.read` - Read line items
- `crm.objects.subscriptions.read` - Read subscriptions
- `crm.objects.carts.read` - Read carts

## Project Structure

```
intenova-hubspot-app/
├── hsproject.json                    # HubSpot project configuration
├── src/
│   └── app/
│       └── user-level-app-hsmeta.json  # HubSpot app metadata & OAuth config
├── public/                           # Firebase Hosting static files
│   ├── index.html                    # Main web app (Google SSO + HubSpot connect)
│   └── hubspot-callback.html         # HubSpot OAuth callback handler
├── firebase.json                     # Firebase configuration
├── firestore.rules                   # Firestore security rules
├── firestore.indexes.json            # Firestore indexes
├── .firebaserc                       # Firebase project alias
├── README.md                         # This file
└── .gitignore                        # Git ignore rules
```

## Security Considerations

### Production Recommendations

1. **Move Token Exchange to Cloud Functions**

   The current callback page includes client-side token exchange for simplicity. For production:

   ```bash
   # Initialize Cloud Functions
   firebase init functions
   ```

   Create a function to handle token exchange securely:

   ```javascript
   // functions/index.js
   exports.exchangeHubspotToken = functions.https.onCall(async (data, context) => {
       // Verify user is authenticated
       if (!context.auth) throw new Error('Unauthorized');

       // Exchange code for tokens server-side
       // Store in Firestore
       // Return success
   });
   ```

2. **Environment Variables**

   Never commit secrets. Use Firebase environment config:

   ```bash
   firebase functions:config:set hubspot.client_id="xxx" hubspot.client_secret="xxx"
   ```

3. **Token Refresh**

   HubSpot tokens expire. Implement automatic refresh:
   - Check token expiration before MCP calls
   - Use refresh_token to get new access_token
   - Update Firestore with new tokens

### Firestore Security Rules

The current rules ensure users can only access their own tokens:

```
match /users/{userId}/hubspot_tokens/{tokenId} {
    allow read, write: if request.auth != null && request.auth.uid == userId;
}
```

## Development

### Local Development

```bash
# Start local server
npx serve public -l 3000

# Or use Firebase emulators
firebase emulators:start
```

### Branches

- `main` - Production-ready code
- `develop` - Development branch for integration

This project follows gitflow branching model.

## Troubleshooting

### "HubSpot CLI version too old"
```bash
npm install -g @hubspot/cli@latest
```

### "Firebase project not found"
```bash
firebase login --reauth
firebase projects:list
```

### "OAuth redirect_uri mismatch"
Ensure `redirectUrls` in `user-level-app-hsmeta.json` includes both:
- `http://localhost:3000/hubspot-callback` (local dev)
- `https://test.intenova.ai/hubspot-callback` (production)

### "Token exchange failed"
- Verify HubSpot Client ID and Secret are correct
- Check that the redirect URI matches exactly
- Ensure the authorization code hasn't expired (5 minutes limit)

## Resources

- [HubSpot MCP Server Documentation](https://developers.hubspot.com/mcp)
- [HubSpot OAuth Guide](https://developers.hubspot.com/docs/api/oauth/tokens)
- [User-Level App Template](https://github.com/hubspotdev/user-level-app-template)
- [Firebase Hosting Documentation](https://firebase.google.com/docs/hosting)
- [Firestore Security Rules](https://firebase.google.com/docs/firestore/security/get-started)

## License

Proprietary - Intenova / GlobalGig Inc.
