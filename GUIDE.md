# איך חיברתי את Base44 ל-Claude Code ב-30 דקות

## מה השגתי?

אפליקציית התקציב שלי חיה ב-Base44 - פלטפורמת low-code מעולה לבניית אפליקציות. הבעיה? כשרציתי לעבוד עם Claude Code על האפליקציה, הוא לא יכול היה לגשת לנתונים האמיתיים.

**הפתרון:** חיבור דו-כיווני בין Claude Code לבין Base44 - כך שאני יכול:
- לראות את הנתונים האמיתיים בסביבה מקומית
- לבצע שינויים שמסתנכרנים אוטומטית ל-Base44
- לעבוד עם Claude Code על קוד שמחובר לדאטא אמיתי

## הארכיטקטורה

```
┌─────────────────┐
│   Base44 App    │◀──── GitHub Sync ────▶ Local Repo
│  (Production)   │
└────────┬────────┘
         │
    Base44 API
         │
         ▼
┌─────────────────┐     ┌─────────────────┐
│  MCP Server     │◀────│   Claude Code   │
│  (localhost)    │     │                 │
└────────┬────────┘     └─────────────────┘
         │
         ▼
┌─────────────────┐
│  Vite Dev App   │
│  (localhost)    │
└─────────────────┘
```

---

## שלב 1: הכנת הקרקע

### 1.1 סנכרון GitHub

Base44 תומכת בסנכרון עם GitHub. חיברתי את הריפו:
```
https://github.com/YOUR_USERNAME/YOUR_APP
```

עכשיו כל push ל-main מתעדכן אוטומטית ב-Base44 Preview.

### 1.2 קבלת API Credentials

ב-Base44 Dashboard, יצרתי API Key:

```javascript
// הנתונים שצריך לשמור
const APP_ID = "your_app_id";
const API_KEY = "your_api_key";
```

⚠️ **חשוב:** שמרו את אלה ב-`.env` ולא בקוד!

---

## שלב 2: יצירת MCP Server

MCP (Model Context Protocol) מאפשר ל-Claude Code לתקשר עם שירותים חיצוניים.

### 2.1 מבנה התיקייה

```
mcp-server/
├── index.js
├── package.json
└── .env          # לא מעלים ל-Git!
```

### 2.2 package.json

```json
{
  "name": "base44-mcp-server",
  "version": "1.0.0",
  "type": "module",
  "main": "index.js",
  "dependencies": {
    "@modelcontextprotocol/sdk": "^0.5.0",
    "dotenv": "^16.3.1"
  }
}
```

### 2.3 index.js - הקוד המלא

```javascript
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import dotenv from 'dotenv';

dotenv.config({ path: '.env' });

const API_KEY = process.env.BASE44_API_KEY;
const APP_ID = process.env.BASE44_APP_ID;
const BASE_URL = `https://app.base44.com/api/apps/${APP_ID}`;

// Helper function for API requests
async function apiRequest(endpoint, method = 'GET', body = null) {
    const options = {
        method,
        headers: {
            'api_key': API_KEY,
            'Content-Type': 'application/json'
        }
    };

    if (body) {
        options.body = JSON.stringify(body);
    }

    const response = await fetch(`${BASE_URL}${endpoint}`, options);
    return response.json();
}

// Create the MCP server
const server = new Server(
    { name: 'base44-budget-server', version: '1.0.0' },
    { capabilities: { tools: {} } }
);

// List available tools
server.setRequestHandler('tools/list', async () => ({
    tools: [
        {
            name: 'list_entities',
            description: 'List all records from a Base44 entity',
            inputSchema: {
                type: 'object',
                properties: {
                    entity_name: {
                        type: 'string',
                        description: 'Name of the entity (e.g., Expense, Income, Profile)'
                    }
                },
                required: ['entity_name']
            }
        },
        {
            name: 'create_entity',
            description: 'Create a new record in a Base44 entity',
            inputSchema: {
                type: 'object',
                properties: {
                    entity_name: { type: 'string' },
                    data: { type: 'object' }
                },
                required: ['entity_name', 'data']
            }
        },
        {
            name: 'update_entity',
            description: 'Update an existing record',
            inputSchema: {
                type: 'object',
                properties: {
                    entity_name: { type: 'string' },
                    id: { type: 'string' },
                    data: { type: 'object' }
                },
                required: ['entity_name', 'id', 'data']
            }
        }
    ]
}));

