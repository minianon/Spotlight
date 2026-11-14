# Spotlight: Low-Level Design

For the big picture see the [High-Level Design](hld.md).

## 1. Data model (`prisma/schema.prisma`)

```mermaid
erDiagram
  USER ||--o{ WEBINAR : presents
  WEBINAR ||--o{ ATTENDANCE : has
  ATTENDEE ||--o{ ATTENDANCE : records
  USER {
    uuid id
    string clerkId
    string email
    string name
    string stripeConnectId
    bool subscription
  }
  WEBINAR {
    uuid id
    string title
    datetime startTime
    int duration
    enum webinarStatus
    enum ctaType
    string ctaLabel
    string priceId
    string couponCode
    bool lockChat
    uuid aiAgentId
  }
  ATTENDEE {
    uuid id
    string email
    string name
    enum callStatus
  }
  ATTENDANCE {
    uuid id
    uuid webinarId
    uuid attendeeId
    enum attendedType
    datetime joinedAt
    datetime leftAt
  }
```

| Enum | Values |
|---|---|
| `WebinarStatusEnum` | SCHEDULED, WAITING_ROOM, LIVE, ENDED, CANCELLED |
| `CtaTypeEnum` | BUY_NOW, BOOK_A_CALL |
| `AttendedTypeEnum` | REGISTERED, ATTENDED, ADDED_TO_CART, FOLLOW_UP, BREAKOUT_ROOM, CONVERTED |
| `CallStatusEnum` | PENDING, InProgress, COMPLETED |

## 2. Server actions

| Action | Behavior |
|---|---|
| `onAuthenticateUser()` | Reads the Clerk user; returns the existing `User` or creates one |
| `createWebinar(form)` | Validates the multi-step form (basic info, CTA, additional info) and inserts a webinar |
| `getWebinarByPresenterId`, `getWebinarById` | Dashboard and live-page queries |
| `changeWebinarStatus(id, status)` | Moves the webinar through its lifecycle |
| `registerAttendee({ webinarId, email, name })` | Finds or creates the attendee, rejects a second registration for the same webinar, records `REGISTERED` attendance |
| `getWebinarAttendance(webinarId)` | `groupBy` attendance type → pipeline columns with counts and people |

## 3. Lead pipeline (typical progression)

The pipeline page shows one column per `AttendedTypeEnum` value; leads usually move like this:

```mermaid
flowchart LR
  R[REGISTERED] --> AT[ATTENDED] --> C[ADDED_TO_CART]
  AT --> B[BREAKOUT_ROOM]
  C --> F[FOLLOW_UP]
  B --> F
  F --> CV[CONVERTED]
```

## 4. Going live

```mermaid
sequenceDiagram
  actor A as Attendee
  participant Page as /live-webinar/[id]
  participant Act as server actions
  participant SV as Stream Video
  participant SC as Stream Chat
  A->>Page: open link
  Page->>Act: getWebinarById
  alt SCHEDULED / WAITING_ROOM
    Page-->>A: countdown + register form
    A->>Act: registerAttendee
  else LIVE
    Page->>SV: join livestream call
    Page->>SC: join webinar channel (unless chat locked)
    Page-->>A: video, chat and CTA button
  end
```

## 5. Stripe toggle

`POST /api/toggle-stripe` (signed-in presenter) flips `User.stripeConnectId` between `null` and a placeholder value; it marks Stripe as connected for the UI rather than running Stripe Connect onboarding.
