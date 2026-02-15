# Requirements Document

## Introduction

Study Garden is a full-stack web application designed to help CS/BTech students manage multiple learning goals across various platforms in a unified interface. The system distinguishes between platform-linked goals (which automatically sync progress via Chrome extension) and custom goals (which require manual tracking). All data persists to MongoDB for cross-device synchronization.

## Glossary

- **System**: The complete Study Garden application including frontend, backend, database, and Chrome extension
- **Frontend**: React single-page application for viewing and managing goals
- **Backend**: Node.js + Express REST API server
- **Database**: MongoDB instance storing user and goal data
- **Extension**: Chrome browser extension for automatic progress detection
- **User**: CS student using the application
- **Platform_Goal**: Goal linked to a supported study platform with automatic progress tracking
- **Custom_Goal**: User-created goal requiring manual progress updates
- **Activity_History**: Record of daily task completions over 30-day period
- **Garden_Visualization**: Animated plant growth system based on user activity
- **JWT**: JSON Web Token used for authentication
- **Supported_Platform**: One of LeetCode, Google Classroom, GitHub, Codeforces, NPTEL, or Coursera

## Requirements

### Requirement 1: User Authentication

**User Story:** As a student, I want to securely register and log in to my account, so that my learning goals and progress are protected and accessible only to me.

#### Acceptance Criteria

1. WHEN a user submits registration with email and password, THE System SHALL hash the password using bcrypt and store the user credentials in the Database
2. WHEN a user submits valid login credentials, THE Backend SHALL generate a JWT token and return it to the Frontend
3. WHEN a user logs out, THE Backend SHALL invalidate the user's JWT token
4. WHEN an API request is made to protected endpoints, THE Backend SHALL validate the JWT token before processing the request
5. IF a JWT token is expired or invalid, THEN THE Backend SHALL return an authentication error and reject the request

### Requirement 2: User Profile and Onboarding

**User Story:** As a new user, I want to set up my academic profile and select relevant goal categories, so that the application is personalized to my learning context.

#### Acceptance Criteria

1. WHEN a new user completes registration, THE Frontend SHALL display an onboarding flow requesting academic year and major
2. THE System SHALL provide academic year options: Freshman, Sophomore, Junior, Senior, Graduate
3. THE System SHALL provide major options: CS, IT, SE, Other Engineering, Other
4. WHEN a user selects goal categories during onboarding, THE System SHALL present six options: Classroom Assignments, Online Courses, DSA Sheets, Projects, Interview Prep, Competitive Programming
5. WHEN onboarding is completed, THE Backend SHALL persist all selections to the user profile in the Database
6. WHEN a user requests their profile, THE Backend SHALL return all profile data including onboarding selections

### Requirement 3: Goal Creation and Management

**User Story:** As a student, I want to create and manage learning goals with different tracking methods, so that I can organize my study activities across multiple platforms.

#### Acceptance Criteria

1. WHEN a user creates a goal with a URL matching a Supported_Platform, THE System SHALL classify it as a Platform_Goal
2. WHEN a user creates a goal without a URL or with a non-platform URL, THE System SHALL classify it as a Custom_Goal
3. WHEN a goal is created, THE System SHALL require: category, title, task count, and deadline
4. WHERE a goal is a Platform_Goal, THE System SHALL require a resource URL
5. WHEN a user edits a goal, THE Backend SHALL update the goal document in the Database and return the updated goal
6. WHEN a user deletes a goal, THE Backend SHALL remove the goal document from the Database
7. WHEN a user requests their goals, THE Backend SHALL return all goals associated with the authenticated user

### Requirement 4: Automatic Progress Tracking via Chrome Extension

**User Story:** As a student, I want my progress on study platforms to be automatically detected and synced, so that I don't have to manually update my completion status.

#### Acceptance Criteria

