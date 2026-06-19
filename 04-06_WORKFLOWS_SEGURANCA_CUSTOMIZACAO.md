# 4️⃣-6️⃣ WIKI.JS: WORKFLOWS, SEGURANÇA & CUSTOMIZAÇÃO

---

# 4️⃣ WORKFLOWS & COLABORAÇÃO

## 📦 EXEMPLO 1: Editorial Workflow Completo

**Arquivo:** `editorial-workflow.md`

```yaml
# Editorial Process

## Stages
1. Draft (Autor escreve)
2. Review (Revisor aprova)
3. Publish (Editor publica)
4. Archive (Ao expirar)

## Draft Stage
- Autor cria página em namespace: /drafts/{author}/{title}
- Salva como rascunho
- Aviso automático: "Esta é uma página de rascunho"
- Comentários habilitados para feedback

## Review Stage
- Autor move para /in-review/{title}
- Notifica revisor via email & Slack
- Revisor pode:
  - Aprovar (move para /published)
  - Solicitar mudanças (comentário)
  - Rejeitar (volta para /drafts)

## Publish Stage
- Editor move para /docs/{category}/{title}
- Auto-commit para GitLab
- Dispara CI/CD (validação)
- Notifica time via Slack

## Archive Stage
- Páginas não atualizadas há 1 ano
- Move para /archived/{category}
- Links antigos redirecionam
```

**Configuração no Wiki.js:**

```json
{
  "workflow": {
    "enabled": true,
    "stages": [
      {
        "id": "draft",
        "name": "Draft",
        "color": "blue",
        "icon": "pencil",
        "permissions": ["author"],
        "autoNotify": false
      },
      {
        "id": "review",
        "name": "Under Review",
        "color": "orange",
        "icon": "eye",
        "permissions": ["reviewer"],
        "autoNotify": true,
        "notifyRole": "editor",
        "notifyChannel": "slack"
      },
      {
        "id": "published",
        "name": "Published",
        "color": "green",
        "icon": "check",
        "permissions": ["viewer"],
        "autoNotify": false,
        "webhookURL": "https://gitlab.example.com/api/webhook"
      }
    ]
  }
}
```

---

## 📦 EXEMPLO 2: Comments & Discussions

**Setup de Comentários:**

```javascript
// Configuração no Wiki.js Admin

comments: {
  enabled: true,
  allowGuests: false,
  autoApprove: false,
  moderationRequired: true,
  
  notifications: {
    enabled: true,
    notifyAuthor: true,
    notifyWatchers: true,
    channels: ["email", "slack"]
  },
  
  threads: {
    enabled: true,
    sorting: "newest",
    collapsing: true
  },
  
  threadingRules: {
    minLevel: "user",  // Quem pode comentar
    requireAuth: true,
    spamFilter: true
  }
}
```

**Uso em Página:**

```markdown
# API Documentation

This page is **open for discussion**. Please share your feedback!

> [!discussion]
> **What should be the default timeout?**
> 
> Currently set to 30 seconds. Should we change it?
> - Thread started by @john on 2024-01-15
> - 5 comments so far

## Endpoints

### GET /users

Get list of users.

{#endpoint-get-users}

> [!discussion]
> Example discussion on this section
```

---

## 📦 EXEMPLO 3: Multi-author Editing (Resolving Conflicts)

**Estratégia Git:**

```bash
# Wiki.js usa Git internamente para sync
# Quando múltiplos autores editam:

1. Author A edita page1.md
2. Author B edita page1.md (simultaneamente)
3. Wiki.js tenta merge automático
4. Se conflito:
   - Notifica ambos
   - Mostra diffs
   - Pede resolução manual

# Arquivo de conflito gerado:
<<<<<<< yours
## Version A
Content from Author A
=======
## Version B
Content from Author B
>>>>>>> theirs
```

**Resolução no Wiki.js:**

