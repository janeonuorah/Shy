# Shy - System Architecture Documentation

## Overview

Shy is a mobile-first application designed to connect socially anxious individuals attending the same events. This document outlines the technical architecture, technology stack, and design decisions that enable Shy to provide a low-pressure, anxiety-friendly event buddy matching experience.

---

## Tech Stack

### Frontend
- **Framework**: React Native
- **Development Tool**: Expo (for faster development and easier deployment)
- **State Management**: React Context API (lightweight for MVP)
- **Navigation**: Expo Router
- **UI Components**: React Native Paper + custom components
- **API Client**: Axios

**Why React Native?**
- Cross-platform development (iOS + Android from single codebase)
- Native mobile features essential for Shy (push notifications, location services, camera)
- Large community and extensive library ecosystem
- Faster iteration compared to native development
- Cost-effective for MVP stage

### Backend
- **Runtime**: Node.js (v18+)
- **Framework**: Express.js
- **API Architecture**: RESTful API
- **Real-time Communication**: Socket.io (for chat)
- **Authentication**: Supabase Auth
- **Database**: MongoDB
- **API Documentation**: Swagger/OpenAPI

**Why Node.js + Express?**
- JavaScript across entire stack (easier development and maintenance)
- Excellent performance for I/O operations (API calls, database queries)
- Socket.io integration for real-time chat is seamless
- Large ecosystem of npm packages
- Asynchronous by nature (handles concurrent users well)

### Database
- **Primary Database**: MongoDB Atlas (cloud-hosted)
- **Caching Layer**: Redis (for sessions and frequently accessed data)
- **File Storage**: Supabase Storage (for profile photos)

**Why MongoDB?**
- Flexible document schema (user preferences and interests vary widely)
- Native JSON support aligns perfectly with Node.js
- Excellent for storing nested data structures (conversations, user connections)
- Easy horizontal scaling as user base grows
- Fast queries for event-based matching

### Authentication & Authorization
- **Service**: Supabase Auth
- **Token Type**: JWT (JSON Web Tokens)
- **Session Management**: Redis

**Why Supabase Auth?**
- Secure, production-ready authentication out of the box
- Built-in email verification and password reset
- Social login support (Google, Apple) for future expansion
- Easy integration with React Native
- Handles token refresh automatically

### Real-Time Features
- **Chat System**: Socket.io (WebSocket protocol)
- **Presence Detection**: Socket.io rooms

**Why Socket.io?**
- Reliable real-time bidirectional communication
- Automatic fallback to polling if WebSocket unavailable
- Room-based architecture perfect for one-to-one chats
- Built-in reconnection logic
- Proven technology with strong community support

### Infrastructure & Services
- **Backend Hosting**: Railway or Render
- **Database Hosting**: MongoDB Atlas (free tier → paid as we scale)
- **File Storage**: Supabase Storage
- **Push Notifications**: Firebase Cloud Messaging (FCM)
- **Analytics**: Mixpanel (user behavior tracking)

---

## System Architecture Diagram
```
                       Mobile Apps                              
                  (React Native - iOS & Android)                  
                                                             
   ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐     
   │ Event Browse │  │ Chat Screen  │  │ Profile Management │     
   └──────────────┘  └──────────────┘  └────────────────────┘     
 ────────────┬───────────────┬────────────────┬──────────────
             │               │                │
             │ REST API      │ WebSocket      │ REST API
             │ (HTTPS)       │ (WSS)          │ (HTTPS)
             ▼               ▼                ▼
┌────────────────────────────────────────────────────────────┐
│                    API Gateway / Load Balancer             │
└────────────┬───────────────┬────────────────┬──────────────┘
             │               │                │
             ▼               ▼                ▼
┌──────────────────┐  ┌──────────────┐  ┌──────────────────┐
│  Express.js      │  │  Socket.io   │  │  Supabase Auth   │
│  REST API Server │  │  Chat Server │  │  Service         │
│                  │  │              │  │                  │
│  - User Service  │  │  - Rooms     │  │  - Login/Signup  │
│  - Event Service │  │  - Messages  │  │  - JWT Tokens    │
│  - Match Service │  │  - Presence  │  │  - Verification  │
└────────┬─────────┘  └──────┬───────┘  └──────────────────┘
         │                   │
         └──────────┬────────┘
                    │
         ┌──────────┴────────────┬─────────────┐
         ▼                       ▼             ▼
┌─────────────────┐  ┌─────────────────┐  ┌──────────────┐
│   MongoDB       │  │     Redis       │  │   Supabase   │
│   (Atlas)       │  │   (Cache)       │  │   Storage    │
│                 │  │                 │  │              │
│  - Users        │  │  - Sessions     │  │  - Profile   │
│  - Events       │  │  - Rate Limits  │  │    Photos    │
│  - Messages     │  │  - Hot Data     │  │  - Event     │
│                 │  │                 │  │    Images    │
└─────────────────┘  └─────────────────┘  └──────────────┘
         │
         ▼
┌─────────────────┐
│    Firebase     │
│       FCM       │
│ (Push Notifs)   │
└─────────────────┘
```

