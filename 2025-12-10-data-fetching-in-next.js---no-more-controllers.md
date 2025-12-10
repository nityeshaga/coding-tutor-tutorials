---
concepts: Data Fetching,Server Component Fetching,React Query,useQuery,API Routes,Client-side Fetching
source_repo: form-flow
description: How to get data in Next.js when there's no controller layer. Covers the three main patterns - fetching directly in Server Components, using React Query in Client Components, and building API routes. Uses real form-flow examples to show when to use each approach.
understanding_score: null
last_quizzed: null
prerequisites: [2025-12-09-next.js-for-rails-developers---the-mental-model-shift.md, 2025-12-09-server-vs-client-components---the-split-that-changes-everything.md]
created: 10-12-2025
last_updated: 10-12-2025
---

# Data Fetching in Next.js: No More Controllers

In Rails, data fetching has one home: the controller.

```ruby
class FormsController < ApplicationController
  def show
    @form = Form.find(params[:id])  # Data fetching happens HERE
  end
end
```

The view receives `@form` and renders it. Clean, predictable, one place to look.

In Next.js, there's no controller layer. Data fetching is distributed across your application based on *where* and *when* you need the data. This feels chaotic at first, but there's a clear mental model once you see the three patterns.

This tutorial maps the Rails approach you know onto the three Next.js patterns, using real form-flow code.

## The Three Patterns

Here's the mental model:

```
┌─────────────────────────────────────────────────────────────────┐
│                    WHERE DO YOU NEED DATA?                       │
└─────────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
   │   SERVER    │    │   CLIENT    │    │   CLIENT    │
   │  COMPONENT  │    │  COMPONENT  │    │  COMPONENT  │
   │             │    │  (initial)  │    │ (reactive)  │
   └─────────────┘    └─────────────┘    └─────────────┘
          │                   │                   │
          ▼                   ▼                   ▼
   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
   │   fetch()   │    │   Props     │    │ React Query │
   │   directly  │    │   from      │    │  useQuery   │
   │   or Prisma │    │   Server    │    │             │
   └─────────────┘    └─────────────┘    └─────────────┘

   "I need data     "Parent Server    "I need data that
    to render        Component         updates, refreshes,
    this page"       already got it"   or responds to user"
```

Let's see each pattern in form-flow.

## Pattern 1: Fetch Directly in Server Components

This is the most Rails-like pattern. A Server Component can fetch data directly - using `fetch()`, Prisma, or any async operation - because it runs on the server.

### Example: `app/forms/[formId]/page.tsx`

```tsx
// No 'use client' - this is a Server Component
export default async function FormPage({ params }: { params: Promise<{ formId: string }> }) {
  const formId = (await params).formId

  // Fetch data directly - this runs on the SERVER
  const res = await fetch(`${process.env.NEXT_PUBLIC_APP_URL}/api/public/forms/${formId}`)
  const form = await res.json()

  if (!form || form.error) {
    notFound()
  }

  // Can also check session - server-only!
  const session = await getServerSession(authOptions)
  const isOwner = session?.user?.email === form.user?.email

  // Pass pre-fetched data to Client Component as props
  return (
    <FormResponse
      formId={formId}
      initialBlocks={draftBlocks}
      title={form.title}
      theme={form.theme}
    />
  )
}
```

**Rails equivalent:**

```ruby
class FormsController < ApplicationController
  def show
    @form = Form.find(params[:id])
    @is_owner = @form.user == current_user
    # View receives @form and @is_owner
  end
end
```

