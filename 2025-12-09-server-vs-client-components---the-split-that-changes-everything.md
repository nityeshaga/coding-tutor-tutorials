---
concepts: Server Components,Client Components,use client,React Server Components,Hydration
source_repo: form-flow
description: The concept that has no Rails equivalent - understanding how Next.js splits components between server and client execution. Covers why the split exists, how to identify which is which, the 'use client' directive, and the mental model for deciding where code should run.
understanding_score: null
last_quizzed: null
prerequisites: [2025-12-09-next.js-for-rails-developers---the-mental-model-shift.md]
created: 09-12-2025
last_updated: 09-12-2025
---

# Server vs Client Components: The Split That Changes Everything

This is the concept that will feel the most alien coming from Rails. There's simply nothing equivalent in the Rails world, because Rails made a different architectural choice decades ago.

In Rails, there's a bright line between server and client:
- **Server**: Ruby code runs, renders HTML, sends it to the browser
- **Client**: JavaScript runs in the browser, maybe enhances the HTML

These are different languages, different runtimes, different files. You never confuse which is which.

In Next.js with React Server Components, **the same language (JavaScript/TypeScript) runs in both places, and a single file can contain code that runs in EITHER place - or both**. The framework decides at build time which code runs where.

This sounds confusing because it IS confusing until you build the right mental model. Let's build it.

## Why Does This Split Even Exist?

Before we dive into mechanics, let's understand the *why*. What problem is this solving?

**The Traditional SPA Problem:**

When React first became popular, the standard approach was a Single Page Application (SPA):
1. Server sends a nearly-empty HTML page with a JavaScript bundle
2. Browser downloads the JavaScript (could be several MB)
3. JavaScript executes and renders the entire UI
4. User finally sees something

This meant: slow initial loads, no content until JavaScript runs, bad for SEO, huge bundle sizes.

**The Traditional Server Rendering Problem:**

The alternative was server-side rendering (what Rails does):
1. Server renders full HTML
2. Browser displays it immediately
3. If you want interactivity, you need to write separate JavaScript

This meant: fast initial load, good SEO, but interactivity requires duplicating logic between server and client.

**Server Components: The Best of Both:**

React Server Components (RSC) try to give you both:
- **Server Components**: Render on the server, send HTML, never ship their code to the browser
- **Client Components**: Ship to the browser, handle interactivity

The key insight: **most of your UI doesn't need to be interactive**. A blog post, a product listing, a form label - these just display data. Why ship JavaScript for them?

Server Components let you write React that renders to HTML on the server and STAYS there. Only the interactive bits ship JavaScript to the browser.

## The Mental Model: Two Worlds

Think of it like this:

```
┌─────────────────────────────────────────────────────────────┐
│                         SERVER                               │
│                                                              │
│   Server Components run here                                 │
│   - Can access database directly                             │
│   - Can read files from disk                                 │
│   - Can use server-only secrets                              │
│   - Their code NEVER goes to the browser                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              │  (sends HTML + minimal JS)
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                         BROWSER                              │
│                                                              │
│   Client Components run here                                 │
│   - Can use useState, useEffect                              │
│   - Can handle clicks, typing, user events                   │
│   - Can access browser APIs (localStorage, etc.)             │
│   - Their code IS shipped to the browser                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## How Do You Know Which Is Which?

Here's the simple rule:

**By default, everything is a Server Component.**

If you want a Client Component, you put `'use client'` at the top of the file. That's it.

Let's look at form-flow:

### Server Component: `app/page.tsx`

```tsx
// No 'use client' directive - this is a Server Component
import { getServerSession } from 'next-auth'
import { redirect } from 'next/navigation'
import { authOptions } from './api/auth/[...nextauth]/options'
import { LandingPage } from './components/LandingPage'

export default async function Home() {
  // This runs on the SERVER
  const session = await getServerSession(authOptions)

  if (session) {
    redirect('/dashboard')  // Server-side redirect
  }

  return <LandingPage />
}
```

Notice:
- No `'use client'` at the top
- The function is `async` (you can't use async in Client Components directly)
- It calls `getServerSession()` - a server-only function that reads cookies/session
- It uses `redirect()` from `next/navigation` - a server-side redirect

This component runs entirely on the server. When someone visits `/`, Next.js runs this code on the server, checks if they're logged in, redirects if they are, or renders the HTML if they're not.

### Client Component: `app/dashboard/page.tsx`

```tsx
'use client'  // <-- This makes it a Client Component