1. WHEN the Extension detects a completion event on a Supported_Platform page, THE Extension SHALL identify the matching Platform_Goal by URL
2. WHEN a LeetCode problem submission is accepted, THE Extension SHALL send an authenticated progress update request to the Backend
3. WHEN a Google Classroom assignment is turned in, THE Extension SHALL send an authenticated progress update request to the Backend
4. WHEN a GitHub commit is pushed or PR is merged, THE Extension SHALL send an authenticated progress update request to the Backend
5. WHEN a Codeforces submission is accepted, THE Extension SHALL send an authenticated progress update request to the Backend
6. WHEN an NPTEL lecture is marked complete, THE Extension SHALL send an authenticated progress update request to the Backend
7. WHEN a Coursera lesson is completed, THE Extension SHALL send an authenticated progress update request to the Backend
8. WHEN the Backend receives a progress update from the Extension, THE Backend SHALL increment the completed_tasks field and update the Activity_History
9. IF the Extension cannot authenticate, THEN THE Extension SHALL prompt the user to log in

### Requirement 5: Manual Progress Tracking

**User Story:** As a student, I want to manually update progress on custom goals, so that I can track learning activities that aren't on supported platforms.

#### Acceptance Criteria

1. WHERE a goal is a Custom_Goal, THE Frontend SHALL display increment and decrement buttons on the goal card
2. WHEN a user clicks the increment button, THE Frontend SHALL send a progress update request with increment value 1 to the Backend
3. WHEN a user clicks the decrement button, THE Frontend SHALL send a progress update request with increment value -1 to the Backend
4. WHEN the Backend processes a progress update, THE Backend SHALL validate that completed_tasks does not exceed total_tasks
5. WHEN the Backend processes a progress update, THE Backend SHALL validate that completed_tasks does not go below zero
6. IF validation fails, THEN THE Backend SHALL return an error and not update the Database
7. WHEN a valid progress update is processed, THE Backend SHALL persist the change to the Database immediately

### Requirement 6: Progress Visualization

**User Story:** As a student, I want to see visual representations of my progress, so that I can quickly understand my completion status and stay motivated.

#### Acceptance Criteria

1. WHEN displaying a goal, THE Frontend SHALL show a progress bar indicating completion percentage
2. WHEN displaying a goal, THE Frontend SHALL show completed tasks over total tasks as a fraction
3. WHEN displaying a goal, THE Frontend SHALL calculate and show days remaining until deadline
4. WHEN a goal has 3 or fewer days until deadline, THE Frontend SHALL display an urgency indicator
5. WHEN a goal is overdue, THE Frontend SHALL display an overdue indicator
6. WHEN a goal reaches 100% completion, THE Frontend SHALL display a completion badge
7. WHEN displaying a goal, THE Frontend SHALL show the goal-specific streak of consecutive days with task completion
8. THE Frontend SHALL display a global streak counter showing consecutive days with any task completion across all goals

### Requirement 7: Garden Visualization and Gamification

**User Story:** As a student, I want to see my study activity represented as a growing garden, so that I feel motivated to maintain consistent learning habits.

#### Acceptance Criteria

1. WHEN the System calculates garden growth, THE System SHALL use the 30-day Activity_History
2. WHEN a user has 1 or more active days, THE Frontend SHALL display a Seed stage plant (🌱)
3. WHEN a user has 5 or more active days, THE Frontend SHALL display a Sprout stage plant (🌿)
4. WHEN a user has 10 or more active days, THE Frontend SHALL display a Young Plant stage plant (🌳)
5. WHEN a user has 15 or more active days, THE Frontend SHALL display a Flowering stage plant (🌺)
6. WHEN a user has 20 or more active days, THE Frontend SHALL display a Mature stage plant (🌻)
7. WHEN a user has 25 or more active days, THE Frontend SHALL display a Thriving stage plant (🍀)
8. WHEN a user has 28 or more active days, THE Frontend SHALL display a Full Bloom stage plant (🌸)
9. THE Frontend SHALL display a 30-day activity heatmap showing daily task completion intensity
10. WHEN calculating active days, THE System SHALL count both automatic and manual task completions
11. WHEN activity data is updated, THE Backend SHALL persist changes to the user's Activity_History in the Database

### Requirement 8: Category-Based Organization

