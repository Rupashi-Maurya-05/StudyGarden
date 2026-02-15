# Design Document: Study Garden

## Overview

Study Garden is a full-stack learning management system consisting of four main components:

1. **React Frontend (SPA)**: User interface for goal management and progress visualization
2. **Node.js/Express Backend**: REST API server handling authentication, business logic, and data persistence
3. **MongoDB Database**: Document store for users, goals, and activity data
4. **Chrome Extension**: Content scripts for automatic progress detection on study platforms

The system's core innovation is the dual-tracking model: platform-linked goals automatically sync progress via the Chrome extension, while custom goals use manual tracking. All data flows through the authenticated REST API and persists to MongoDB for cross-device access.

### Key Design Principles

- **Authentication-first**: All data operations require JWT validation
- **Offline-capable**: Frontend caches data in LocalStorage for offline viewing
- **Real-time sync**: Extension pushes updates immediately upon detecting completion events
- **Visual distinction**: Platform goals and custom goals have different UI affordances
- **Gamification**: Activity tracking drives garden visualization for user motivation

## Architecture

### System Architecture

```
┌─────────────────┐         ┌──────────────────┐
│  React Frontend │◄────────┤ Chrome Extension │
│      (SPA)      │         │  (Content Script)│
└────────┬────────┘         └────────┬─────────┘
         │                           │
         │ HTTPS/REST                │ HTTPS/REST
         │ (JWT Auth)                │ (JWT Auth)
         │                           │
         ▼                           ▼
    ┌────────────────────────────────────┐
    │   Node.js/Express Backend (API)    │
    │  - Authentication (bcrypt + JWT)   │
    │  - Goal Management                 │
    │  - Progress Tracking               │
    │  - Activity Recording              │
    └──────────────┬─────────────────────┘
                   │
                   │ MongoDB Driver
                   │
                   ▼
            ┌──────────────┐
            │   MongoDB    │
            │  - users     │
            │  - goals     │
            └──────────────┘
```

### Data Flow

**Platform Goal Progress Update:**
1. User completes task on study platform (e.g., solves LeetCode problem)
2. Extension content script detects completion event via DOM observation
3. Extension identifies matching goal by URL pattern
4. Extension sends PATCH /goals/:id/progress with JWT token
5. Backend validates token, increments completed_tasks, updates activity_history
6. Backend persists to MongoDB
7. Frontend polls or receives WebSocket update, refreshes UI

**Custom Goal Progress Update:**
1. User clicks +/- button on goal card
2. Frontend sends PATCH /goals/:id/progress with increment value
3. Backend validates token and bounds (0 ≤ completed ≤ total)
4. Backend persists to MongoDB
5. Frontend updates UI optimistically

### Technology Stack

- **Frontend**: React 18, React Router, Axios, LocalStorage API
- **Backend**: Node.js 18+, Express 4, bcrypt, jsonwebtoken, mongoose
- **Database**: MongoDB 6+
- **Extension**: Chrome Extension Manifest V3, Content Scripts API
- **Authentication**: JWT with RS256 signing, 24-hour expiration
- **Styling**: CSS Modules or Styled Components

## Components and Interfaces

### Frontend Components

#### 1. Authentication Components

**LoginForm**
- Input fields: email, password
- Submits POST /auth/login
- Stores JWT token in LocalStorage
- Redirects to dashboard on success

**RegisterForm**
- Input fields: email, password, confirm password
- Validates password strength (min 8 chars, 1 uppercase, 1 number)
- Submits POST /auth/register
- Redirects to onboarding on success

**OnboardingFlow**
- Multi-step form: academic year → major → goal categories
- Submits PUT /users/me/onboarding
- Redirects to dashboard on completion

#### 2. Dashboard Components

**Dashboard**
- Container component managing state for all goals
- Fetches GET /goals on mount
- Groups goals by category
- Renders CategorySection for each selected category
- Displays GardenVisualization and GlobalStreak

**CategorySection**
- Props: category name, goals array, isExpanded
- Displays category header with icon, goal count, aggregate completion %
- Collapsible section (state cached in LocalStorage)
- Renders GoalCard for each goal in category

**GoalCard**
- Props: goal object (id, type, title, url, completed_tasks, total_tasks, deadline, next_step)
- Displays progress bar, completion fraction, deadline countdown
- Conditional rendering:
  - Platform goals: Show "Auto-synced ✓" badge
  - Custom goals: Show +/- buttons