```javascript
// Admin → Pages → Conflicts

conflicts: {
  strategy: "collaborative",  // ou "manual" ou "latest-wins"
  
  autoMerge: {
    enabled: true,
    strategy: "three-way",  // Merge inteligente
    timeout: 300  // segundos
  },
  
  notifications: {
    onConflict: true,
    channels: ["email", "slack", "in-app"]
  },
  
  resolution: {
    requireApproval: true,
    approvers: ["editor", "author"]
  }
}
```

---

# 5️⃣ SEGURANÇA & PERMISSÕES

## 📦 EXEMPLO 4: RBAC (Role-Based Access Control)

**Arquivo:** `rbac-setup.js`

```javascript
// Wiki.js RBAC Configuration

roles: [
  {
    id: "guest",
    name: "Guest",
    permissions: [
      "pages:read",
      "search:read"
    ]
  },
  
  {
    id: "user",
    name: "User (Authenticated)",
    permissions: [
      "pages:read",
      "pages:comment",
      "search:read",
      "history:read"
    ]
  },
  
  {
    id: "author",
    name: "Author",
    permissions: [
      "pages:read",
      "pages:create",
      "pages:edit-own",
      "pages:comment",
      "files:upload",
      "search:read",
      "history:read"
    ]
  },
  
  {
    id: "reviewer",
    name: "Reviewer/Editor",
    permissions: [
      "pages:read",
      "pages:create",
      "pages:edit",
      "pages:delete",
      "pages:move",
      "pages:approve",
      "comments:moderate",
      "users:read",
      "search:read",
      "history:read"
    ]
  },
  
  {
    id: "admin",
    name: "Administrator",
    permissions: [
      "*"  // All permissions
    ]
  }
]

// Group-based RBAC
groups: [
  {
    id: "backend-team",
    name: "Backend Team",
    roles: ["author"],
    namespaces: [
      "/docs/backend/**",
      "/processes/deployment/**"
    ]
  },
  
  {
    id: "frontend-team",
    name: "Frontend Team",
    roles: ["author"],
    namespaces: [
      "/docs/frontend/**",
      "/docs/components/**"
    ]
  },
  
  {
    id: "security-team",
    name: "Security Team",
    roles: ["reviewer"],
    namespaces: [
      "/docs/security/**",
      "/docs/compliance/**"
    ]
  }
]

// Page-level Permissions
pages: [
  {
    path: "/docs/secret/internal-salaries",
    permission: "admin-only"
  },
  
  {
    path: "/docs/security/**",
    readPermission: "user",
    editPermission: "reviewer"
  },
  
  {
    path: "/drafts/**",
    readPermission: "author-only",
    editPermission: "author-only"
  }
]
```

---

## 📦 EXEMPLO 5: User Visibility & Group Management

**Arquivo:** `user-groups.yml`

```yaml
groups:
  # Public
  - id: "public-viewers"
    name: "Public Viewers"
    description: "Public website visitors"
    visibility: "public"
    members: []  # Anyone can view
    permissions:
      - pages:read
      - search:read

  # Company
  - id: "employees"
    name: "All Employees"
    description: "Internal company wiki"
    visibility: "internal"
    members: []  # Anyone logged in
    permissions:
      - pages:read
      - pages:comment
      - search:read

  # Department
  - id: "engineering"
    name: "Engineering Department"
    visibility: "internal"
    members:
      - john@company.com (owner)
      - jane@company.com (member)
      - bob@company.com (member)
    permissions:
      - pages:read
      - pages:create
      - pages:edit
      - files:upload

  # Team
  - id: "platform-team"
    name: "Platform Team"
    parent: "engineering"
    visibility: "internal"
    members:
      - john@company.com (owner)
      - alice@company.com (member)
    permissions:
      - pages:*
      - search:*
      - users:read

# User Visibility Matrix
user_visibility:
  "public-viewers": ["public"]
  "employees": ["public", "internal"]
  "engineering": ["public", "internal", "engineering"]
  "platform-team": ["public", "internal", "engineering", "confidential"]
```

