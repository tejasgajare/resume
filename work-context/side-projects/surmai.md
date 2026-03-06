# Surmai — Travel Planning PWA

> **Type**: Personal / Side Project
> **Stack**: Go/PocketBase (backend), React/TypeScript (frontend), PWA
> **Repo**: Private

---

## Overview

I built Surmai as a full-stack travel planning application. The name comes from the Surmai fish (King Mackerel), common in Indian coastal cuisine — a nod to my travel style of exploring places through their food.

Surmai is a Progressive Web App (PWA) that lets users plan trips collaboratively: create itineraries, organize activities by day, track reservations, and share plans with travel companions. The backend is Go with PocketBase, and the frontend is React/TypeScript.

---

## Architecture

### Full Stack Diagram

```
┌──────────────────────────────────────┐
│   React/TypeScript Frontend (PWA)    │
│   ├── Trip Dashboard                 │
│   ├── Itinerary Builder              │
│   ├── Activity Cards                 │
│   └── Sharing / Collaboration        │
├──────────────────────────────────────┤
│          PocketBase API              │
│   ├── Auth (email/password, OAuth)   │
│   ├── Collections (REST + Realtime)  │
│   └── File Storage                   │
├──────────────────────────────────────┤
│         Go / PocketBase              │
│   ├── Custom Endpoints               │
│   ├── Hooks & Event Handlers         │
│   └── SQLite (embedded)              │
└──────────────────────────────────────┘
```

### Why PocketBase?

I chose PocketBase for several reasons:
1. **Single binary**: Go binary with embedded SQLite — no external dependencies to deploy
2. **Instant CRUD**: Define collections and get REST + realtime APIs for free
3. **Auth built-in**: Email/password and OAuth providers out of the box
4. **Extensibility**: Custom Go routes and hooks for business logic
5. **Admin UI**: Built-in admin dashboard for data management

PocketBase is essentially "Firebase that you self-host as a single Go binary." For a solo project, this drastically reduced boilerplate.

### Data Model

```
Trip
├── title, description, dates
├── owner (User)
├── collaborators [User]
└── itinerary_days
    └── Day
        ├── date, notes
        └── activities
            └── Activity
                ├── title, description
                ├── time, duration
                ├── location (lat/lng, address)
                ├── category (food, transport, sightseeing, ...)
                ├── reservations (confirmation #, etc.)
                └── attachments (photos, PDFs)
```

---

## Backend — Go / PocketBase

### Custom Endpoints

While PocketBase handles CRUD automatically, I added custom Go endpoints for:

- **Trip cloning**: Deep-copy a trip template with all days and activities
- **Itinerary optimization**: Reorder activities by proximity (basic TSP heuristic)
- **Export**: Generate PDF itinerary for offline use
- **Sharing**: Generate shareable links with configurable permissions (view/edit)

### Hooks and Event Handlers

PocketBase hooks let me intercept database operations:

```go
app.OnRecordBeforeCreateRequest("activities").Add(func(e *core.RecordCreateEvent) error {
    // Validate time doesn't overlap with existing activities on same day
    // Set default duration if not specified
    return nil
})

app.OnRecordAfterCreateRequest("trips").Add(func(e *core.RecordCreateEvent) error {
    // Create default itinerary days based on trip date range
    return nil
})
```

### Real-Time Collaboration

PocketBase provides real-time subscriptions via SSE (Server-Sent Events). I use this for:
- Live itinerary updates when collaborators make changes
- Activity status changes (confirmed, cancelled)
- Chat-like comments on activities

[KEY_POINTS]
- PocketBase = Firebase alternative as a single Go binary
- Real-time collaboration via SSE subscriptions
- Custom Go hooks for business logic (validation, auto-creation)
- SQLite embedded — zero external dependencies
[/KEY_POINTS]

---

## Frontend — React / TypeScript PWA

