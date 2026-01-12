---
name: base44-local-dev
description: Connect Claude Code with Base44 apps for local development with live data sync via REST API.
---

# Base44 Local Development Integration

## Overview

This skill helps you connect Base44 applications to Claude Code for local development with real data synchronization.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    CODE SYNC (GitHub)                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Base44 App  ◄────── GitHub Sync ──────►  Claude Code      │
│   (Production)                              (reads/edits)    │
│                                                              │
├─────────────────────────────────────────────────────────────┤
│                    DATA SYNC (REST API)                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Base44 API  ◄────── REST API ──────►  Local Dev Server    │
│   (Real Data)        (api_key auth)      (npm run dev)       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## What This Is

This is:
- Direct REST API integration between your React app and Base44
- A "Dev Mode" configuration that allows local development with real data
- Backend function modifications for profile-based authentication

## Prerequisites

1. **Base44 App with GitHub Sync**
2. **API Credentials from Base44 Dashboard:**
   - `app_id`
   - `api_key`
3. **Your Profile ID** (from Base44 data)

## Setup Steps

### 1. Environment Variables

Create `.env` in project root (add to `.gitignore`!):

```env
VITE_DEV_MODE=true
VITE_BASE44_APP_ID=your_app_id
VITE_BASE44_ACCESS_TOKEN=your_api_key
VITE_DEV_PROFILE_ID=your_profile_id
VITE_DEV_USER_EMAIL=your@email.com
VITE_BASE44_FUNCTIONS_URL=https://preview--app-xxxxx.base44.app/api/functions
```

### 2. Safe Dev Mode Check

**CRITICAL**: Use this safe check to prevent crashes in Base44 Production:

```javascript
// WRONG - crashes in Base44:
const IS_DEV = import.meta.env.VITE_DEV_MODE === 'true';

// CORRECT - safe everywhere:
const IS_DEV = typeof import.meta !== 'undefined' &&
               import.meta.env &&
               import.meta.env.VITE_DEV_MODE === 'true';
```

### 3. API Client Modification

Update your Base44 client to support Dev Mode:

```javascript
if (IS_DEV_MODE) {
    const APP_ID = import.meta.env.VITE_BASE44_APP_ID;
    const API_KEY = import.meta.env.VITE_BASE44_ACCESS_TOKEN;
    const BASE_URL = `https://app.base44.com/api/apps/${APP_ID}`;

    // Custom fetch with api_key header
    const request = async (endpoint, method = 'GET', body = null) => {
        const response = await fetch(`${BASE_URL}${endpoint}`, {
            method,
            headers: {
                'api_key': API_KEY,
                'Content-Type': 'application/json'
            },
            body: body ? JSON.stringify(body) : null
        });
        return response.json();
    };

    // Entity proxy
    const createEntityProxy = (entityName) => ({
        list: () => request(`/entities/${entityName}`),
        filter: (filters) => request(`/entities/${entityName}?${new URLSearchParams(filters)}`),
        create: (data) => request(`/entities/${entityName}`, 'POST', data),
        update: (id, data) => request(`/entities/${entityName}/${id}`, 'PUT', data),
        delete: (id) => request(`/entities/${entityName}/${id}`, 'DELETE'),
    });
}
```

### 4. Backend Functions Pattern

Update all backend functions to support both Dev Mode and Production:

```javascript
import { createClientFromRequest } from 'npm:@base44/sdk@0.7.1';

Deno.serve(async (req) => {
    const base44 = createClientFromRequest(req);
    const body = await req.json().catch(() => ({}));

    let profile_id;

    // Dev mode - accepts profile_id in request body
    if (body.profile_id) {
        profile_id = body.profile_id;
    } else {
        // Production - uses authenticated user
        const user = await base44.auth.me();
        if (!user) {
            return Response.json({ error: 'Unauthorized' }, { status: 401 });
        }
        const profiles = await base44.asServiceRole.entities.Profile.filter({
            user_email: user.email
        });
        profile_id = profiles[0].id;
    }

    // Use profile_id for all queries
    const data = await base44.asServiceRole.entities.YourEntity.filter({
        profile_id: profile_id
    });

    return Response.json({ success: true, data });
});
```

## Common Issues

| Issue | Symptom | Solution |
|-------|---------|----------|
| Infinite redirect | URL grows on each refresh | Add Dev Mode with mock user in AuthContext |
| Empty profile | No data loads | Use real email in VITE_DEV_USER_EMAIL |
| Functions fail | "Unauthorized" error | Send profile_id in request body |
| Production crash | "useAuth must be used within..." | Use safe typeof check for import.meta |

## Security Notes

1. **Never commit `.env`** - Keep it in `.gitignore`
2. **Safe environment checks** - Prevent crashes in Production
3. **Backwards compatibility** - Functions work with both auth methods
4. **Profile isolation** - Always filter by profile_id

## Full Guide

See the complete guide with all prompts and troubleshooting:
https://lior6221.github.io/base44-claude-code-guide/

---

**Key Insight:** Give Claude Code one API example - it will understand the pattern for all entities.