**User Story:** As a student, I want my goals organized by category, so that I can focus on specific types of learning activities.

#### Acceptance Criteria

1. WHEN displaying the dashboard, THE Frontend SHALL group goals into sections by category
2. WHEN displaying a category section, THE Frontend SHALL show the category icon, total goal count, and aggregate completion percentage
3. WHEN a user expands or collapses a category section, THE Frontend SHALL cache the state in browser LocalStorage
4. THE Frontend SHALL only display categories that the user selected during onboarding
5. WHEN the Frontend loads, THE Frontend SHALL restore category expand/collapse states from LocalStorage

### Requirement 9: Data Persistence and Synchronization

**User Story:** As a student, I want my data to be saved in the cloud and accessible across devices, so that I can access my goals from anywhere.

#### Acceptance Criteria

1. WHEN user data is created or modified, THE Backend SHALL persist changes to the Database immediately
2. THE Database SHALL store user profiles, onboarding data, and Activity_History in a users collection
3. THE Database SHALL store all goal documents in a goals collection
4. WHEN storing goals, THE Backend SHALL associate each goal with the user_id from the JWT token
5. WHEN the Frontend is online, THE Frontend SHALL fetch current data from the Backend
6. WHEN the Frontend is offline, THE Frontend SHALL display cached goal data from LocalStorage in read-only mode
7. WHEN network connectivity is restored, THE Frontend SHALL sync any pending changes to the Backend
8. IF concurrent updates occur, THEN THE System SHALL apply last-write-wins conflict resolution

### Requirement 10: REST API Endpoints

**User Story:** As a developer, I want a well-defined REST API, so that the Frontend and Extension can reliably communicate with the Backend.

#### Acceptance Criteria

1. THE Backend SHALL provide a POST /auth/register endpoint for user account creation
2. THE Backend SHALL provide a POST /auth/login endpoint that returns a JWT token
3. THE Backend SHALL provide a POST /auth/logout endpoint that invalidates tokens
4. THE Backend SHALL provide a GET /users/me endpoint that returns user profile and onboarding data
5. THE Backend SHALL provide a PUT /users/me/onboarding endpoint for updating onboarding selections
6. THE Backend SHALL provide a GET /goals endpoint that returns all goals for the authenticated user
7. THE Backend SHALL provide a POST /goals endpoint for creating new goals
8. THE Backend SHALL provide a GET /goals/:id endpoint for fetching a specific goal
9. THE Backend SHALL provide a PUT /goals/:id endpoint for updating goals
10. THE Backend SHALL provide a DELETE /goals/:id endpoint for deleting goals
11. THE Backend SHALL provide a PATCH /goals/:id/progress endpoint for incrementing or decrementing completed_tasks
12. THE Backend SHALL provide a GET /activity endpoint that returns 30-day activity history
13. THE Backend SHALL provide a POST /activity endpoint for recording daily activity
14. WHEN a request is made to any endpoint except /auth/register and /auth/login, THE Backend SHALL require valid JWT authentication
15. THE Backend SHALL implement rate limiting on all endpoints to prevent abuse

### Requirement 11: Visual Design and User Experience

**User Story:** As a student, I want an aesthetically pleasing and intuitive interface, so that using the application is enjoyable and stress-free.

#### Acceptance Criteria

1. THE Frontend SHALL use an earthy journal aesthetic with warm browns (#c4a882) and tans (#b8976a)
2. THE Frontend SHALL use a white/cream background color (#faf8f5)
3. THE Frontend SHALL apply rounded corners (10-20px border-radius) to all component containers
4. THE Frontend SHALL use a Display serif font for headings
5. THE Frontend SHALL use a sans-serif font (Segoe UI) for body text
6. THE Frontend SHALL implement smooth transitions and micro-animations for user interactions
7. WHERE a goal is a Platform_Goal, THE Frontend SHALL display an auto-sync badge (✓)
8. WHERE a goal is a Custom_Goal, THE Frontend SHALL display increment and decrement buttons
9. THE Frontend SHALL implement a responsive layout optimized for desktop viewing

