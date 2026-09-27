# Architecture Overview: MVVM + RTK Query in React

This document outlines how the **Model-View-ViewModel (MVVM)** pattern is structured and integrated with **RTK Query** in this project's `src` directory.

## Folder Structure Reference

```text
src/
├── models/
├── slice/
├── view-models/
├── views/
├── routes/
├── utils/
└── mock/
```

| Folder         | Responsibility                                                                                                                   |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `models/`      | TypeScript interfaces and types representing business data structures (e.g., `user.model.ts`).                                   |
| `slice/`       | RTK Query API slices and Redux Toolkit slices handling server-state caching, fetching, and mutations.                            |
| `view-models/` | Custom hooks (`use*.ts`) that encapsulate business logic, UI state manipulation, data filtering, and connect to RTK Query hooks. |
| `views/`       | Pure UI React components responsible only for layout, rendering, and passing user interactions to the ViewModel.                 |
| `routes/`      | Application routing configuration.                                                                                               |
| `utils/`       | Shared helper functions and utility modules.                                                                                     |
| `mock/`        | Mock data or MSW handlers for testing and local development.                                                                     |

---

## Data Flow Architecture

```text
[ View (UI Component) ]
          │
          ▼
    Invokes & Observes
          │
          ▼
[ ViewModel (Custom Hook) ]
          │
          ├─────────────────────────┐
          ▼                         ▼
[ RTK Query (`slice/`) ]     [ Local UI State (`useState`) ]
          │
          ▼
[ Backend API / Mock ]
```

### Flow Description

1. **View** calls the corresponding ViewModel hook, such as `useUserViewModel`.
2. **ViewModel** triggers RTK Query hooks from `slice/` to fetch or mutate server data.
3. **ViewModel** manages local UI state such as:
4. **ViewModel** processes, transforms, or filters the data.
5. **ViewModel** exposes a clean, flat state object to the View.
6. **View** renders the UI based solely on the state and actions provided by the ViewModel.

---

# Code Implementation Pattern

## 1. Model

**`src/models/user.model.ts`**

The Model defines the shape of the business entity.

```typescript
export interface User {
  id: string;
  name: string;
  email: string;
}
```

The model should contain the data structure and types required by the application without containing UI-specific logic.

---

## 2. RTK Query Slice

**`src/slice/userApi.ts`**

The RTK Query slice is responsible for API communication, server-state management, and caching.

```typescript
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';
import { User } from '../models/user.model';

export const userApi = createApi({
  reducerPath: 'userApi',
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),

  endpoints: (builder) => ({
    getUsers: builder.query<User[], void>({
      query: () => '/users',
    }),
  }),
});

export const { useGetUsersQuery } = userApi;
```

RTK Query handles:

* API requests
* Loading and error states
* Server-state caching
* Cache invalidation
* Refetching
* Query and mutation lifecycle management

---

## 3. ViewModel

**`src/view-models/useUserViewModel.ts`**

The ViewModel encapsulates data fetching, business logic, data transformation, and local UI state.

```typescript
import { useState } from 'react';
import { useGetUsersQuery } from '../slice/userApi';

export const useUserViewModel = () => {
  const { data: users, isLoading, error } = useGetUsersQuery();

  const [searchQuery, setSearchQuery] = useState('');

  const filteredUsers = users?.filter((user) =>
    user.name.toLowerCase().includes(searchQuery.toLowerCase())
  );

  return {
    users: filteredUsers,
    isLoading,
    error,
    searchQuery,
    setSearchQuery,
  };
};
```

The ViewModel acts as the bridge between the View and the application's data sources.

It can be responsible for:

* Calling RTK Query hooks
* Managing local UI state
* Filtering and sorting data
* Transforming API responses
* Handling user interactions
* Coordinating UI-related side effects
* Exposing a simplified interface to the View

---

## 4. View

**`src/views/UserView.tsx`**

The View is responsible for rendering the UI and handling user interactions through the ViewModel.

```tsx
import React from 'react';
import { useUserViewModel } from '../view-models/useUserViewModel';

export const UserView: React.FC = () => {
  const {
    users,
    isLoading,
    error,
    searchQuery,
    setSearchQuery,
  } = useUserViewModel();

  if (isLoading) {
    return <div>Loading...</div>;
  }

  if (error) {
    return <div>Failed to load users.</div>;
  }

  return (
    <div>
      <input
        type="text"
        value={searchQuery}
        onChange={(e) => setSearchQuery(e.target.value)}
        placeholder="Filter users..."
      />

      <ul>
        {users?.map((user) => (
          <li key={user.id}>{user.name}</li>
        ))}
      </ul>
    </div>
  );
};
```

The View should primarily focus on:

* Layout
* Rendering
* Accessibility
* Displaying ViewModel state
* Passing user interactions back to the ViewModel

The View should avoid containing complex business logic or directly interacting with APIs.

---

# Responsibility Breakdown

```text
┌─────────────────────────────────────────────┐
│                    VIEW                     │
│                                             │
│  • UI rendering                             │
│  • User interactions                        │
│  • Layout                                   │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│                 VIEWMODEL                   │
│                                             │
│  • UI state                                 │
│  • Business logic                           │
│  • Filtering / transformation               │
│  • Coordinates data access                  │
└───────────────┬─────────────────┬───────────┘
                │                 │
                ▼                 ▼
┌───────────────────────┐   ┌─────────────────┐
│      RTK QUERY        │   │   LOCAL STATE   │
│                       │   │                 │
│ • API requests        │   │ • Modals        │
│ • Caching             │   │ • Filters       │
│ • Server state        │   │ • Pagination    │
│ • Mutations           │   │ • UI state      │
└───────────┬───────────┘   └─────────────────┘
            │
            ▼
┌─────────────────────────────────────────────┐
│              BACKEND / MOCK                 │
└─────────────────────────────────────────────┘
```

---

# Benefits of This Architecture

## 1. Strict Separation of Concerns

Components remain clean and declarative.

* **Views** handle presentation.
* **ViewModels** handle business and UI logic.
* **RTK Query** handles server-state management.
* **Models** define application data structures.

This prevents API calls and complex business logic from being scattered throughout UI components.

## 2. Enhanced Testability

ViewModels can be tested independently of the complete React component tree using standard hook testing utilities.

For example, filtering logic can be tested without rendering the entire page.

## 3. Scalability

As the application grows, complex client-side transformations and UI behavior can remain inside ViewModels instead of cluttering React components or global Redux state.

## 4. Centralized Server-State Management

RTK Query provides a consistent approach for:

* Fetching server data
* Caching responses
* Managing loading states
* Handling errors
* Performing mutations
* Invalidating and refreshing cached data

## 5. Cleaner Views

Views receive a simple interface from the ViewModel:

```typescript
const {
  users,
  isLoading,
  error,
  searchQuery,
  setSearchQuery,
} = useUserViewModel();
```

The View does not need to know how the data is fetched, filtered, or transformed.

---

# Architectural Principle

The overall responsibility can be summarized as:

```text
Model
  ↓
Defines data structures

RTK Query / Slice
  ↓
Manages server state and API communication

ViewModel
  ↓
Combines server data with business and UI logic

View
  ↓
Renders the UI
```

The goal is to keep each layer focused on a single responsibility while allowing the **ViewModel to act as the primary bridge between the UI and application logic**.