### Stack
- **React 18** with TypeScript
- **Vite** for build tooling
- **TanStack Query** for server state management
- **Tailwind CSS** for styling
- **PWA**: Service worker for offline support, installable

### PWA Features

The PWA aspect is critical for a travel app — users need access to their itinerary even without internet:

1. **Service Worker**: Caches API responses for offline viewing
2. **Installable**: Can be added to home screen on mobile
3. **Background Sync**: Queue changes made offline, sync when connectivity returns
4. **Push Notifications**: Reminders for upcoming activities (via PocketBase hooks)

### Component Architecture

```
App
├── TripDashboard
│   ├── TripCard (summary view)
│   └── CreateTripModal
├── TripDetail
│   ├── ItineraryTimeline
│   │   ├── DaySection
│   │   │   └── ActivityCard
│   │   │       ├── ActivityActions (edit, delete, move)
│   │   │       └── ReservationBadge
│   │   └── AddActivityButton
│   ├── TripMap (all activities plotted)
│   └── CollaboratorList
├── ActivityDetail
│   ├── ActivityForm
│   ├── LocationPicker
│   └── AttachmentGallery
└── Settings
    ├── ProfileEditor
    └── NotificationPreferences
```

### Drag-and-Drop Itinerary

I implemented drag-and-drop for reordering activities within a day and moving activities between days. This uses `@dnd-kit/sortable` for smooth, accessible drag interactions:

- Drag within a day: reorder activities
- Drag between days: move activity to another day
- Visual indicators for drop targets
- Optimistic updates with rollback on API failure

[COMMON_MISTAKES]
- Not handling offline-to-online sync conflicts: Two collaborators edit the same activity offline → conflict on sync. I use last-write-wins with conflict notification.
- PWA caching too aggressively: Stale itinerary data shown after collaborator makes changes. Service worker uses network-first strategy for API calls.
- PocketBase realtime subscriptions need reconnection logic: SSE connections drop on mobile when the app is backgrounded.
[/COMMON_MISTAKES]

---

## Deployment

### Single Binary Deployment

```bash
# Build
go build -o surmai ./cmd/surmai

# Deploy — that's it. Single binary includes:
# - Go HTTP server
# - PocketBase admin UI
# - SQLite database
# - Static frontend files (embedded via go:embed)
./surmai serve --http=0.0.0.0:8090
```

The entire application — backend, database, admin UI, and frontend — deploys as a single Go binary. Frontend assets are embedded using Go's `embed` package.

---

## Interview Talking Points

**"Tell me about a full-stack project."**
Surmai is a travel planning PWA I built with Go/PocketBase on the backend and React/TypeScript on the frontend. The interesting architectural decision was using PocketBase — it gave me instant CRUD APIs, auth, and real-time subscriptions as a single Go binary with embedded SQLite. The frontend is a PWA with offline support, critical for a travel app.

**"How do you handle offline support?"**
Service worker with network-first strategy for API calls, falling back to cached responses when offline. Changes made offline queue in IndexedDB and sync when connectivity returns. For conflicts, I use last-write-wins with notification to the user.

**"How do you choose your tech stack?"**
For Surmai, I optimized for deployment simplicity and development speed. PocketBase eliminated the need for a separate database server, auth service, and admin interface. The tradeoff is less flexibility than a custom Go server, but for a solo project, the productivity gain was massive.

[FOLLOW_UP]
- How would you scale this beyond a single PocketBase instance?
- What's the tradeoff between PocketBase and a custom Go API?
- How do you handle real-time sync with multiple collaborators editing simultaneously?
[/FOLLOW_UP]

---

## Cross-References

- See [Side Projects → ClipStash](clipstash.md) for another project with SQLite (via GRDB)
- See [Side Projects → Kairo](kairo.md) for another project with TypeScript/React frontend (Tauri port)
- See [Nimble Download → Global Trade Compliance](../nimble-download/global-trade-compliance.md) for Go backend experience at scale

---

*Last updated: Auto-generated from work-context-refresh skill*