- Handles click events for +/- buttons → calls updateProgress()
- Shows urgency indicator if deadline ≤ 3 days
- Shows completion badge if 100% complete

**GardenVisualization**
- Fetches GET /activity on mount
- Calculates plant stage from 30-day activity count
- Renders animated plant emoji with growth transitions
- Displays 30-day activity heatmap (calendar grid)
- Shows global streak counter

#### 3. Goal Management Components

**AddGoalForm**
- Input fields: category, title, url (optional), task_count, deadline, next_step
- URL validation: checks if matches Supported_Platform patterns
- Auto-detects goal type based on URL
- Submits POST /goals
- Closes modal and refreshes goal list on success

**EditGoalModal**
- Pre-populated form with existing goal data
- Allows editing all fields
- Submits PUT /goals/:id
- Updates goal list on success

### Backend API Structure

#### Authentication Module

**POST /auth/register**
```javascript
Request: { email: string, password: string }
Process:
  1. Validate email format and password strength
  2. Check if email already exists
  3. Hash password with bcrypt (10 rounds)
  4. Create user document in MongoDB
  5. Return success message
Response: { message: "User created successfully" }
```

**POST /auth/login**
```javascript
Request: { email: string, password: string }
Process:
  1. Find user by email
  2. Compare password with bcrypt
  3. Generate JWT token (payload: { userId, email }, expires: 24h)
  4. Return token
Response: { token: string, user: { id, email } }
```

**POST /auth/logout**
```javascript
Request: { token: string }
Process:
  1. Add token to blacklist (Redis or in-memory cache)
  2. Return success
Response: { message: "Logged out successfully" }
```

#### User Profile Module

**GET /users/me**
```javascript
Headers: { Authorization: "Bearer <token>" }
Process:
  1. Verify JWT token
  2. Extract userId from token
  3. Fetch user document from MongoDB
  4. Return user profile
Response: {
  id: string,
  email: string,
  academic_year: string,
  major: string,
  selected_categories: string[],
  activity_history: { date: string, task_count: number }[]
}
```

**PUT /users/me/onboarding**
```javascript
Headers: { Authorization: "Bearer <token>" }
Request: {
  academic_year: string,
  major: string,
  selected_categories: string[]
}
Process:
  1. Verify JWT token
  2. Update user document with onboarding data
  3. Return updated user
Response: { user: UserObject }
```

#### Goal Management Module

**GET /goals**
```javascript
Headers: { Authorization: "Bearer <token>" }
Process:
  1. Verify JWT token
  2. Query goals collection where user_id = userId
  3. Return all goals
Response: { goals: GoalObject[] }
```

**POST /goals**
```javascript
Headers: { Authorization: "Bearer <token>" }
Request: {
  category: string,
  title: string,
  url?: string,
  total_tasks: number,
  deadline: ISO8601 date string,
  next_step?: string
}
Process:
  1. Verify JWT token
  2. Determine goal type from URL (platform vs custom)
  3. Create goal document with user_id, completed_tasks=0, created_at
  4. Insert into MongoDB
  5. Return created goal
Response: { goal: GoalObject }
```

**PUT /goals/:id**
```javascript
Headers: { Authorization: "Bearer <token>" }
Request: { fields to update }
Process:
  1. Verify JWT token
  2. Verify goal belongs to user
  3. Update goal document
  4. Return updated goal
Response: { goal: GoalObject }
```

**DELETE /goals/:id**
```javascript
Headers: { Authorization: "Bearer <token>" }
Process:
  1. Verify JWT token
  2. Verify goal belongs to user
  3. Delete goal document
  4. Return success
Response: { message: "Goal deleted" }
```

**PATCH /goals/:id/progress**
```javascript
Headers: { Authorization: "Bearer <token>" }
Request: { increment: number }  // +1 or -1
Process:
  1. Verify JWT token
  2. Verify goal belongs to user
  3. Calculate new completed_tasks = current + increment
  4. Validate: 0 <= new_value <= total_tasks
  5. Update goal document
  6. Update activity_history for current date
  7. Return updated goal
Response: { goal: GoalObject }
```

#### Activity Module

**GET /activity**
```javascript
Headers: { Authorization: "Bearer <token>" }
Process:
  1. Verify JWT token
  2. Fetch user's activity_history (last 30 days)
  3. Return activity data
Response: {
  activity: { date: string, task_count: number }[],
  global_streak: number
}
```