---

## 📦 EXEMPLO 6: Audit Logging & Compliance

**Configuração:**

```json
{
  "audit": {
    "enabled": true,
    "logLevel": "verbose",
    "retention": 365,  // dias
    "storage": "database",
    
    "events": {
      "authentication": true,
      "pageCreate": true,
      "pageEdit": true,
      "pageDelete": true,
      "pageMove": true,
      "userCreate": true,
      "userDelete": true,
      "permissionChange": true,
      "settingChange": true,
      "fileUpload": true,
      "fileDelete": true,
      "groupCreate": true,
      "groupDelete": true
    },
    
    "export": {
      "enabled": true,
      "formats": ["json", "csv"],
      "schedule": "daily"
    },
    
    "notification": {
      "enabled": true,
      "channels": ["email", "slack"],
      "alertOn": [
        "deleteUser",
        "deleteGroup",
        "permissionEscalation",
        "massDelete"
      ]
    }
  }
}
```

**Logs de Exemplo:**

```json
{
  "timestamp": "2024-01-15T10:30:45Z",
  "event": "pageEdit",
  "userId": 123,
  "userName": "john.doe@company.com",
  "userRole": "author",
  "pageId": 456,
  "pagePath": "/docs/api/users",
  "action": "edit",
  "changes": {
    "title": {
      "before": "Users API",
      "after": "User Management API"
    },
    "content": {
      "added": 150,
      "removed": 30,
      "modified": 45
    }
  },
  "ipAddress": "192.168.1.100",
  "userAgent": "Mozilla/5.0..."
}
```

---

# 6️⃣ CUSTOMIZAÇÃO & DESIGN

## 📦 EXEMPLO 7: Theme Customization

**Arquivo:** `custom-theme.css`

```css
/* Wiki.js Custom Theme */

:root {
  /* Colors */
  --primary: #2563eb;
  --primary-dark: #1e40af;
  --secondary: #64748b;
  --success: #10b981;
  --warning: #f59e0b;
  --danger: #ef4444;
  
  /* Typography */
  --font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  --font-size-base: 16px;
  --line-height: 1.6;
  
  /* Spacing */
  --spacing-unit: 4px;
  --spacing-xs: calc(var(--spacing-unit) * 1);
  --spacing-sm: calc(var(--spacing-unit) * 2);
  --spacing-md: calc(var(--spacing-unit) * 4);
  --spacing-lg: calc(var(--spacing-unit) * 8);
  
  /* Border Radius */
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;
}

/* Header Customization */
.wiki-header {
  background: linear-gradient(135deg, var(--primary) 0%, var(--primary-dark) 100%);
  color: white;
  padding: var(--spacing-lg);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.wiki-logo {
  font-size: 24px;
  font-weight: 700;
  letter-spacing: -1px;
}

/* Navigation */
.wiki-nav {
  background: #f8fafc;
  border-right: 1px solid #e2e8f0;
  padding: var(--spacing-md);
}

.wiki-nav-item {
  padding: var(--spacing-md);
  border-radius: var(--radius-md);
  color: var(--secondary);
  transition: all 0.2s ease;
}

.wiki-nav-item:hover {
  background: #e2e8f0;
  color: var(--primary);
}

/* Content Area */
.wiki-content {
  max-width: 900px;
  margin: 0 auto;
  padding: var(--spacing-lg);
}

/* Headings */
h1, h2, h3, h4, h5, h6 {
  font-weight: 600;
  line-height: 1.3;
  margin-top: var(--spacing-lg);
  margin-bottom: var(--spacing-md);
  color: #0f172a;
}

h1 { font-size: 32px; }
h2 { font-size: 24px; }
h3 { font-size: 20px; }

/* Code Blocks */
pre {
  background: #1e293b;
  color: #e2e8f0;
  padding: var(--spacing-md);
  border-radius: var(--radius-md);
  overflow-x: auto;
  font-size: 14px;
  line-height: 1.5;
}

code {
  font-family: "Monaco", "Courier New", monospace;
}

/* Links */
a {
  color: var(--primary);
  text-decoration: none;
  border-bottom: 1px solid transparent;
  transition: all 0.2s ease;
}

a:hover {
  border-bottom-color: var(--primary);
}

/* Alerts/Callouts */
.wiki-callout {
  padding: var(--spacing-md);
  border-left: 4px solid var(--primary);
  background: #eff6ff;
  border-radius: var(--radius-md);
  margin: var(--spacing-md) 0;
}

.wiki-callout.warning {
  border-color: var(--warning);
  background: #fffbeb;
}

.wiki-callout.danger {
  border-color: var(--danger);
  background: #fef2f2;
}

/* Mobile Responsive */
@media (max-width: 768px) {
  .wiki-nav {
    display: none;
  }
  
  .wiki-content {
    padding: var(--spacing-md);
  }
  
  h1 { font-size: 24px; }
  h2 { font-size: 20px; }
  h3 { font-size: 16px; }
}
```

