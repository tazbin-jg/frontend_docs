# 👋 JustGo Frontend — Onboarding Guide

Welcome to the JustGo Frontend team! This guide will get you up and running, and walk you through creating your first widget. No scary internals — just the things you need to know to start contributing.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Clone & Rename the Repo](#2-clone--rename-the-repo)
3. [Install Dependencies](#3-install-dependencies)
4. [Environment Setup](#4-environment-setup)
5. [Run the App Locally](#5-run-the-app-locally)
6. [Project Structure at a Glance](#6-project-structure-at-a-glance)
7. [Creating a New Widget — Step by Step](#7-creating-a-new-widget--step-by-step)
8. [Using Planet Icons](#8-using-planet-icons)
9. [Useful Path Aliases](#9-useful-path-aliases)
10. [Common Scripts](#10-common-scripts)

---

## 1. Prerequisites

Before you start, make sure you have the following installed on your machine:

| Tool | Minimum Version | Notes |
|------|----------------|-------|
| **Node.js** | v20 LTS or higher | Download from [nodejs.org](https://nodejs.org) |
| **Yarn** | v1.x | Run `npm install -g yarn` after Node is installed |
| **Git** | Any recent version | To clone the repo |

> **Tip:** To check your Node version, run `node -v` in your terminal. If it shows anything older than v20, please update before continuing.

---

## 2. Clone & Rename the Repo

### Clone

```bash
git clone https://azolvedevops@dev.azure.com/azolvedevops/JustGo/_git/React%20Development%20V2
```

### ⚠️ Important: Rename the folder

After cloning, rename the folder to something clean for your local machine. The cloned folder name contains spaces and URL encoding, which can cause issues.

**On Windows (Command Prompt):**
```cmd
ren "React Development V2" ReactDevelopmentV2
```

Or simply rename it through File Explorer.

### Switch to the main working branch

```bash
cd ReactDevelopmentV2
git checkout Dev/286_Asset
```

> **Important:** Always make sure you're on the `Dev/286_Asset` branch before starting work. This is the active main branch.

---

## 3. Install Dependencies

From inside the project folder, run:

```bash
yarn install
```

This will install all the project packages. It may take a few minutes the first time.

> **Note:** We use **Yarn** (not `npm install`). Using `npm install` may create a `package-lock.json` and cause conflicts. Stick with Yarn.

---

## 4. Environment Setup

Create a file called `.env.local` in the root of the project. This file holds your local environment variables.

The minimum variable you need to run the app locally:

```env
REACT_APP_BACKEND_URL=https://development-qa-286.justgo.com
```

> **Note:** Your team lead will share the full `.env.local` values with you. Do not commit this file to Git — it is already in `.gitignore`.

---

## 5. Run the App Locally

Once dependencies are installed and your `.env.local` is in place, start the dev server:

```bash
yarn dev
```

The app will open automatically in your browser at:

```
http://localhost:3000
```

> **Tip:** If port 3000 is already in use, the app will fail to start (strict port mode is on). Stop whatever is using port 3000 first.

---

## 6. Project Structure at a Glance

Here is a simplified overview of the folders you'll work with most:

```
src/
├── JG_V2_8/                   ← Main application code lives here
│   ├── widgets/               ← Each feature lives here as a "Widget"
│   │   ├── AssetManagement/   ← Example feature widget
│   │   ├── GettingStarted/    ← Another example widget
│   │   └── index.ts          ← Widget exports (register new widgets here)
│   │
│   ├── views/
│   │   ├── Protected/         ← Pages that need login
│   │   │   ├── AssetManagement/
│   │   │   └── index.ts      ← View exports (register new views here)
│   │   └── Public/            ← Pages that don't need login
│   │
│   ├── ViewRegister.ts        ← Maps URL routes to widget names
│   ├── JG_APP.tsx             ← Root application component
│   └── _core/                 ← Framework internals (don't touch unless advised)
│
├── comps/
│   ├── uiComps/               ← Shared UI components (buttons, inputs, etc.)
│   └── designSystem/          ← Design tokens and base styles
│
└── index.jsx                  ← App entry point
```

### How it all connects

Think of the architecture in three simple steps:

```
Route URL  →  ViewRegister.ts  →  View file (views/)  →  Widget (widgets/)
```

1. **ViewRegister.ts** — says "when the user visits `/AssetHub/`, show `AssetManagement`"
2. **View file** — a tiny connector that loads and mounts the right Widget
3. **Widget file** — where the actual feature lives (layout, pages, sub-routes)

---

## 7. Creating a New Widget — Step by Step

> **Important:** Widget creation in this project follows a **specific 4-step process**. It is different from how most React apps are structured. Please follow all 4 steps in order.

Let's say you want to create a new widget called **`MyFeature`**.

---

### Step 1 — Create the Widget Folder & File

Create a new folder inside `src/JG_V2_8/widgets/`:

```
src/JG_V2_8/widgets/MyFeature/
```

Inside that folder, create the widget file: **`MyFeatureWidget.tsx`**

```tsx
// src/JG_V2_8/widgets/MyFeature/MyFeatureWidget.tsx

import { createWidget } from 'jg-widget'
import { lazy, Suspense } from 'react'

// Lazily import your pages so they load only when needed
const MyPage = lazy(() => import('./pages/MyPage'))

const MyFeatureWidget = createWidget(({ routePath, getRootRoute }) => {
  return {
    path: routePath,
    children: [
      getRootRoute({
        element: (
          <Suspense fallback={<div>Loading...</div>}>
            <MyPage />
          </Suspense>
        ),
      }),
    ],
  }
}, 'MyFeatureWidget') // <-- Second argument is the widget name (used for debugging)

export default MyFeatureWidget
```

**What does this do?**
- `createWidget` is how we create every widget in this project
- `routePath` — where to mount this widget in the URL (provided automatically)
- `getRootRoute(...)` — sets what to show at the root path of this widget
- `lazy(...)` — makes the page load only when visited (for performance)

> **Tip:** You can add nested routes (sub-pages) by adding more children alongside `getRootRoute`. Look at `GettingStartedWidget.tsx` for an example with multiple sub-pages.

---

### Step 2 — Create Your Page Component

Inside your widget folder, create a pages directory and your first page:

```
src/JG_V2_8/widgets/MyFeature/pages/MyPage.tsx
```

```tsx
// src/JG_V2_8/widgets/MyFeature/pages/MyPage.tsx

const MyPage = () => {
  return (
    <div className="p-6">
      <h1>My Feature Page</h1>
      <p>Welcome to My Feature!</p>
    </div>
  )
}

export default MyPage
```

---

### Step 3 — Register the Widget & Create the View Connector

This is a **two-part** step that tells the app about your widget.

#### 3a. Export the widget from `widgets/index.ts`

Open `src/JG_V2_8/widgets/index.ts` and add your widget:

```ts
// Add your import at the top
import MyFeatureWidget from './MyFeature/MyFeatureWidget'

// Add your widget to the export list at the bottom
export {
  // ... existing widgets ...
  MyFeatureWidget,  // ← add this
}
```

#### 3b. Create a View Connector file

Create a new folder inside `src/JG_V2_8/views/Protected/`:

```
src/JG_V2_8/views/Protected/MyFeature/
```

Inside it, create **`MyFeature.tsx`**:

```tsx
// src/JG_V2_8/views/Protected/MyFeature/MyFeature.tsx

import { MyFeatureWidget } from '@/JG_V2_8/widgets'
import createView from 'jg-view'

export default createView(({ routePath, config, initWidget }) =>
  initWidget(MyFeatureWidget, routePath, config)
)
```

> **Note:** This small file is called a "View Connector". It's a bridge between the routing system and your widget. Every widget needs one.

#### 3c. Export from `views/Protected/index.ts`

Open `src/JG_V2_8/views/Protected/index.ts` and add your view:

```ts
// Import at the top
import MyFeature from './MyFeature/MyFeature'

// Add to the export list
export {
  // ... existing views ...
  MyFeature,  // ← add this
}
```

---

### Step 4 — Register the Route in ViewRegister.ts

Open `src/JG_V2_8/ViewRegister.ts` and add a route entry inside `RegisterViews('Protected', [...])`:

```ts
RegisterViews('Protected', [
  // ... existing routes ...

  {
    path: '/MyFeature/',       // ← The URL path in the browser
    name: 'MyFeature',         // ← Must exactly match your export name in views/Protected/index.ts
    title: 'My Feature',       // ← (Optional) Human-readable name shown in headers
    config: {},                // ← Extra options you can read inside the widget
  },
])
```

> **Important:** The `name` field in `ViewRegister.ts` **must exactly match** the name you exported from `views/Protected/index.ts`. A mismatch here means your page won't load.

---

### ✅ Summary Checklist

After following all 4 steps, you should have:

- [ ] `src/JG_V2_8/widgets/MyFeature/MyFeatureWidget.tsx` — created
- [ ] `src/JG_V2_8/widgets/MyFeature/pages/MyPage.tsx` — created
- [ ] `src/JG_V2_8/widgets/index.ts` — widget exported
- [ ] `src/JG_V2_8/views/Protected/MyFeature/MyFeature.tsx` — View Connector created
- [ ] `src/JG_V2_8/views/Protected/index.ts` — view exported
- [ ] `src/JG_V2_8/ViewRegister.ts` — route registered

Visit `http://localhost:3000/MyFeature/` after saving to see your new page! 🎉

---

## 8. Using Planet Icons

This project uses a custom icon library called **`@justgo/planet-icons`**. All icons in the app come from this package — do **not** use generic icon libraries like `react-icons` for new work.

### Setup

The package is already installed. No extra setup needed, but please read the official setup guide from your team lead:

> 📄 **Planet Icon Setup Guide:** https://docs.google.com/document/d/1ik9XOYKYjNkS828Pmf-HxxAJ3i0yOXB5B_WktwT0SJ0/edit?tab=t.0
>
> *(Requires team Google account access — ask your team lead to share access if you cannot open it.)*

### How to Use Icons in Your Code

Icons are imported from the `@justgo/planet-icons` package by name:

```tsx
import { IconSearch, IconSaveFill, IconTranslate } from '@justgo/planet-icons'

// Use in JSX like any React component
const MyComponent = () => (
  <div>
    <IconSearch size="sm" />
    <IconSaveFill size="md" className="text-jg-green-500" />
    <IconTranslate size="lg" />
  </div>
)
```

### Icon Props

| Prop | Options | Description |
|------|---------|-------------|
| `size` | `"sm"` `"md"` `"lg"` | Controls icon size |
| `className` | Any Tailwind class | For color, spacing, etc. |

### Icon Naming Pattern

All icon names follow this pattern: `Icon` + `[Name]` + `[Style]`

Examples:
- `IconSearch` — search icon
- `IconEditOutline` — edit icon, outline style
- `IconSaveFill` — save icon, filled style
- `IconInfoStyleOutline` — info icon, outline style

> **Tip:** To find available icons, search for `from '@justgo/planet-icons'` anywhere in the codebase to see how existing icons are used.

---

## 9. Useful Path Aliases

Instead of writing long relative paths, you can use these shortcuts:

| Alias | Points to | Example |
|-------|-----------|---------|
| `@/*` | `src/*` | `@/JG_V2_8/widgets` |
| `@jg/*` | `src/JG_V2_8/*` | `@jg/widgets/MyFeature` |
| `@comps/*` | `src/comps/*` | `@comps/uiComps` |
| `jg-widget` | `_core/CreateWidget.ts` | `import { createWidget } from 'jg-widget'` |
| `jg-view` | `_core/CreateView.ts` | `import createView from 'jg-view'` |

---

## 10. Common Scripts

Run these from the project root:

| Command | What it does |
|---------|-------------|
| `yarn dev` | Starts the local dev server at `http://localhost:3000` |
| `yarn build` | Creates a production build |
| `yarn lint` | Checks your code for style/quality issues |
| `yarn lint:fix` | Auto-fixes lint issues where possible |
| `yarn typecheck` | Runs TypeScript type checking without building |
| `yarn storybook` | Opens Storybook component explorer at port 6006 |

---

## Need Help?

- Talk to your team lead or a senior developer — don't guess on things you're unsure about.
- The `docs/` folder in the project root contains additional technical guides (localization, etc.).
- Look at existing simple widgets like `GettingStartedWidget.tsx` or `LanguageSettings/index.tsx` as reference examples.

Happy coding! 🚀