**When to use this pattern:**
- Initial page load data
- Data that doesn't need to update without a page refresh
- SEO-critical content (it's in the initial HTML)
- When you need server-side auth checks before showing data

**The key insight:** Server Components can be `async` and directly `await` data. The component doesn't render until the data is ready - just like a Rails controller action.

### Example: `app/page.tsx` (Home Page)

```tsx
export default async function Home() {
  // Server-side session check
  const session = await getServerSession(authOptions)

  if (session) {
    redirect('/dashboard')  // Server redirect, no client JS needed
  }

  return <LandingPage />
}
```

This is data fetching in its simplest form - checking session data to decide what to render.

## Pattern 2: Pass Data from Server to Client via Props

Often you fetch data in a Server Component but need interactivity in the UI. The solution: fetch on server, pass to Client Component as props.

### Example: `app/forms/[formId]/page.tsx` → `<FormResponse />`

The Server Component fetches:
```tsx
// Server Component
const form = await res.json()
const { draftBlocks } = transformApiFormToFormData(form)

return (
  <FormResponse
    formId={formId}
    initialBlocks={draftBlocks}  // ← Data passed as prop
    title={form.title}
    theme={form.theme}
  />
)
```

The Client Component receives:
```tsx
'use client'

export function FormResponse({ formId, initialBlocks, title, theme }) {
  // initialBlocks is already available - no fetch needed!
  const [currentBlock, setCurrentBlock] = useState(0)

  // Interactive logic here...
}
```

**Rails equivalent:** This is like passing `@form` to a view, but the view has JavaScript that makes it interactive.

**When to use this pattern:**
- Initial data for an interactive component
- When you want fast initial render (data is in the HTML)
- When the component needs to manipulate or display the data interactively

## Pattern 3: Client-Side Fetching with React Query

Sometimes you need data that:
- Updates based on user actions
- Needs to refresh periodically
- Is fetched after the initial page load

This is where React Query (TanStack Query) comes in. It's like having a smart caching layer for your API calls.

### Example: `app/components/FormsList.tsx`

```tsx
'use client'

import { useQuery } from '@tanstack/react-query'
import { api } from '@/lib/api'

export function FormsList() {
  const { data: forms, isLoading, error } = useQuery<Form[]>({
    queryKey: ['forms'],           // Cache key
    queryFn: () => api.getForms()  // Function that fetches data
  })

  if (isLoading) {
    return <Skeleton />  // Show loading state
  }

  if (error) {
    return <div>Failed to load forms</div>
  }

  return (
    <div>
      {forms.map((form) => (
        <FormCard key={form.id} form={form} />
      ))}
    </div>
  )
}
```

**What React Query gives you:**
- **Automatic caching**: If you navigate away and back, data is instant from cache
- **Background refetching**: Data stays fresh
- **Loading/error states**: Built-in, no manual `useState` for these
- **Deduplication**: Multiple components requesting same data = one fetch

**Rails equivalent:** There isn't a direct one. In Rails, you'd either:
- Refresh the whole page (server re-renders with fresh data)
- Write custom JavaScript to fetch and update the DOM

React Query is the "batteries included" solution for this in React-land.

### Example: `app/hooks/useFormData.ts`

This is a more sophisticated example - a custom hook that combines React Query with other state:

```tsx
'use client'

import { useQuery, useMutation } from '@tanstack/react-query'
import { api } from '@/lib/api'

export function useFormData(formId: string) {
  const [saveStatus, setSaveStatus] = useState<SaveStatus>('idle')

  // Fetch form data
  const { isLoading, data: formData } = useQuery<ApiForm>({
    queryKey: ['form', formId],  // Unique cache key per form
    queryFn: async () => {
      const data = await api.getForm(formId)
      return data
    },
    staleTime: 0,  // Always refetch when component mounts
  })

  // Setup save mutation (for updating data)
  const { mutateAsync: saveForm } = useMutation({
    mutationFn: async (data) => {
      return api.updateForm(formId, data)
    },
    onMutate: () => setSaveStatus('saving'),
    onSuccess: () => {
      setSaveStatus('saved')
      setTimeout(() => setSaveStatus('idle'), 2000)
    },
    onError: () => setSaveStatus('error'),
  })

  return {
    formData,
    isLoading,
    saveStatus,
    saveForm
  }
}
```

**What this hook does:**
1. **Fetches** form data with `useQuery`
2. **Caches** it under the key `['form', formId]`
3. **Provides** a `saveForm` function via `useMutation`
4. **Tracks** save status (saving, saved, error)

This is the pattern form-flow uses throughout - the `FormBuilder` component uses this hook:

```tsx
'use client'

export default function FormBuilder({ formId, mode }) {
  const {
    formData,
    isLoading,
    saveStatus,
    handleSaveTitle,
    handleSaveTheme,
  } = useFormData(formId)  // ← Custom hook wrapping React Query

  if (isLoading || !formData) {
    return <div>Loading...</div>
  }

  return (
    <div>
      <Header saveStatus={saveStatus} />
      {/* ... rest of form builder */}
    </div>
  )
}
```

## The API Layer: Where Fetch Actually Hits

All these patterns need somewhere to fetch *from*. That's the `lib/api.ts` file:

```tsx
export const api = {
  async getForm(formId: string): Promise<ApiForm> {
    const response = await fetch(`/api/forms/${formId}`)
    return handleResponse(response)
  },

  async getForms() {
    const response = await fetch('/api/forms')
    return handleResponse(response)
  },

  async updateForm(formId: string, data: FormData) {
    const response = await fetch(`/api/forms/${formId}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    })
    return handleResponse(response)
  },

  async createForm(data: FormData) {
    const response = await fetch('/api/forms', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    })
    return handleResponse(response)
  },
}
```

This is a thin wrapper around `fetch()`. It:
- Centralizes API URLs
- Handles response parsing
- Provides type safety

**Rails equivalent:** This is like a service object or API client class.

## The API Routes: Your "Controllers"

These API calls hit the routes in `app/api/`. Here's where the database actually gets touched:

```tsx
// app/api/forms/route.ts

