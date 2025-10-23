# Authentication Hook Deep Dive

This document explains the authentication system in `frontend/src/hooks/useAuth.ts` for experienced programmers new to web development.

## Overview

The `useAuth` hook manages user authentication state, login/logout operations, and user data fetching. It combines React hooks, TanStack Query for data management, and browser APIs for token storage.

## Import Block Analysis

```typescript
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query"
import { useNavigate } from "@tanstack/react-router"
import { useState } from "react"
```

### TanStack Query Imports
- **useQuery**: Fetches and caches data from APIs
- **useMutation**: Handles data modifications (create, update, delete)
- **useQueryClient**: Access to the global query cache

### TanStack Router Import
- **useNavigate**: Programmatic navigation between pages

### React Import
- **useState**: Local component state management

## Type Imports Explained

```typescript
import {
  type Body_login_login_access_token as AccessToken,
  type ApiError,
  LoginService,
  type UserPublic,
  type UserRegister,
  UsersService,
} from "@/client"
```

The `type` prefix indicates TypeScript type-only imports. These are compile-time constructs that don't exist in the JavaScript output.

- **AccessToken**: Type for login form data
- **ApiError**: Error response structure from API
- **UserPublic**: User data structure (safe to expose)
- **UserRegister**: User registration form data
- **LoginService** & **UsersService**: Actual service classes with API methods

## Authentication State Check

```typescript
const isLoggedIn = () => {
  return localStorage.getItem("access_token") !== null
}
```

### localStorage Fundamentals
- **What**: Browser API for persistent key-value storage
- **Scope**: Per browser, per domain (not per IP or user account)
- **Persistence**: Survives page refreshes, browser restarts
- **Capacity**: ~5-10MB per domain
- **Security**: Accessible only to scripts from the same origin

### Function Purpose
Checks if an access token exists, indicating an authenticated session.

## The useAuth Hook Function

```typescript
const useAuth = () => {
  const [error, setError] = useState<string | null>(null)
  const navigate = useNavigate()
  const queryClient = useQueryClient()
```

### React Hooks Explained

**useState**: Creates local state that persists between renders
- **Initial value**: `null`
- **Returns**: `[currentValue, setterFunction]` tuple
- **Scope**: Per component instance, per browser tab

**useNavigate**: Returns navigation function for changing routes
- **Usage**: `navigate({ to: "/path" })`
- **Effect**: Updates URL and renders new component

**useQueryClient**: Provides access to TanStack Query's cache
- **Purpose**: Invalidate cached data after mutations

## User Data Fetching

```typescript
const { data: user } = useQuery<UserPublic | null, Error>({
  queryKey: ["currentUser"],
  queryFn: UsersService.readUserMe,
  enabled: isLoggedIn(),
})
```

### useQuery Parameters
- **queryKey**: Unique cache identifier `["currentUser"]`
- **queryFn**: Async function that fetches data
- **enabled**: Conditionally runs query only when logged in

### Caching Behavior
- **Cache key**: `"currentUser"`
- **Stale time**: 0 (refetches on window focus by default)
- **Scope**: Shared across all components using same key
- **Persistence**: Memory-only, lost on page refresh

## User Registration Mutation

```typescript
const signUpMutation = useMutation({
  mutationFn: (data: UserRegister) =>
    UsersService.registerUser({ requestBody: data }),

  onSuccess: () => {
    navigate({ to: "/login" })
  },
  onError: (err: ApiError) => {
    handleError(err)
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ["users"] })
  },
})
```

### Mutation Lifecycle
1. **mutationFn**: Executes the API call
2. **onSuccess**: Runs after successful registration, redirects to login
3. **onError**: Handles API errors
4. **onSettled**: Runs regardless of outcome, clears user cache

### Cache Invalidation
`invalidateQueries` marks cached data as stale, triggering refetches in components using the `"users"` key.

## Login Function

```typescript
const login = async (data: AccessToken) => {
  const response = await LoginService.loginAccessToken({
    formData: data,
  })
  localStorage.setItem("access_token", response.access_token)
}
```

### Authentication Flow
1. **API call**: Sends credentials to backend
2. **Token storage**: Saves JWT access token to localStorage
3. **No navigation**: Function only handles authentication, navigation handled by mutation

## Login Mutation

```typescript
const loginMutation = useMutation({
  mutationFn: login,
  onSuccess: () => {
    navigate({ to: "/" })
  },
  onError: (err: ApiError) => {
    handleError(err)
  },
})
```

### Differences from signUpMutation
- **Simpler**: No cache invalidation needed
- **Navigation**: Redirects to homepage on success
- **Same error handling**: Consistent error experience

## Logout Function

```typescript
const logout = () => {
  localStorage.removeItem("access_token")
  navigate({ to: "/login" })
}
```

### Cleanup Process
1. **Token removal**: Clears authentication from localStorage
2. **Navigation**: Redirects to login page
3. **No cache clearing**: User data naturally becomes stale on next check

## Return Object

```typescript
return {
  signUpMutation,
  loginMutation,
  logout,
  user,
  error,
  resetError: () => setError(null),
}
```

### Exposed API
- **Mutations**: For triggering auth operations
- **State**: Current user data and error state
- **Utilities**: Error reset function

## Usage Patterns

### Component Integration
```typescript
const { user, loginMutation, logout } = useAuth()

// Check authentication
if (!user) return <LoginForm />

// Handle login
loginMutation.mutate({ username, password })

// Handle logout
<button onClick={logout}>Logout</button>
```

### State Management Scope
- **Per tab**: Each browser tab has independent auth state
- **Shared within tab**: All components in same tab share auth state
- **Lost on refresh**: localStorage persists, but React state resets

## Security Considerations

### Token Storage
- **localStorage**: Vulnerable to XSS attacks
- **No httpOnly**: Accessible to JavaScript
- **No encryption**: Stored in plain text
- **Domain scope**: Shared across all pages on same domain

### Alternatives
- **httpOnly cookies**: Server-only access, XSS protection
- **Session storage**: Clears on tab close
- **Memory only**: No persistence across refreshes

## Error Handling Integration

The hook uses a centralized error handler:
```typescript
import { handleError } from "@/utils"
```

This provides consistent error messaging across the application.

## Testing Implications

### Mock Requirements
- **localStorage**: Needs mocking in test environment
- **API services**: Should be mocked to avoid real API calls
- **Navigation**: Router needs test setup

### State Isolation
Each test should clean up:
- localStorage state
- Query cache
- React component state

## Performance Considerations

### Query Optimization
- **Conditional fetching**: User query only runs when logged in
- **Cache sharing**: Multiple components share same user data
- **Background refetch**: Happens on window focus

### Bundle Size
- **Tree shaking**: Unused mutations can be eliminated
- **Code splitting**: Auth logic can be lazy loaded

## Migration Patterns

### From Class Components
```typescript
// Class component pattern
this.state = { user: null, loading: false }

// Hook equivalent
const { user } = useAuth()
```

### From Redux
```typescript
// Redux pattern
const user = useSelector(state => state.auth.user)
const dispatch = useDispatch()

// Hook equivalent
const { user, loginMutation } = useAuth()
```

This hook provides a simpler, more focused alternative to global state management for authentication concerns.