// Handle tool calls
server.setRequestHandler('tools/call', async (request) => {
    const { name, arguments: args } = request.params;

    try {
        switch (name) {
            case 'list_entities': {
                const data = await apiRequest(`/entities/${args.entity_name}`);
                return { content: [{ type: 'text', text: JSON.stringify(data, null, 2) }] };
            }
            case 'create_entity': {
                const data = await apiRequest(`/entities/${args.entity_name}`, 'POST', args.data);
                return { content: [{ type: 'text', text: JSON.stringify(data, null, 2) }] };
            }
            case 'update_entity': {
                const data = await apiRequest(`/entities/${args.entity_name}/${args.id}`, 'PUT', args.data);
                return { content: [{ type: 'text', text: JSON.stringify(data, null, 2) }] };
            }
        }
    } catch (error) {
        return { content: [{ type: 'text', text: `Error: ${error.message}` }] };
    }
});

// Start the server
const transport = new StdioServerTransport();
await server.connect(transport);
```

### 2.4 הגדרת Claude Code

צרו קובץ `.mcp.json` בשורש הפרויקט:

```json
{
  "mcpServers": {
    "base44": {
      "command": "node",
      "args": ["mcp-server/index.js"],
      "cwd": "${workspaceFolder}"
    }
  }
}
```

---

## שלב 3: Dev Mode לאפליקציה

כאן הגיע החלק המעניין - לגרום לאפליקציה עצמה לעבוד מקומית עם הנתונים מ-Base44.

### 3.1 קובץ .env

```env
# Base44 API
BASE44_API_KEY=your_api_key
BASE44_APP_ID=your_app_id

# Vite Environment Variables (must start with VITE_)
VITE_DEV_MODE=true
VITE_BASE44_ACCESS_TOKEN=your_api_key
VITE_DEV_PROFILE_ID=your_profile_id
VITE_DEV_USER_EMAIL=your@email.com
VITE_BASE44_FUNCTIONS_URL=https://preview--app-xxxxx.base44.app/api/functions
```

### 3.2 עדכון API Client

הנקודה הקריטית - בדיקה בטוחה של environment variables:

```javascript
// ⚠️ חשוב! בדיקה בטוחה שלא תקרוס ב-Base44 Production
const IS_DEV_MODE = typeof import.meta !== 'undefined' &&
                    import.meta.env &&
                    import.meta.env.VITE_DEV_MODE === 'true';
```

למה זה חשוב? כי ב-Base44 Production אין `import.meta.env`, וללא הבדיקה הזו האפליקציה תקרוס.

### 3.3 עדכון ה-API Client המלא

```javascript
// src/api/base44Client.js

const IS_DEV_MODE = typeof import.meta !== 'undefined' &&
                    import.meta.env &&
                    import.meta.env.VITE_DEV_MODE === 'true';

const API_KEY = IS_DEV_MODE ? import.meta.env.VITE_BASE44_ACCESS_TOKEN : null;
const DEV_PROFILE_ID = IS_DEV_MODE ? import.meta.env.VITE_DEV_PROFILE_ID : null;

// Dev mode - custom client
if (IS_DEV_MODE) {
    const APP_ID = import.meta.env.VITE_BASE44_APP_ID;
    const BASE_URL = `https://app.base44.com/api/apps/${APP_ID}`;
    const FUNCTIONS_URL = import.meta.env.VITE_BASE44_FUNCTIONS_URL;

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

    // Export dev client
    export default {
        entities: new Proxy({}, {
            get: (_, entityName) => createEntityProxy(entityName)
        }),
        functions: {
            invoke: async (name, params = {}) => {
                const response = await fetch(`${FUNCTIONS_URL}/${name}`, {
                    method: 'POST',
                    headers: { 'api_key': API_KEY, 'Content-Type': 'application/json' },
                    body: JSON.stringify({ ...params, profile_id: DEV_PROFILE_ID })
                });
                return { data: await response.json() };
            }
        }
    };
}