import { prisma } from '@/lib/prisma'

export async function GET() {
  const session = await getServerSession(authOptions)
  if (!session?.user?.email) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }

  const forms = await prisma.form.findMany({
    where: {
      user: { email: session.user.email }
    },
    include: {
      blocks: true,
      theme: true,
      responses: true
    }
  })

  return NextResponse.json(forms)
}
```

**This IS your controller action.** It:
- Checks authentication
- Queries the database (via Prisma)
- Returns JSON

The difference from Rails: this runs as a serverless function, not a long-running process.

## Decision Tree: Which Pattern to Use?

```
Do you need this data to render the initial HTML?
│
├─ YES → Is the component interactive (needs hooks/events)?
│        │
│        ├─ NO  → Pattern 1: Fetch in Server Component
│        │
│        └─ YES → Pattern 2: Fetch in Server Component,
│                            pass to Client Component as props
│
└─ NO (data comes later, updates, or responds to user)
         │
         └─────→ Pattern 3: React Query in Client Component
```

**Examples from form-flow:**

| Page | Pattern | Why |
|------|---------|-----|
| `/` (home) | Server fetch | Just needs session check |
| `/forms/:id` (public form) | Server fetch → props | Initial blocks needed, but form is interactive |
| `/dashboard` | React Query | List updates when you create/delete forms |
| Form Builder | React Query | Real-time saving, complex state |

## The Prisma Connection

You might wonder: "Can I use Prisma directly in a Server Component?"

**Yes!** Server Components run on the server, so they have access to the database:

```tsx
// This would work in a Server Component
import { prisma } from '@/lib/prisma'

export default async function FormsPage() {
  const forms = await prisma.form.findMany()  // Direct DB access!

  return (
    <ul>
      {forms.map(form => <li key={form.id}>{form.title}</li>)}
    </ul>
  )
}
```

Form-flow doesn't do this directly in page components (it goes through API routes), but it's a valid pattern for simpler apps.

**Why use API routes instead of direct Prisma?**
- If Client Components also need the data (they can't call Prisma)
- To centralize business logic
- To have a clear API contract

## Try It Yourself

1. Open `app/components/FormsList.tsx`
2. Find the `useQuery` call - this is React Query fetching forms
3. Now open `lib/api.ts` and find `getForms()` - this is what useQuery calls
4. Now open `app/api/forms/route.ts` and find the `GET` function - this handles the request
5. Trace the data flow: Component → api.ts → API route → Prisma → Database

Do the same trace for the form builder:
1. `FormBuilder.tsx` uses `useFormData(formId)`
2. `useFormData.ts` calls `api.getForm(formId)`
3. `api.ts` fetches `/api/forms/${formId}`
4. `app/api/forms/[formId]/route.ts` handles it

## Summary

- **No controller layer** - data fetching is distributed based on where you need it
- **Pattern 1: Server Component fetch** - `async` component, direct fetch or Prisma, data in initial HTML
- **Pattern 2: Server → Client props** - Server fetches, passes to interactive Client Component
- **Pattern 3: React Query** - Client-side fetching with caching, for dynamic/reactive data
- **API routes** (`app/api/`) are your "controllers" - they handle requests and talk to the database
- **The api.ts file** is your client-side API wrapper - centralizes fetch calls

**The mental shift from Rails:** Instead of "all data goes through controllers," think "where does this component live (server/client) and when does it need data (initial/reactive)?" The answer determines the pattern.

You now have the three foundational tutorials:
1. **The Mental Model** - how Next.js is structured
2. **Server vs Client** - where code runs
3. **Data Fetching** - how to get data in each context

With these three, you can read and understand any Next.js codebase. The advanced patterns (Zustand, custom hooks, the draft/publish pattern) all build on this foundation.

---

## Q&A

[Questions and answers will be added here as the learner asks them during the tutorial]

## Quiz History

[Quiz sessions will be recorded here after the learner is quizzed on this topic]
