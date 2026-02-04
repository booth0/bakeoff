# Quickstart: Recipe Sharing Web Application

**Feature**: 001-recipe-sharing
**Date**: 2026-02-03

This guide walks you through setting up and running the Recipe Sharing application locally.

## Prerequisites

- Node.js 18.17 or later
- npm 9+ or pnpm 8+
- Supabase account (free tier works)
- Git

## 1. Clone and Install

```bash
# Clone the repository
git clone <repository-url>
cd bakeoff

# Install dependencies
npm install
```

## 2. Supabase Setup

### Create a Supabase Project

1. Go to [supabase.com](https://supabase.com) and sign in
2. Click "New Project"
3. Fill in:
   - **Name**: recipe-sharing (or your preference)
   - **Database Password**: Generate a strong password (save this!)
   - **Region**: Choose closest to your users
4. Click "Create new project" and wait for setup (~2 minutes)

### Get Your API Keys

1. In your Supabase project, go to **Settings > API**
2. Copy these values:
   - **Project URL** (e.g., `https://xxxxx.supabase.co`)
   - **anon public key** (safe for client-side)
   - **service_role key** (server-side only, keep secret!)

### Configure Environment Variables

Create a `.env.local` file in the project root:

```bash
# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# App
NEXT_PUBLIC_SITE_URL=http://localhost:3000
```

### Run Database Migrations

```bash
# Install Supabase CLI (if not installed)
npm install -g supabase

# Link to your project
supabase link --project-ref xxxxx

# Run migrations
supabase db push
```

Alternatively, run the SQL migrations manually in **Supabase Dashboard > SQL Editor**.

### Configure Storage Bucket

1. Go to **Storage** in Supabase Dashboard
2. Click "New bucket"
3. Configure:
   - **Name**: `recipe-images`
   - **Public bucket**: Yes
   - **File size limit**: 5MB
   - **Allowed MIME types**: `image/jpeg, image/png`
4. Click "Create bucket"

### Configure Authentication

1. Go to **Authentication > Providers**
2. Ensure **Email** is enabled
3. Go to **Authentication > URL Configuration**
4. Set **Site URL**: `http://localhost:3000`
5. Add to **Redirect URLs**:
   - `http://localhost:3000/auth/callback`
   - `http://localhost:3000/auth/verify`

## 3. Run Development Server

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) in your browser.

## 4. Verify Setup

### Test Authentication

1. Go to `/register`
2. Create an account with a valid email
3. Check your email for verification link
4. Click the link to verify
5. Log in at `/login`

### Test Recipe Creation

1. After logging in, go to `/recipes/new`
2. Create a test recipe:
   - **Title**: "Test Recipe"
   - **Ingredients**: Add at least one
   - **Instructions**: Add at least one step
3. Save the recipe
4. Verify it appears in `/my-recipes`

### Test Recipe Sharing

1. From your recipe detail page, click "Publish"
2. Go to the homepage (`/`)
3. Verify your recipe appears in the public feed
4. Copy the share link and test it in an incognito window

## Common Issues

### "Invalid API key"

- Check that `.env.local` has correct values
- Ensure no extra spaces or quotes around keys
- Restart the dev server after changing `.env.local`

### "Email not sending"

- Supabase free tier has email limits
- Check **Authentication > Logs** in Supabase Dashboard
- For local testing, use [Inbucket](https://inbucket.org/) or check Supabase logs

### "Storage upload failed"

- Verify the `recipe-images` bucket exists and is public
- Check file size (max 5MB) and type (JPEG/PNG only)
- Verify storage policies are correctly set

### "RLS policy error"

- Run all migrations in order
- Check that RLS is enabled on tables
- Verify policies exist in **Authentication > Policies**

## Development Commands

```bash
# Development server
npm run dev

# Run tests
npm test              # Unit tests
npm run test:e2e      # E2E tests (requires dev server)

# Type checking
npm run type-check

# Linting
npm run lint

# Format code
npm run format

# Build for production
npm run build
```

## Project Structure

```text
src/
├── app/              # Next.js pages and routes
├── components/       # React components
├── lib/              # Utilities and Supabase client
├── hooks/            # Custom React hooks
└── types/            # TypeScript definitions

supabase/
├── migrations/       # Database migrations
└── seed.sql          # Test data

tests/
├── e2e/              # Playwright tests
├── integration/      # Component tests
└── unit/             # Unit tests
```

## Next Steps

- Read [spec.md](./spec.md) for full feature requirements
- Read [data-model.md](./data-model.md) for database schema
- Read [contracts/api-routes.md](./contracts/api-routes.md) for API contracts
- Run `/speckit.tasks` to generate implementation tasks