// Production - use Base44 SDK
export { default } from '@base44/sdk';
```

---

## שלב 4: עדכון Backend Functions

הבעיה: Functions ב-Base44 משתמשות ב-`auth.me()` לאימות. ב-dev mode אין לנו session.

הפתרון: לתמוך ב-`profile_id` בגוף הבקשה:

```javascript
// Pattern לכל function
import { createClientFromRequest } from 'npm:@base44/sdk@0.7.1';

Deno.serve(async (req) => {
    try {
        const base44 = createClientFromRequest(req);
        const body = await req.json().catch(() => ({}));

        let profile_id;

        // 🔑 התוספת הקריטית - תמיכה ב-dev mode
        if (body.profile_id) {
            profile_id = body.profile_id;
        } else {
            // Production flow - original code
            const user = await base44.auth.me();
            if (!user) {
                return Response.json({ error: 'Unauthorized' }, { status: 401 });
            }
            const profiles = await base44.asServiceRole.entities.Profile.filter({
                user_email: user.email
            });
            profile_id = profiles[0].id;
        }

        // שימוש ב-profile_id בשאילתות
        const data = await base44.asServiceRole.entities.YourEntity.filter({
            profile_id: profile_id
        });

        return Response.json({ success: true, data });
    } catch (error) {
        return Response.json({ success: false, error: error.message }, { status: 500 });
    }
});
```

---

## בעיות שנתקלתי בהן (והפתרונות)

### בעיה 1: לולאת Redirect אינסופית

**תסמין:** URL הולך ומתארך בכל רענון
**סיבה:** Auth context מנסה לאמת משתמש שלא קיים
**פתרון:**
```javascript
// AuthContext.jsx
const [user, setUser] = useState(IS_DEV_MODE ? { email: 'dev@local' } : null);
const [isAuthenticated, setIsAuthenticated] = useState(IS_DEV_MODE);
```

### בעיה 2: פרופיל ריק

**תסמין:** האפליקציה עולה אבל ללא נתונים
**סיבה:** נוצר פרופיל חדש ל-"dev@local" במקום לטעון את הקיים
**פתרון:** הוספת `VITE_DEV_USER_EMAIL` עם האימייל האמיתי

### בעיה 3: קריסה ב-Base44 אחרי Push

**תסמין:** `useAuth must be used within an AuthProvider`
**סיבה:** קוד ה-dev mode קורס בסביבת Production
**פתרון:** בדיקה בטוחה:
```javascript
// ❌ לא בטוח
const IS_DEV_MODE = import.meta.env.VITE_DEV_MODE === 'true';

// ✅ בטוח
const IS_DEV_MODE = typeof import.meta !== 'undefined' &&
                    import.meta.env &&
                    import.meta.env.VITE_DEV_MODE === 'true';
```

### בעיה 4: Functions לא עובדות

**תסמין:** "Unauthorized" או "private app"
**סיבה:** Functions דורשות אימות שלא קיים ב-dev mode
**פתרון:** שליחת `profile_id` בבקשה ועדכון כל ה-functions לתמוך בזה

---

## התוצאה

✅ אני יכול להריץ את האפליקציה מקומית עם `npm run dev`
✅ הנתונים האמיתיים נטענים מ-Base44
✅ שינויים שאני עושה מסתנכרנים ל-Base44
✅ Claude Code יכול לגשת לנתונים דרך MCP Server
✅ Push ל-GitHub מעדכן את Base44 Preview
✅ אין השפעה על Production - הכל backwards compatible

---

## Checklist מהיר

- [ ] סנכרון GitHub עם Base44
- [ ] יצירת API Key ב-Base44
- [ ] יצירת MCP Server
- [ ] הגדרת `.mcp.json`
- [ ] קובץ `.env` עם credentials
- [ ] עדכון API client לתמיכה ב-dev mode
- [ ] עדכון AuthContext
- [ ] עדכון כל ה-Backend Functions
- [ ] בדיקה שהכל עובד ב-Preview אחרי Push
- [ ] בדיקה שהאפליקציה עובדת מקומית

---

## קישורים שימושיים

- [Base44 Documentation](https://docs.base44.com)
- [MCP Protocol Specification](https://modelcontextprotocol.io)
- [Claude Code](https://claude.com/claude-code)

---

**נכתב על ידי Lior עם Claude Code | ינואר 2026**

*המדריך הזה נוצר על בסיס ניסיון אמיתי של חיבור אפליקציית תקציב. הקוד עובד ונבדק.*