**POST /activity**
```javascript
Headers: { Authorization: "Bearer <token>" }
Request: { date: string, task_count: number }
Process:
  1. Verify JWT token
  2. Upsert activity record for date
  3. Recalculate global streak
  4. Return updated activity
Response: { activity: ActivityObject }
```

### Chrome Extension Architecture

#### Manifest Configuration (manifest.json)

```json
{
  "manifest_version": 3,
  "name": "Study Garden Tracker",
  "version": "1.0.0",
  "permissions": ["storage", "activeTab"],
  "host_permissions": [
    "https://leetcode.com/*",
    "https://classroom.google.com/*",
    "https://github.com/*",
    "https://codeforces.com/*",
    "https://onlinecourses.nptel.ac.in/*",
    "https://www.coursera.org/*"
  ],
  "content_scripts": [
    {
      "matches": ["https://leetcode.com/*"],
      "js": ["content-scripts/leetcode.js"]
    },
    {
      "matches": ["https://classroom.google.com/*"],
      "js": ["content-scripts/classroom.js"]
    },
    {
      "matches": ["https://github.com/*"],
      "js": ["content-scripts/github.js"]
    },
    {
      "matches": ["https://codeforces.com/*"],
      "js": ["content-scripts/codeforces.js"]
    },
    {
      "matches": ["https://onlinecourses.nptel.ac.in/*"],
      "js": ["content-scripts/nptel.js"]
    },
    {
      "matches": ["https://www.coursera.org/*"],
      "js": ["content-scripts/coursera.js"]
    }
  ],
  "background": {
    "service_worker": "background.js"
  }
}
```

#### Content Script Pattern (Platform-Specific)

Each content script follows this pattern:

```javascript
// Example: leetcode.js

// 1. Detect completion event
function detectCompletion() {
  // Watch for "Accepted" status in submission result
  const observer = new MutationObserver((mutations) => {
    for (const mutation of mutations) {
      if (isAcceptedSubmission(mutation)) {
        handleCompletion();
      }
    }
  });
  
  observer.observe(document.body, {
    childList: true,
    subtree: true
  });
}

// 2. Identify matching goal
async function findMatchingGoal() {
  const currentUrl = window.location.href;
  const token = await getStoredToken();
  
  const response = await fetch(`${API_BASE}/goals`, {
    headers: { 'Authorization': `Bearer ${token}` }
  });
  
  const { goals } = await response.json();
  return goals.find(goal => 
    goal.type === 'platform' && 
    currentUrl.includes(goal.url)
  );
}

// 3. Send progress update
async function handleCompletion() {
  const goal = await findMatchingGoal();
  if (!goal) return;
  
  const token = await getStoredToken();
  
  await fetch(`${API_BASE}/goals/${goal.id}/progress`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ increment: 1 })
  });
  
  showNotification('Progress updated! 🌱');
}

// 4. Token management
async function getStoredToken() {
  return new Promise((resolve) => {
    chrome.storage.local.get(['jwt_token'], (result) => {
      resolve(result.jwt_token);
    });
  });
}

// Initialize
detectCompletion();
```

#### Platform-Specific Detection Logic

**LeetCode**: Watch for `<span class="text-green">Accepted</span>` in submission result panel

**Google Classroom**: Watch for "Turned in" status change on assignment page

**GitHub**: Watch for successful commit push notification or PR merge event

**Codeforces**: Watch for "Accepted" verdict in submission status

**NPTEL**: Watch for checkmark icon on lecture completion

**Coursera**: Watch for lesson completion animation/state change

## Data Models

### MongoDB Collections

#### users Collection

```javascript
{
  _id: ObjectId,
  email: string,              // unique, indexed
  password_hash: string,      // bcrypt hash
  academic_year: string,      // "Freshman" | "Sophomore" | "Junior" | "Senior" | "Graduate"
  major: string,              // "CS" | "IT" | "SE" | "Other Engineering" | "Other"
  selected_categories: [      // Array of selected goal categories
    string                    // "Classroom Assignments" | "Online Courses" | etc.
  ],
  activity_history: [         // Last 30 days of activity
    {
      date: ISODate,          // Date of activity
      task_count: number      // Number of tasks completed that day
    }
  ],
  global_streak: number,      // Consecutive days with any task completion
  created_at: ISODate,
  updated_at: ISODate
}
```

**Indexes:**
- `email`: unique index for fast lookup during authentication
- `_id`: default primary key index

#### goals Collection

