---
name: firebase-deploy
description: >
  Deploy and manage Firebase projects including Hosting (static sites & SPAs),
  Cloud Functions, Firestore security rules, Storage rules, and Realtime Database.
  Use when the user mentions Firebase, Firestore, Firebase Hosting, Firebase Auth,
  firebase deploy, firebase.json, Realtime Database, or Firebase Emulator Suite.
license: Apache-2.0
compatibility: Requires Node.js 18+, firebase-tools (`npm install -g firebase-tools`)
metadata:
  author: google-agent-skills
  version: "1.0"
  gcp-services: Firebase Hosting, Cloud Functions, Firestore, Firebase Auth, Firebase Storage
---

# Firebase Deploy

## Setup

```bash
# Install Firebase CLI
npm install -g firebase-tools

# Log in
firebase login

# Initialize project in current directory
firebase init

# Select project
firebase use my-project-id
firebase use --add   # add a new alias
```

## Firebase Hosting

### Deploy

```bash
# Build your app first (React, Next.js, Vue, etc.)
npm run build

# Deploy hosting only
firebase deploy --only hosting

# Deploy with a preview channel (shareable URL for review)
firebase hosting:channel:deploy preview-branch --expires 7d
```

### `firebase.json` for SPAs

```json
{
  "hosting": {
    "public": "build",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "rewrites": [
      {
        "source": "**",
        "destination": "/index.html"
      }
    ],
    "headers": [
      {
        "source": "**/*.@(js|css)",
        "headers": [{ "key": "Cache-Control", "value": "max-age=31536000" }]
      }
    ]
  }
}
```

### Custom Domain

```bash
firebase hosting:sites:create my-site
firebase target:apply hosting production my-site

# Then add custom domain in Firebase console or:
firebase hosting:channel:open live
```

## Cloud Functions

### Write a Function (Node.js)

```javascript
// functions/index.js
const { onRequest } = require("firebase-functions/v2/https");
const { onDocumentCreated } = require("firebase-functions/v2/firestore");
const admin = require("firebase-admin");

admin.initializeApp();

// HTTP function
exports.helloWorld = onRequest((req, res) => {
  res.json({ message: "Hello from Firebase!" });
});

// Firestore trigger — runs when a new user doc is created
exports.onUserCreated = onDocumentCreated("users/{userId}", async (event) => {
  const user = event.data.data();
  console.log("New user:", user.email);
  // Send welcome email, update stats, etc.
});
```

### Deploy Functions

```bash
# Deploy all functions
firebase deploy --only functions

# Deploy a single function
firebase deploy --only functions:helloWorld

# Set environment config
firebase functions:secrets:set API_KEY
firebase functions:config:set stripe.key="sk_live_xxx"
```

## Firestore Security Rules

```javascript
// firestore.rules
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Users can only read/write their own documents
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }

    // Posts: anyone can read, only authenticated users can write
    match /posts/{postId} {
      allow read: if true;
      allow create: if request.auth != null
        && request.resource.data.authorId == request.auth.uid;
      allow update, delete: if request.auth != null
        && resource.data.authorId == request.auth.uid;
    }

    // Admin-only collection
    match /admin/{document=**} {
      allow read, write: if request.auth.token.admin == true;
    }
  }
}
```

```bash
# Deploy rules only
firebase deploy --only firestore:rules

# Test rules before deploying
firebase emulators:start --only firestore
```

## Firebase Storage Rules

```javascript
// storage.rules
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /users/{userId}/{allPaths=**} {
      allow read: if request.auth != null;
      allow write: if request.auth != null
        && request.auth.uid == userId
        && request.resource.size < 5 * 1024 * 1024  // 5MB limit
        && request.resource.contentType.matches('image/.*');
    }
  }
}
```

## Firebase Emulator Suite (Local Dev)

```bash
# Start all emulators
firebase emulators:start

# Start specific emulators
firebase emulators:start --only auth,firestore,functions,hosting

# Import seed data
firebase emulators:start --import=./seed-data

# Export emulator data (for future imports)
firebase emulators:export ./seed-data
```

Point your app at emulators in dev:
```javascript
import { connectFirestoreEmulator, getFirestore } from "firebase/firestore";
import { connectAuthEmulator, getAuth } from "firebase/auth";

if (location.hostname === "localhost") {
  connectFirestoreEmulator(getFirestore(), "localhost", 8080);
  connectAuthEmulator(getAuth(), "http://localhost:9099");
}
```

## Full Deploy Command

```bash
# Deploy everything at once
firebase deploy

# Deploy specific targets
firebase deploy --only hosting,functions,firestore:rules,storage
```

## Useful Commands

```bash
firebase projects:list          # List all projects
firebase use                    # Show current project
firebase open hosting:site      # Open deployed site in browser
firebase functions:log          # View function logs
firebase firestore:delete /path --recursive  # Delete documents
```