---

## Core Components

### 1. User Service

**Responsibilities:**
- User registration and authentication flow
- Profile management
- User preferences
- Account settings and privacy controls

**Key API Endpoints:**
```
POST   /api/auth/register        - Create new account
POST   /api/auth/login           - User login
POST   /api/auth/logout          - User logout
GET    /api/users/me             - Get current user profile
PUT    /api/users/me             - Update profile
PATCH  /api/users/comfort-level  - Update comfort level
DELETE /api/users/me             - Delete account
```

**Data Model (MongoDB):**
```javascript
User {
  _id: ObjectId
  email: String (unique, indexed)
  name: String
  age: Number
  location: {
    city: String,
    country: String
  }
  bio: String
  profilePhoto: String (URL)
  comfortLevel: Enum ['social', 'moderate', 'high-anxiety']
  preferences: {
    textBeforeMeeting: Boolean,
    smallGroups: Boolean,
    planAhead: Boolean
  }
  interests: Array<String> ['live-music', 'coffee', 'art', 'tech']
  createdAt: Date
  updatedAt: Date
  lastActive: Date
}
```

---

<!-- ### 2. Event Service

**Responsibilities:**
- Event discovery and search
- Event detail retrieval
- Tracking user attendance (who's going)
- Event creation (manual entry for MVP)

**Key API Endpoints:**
```
GET    /api/events                    - Browse/search events
GET    /api/events/:id                - Get event details
POST   /api/events                    - Create event (manual)
POST   /api/events/:id/attend         - Mark attendance
DELETE /api/events/:id/attend         - Remove attendance
GET    /api/events/:id/attendees      - List attendees
```

**Data Model (MongoDB):**
```javascript
Event {
  _id: ObjectId
  name: String
  date: Date (indexed)
  location: {
    venue: String,
    address: String,
    city: String,
    coordinates: {
      lat: Number,
      lng: Number
    }
  }
  description: String
  category: Enum ['concert', 'workshop', 'networking', 'social', 'sports']
  imageUrl: String
  attendeeCount: Number
  attendees: Array<ObjectId> (references User._id)
  createdBy: ObjectId (user who added it)
  createdAt: Date
}
```

---

### 3. Matching Service

**Responsibilities:**
- Find potential buddies for an event
- Calculate compatibility scores
- Manage buddy requests (send, accept, decline)
- Track connection status

**Key API Endpoints:**
```
GET    /api/events/:id/potential-buddies  - Get match suggestions
POST   /api/connections/request           - Send buddy request
PUT    /api/connections/:id/accept        - Accept request
PUT    /api/connections/:id/decline       - Decline request
GET    /api/connections                   - Get all connections
DELETE /api/connections/:id               - Unmatch/remove
```

**Matching Algorithm:**
```javascript
// Pseudocode for matching
function calculateCompatibilityScore(user1, user2) {
  let score = 0;
  
  // Same event attendance (required)
  if (!attendingSameEvent(user1, user2)) return 0;
  
  // Comfort level compatibility (30% weight)
  const comfortScore = compareComfortLevels(user1.comfortLevel, user2.comfortLevel);
  score += comfortScore * 30;
  
  // Interest overlap (40% weight)
  const interestScore = calculateInterestOverlap(user1.interests, user2.interests);
  score += interestScore * 40;
  
  // Preference alignment (30% weight)
  const prefScore = comparePreferences(user1.preferences, user2.preferences);
  score += prefScore * 30;
  
  return score; // 0-100
}

// Sort potential buddies by score, return top matches
```

**Data Model (MongoDB):**
```javascript
Connection {
  _id: ObjectId
  eventId: ObjectId (reference to Event)
  users: [ObjectId, ObjectId] (two user IDs)
  requestedBy: ObjectId (who initiated)
  status: Enum ['pending', 'accepted', 'declined', 'completed']
  compatibilityScore: Number
  createdAt: Date
  acceptedAt: Date
  eventDate: Date (denormalized for queries)
}
```

---

### 4. Chat Service

**Responsibilities:**
- Real-time message delivery between matched users
- Message persistence
- Typing indicators
- Read receipts
- Connection-based access control

**Technology**: Socket.io for WebSocket communication

**Key Socket Events:**
```javascript
// Client → Server
'join-chat'      - Join specific connection room
'send-message'   - Send message to buddy
'typing'         - Notify buddy user is typing
'mark-read'      - Mark messages as read

// Server → Client
'receive-message' - New message from buddy
'buddy-typing'    - Buddy is typing
'message-read'    - Buddy read your message
```

**Data Model (MongoDB):**
```javascript
Message {
  _id: ObjectId
  connectionId: ObjectId (reference to Connection)
  senderId: ObjectId (reference to User)
  content: String
  timestamp: Date (indexed)
  read: Boolean
  readAt: Date
}
```

**Chat Flow:**
```
1. User A and User B are matched (Connection status: accepted)
2. Both users open chat screen
3. Mobile app connects to Socket.io server via WebSocket
4. Server authenticates user via JWT token
5. Users join room: `connection_${connectionId}`
6. User A types message → emits 'send-message'
7. Server validates, saves to MongoDB
8. Server emits 'receive-message' to User B in same room
9. User B's app shows message in real-time
10. User B reads message → emits 'mark-read'
11. Server updates message.read = true
12. Server notifies User A via 'message-read' event
```

---

### 5. Notification Service

**Responsibilities:**
- Send push notifications for key events
- In-app notification center
- Notification preferences management

**Technology**: Firebase Cloud Messaging (FCM)

**Notification Triggers:**
- Buddy request received
- Buddy request accepted
- New message received (when app in background)
- Event reminder (24 hours before, 2 hours before)

**Data Model (MongoDB):**
```javascript
Notification {
  _id: ObjectId
  userId: ObjectId
  type: Enum ['buddy-request', 'request-accepted', 'message', 'event-reminder']
  title: String
  body: String
  data: Object (context-specific data)
  read: Boolean
  createdAt: Date
}
```

---

## Data Flow Examples

### Example 1: User Finds and Matches with a Buddy
```
1. User opens Shy app
   → Mobile app authenticates via Supabase Auth (JWT token)

2. User browses events
   → GET /api/events?city=Lagos
   → Backend queries MongoDB for events
   → Returns paginated list of events

3. User selects "Indie Concert - March 15"
   → GET /api/events/abc123
   → Backend returns event details + attendee count

4. User marks attendance
   → POST /api/events/abc123/attend
   → Backend adds user to event.attendees array
   → Returns success

5. User views potential buddies
   → GET /api/events/abc123/potential-buddies
   → Backend:
     a. Finds all users attending same event
     b. Calculates compatibility scores
     c. Sorts by score (descending)
     d. Returns top 10 matches
   → Mobile app displays buddy cards

6. User sends buddy request to Sarah
   → POST /api/connections/request
     Body: { eventId: 'abc123', requestedUserId: 'sarah_id' }
   → Backend:
     a. Creates Connection document (status: pending)
     b. Saves to MongoDB
     c. Triggers push notification to Sarah via FCM
   → Returns success

7. Sarah receives notification
   → FCM delivers push notification to Sarah's device
   → "New buddy request from John for Indie Concert!"

8. Sarah opens app and accepts request
   → PUT /api/connections/xyz789/accept
   → Backend:
     a. Updates Connection.status = 'accepted'
     b. Saves to MongoDB
     c. Triggers push notification to John via FCM
   → Returns success

9. Both users can now chat
   → Chat screen opens
   → WebSocket connection established via Socket.io
   → Both join room: `connection_xyz789`
   → Real-time chat enabled
```

---

### Example 2: Real-Time Chat
```
1. John (matched with Sarah) opens chat screen
   → Mobile app establishes WebSocket connection
   → Socket.io client connects to backend
   → Authentication via JWT token in handshake

2. Server authenticates and adds John to room
   → Server validates JWT
   → Server joins John to Socket.io room: `connection_xyz789`
   → Server confirms connection success

3. Sarah also opens chat screen
   → Same WebSocket connection process
   → Sarah joins same room: `connection_xyz789`

4. John types a message: "Excited for the concert!"
   → Mobile app emits 'typing' event
   → Server broadcasts to room (Sarah sees "John is typing...")

5. John sends the message
   → Mobile app emits 'send-message' event
     Data: { connectionId: 'xyz789', content: 'Excited for the concert!' }
   → Server receives event:
     a. Validates John is in this connection
     b. Saves message to MongoDB:
        {
          connectionId: 'xyz789',
          senderId: 'john_id',
          content: 'Excited for the concert!',
          timestamp: Date.now(),
          read: false
        }
     c. Broadcasts 'receive-message' to room
   → Sarah's app receives event in real-time
   → Sarah's chat screen displays message instantly

6. Sarah reads the message
   → Mobile app emits 'mark-read' event
     Data: { messageId: 'msg123' }
   → Server updates MongoDB: message.read = true
   → Server broadcasts 'message-read' to room
   → John's app shows read receipt (✓✓)

7. If Sarah's app is in background
   → Server sends push notification via FCM
   → "New message from John: Excited for the concert!"
```

---

## Security & Privacy

### Authentication Flow
1. User signs up → Supabase creates account, sends verification email
2. User logs in → Supabase returns JWT access token + refresh token
3. Mobile app stores tokens securely (iOS Keychain / Android Keystore)
4. All API requests include `Authorization: Bearer <jwt_token>` header
5. Backend validates JWT on every request
6. Tokens expire after 1 hour, refresh tokens used to get new access tokens

### Authorization Rules
- Users can only view profiles of people attending the same event
- Chat access requires accepted connection (status: 'accepted')
- Users cannot see other people's conversations
- Event attendee lists only show users who opted in to be visible

### Data Protection
- All API communication over HTTPS (TLS 1.3)
- Database connections encrypted (MongoDB Atlas SSL)
- User passwords hashed with bcrypt (handled by Supabase)
- Profile photos stored in Supabase with signed URLs (expire after 1 hour)
- Sensitive user data (email) not exposed in public APIs

### Privacy Controls
- Users control visibility per event (show/hide from buddy list)
- Option to report/block users
- All personal contact sharing (phone, social media) is opt-in
- Users can delete their account (hard delete after 30 days)

### Rate Limiting
- Redis-based rate limiting: 100 requests per 15 minutes per user
- Prevents spam buddy requests: Max 10 requests per day per user
- Chat message limit: 100 messages per hour per connection

---

## Scalability Considerations

### Current Architecture (MVP - 1K-10K Users)
- **Backend**: Single Node.js server on Railway/Render
- **Database**: MongoDB Atlas M0 (free tier) → M10 as we grow
- **Redis**: Free tier (Upstash or Redis Cloud)
- **Cost**: ~$25-50/month

**Expected Performance:**
- API response time: < 200ms
- WebSocket latency: < 50ms
- Concurrent users: 1,000+
- Messages per second: 100+

### Future Scaling (10K-100K Users)

**Backend Scaling:**
- Horizontal scaling: Multiple Node.js instances behind load balancer
- Stateless API design allows easy replication
- Socket.io with Redis adapter for cross-server WebSocket communication

**Database Scaling:**
- MongoDB sharding by geographic region (users in Lagos → Lagos shard)
- Read replicas for queries (event browsing, user searches)
- Write operations go to primary, reads to replicas

**Caching Strategy:**
- Redis caching for:
  - Event lists (cache 5 minutes)
  - User profiles (cache 10 minutes)
  - Compatibility scores (cache 1 hour)
- Reduces database load by 60-70%

**CDN for Static Assets:**
- Cloudflare or AWS CloudFront
- Serve profile photos, event images from edge locations
- Reduces latency for image loading

---

## Development Workflow

### Local Development Setup
```bash
# Clone repository
git clone https://github.com/yourusername/shy-event-buddy.git
cd shy-event-buddy

# Install backend dependencies
cd backend
npm install

# Set up environment variables
cp .env.example .env
# Edit .env with your MongoDB, Supabase, etc. credentials

# Run backend locally
npm run dev  # Runs on http://localhost:5000

# In new terminal: Install mobile app dependencies
cd ../mobile
npm install

# Run mobile app (with Expo)
npx expo start
# Scan QR code with Expo Go app on your phone
```

### Git Workflow
- **main** branch: Production-ready code
- **develop** branch: Integration branch for features
- **feature/** branches: Individual features (e.g., `feature/chat-system`)

### Commit Message Format
```
feat: Add buddy matching algorithm
fix: Resolve WebSocket reconnection issue
docs: Update system architecture diagram
refactor: Optimize database queries
```

---

## Testing Strategy

### Unit Tests
- **Backend**: Jest for API route handlers, matching algorithm logic
- **Mobile**: Jest + React Native Testing Library for components

### Integration Tests
- **API Tests**: Supertest for endpoint testing
- **Database Tests**: Test MongoDB operations with in-memory MongoDB

### End-to-End Tests
- **Mobile**: Detox for user flow testing (signup → find buddy → chat)

### Manual Testing
- Beta user group (20-30 anxious event-goers)
- Real-world event testing (actual concerts, workshops)
- User feedback surveys after each event

---

## Monitoring & Analytics

### Error Tracking
- **Sentry**: Tracks backend errors, frontend crashes
- Real-time alerts for critical issues
- Performance monitoring (slow API calls)

### Logging
- **Backend**: Winston logger (info, warn, error levels)
- Logs stored in files, searchable via log aggregation tool

### Analytics
- **Mixpanel**: User behavior tracking
  - Events tracked: Signup, Event viewed, Buddy request sent/accepted, Message sent, Event attended
  - Funnels: Signup → Browse → Match → Attend
- **Firebase Analytics**: App usage, screen views, crash reports

### Key Metrics (KPIs)
- User registrations (weekly)
- Events browsed vs. events attended (conversion rate)
- Buddy requests sent vs. accepted (match rate)
- Messages sent per connection (engagement)
- Event completion rate (did users actually attend?)
- Retention: Day 7, Day 30 active users

---

## Why This Architecture Works for Shy

### 1. Mobile-First Design
Real-world event attendance requires:
- **Push notifications** for timely buddy requests and messages
- **Location services** for "events near you" and venue coordination
- **Camera access** for easy profile photo uploads
- **Always available** in user's pocket at events

A web app would compromise these essential features.

### 2. Real-Time Communication
- Socket.io provides low-latency chat experience
- Anxious users need immediate responses to feel comfortable
- Typing indicators and read receipts reduce uncertainty

### 3. Flexible Data Model
- MongoDB's document structure accommodates varied user preferences
- Easy to add new fields (e.g., new anxiety indicators) without schema migrations
- Nested data (messages in conversations) stored naturally

### 4. Proven Technology Stack
- React Native + Node.js = battle-tested, widely adopted
- Large communities → easy to find help, libraries, developers
- Supabase Auth → secure authentication without building from scratch
- MongoDB Atlas → managed database, automatic backups

### 5. Cost-Effective MVP
- Start with free tiers (MongoDB, Supabase, Railway free credits)
- Scale up only when user base grows
- No upfront infrastructure costs

### 6. Security & Privacy First
- Anxious users need trust → Supabase Auth, encrypted connections, privacy controls
- Event-scoped connections → data minimization (only share what's needed)
- User control over visibility → empowers anxious individuals

---

## Open Questions & Future Considerations

### Event Verification
**Problem**: How do we confirm users actually attended the event?
**Potential Solutions**:
- Location-based check-in (GPS at venue during event time)
- QR code scanning (if event organizer provides)
- Post-event confirmation (both buddies confirm attendance)

### Abuse Prevention
**Problem**: What if users accept requests but ghost their buddy?
**Potential Solutions**:
- Reputation system (buddies rate each other post-event)
- Strike system (3 no-shows = account warning)
- Require verified phone number for accountability

### Monetization
**Current**: Free MVP to validate concept
**Future Options**:
- Freemium: Free basic matching, premium for advanced filters
- Event partnerships: Fee from event organizers for promoted listings
- Sponsored events: Companies pay to host "Shy-friendly" events

### Internationalization
- Multi-language support (English, French, Spanish, etc.)
- Localized event discovery (timezone-aware)
- Cultural customization (comfort level norms vary by region)

### Offline Support
- Cache event data for offline browsing
- Queue messages when offline, send when reconnected

---

## Conclusion

This architecture provides a solid foundation for Shy's MVP while allowing room to scale as the user base grows. The mobile-first approach ensures the best experience for anxious event-goers, and the technology choices balance ease of development with production-readiness.

The modular design (separate services for users, events, matching, chat) allows the team to iterate quickly and add features without breaking existing functionality.

**Next Steps**:
1. Set up development environment
2. Build authentication flow (Supabase integration)
3. Implement event browsing and attendance marking
4. Develop matching algorithm
5. Build chat system with Socket.io
6. Beta test with real users at local events

---

**Last Updated**: [Today's Date]  
**Version**: 1.0 (MVP Architecture)  
**Status**: Ready for Development -->