import { FormsList } from "@/app/components/FormsList"
import { UserNav } from "@/app/components/UserNav"
import { Button } from "@/components/ui/button"
import { useRouter } from "next/navigation"

export default function DashboardPage() {
  const router = useRouter()  // Client-side hook

  const handleNewForm = async () => {
    // This runs in the BROWSER when you click
    const newForm = await api.createForm({...})
    router.push(`/form/${newForm.id}`)  // Client-side navigation
  }

  return (
    <div>
      <Button onClick={handleNewForm}>+ New Form</Button>
      <FormsList />
    </div>
  )
}
```

Notice:
- `'use client'` at the very top
- Uses `useRouter()` - a hook that only works in the browser
- Has an `onClick` handler - events only exist in the browser
- This JavaScript is shipped to the user's browser

### The Hybrid: Server Component Rendering a Client Component

Here's where it gets interesting. Look at `app/page.tsx` again:

```tsx
// Server Component
export default async function Home() {
  const session = await getServerSession(authOptions)  // Server code

  if (session) {
    redirect('/dashboard')
  }

  return <LandingPage />  // <-- This is a Client Component!
}
```

The `<LandingPage />` component has `'use client'` in it:

```tsx
"use client";  // LandingPage is a Client Component

import { useQuery } from '@tanstack/react-query';

export function LandingPage() {
  const { data: stats } = useQuery({...})  // Client-side data fetching

  return (
    <div>
      {/* Interactive UI */}
    </div>
  )
}
```

**What happens here:**

1. Server runs `Home()` (Server Component)
2. Server checks session, decides to render `<LandingPage />`
3. Server renders the initial HTML for LandingPage
4. Server sends HTML to browser PLUS the LandingPage JavaScript code
5. Browser displays HTML immediately
6. Browser "hydrates" LandingPage - attaches event handlers, runs useQuery, makes it interactive

This is the magic: **Server Components can render Client Components as children**. The server part runs first, then hands off to the client.

## Another Real Example: `app/forms/[formId]/page.tsx`

This is a Server Component that does real server work:

```tsx
// No 'use client' - Server Component
import { getServerSession } from 'next-auth'

export default async function FormPage({ params }: { params: Promise<{ formId: string }> }) {
  const formId = (await params).formId

  // Fetch data on the SERVER
  const res = await fetch(`${process.env.NEXT_PUBLIC_APP_URL}/api/public/forms/${formId}`)
  const form = await res.json()

  if (!form || form.error) {
    notFound()  // Server-side 404
  }

  // Check ownership on the SERVER (has access to session)
  const session = await getServerSession(authOptions)
  const isOwner = session?.user?.email === form.user?.email

  return (
    <>
      {isOwner && hasUnpublishedChanges && <UnpublishedChangesBanner />}
      <FormResponse
        formId={formId}
        initialBlocks={draftBlocks}
        title={form.title}
        theme={form.theme}
      />
    </>
  )
}
```

**Why is this a Server Component?**

1. It fetches form data - no need to ship fetching code to browser
2. It checks session/ownership - security logic stays on server
3. It passes already-fetched data to `<FormResponse />` - the client doesn't need to re-fetch

The `<FormResponse />` component (a Client Component) receives the data as props. It doesn't need to know where the data came from - it just renders and handles interactions.

## What Can Each Type Do?

Here's your reference table:

| Capability | Server Component | Client Component |
|-----------|-----------------|------------------|
| Render on server | Yes | Yes (initial render) |
| `async`/`await` at component level | Yes | No |
| Access database directly | Yes | No |
| Access filesystem | Yes | No |
| Use server secrets (env vars) | Yes | No |
| `useState`, `useEffect` | No | Yes |
| Event handlers (`onClick`, etc.) | No | Yes |
| Browser APIs (`localStorage`, etc.) | No | Yes |
| Custom hooks | No | Yes |
| Render Client Components | Yes | Yes |
| Render Server Components | Yes | No (can't import) |
| Code shipped to browser | No | Yes |

## The Decision Framework

When you're writing a component, ask:

**Does this component need interactivity?**
- Click handlers?
- Form inputs with state?
- Animations on user action?
- Hooks like `useState`, `useEffect`, `useQuery`?

If YES → **Client Component** (`'use client'`)

If NO → **Server Component** (default, no directive needed)

**The form-flow pattern:**

Looking at the codebase, you can see a pattern:
- **Pages that just coordinate** (`app/forms/[formId]/page.tsx`) → Server Component
- **Pages with interactivity** (`app/dashboard/page.tsx`) → Client Component
- **Complex interactive UI** (`FormBuilder`, `FormsList`) → Client Component
- **Layout wrapper** (`app/layout.tsx`) → Server Component (but wraps client Providers)

## The Providers Pattern

Look at `app/layout.tsx`:

```tsx
// Server Component (no 'use client')
export default async function RootLayout({ children }) {
  const session = await getServerSession(authOptions)  // Server!

  return (
    <html lang="en">
      <body>
        <Providers session={session}>  {/* Client Component! */}
          {children}
        </Providers>
      </body>
    </html>
  )
}
```

And `app/providers.tsx`:

```tsx
'use client'  // Client Component