```javascript
{
  _id: ObjectId,
  user_id: ObjectId,          // Reference to users._id, indexed
  type: string,               // "platform" | "custom"
  category: string,           // One of 6 categories from onboarding
  title: string,              // User-defined goal title
  url: string | null,         // Platform URL for platform goals, null for custom
  total_tasks: number,        // Total number of tasks to complete
  completed_tasks: number,    // Current completion count
  deadline: ISODate,          // Target completion date
  next_step: string | null,   // Optional notes about next action
  goal_streak: number,        // Consecutive days with progress on this goal
  last_activity_date: ISODate | null,  // Last date progress was made
  created_at: ISODate,
  updated_at: ISODate
}
```

**Indexes:**
- `user_id`: index for fast filtering by user
- `user_id + category`: compound index for category-based queries
- `user_id + deadline`: compound index for deadline sorting

### Data Validation Rules

**User Document:**
- `email`: Must match email regex pattern
- `password_hash`: Must be bcrypt hash (60 characters)
- `academic_year`: Must be one of 5 allowed values
- `major`: Must be one of 5 allowed values
- `selected_categories`: Array length 1-6, each element must be valid category
- `activity_history`: Max 30 entries, sorted by date descending
- `global_streak`: Non-negative integer

**Goal Document:**
- `user_id`: Must reference existing user
- `type`: Must be "platform" or "custom"
- `category`: Must be in user's selected_categories
- `title`: Non-empty string, max 200 characters
- `url`: If type="platform", must match Supported_Platform pattern; if type="custom", must be null
- `total_tasks`: Positive integer, max 10000
- `completed_tasks`: Non-negative integer, must be ≤ total_tasks
- `deadline`: Must be future date (at creation time)
- `goal_streak`: Non-negative integer

### URL Pattern Matching for Platform Detection

```javascript
const PLATFORM_PATTERNS = {
  leetcode: /^https?:\/\/(www\.)?leetcode\.com\/.+/,
  classroom: /^https?:\/\/classroom\.google\.com\/.+/,
  github: /^https?:\/\/(www\.)?github\.com\/.+/,
  codeforces: /^https?:\/\/(www\.)?codeforces\.com\/.+/,
  nptel: /^https?:\/\/onlinecourses\.nptel\.ac\.in\/.+/,
  coursera: /^https?:\/\/(www\.)?coursera\.org\/.+/
};

function detectGoalType(url) {
  if (!url) return 'custom';
  
  for (const [platform, pattern] of Object.entries(PLATFORM_PATTERNS)) {
    if (pattern.test(url)) {
      return 'platform';
    }
  }
  
  return 'custom';
}
```


## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Property 1: Password Security (Bcrypt Hashing)

*For any* user registration with valid email and password, the stored password in the database should be a bcrypt hash (not plaintext), and verifying the original password against the stored hash should succeed.

**Validates: Requirements 1.1**

### Property 2: JWT Authentication Round-Trip

*For any* valid user credentials, logging in should generate a JWT token that can be decoded to contain the correct user ID and email, and using that token for authenticated requests should succeed.

**Validates: Requirements 1.2, 1.4, 10.14**

### Property 3: Token Invalidation

*For any* valid JWT token, after calling the logout endpoint with that token, subsequent requests using the same token should be rejected with an authentication error.

**Validates: Requirements 1.3**

### Property 4: Invalid Token Rejection

*For any* expired or malformed JWT token, requests to protected endpoints should be rejected with an authentication error.

**Validates: Requirements 1.5**

### Property 5: Data Persistence Round-Trip

*For any* valid data modification (user profile update, goal creation, goal update, activity recording), immediately fetching the modified data should return the updated values.

**Validates: Requirements 2.5, 3.5, 5.7, 7.11, 9.1**

### Property 6: Profile Data Completeness

*For any* user who has completed onboarding, fetching their profile should return all required fields: email, academic_year, major, selected_categories, and activity_history.

**Validates: Requirements 2.6**

### Property 7: Goal Type Classification

*For any* goal creation request, if the URL matches a supported platform pattern (LeetCode, Classroom, GitHub, Codeforces, NPTEL, Coursera), the goal type should be "platform"; otherwise (including null URL), the goal type should be "custom".

**Validates: Requirements 3.1, 3.2**

### Property 8: Required Fields Validation

*For any* goal creation request missing required fields (category, title, task count, or deadline), the request should be rejected with a validation error.

**Validates: Requirements 3.3**

### Property 9: Goal Deletion

*For any* goal belonging to a user, after successfully deleting that goal, attempting to fetch it should return a 404 error, and it should not appear in the user's goal list.

**Validates: Requirements 3.6**

