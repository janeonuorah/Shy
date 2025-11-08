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
  comfortLevel: Enum ['social', 'moderate']
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

 ### 2. Event Service

**Responsibilities:**
- Event discovery and search
- Event detail retrieval
- Tracking user attendance (who's going)
- Event creation (manual entry for MVP)

**Key API Endpoints:**
```
GET    /api/events                    - Browse/search events
GET    /api/events/:id                - Get event details
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
    city: String
  }
  description: String
  category: String // ['concert', 'workshop', 'networking', 'social', 'sports']
  imageUrl: String
  attendeeCount: Number
  attendees: Array<ObjectId> (references User._id)
  createdBy: ObjectId (user who added it)
  createdAt: Date
  updatedAt: Date
}
```
---
### 3. Matching Service

**Responsibilities:**
- Find potential buddies for an event
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
  updatedAt: Date
  eventDate: Date
}
```

---

### 4. Chat Service

**Responsibilities:**
- Real-time message delivery between matched users
- Message persistence
- Typing indicators
- Read receipts

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