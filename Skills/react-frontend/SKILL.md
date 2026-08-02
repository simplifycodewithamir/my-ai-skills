---
name: react-frontend
description: Use this skill whenever writing or reviewing React/TypeScript frontend code — components, hooks, state management, styling, or accessibility. Trigger for "build a component", "create a page", "add a form", or any UI work, even if the user doesn't say "React" explicitly.
---

# React Frontend Conventions

## Do
- React with **TypeScript**, functional components and hooks.
- Keep **server state** in a query library (e.g. TanStack Query); keep **local state** in components.
- Accessibility floor is non-negotiable: real `<label>`s, keyboard reachability, visible focus states.
- **Tailwind CSS** for styling.
- Build **responsive** UI by default.

## Don't
- Don't write class components.
- Don't reach for global state (Redux/Context-as-global-store) early — local/component state and a query library cover most cases.
- Don't ship a component with no keyboard access or no visible focus indicator.
- Don't skip labels on form inputs, even for "obvious" fields.

## Output expectations
- Components are functional, typed, accessible, and responsive out of the box.
- Server-derived data goes through the query library's caching/loading/error states, not ad-hoc `useEffect` fetches.
