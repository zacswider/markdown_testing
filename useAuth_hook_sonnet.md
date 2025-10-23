# Understanding the useAuth Hook

## What Are React Hooks?

Think of React hooks as specialized tools in a craftsman's workshop. Just as a carpenter has different tools for different jobs (hammer for nails, saw for cutting, drill for holes), React has different hooks for different UI tasks. Hooks are functions that let you "hook into" React's features without writing a class component.

The `useAuth` hook is a **custom hook** - like a specialized tool that combines several basic tools into one convenient package. It handles all authentication-related functionality: logging in, signing up, logging out, and tracking the current user's state.

## The useAuth Hook Overview

Located at `frontend/src/hooks/useAuth.ts:19`, the `useAuth` hook is the central authentication manager for this application. It combines several React Query hooks and React state management to provide a clean interface for authentication operations.

```typescript
const useAuth = () => {
  // Hook implementation
  return {
    signUpMutation,
    loginMutation,
    logout,
    user,
    error,
    resetError: () => setError(null),
  }
}
```

## Core Dependencies

### React Query Integration

The hook heavily uses `@tanstack/react-query`, which is like having a smart cache manager for server data. Instead of manually tracking loading states, errors, and data freshness, React Query handles this automatically.

```typescript
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query"
```

### Navigation Integration

```typescript
import { useNavigate } from "@tanstack/react-router"
```

The `useNavigate` hook works like a GPS system for your app - it programmatically redirects users to different pages after authentication actions.

## Authentication State Management

### User Data Query

```typescript
const { data: user } = useQuery<UserPublic | null, Error>({
  queryKey: ["currentUser"],
  queryFn: UsersService.readUserMe,
  enabled: isLoggedIn(),
})
```

This query automatically fetches the current user's data when they're logged in. The `enabled: isLoggedIn()` acts like a conditional switch - it only runs the query if there's an access token in localStorage.

### Login State Detection

```typescript
const isLoggedIn = () => {
  return localStorage.getItem("access_token") !== null
}
```

This utility function checks if an access token exists in the browser's localStorage - like checking if you have a valid keycard before trying to enter a building.

## Mutation Operations

### Sign Up Mutation

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

Mutations in React Query are like sending a letter through the postal service:
- `mutationFn`: The actual action being performed
- `onSuccess`: What happens when the letter is delivered successfully
- `onError`: What happens if the letter gets lost
- `onSettled`: Cleanup that happens regardless of success or failure

### Login Mutation

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

The login mutation stores the access token in localStorage upon success and redirects to the home page.

## How Components Use useAuth

### Extracting Specific Values

Components extract only what they need from the hook:

#### User Menu (`frontend/src/components/Common/UserMenu.tsx:10`)
```typescript
const { user, logout } = useAuth()
```

This component only needs the user data for display and the logout function for the logout button.

#### Login Page (`frontend/src/routes/login.tsx:31`)
```typescript
const { loginMutation, error, resetError } = useAuth()
```

The login page needs the login mutation to submit credentials and error handling capabilities.

#### Settings Pages (`frontend/src/routes/_layout/settings.tsx:22`)
```typescript
const { user: currentUser } = useAuth()
```

Settings pages typically only need user data to display current information.

### Authentication Guards

```typescript
// In route configuration
beforeLoad: async () => {
  if (isLoggedIn()) {
    throw redirect({ to: "/" })
  }
}
```

The `isLoggedIn` function is used in route guards - like a bouncer at a club checking if someone should be allowed in or redirected elsewhere.

## Error Handling with handleError

### The handleError Utility

Located at `frontend/src/utils.ts:47`, the `handleError` function is a centralized error processor:

```typescript
export const handleError = (err: ApiError) => {
  const { showErrorToast } = useCustomToast()
  const errDetail = (err.body as any)?.detail
  let errorMessage = errDetail || "Something went wrong."
  if (Array.isArray(errDetail) && errDetail.length > 0) {
    errorMessage = errDetail[0].msg
  }
  showErrorToast(errorMessage)
}
```

### How It Works

1. **Extract Error Details**: The function digs into the API error structure to find the actual error message
2. **Handle Different Error Formats**: It can handle both string errors and array-based validation errors
3. **Display User-Friendly Messages**: Uses the toast system to show errors to users

Think of `handleError` like a translator that takes technical error messages from the server and converts them into friendly notifications that users can understand.

### Toast Integration

The `handleError` function uses `useCustomToast` hook which creates toast notifications:

```typescript
const showErrorToast = (description: string) => {
  toaster.create({
    title: "Something went wrong!",
    description,
    type: "error",
  })
}
```

This is like having a helpful assistant that taps you on the shoulder and politely tells you when something goes wrong, rather than crashing the entire application.

## Authentication Flow Example

Here's how a typical login flow works:

1. User fills out login form
2. Form calls `loginMutation.mutate(credentials)`
3. `loginMutation` sends credentials to server
4. On success:
   - Access token stored in localStorage
   - User redirected to home page
   - User query automatically refetches (because `isLoggedIn()` now returns true)
5. On error:
   - `handleError` processes the error
   - Error toast displayed to user

## Local Storage as Authentication State

This application uses localStorage to persist authentication state:

```typescript
// Store token
localStorage.setItem("access_token", response.access_token)

// Check authentication
localStorage.getItem("access_token") !== null

// Clear authentication
localStorage.removeItem("access_token")
```

Think of localStorage like a safe in your browser where the access token is stored. Even if you close and reopen your browser, the token persists, so you stay logged in.

## Benefits of This Architecture

1. **Centralized Logic**: All authentication logic is in one place
2. **Automatic State Management**: React Query handles loading states, caching, and refetching
3. **Consistent Error Handling**: All auth errors are processed the same way
4. **Easy Testing**: The hook can be mocked for testing
5. **Reusable**: Any component can access auth state without prop drilling

This pattern makes authentication feel seamless to users while keeping the code organized and maintainable for developers.