### Property 10: User Goal Isolation

*For any* authenticated user, fetching their goals should return only goals where user_id matches their ID, and should not include goals belonging to other users.

**Validates: Requirements 3.7, 9.4**

### Property 11: Progress Update Bounds Validation

*For any* progress update request (increment or decrement), if the resulting completed_tasks would be negative or exceed total_tasks, the request should be rejected with a validation error and the database should remain unchanged; otherwise, the update should succeed and persist.

**Validates: Requirements 5.4, 5.5, 5.6**

### Property 12: Progress Update Side Effects

*For any* valid progress update (from extension or manual), the completed_tasks field should be incremented by the specified amount, and the activity_history should be updated with a record for the current date.

**Validates: Requirements 4.8**

### Property 13: Days Remaining Calculation

*For any* goal with a deadline, the calculated days remaining should equal the number of days between the current date and the deadline date (negative if overdue).

**Validates: Requirements 6.3**

### Property 14: Streak Calculation Correctness

*For any* goal or user with activity history, the calculated streak should equal the number of consecutive days (ending with today or the most recent activity) where at least one task was completed.

**Validates: Requirements 6.7, 6.8**

### Property 15: Garden Growth Uses 30-Day Window

*For any* user, the garden plant stage calculation should only consider activity records from the last 30 days, ignoring any older activity.

**Validates: Requirements 7.1**

### Property 16: Activity Counting Inclusivity

*For any* day's activity count, both automatically detected completions (from extension) and manual completions (from +/- buttons) should contribute to the total count.

**Validates: Requirements 7.10**

### Property 17: Category-Based Goal Grouping

*For any* set of goals belonging to a user, when displayed on the dashboard, goals should be grouped such that all goals with the same category appear in the same section, and only categories the user selected during onboarding should be displayed.

**Validates: Requirements 8.1, 8.4**

### Property 18: Category State Persistence

*For any* category section expand/collapse action, the state should be stored in browser LocalStorage, and when the page reloads, the stored state should be restored.

**Validates: Requirements 8.3, 8.5**

### Property 19: Offline Mode Data Access

*For any* user with cached goal data in LocalStorage, when the frontend is offline, the cached data should be displayed in read-only mode, and write operations should be disabled or queued.

**Validates: Requirements 9.6, 12.7**

### Property 20: Sync on Reconnection

*For any* queued write operations while offline, when network connectivity is restored, all queued operations should be sent to the backend in order.

**Validates: Requirements 9.7**

### Property 21: Last-Write-Wins Conflict Resolution

*For any* goal that receives concurrent updates from multiple sources (e.g., extension and manual update), the final state should reflect the update with the latest timestamp.

**Validates: Requirements 9.8**

### Property 22: Rate Limiting Enforcement

*For any* API endpoint, when a client exceeds the configured rate limit (e.g., 100 requests per minute), subsequent requests should be rejected with a 429 status code until the rate limit window resets.

**Validates: Requirements 10.15**

## Error Handling

### Authentication Errors

**Invalid Credentials (401)**
- Scenario: User provides incorrect email or password during login
- Response: `{ error: "Invalid credentials" }`
- Action: Frontend displays error message, does not store token

**Expired Token (401)**
- Scenario: User makes request with expired JWT token
- Response: `{ error: "Token expired" }`
- Action: Frontend clears stored token, redirects to login

**Missing Token (401)**
- Scenario: Request to protected endpoint without Authorization header
- Response: `{ error: "Authentication required" }`
- Action: Frontend redirects to login

### Validation Errors

**Missing Required Fields (400)**
- Scenario: Goal creation without required fields
- Response: `{ error: "Missing required fields", fields: ["title", "deadline"] }`
- Action: Frontend highlights missing fields in form

**Invalid Progress Update (400)**
- Scenario: Increment would exceed total_tasks or go negative
- Response: `{ error: "Invalid progress value", current: 5, total: 5, attempted: 6 }`
- Action: Frontend displays error toast, does not update UI optimistically

**Invalid URL Format (400)**
- Scenario: Platform goal with malformed URL
- Response: `{ error: "Invalid URL format" }`
- Action: Frontend displays error message in form

### Authorization Errors

**Goal Not Found or Unauthorized (404)**
- Scenario: User attempts to access/modify goal belonging to another user
- Response: `{ error: "Goal not found" }`
- Action: Frontend displays "Goal not found" message

### Rate Limiting

