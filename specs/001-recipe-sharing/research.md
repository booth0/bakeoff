# Research: Recipe Sharing Web Application

**Feature**: 001-recipe-sharing
**Date**: 2026-02-03
**Status**: Complete

## Technology Decisions

### 1. Framework: Next.js 14 with App Router

**Decision**: Use Next.js 14 with App Router and Server Components

**Rationale**:
- App Router provides built-in layouts, loading states, and error boundaries
- Server Components reduce client-side JavaScript bundle
- Server Actions simplify form handling and data mutations
- Built-in image optimization for recipe photos
- Excellent TypeScript support
- Large ecosystem and community support

**Alternatives Considered**:
- Remix: Good alternative but smaller ecosystem; Next.js more mature
- Vite + React: Would require separate backend; more setup overhead
- SvelteKit: Less familiar ecosystem; fewer UI component libraries

### 2. Backend: Supabase

**Decision**: Use Supabase for authentication, database, and file storage

**Rationale**:
- **Authentication**: Built-in email/password auth with email verification (covers FR-001 through FR-008)
- **Database**: PostgreSQL with Row Level Security for private/public recipes (covers FR-028)
- **Storage**: Built-in file storage for recipe images with automatic resizing
- **Real-time**: Available if needed for future features
- **Generous free tier**: Suitable for initial development and launch

**Alternatives Considered**:
- Firebase: Firestore less suitable for relational recipe data; more complex queries
- Self-hosted PostgreSQL + NextAuth: More setup and maintenance; auth from scratch
- PlanetScale: Good DB but would need separate auth and storage solutions

### 3. Styling: Tailwind CSS

**Decision**: Use Tailwind CSS with custom design tokens

**Rationale**:
- Utility-first approach speeds development
- Custom design tokens ensure consistency (Constitution III)
- Excellent responsive design utilities (mobile-first)
- Good accessibility defaults
- Tree-shaking keeps bundle small

**Alternatives Considered**:
- CSS Modules: More boilerplate; harder to maintain consistency
- Styled Components: Runtime overhead; less performant
- shadcn/ui: Will use as component library on top of Tailwind

### 4. Form Handling: React Hook Form + Zod

**Decision**: Use React Hook Form with Zod validation

**Rationale**:
- Minimal re-renders for complex recipe forms
- Zod provides type-safe validation schemas
- Schemas reusable between client and server
- Good accessibility support for error messages

**Alternatives Considered**:
- Formik: Heavier; more re-renders on large forms
- Native form handling: More boilerplate; less validation support

### 5. Testing Strategy

**Decision**: Vitest (unit) + Testing Library (integration) + Playwright (e2e)

**Rationale**:
- **Vitest**: Fast, Vite-native, excellent TypeScript support
- **Testing Library**: Tests user behavior, not implementation
- **Playwright**: Cross-browser e2e testing; good for auth flows

**Test Coverage Plan**:
| Area | Coverage Target | Test Type |
|------|-----------------|-----------|
| Auth flows | 100% | E2E + Integration |
| Recipe CRUD | 100% | E2E + Integration |
| Form validation | 100% | Unit |
| UI components | 80% | Integration |
| Search/filter | 90% | Integration + E2E |

## Supabase Implementation Details

### Authentication Configuration

```text
Supabase Auth Settings:
- Email provider: Enabled
- Email confirmations: Required (FR-003)
- Password minimum length: 8 characters
- Custom password validation: Add via client-side Zod schema (FR-008)
- Session duration: 30 days (FR-006)
- Password reset: Enabled (FR-005)
```

### Row Level Security (RLS) Policies

```text
recipes table:
- SELECT: published = true OR author_id = auth.uid()
- INSERT: auth.uid() IS NOT NULL
- UPDATE: author_id = auth.uid()
- DELETE: author_id = auth.uid()

saved_recipes table:
- SELECT: user_id = auth.uid()
- INSERT: user_id = auth.uid()
- DELETE: user_id = auth.uid()

profiles table:
- SELECT: true (display names are public)
- UPDATE: id = auth.uid()
```

### Storage Buckets

```text
recipe-images bucket:
- Public: true (published recipe images viewable by all)
- File size limit: 5MB
- Allowed MIME types: image/jpeg, image/png
- Transformations: thumbnail (200x200), medium (800x600)
```

## Search Implementation

**Decision**: PostgreSQL full-text search via Supabase

**Rationale**:
- Built into Supabase, no additional service
- Sufficient for initial scale (10,000 recipes)
- Supports title and ingredient search
- Can add trigram similarity for fuzzy matching

**Implementation**:
- Create GIN index on recipe title and ingredient names
- Use `to_tsvector` and `plainto_tsquery` for search
- Category filter via simple WHERE clause

**Future Enhancement**: If search needs improve, consider Algolia or Meilisearch

## Performance Considerations

### Image Optimization
- Use Next.js Image component with Supabase Storage URLs
- Generate thumbnails on upload (200x200 for cards)
- Lazy load images below the fold
- Use WebP format where supported

### Database Optimization
- Index: `recipes(published, created_at DESC)` for feed
- Index: `recipes(author_id)` for my-recipes
- Index: `saved_recipes(user_id)` for saved collection
- Full-text index on recipe title and ingredients

### Caching Strategy
- Static generation for public recipe pages with ISR (revalidate: 60)
- Client-side caching with React Query or SWR for user-specific data
- Supabase edge caching for API responses

## Security Considerations

### Input Validation
- All user input validated with Zod schemas
- Server-side validation on all mutations
- Sanitize HTML in recipe instructions (prevent XSS)

### Authentication
- Supabase handles password hashing (bcrypt)
- CSRF protection via Supabase client
- Secure session tokens (httpOnly cookies)

### Authorization
- RLS policies enforce data access at database level
- Server Components verify auth before data fetch
- Client-side auth state via Supabase Auth helpers

## Dependencies Summary

```json
{
  "dependencies": {
    "next": "^14.0.0",
    "@supabase/supabase-js": "^2.38.0",
    "@supabase/ssr": "^0.1.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-hook-form": "^7.48.0",
    "@hookform/resolvers": "^3.3.0",
    "zod": "^3.22.0",
    "tailwindcss": "^3.3.0"
  },
  "devDependencies": {
    "typescript": "^5.2.0",
    "@types/react": "^18.2.0",
    "@types/node": "^20.8.0",
    "vitest": "^1.0.0",
    "@testing-library/react": "^14.0.0",
    "@playwright/test": "^1.40.0",
    "eslint": "^8.52.0",
    "eslint-config-next": "^14.0.0",
    "prettier": "^3.0.0"
  }
}
```

## Open Questions Resolved

| Question | Resolution |
|----------|------------|
| Image storage location | Supabase Storage with public bucket |
| Search implementation | PostgreSQL full-text search |
| Session management | Supabase Auth with 30-day sessions |
| Email delivery | Supabase built-in email (can upgrade to custom SMTP) |
| Deployment target | Vercel (optimal for Next.js) |