import { QueryClientProvider } from '@tanstack/react-query'
import { SessionProvider } from "next-auth/react"

export function Providers({ children, session }) {
  return (
    <QueryClientProvider client={queryClient}>
      <SessionProvider session={session}>
        {children}
      </SessionProvider>
    </QueryClientProvider>
  )
}
```

**Why this pattern?**

React Query and NextAuth's session provider need to be Client Components (they use context and hooks). But the root layout wants to be a Server Component (to fetch the session on the server).

Solution: The Server Component fetches data and passes it as props to a Client Component wrapper. The Client Component provides the context. The children (which can be Server or Client Components) use that context.

This is the **"donut" pattern**: Server on the outside, Client in the middle, Server/Client mix on the inside.

## Common Mistakes

### Mistake 1: Adding `'use client'` everywhere

Don't do this:
```tsx
'use client'  // UNNECESSARY

export function BlogPost({ content }) {
  return <article>{content}</article>
}
```

This component has no interactivity. Making it a Client Component means shipping unnecessary JavaScript.

### Mistake 2: Trying to use hooks in Server Components

This will error:
```tsx
// No 'use client' - Server Component
import { useState } from 'react'

export function Counter() {
  const [count, setCount] = useState(0)  // ERROR!
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>
}
```

`useState` only exists in the browser. Add `'use client'`.

### Mistake 3: Importing a Server Component into a Client Component

```tsx
'use client'

import { ServerOnlyThing } from './ServerOnlyThing'  // ERROR!
```

Once you're in Client Component land, you can only import other Client Components. Server Components can't be imported into Client Components.

You CAN pass Server Components as children though:
```tsx
// This works!
<ClientComponent>
  <ServerComponent />  {/* Passed as children prop */}
</ClientComponent>
```

## Try It Yourself

1. Open `app/dashboard/page.tsx` - it's a Client Component
2. Try removing the `'use client'` directive
3. Run the app - you'll get an error because `useRouter()` only works in Client Components
4. Now look at `app/page.tsx` - it's a Server Component
5. Notice it uses `async/await` directly in the component - you can't do this in Client Components

This hands-on experience will cement the difference better than any explanation.

## Summary

- **Server Components are the default** - no directive needed
- **Client Components need `'use client'`** at the top of the file
- **Server Components**: Can be async, access database, use server secrets, but NO hooks or event handlers
- **Client Components**: Can use hooks, handle events, but their code ships to the browser
- **Server Components can render Client Components** - this is the common pattern
- **Client Components cannot import Server Components** - but can receive them as children
- **The decision question**: Does it need interactivity? If yes, Client Component. If no, Server Component.

The mental shift from Rails: Stop thinking "server code goes here, client code goes there." Start thinking "what does this component NEED to do?" The execution environment follows from the requirements.

Next up: **Data Fetching in Next.js** - now that you understand where code runs, we'll cover how to actually get data in both worlds.

---

## Q&A

[Questions and answers will be added here as the learner asks them during the tutorial]

## Quiz History

[Quiz sessions will be recorded here after the learner is quizzed on this topic]