**Too Many Requests (429)**
- Scenario: Client exceeds rate limit
- Response: `{ error: "Rate limit exceeded", retryAfter: 60 }`
- Action: Frontend displays message, disables actions for retryAfter seconds

### Network Errors

**Offline Mode**
- Scenario: No network connectivity
- Response: N/A (request fails before reaching server)
- Action: Frontend detects offline state, shows cached data, queues writes

**Timeout (504)**
- Scenario: Request takes longer than timeout threshold (30 seconds)
- Response: Gateway timeout
- Action: Frontend retries with exponential backoff (3 attempts)

### Extension-Specific Errors

**Extension Not Authenticated**
- Scenario: Extension attempts progress update without stored JWT
- Response: 401 from backend
- Action: Extension displays browser notification: "Please log in to Study Garden"

**Goal Not Found for URL**
- Scenario: Extension detects completion but no matching goal exists
- Response: N/A (extension-side check)
- Action: Extension silently ignores (user may not have created goal yet)

## Testing Strategy

### Dual Testing Approach

The Study Garden application requires both unit testing and property-based testing for comprehensive coverage:

- **Unit tests**: Verify specific examples, edge cases, and error conditions
- **Property tests**: Verify universal properties across all inputs

Both testing approaches are complementary and necessary. Unit tests catch concrete bugs in specific scenarios, while property tests verify general correctness across a wide range of inputs.

### Property-Based Testing Configuration

**Library Selection:**
- **Backend (Node.js)**: Use `fast-check` library for property-based testing
- **Frontend (React)**: Use `fast-check` with React Testing Library for component property tests
- **Extension**: Use `fast-check` for content script logic testing

**Test Configuration:**
- Each property test MUST run minimum 100 iterations (due to randomization)
- Each property test MUST include a comment tag referencing the design property
- Tag format: `// Feature: study-garden, Property {number}: {property_text}`

**Example Property Test Structure:**

```javascript
// Feature: study-garden, Property 5: Data Persistence Round-Trip
test('goal updates persist and are retrievable', async () => {
  await fc.assert(
    fc.asyncProperty(
      fc.record({
        title: fc.string({ minLength: 1, maxLength: 200 }),
        total_tasks: fc.integer({ min: 1, max: 1000 }),
        completed_tasks: fc.integer({ min: 0, max: 1000 })
      }),
      async (goalUpdate) => {
        // Create goal, update it, fetch it, verify changes persisted
        const goal = await createGoal(baseGoal);
        await updateGoal(goal.id, goalUpdate);
        const fetched = await getGoal(goal.id);
        
        expect(fetched.title).toBe(goalUpdate.title);
        expect(fetched.total_tasks).toBe(goalUpdate.total_tasks);
      }
    ),
    { numRuns: 100 }
  );
});
```

### Unit Testing Strategy

**Backend Unit Tests:**
- Authentication middleware: Test token validation, expiration, blacklisting
- Goal type detection: Test URL pattern matching for all supported platforms
- Progress validation: Test boundary conditions (0, total_tasks, negative)
- Streak calculation: Test consecutive day counting with gaps
- Activity aggregation: Test 30-day window filtering
- Rate limiting: Test request counting and throttling

**Frontend Unit Tests:**
- Component rendering: Test conditional rendering (platform vs custom goals)
- Form validation: Test required fields, URL format validation
- Date calculations: Test days remaining, overdue detection
- Garden stage calculation: Test all 7 plant stage thresholds
- LocalStorage caching: Test save/restore of category states
- Offline mode: Test read-only behavior, write queueing

**Extension Unit Tests:**
- URL matching: Test goal identification by URL pattern
- Token storage: Test JWT retrieval from chrome.storage
- Completion detection: Test DOM observation for each platform
- API communication: Test authenticated request construction

**Integration Tests:**
- End-to-end goal creation flow: Register → onboard → create goal → verify in DB
- Progress update flow: Create goal → update progress → verify activity recorded
- Extension sync flow: Mock platform completion → verify backend update
- Offline sync flow: Queue writes offline → reconnect → verify sync

### Test Coverage Goals

- **Backend**: 80% code coverage minimum
- **Frontend**: 70% code coverage minimum (UI components harder to test)
- **Extension**: 60% code coverage minimum (DOM-dependent logic)
- **Property tests**: All 22 correctness properties must have corresponding tests

### Continuous Integration

- Run all tests on every pull request
- Block merges if tests fail or coverage drops below thresholds
- Run property tests with 100 iterations in CI (can use fewer locally for speed)
- Include integration tests in CI pipeline with test MongoDB instance