**Admin → Appearance → Custom CSS → Paste above**

---

## 📦 EXEMPLO 8: Logo & Branding

**Setup no Wiki.js:**

```javascript
// Admin → Branding

branding: {
  siteTitle: "ACME Corporation Wiki",
  siteDescription: "Internal Knowledge Base",
  
  logo: {
    url: "https://cdn.example.com/logo.png",
    alt: "ACME Logo",
    height: "40px",
    darkMode: "https://cdn.example.com/logo-dark.png"
  },
  
  favicon: "https://cdn.example.com/favicon.ico",
  
  colors: {
    primary: "#2563eb",
    accent: "#64748b",
    dark: "#0f172a",
    light: "#f8fafc"
  },
  
  footer: {
    companyName: "ACME Corporation",
    companyUrl: "https://acme.com",
    links: [
      {
        label: "Privacy Policy",
        url: "https://acme.com/privacy"
      },
      {
        label: "Terms of Service",
        url: "https://acme.com/terms"
      },
      {
        label: "Contact Us",
        url: "https://acme.com/contact"
      }
    ]
  },
  
  socialMedia: {
    twitter: "https://twitter.com/acme",
    github: "https://github.com/acme",
    linkedin: "https://linkedin.com/company/acme"
  }
}
```

---

## 📦 EXEMPLO 9: Extensions & Plugins

**Habilitando Extensões:**

```json
{
  "extensions": {
    "enabled": [
      "markdownItMathjax",        // Math equations
      "markdownItMermaid",        // Diagrams
      "markdownItHighlight",      // Code highlighting
      "markdownItToc",            // Table of contents
      "markdownItImplicitFigures", // Auto figure captions
      "markdownItAnchor",         // Heading anchors
      "markdownItFootnote",       // Footnotes
      "markdownItTaskLists"       // Checkboxes
    ],
    
    "markdown-it-mathjax": {
      "delimiters": "dollars"
    },
    
    "markdown-it-mermaid": {
      "theme": "default",
      "securityLevel": "loose"
    }
  }
}
```

**Usando nas Páginas:**

```markdown
# Math Example

$$
E = mc^2
$$

# Diagram Example

\`\`\`mermaid
graph TD
  A[Start] --> B[Process]
  B --> C[End]
\`\`\`

# Footnotes

This is a text[^1]

[^1]: This is the footnote

# TOC

[[toc]]
```

---

## RESUMO CATEGORIAS 4-6

✅ **Workflows & Colaboração:**
- Editorial process completo
- Comments & discussions
- Multi-author editing
- Conflict resolution

✅ **Segurança & Permissões:**
- RBAC granular
- Groups e roles
- Page-level permissions
- Audit logging completo

✅ **Customização & Design:**
- CSS customization
- Logo & branding
- Extensions & plugins
- Temas

Próximas: Content Management, Performance, Automação, Operações
