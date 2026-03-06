---
description: "Build a React component or page: structure, state management, data fetching with React Query, form handling, accessibility, and tests"
argument-hint: "[component description, e.g. 'user profile edit page with avatar upload']"
---

## React Component Builder

Build a production-quality React component or page.

### Phase 1 — Component Design

Before writing code, define:
- **Responsibility**: what does this component do? (single responsibility check)
- **Props interface**: what data/callbacks does it receive?
- **State**: local only (useState), global (Zustand), or server (React Query)?
- **User interactions**: what events does it handle?
- **Edge states**: loading, error, empty, disabled?

### Phase 2 — Data Layer

If the component fetches data:
- Write React Query hook (`useQuery` or `useMutation`)
- Define query key that includes all relevant params
- Set appropriate `staleTime` for the data type
- Handle loading/error/empty states explicitly
- Use optimistic updates for mutations that change visible state

```javascript
// hooks/useUserProfile.js
export function useUserProfile(userId) {
  return useQuery({
    queryKey: ['users', userId],
    queryFn: () => userApi.get(userId),
    enabled: Boolean(userId),
  });
}
```

### Phase 3 — Component Structure

Apply composition patterns:
- Split into container (data) + presentation (UI) components
- Use compound components for complex UI with shared state
- Extract reusable sub-components only if used in 2+ places

File structure for a page:
```
features/profile/
  components/
    ProfileHeader.jsx      # presentation
    ProfileForm.jsx        # form component
    AvatarUpload.jsx       # sub-component
  hooks/
    useUserProfile.js      # data fetching
    useAvatarUpload.js     # upload logic
  ProfilePage.jsx          # container
```

### Phase 4 — Form Handling

If the component has a form:
- Use React Hook Form with Zod schema validation
- Show field-level errors beneath each input
- Handle server errors: map to field errors or global error state
- Disable submit button while submitting
- Show success feedback (toast, redirect, or inline message)

### Phase 5 — Performance

Apply optimizations only where justified:
- `React.memo` — only for components that re-render frequently with same props
- `useMemo` — only for expensive computations (> 1ms)
- `useCallback` — only for functions passed to memoized children
- `key` props — always stable IDs, never array index
- Lazy loading with `React.lazy` + `Suspense` for route-level code splitting

### Phase 6 — Accessibility

Verify against WCAG 2.1 AA:
- All form inputs have associated `<label>` (htmlFor + id)
- Buttons have descriptive text (not just "X" or icon-only)
- Interactive elements keyboard accessible (Tab, Enter, Space, Escape)
- Error messages linked to fields with `aria-describedby`
- Loading states announced with `aria-live` or `role="status"`
- Color contrast meets 4.5:1 minimum (don't rely on color alone)

### Phase 7 — Tests

Write tests covering:
- Happy path: user fills and submits form, sees success state
- Validation errors: empty submit shows field errors
- Server errors: API failure shows error message
- Loading state: skeleton/spinner visible during fetch
- Accessibility: no role/label violations (axe-core)

```javascript
// Avoid testing implementation — test behavior
// Bad: expect(wrapper.state().isLoading).toBe(true)
// Good: expect(screen.getByText(/loading.../i)).toBeInTheDocument()
```

### Output

1. **Component design summary** (responsibility, props, state decisions)
2. **Complete component code** (all files with full implementation)
3. **React Query hooks** if data fetching required
4. **Test file** with 4-6 meaningful test cases
5. **Accessibility checklist** for the specific component
