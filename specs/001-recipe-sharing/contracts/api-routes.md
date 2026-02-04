# API Contracts: Recipe Sharing Web Application

**Feature**: 001-recipe-sharing
**Date**: 2026-02-03
**Architecture**: Next.js Server Actions + Supabase Client

## Overview

This application uses Next.js Server Actions for mutations and Supabase client queries for reads. This document defines the contracts for both patterns.

---

## Authentication (Supabase Auth)

Supabase Auth handles these flows directly. These are client-side operations using the Supabase client.

### Sign Up (FR-001, FR-002, FR-003)

```typescript
// Client-side using Supabase client
supabase.auth.signUp({
  email: string,
  password: string,
  options: {
    emailRedirectTo: string, // Verification redirect URL
    data: {
      display_name: string
    }
  }
})

// Response
{ data: { user, session }, error }
// Note: session is null until email verified
```

### Sign In (FR-004)

```typescript
supabase.auth.signInWithPassword({
  email: string,
  password: string
})

// Response
{ data: { user, session }, error }
```

### Sign Out (FR-007)

```typescript
supabase.auth.signOut()

// Response
{ error }
```

### Password Reset Request (FR-005)

```typescript
supabase.auth.resetPasswordForEmail(email, {
  redirectTo: string // Reset page URL
})

// Response
{ data: {}, error }
```

### Password Reset Confirm (FR-005)

```typescript
supabase.auth.updateUser({
  password: string
})

// Response
{ data: { user }, error }
```

---

## Server Actions

### Profile Actions

#### updateProfile

Updates the current user's profile.

```typescript
// Input
type UpdateProfileInput = {
  display_name: string // 1-100 chars, required
}

// Action
async function updateProfile(input: UpdateProfileInput): Promise<ActionResult<Profile>>

// Success Response
{
  success: true,
  data: {
    id: string,
    display_name: string,
    updated_at: string
  }
}

// Error Response
{
  success: false,
  error: {
    code: 'VALIDATION_ERROR' | 'UNAUTHORIZED' | 'SERVER_ERROR',
    message: string,
    details?: Record<string, string[]>
  }
}
```

---

### Recipe Actions

#### createRecipe (FR-009, FR-010, FR-012, FR-013)

Creates a new recipe for the authenticated user.

```typescript
// Input
type CreateRecipeInput = {
  title: string              // 1-200 chars, required
  description?: string       // 0-2000 chars
  prep_time_minutes?: number // 0-1440
  cook_time_minutes?: number // 0-1440
  servings?: number          // 1-100
  category?: Category        // Predefined enum
  ingredients: {
    quantity?: string        // 0-50 chars
    unit?: string            // 0-50 chars
    name: string             // 1-200 chars, required
  }[]                        // Min 1 ingredient
  instructions: {
    content: string          // 1-2000 chars, required
  }[]                        // Min 1 instruction
}

// Action
async function createRecipe(input: CreateRecipeInput): Promise<ActionResult<Recipe>>

// Success Response
{
  success: true,
  data: {
    id: string,
    title: string,
    // ... full recipe object
    created_at: string
  }
}

// Error Codes
'VALIDATION_ERROR' | 'UNAUTHORIZED' | 'SERVER_ERROR'
```

#### updateRecipe (FR-014)

Updates an existing recipe owned by the authenticated user.

```typescript
// Input
type UpdateRecipeInput = {
  id: string                 // Recipe ID, required
  title?: string
  description?: string
  prep_time_minutes?: number
  cook_time_minutes?: number
  servings?: number
  category?: Category
  ingredients?: {
    id?: string              // Existing ingredient ID (for update)
    quantity?: string
    unit?: string
    name: string
  }[]
  instructions?: {
    id?: string              // Existing instruction ID (for update)
    content: string
  }[]
}

// Action
async function updateRecipe(input: UpdateRecipeInput): Promise<ActionResult<Recipe>>

// Error Codes
'VALIDATION_ERROR' | 'UNAUTHORIZED' | 'NOT_FOUND' | 'FORBIDDEN' | 'SERVER_ERROR'
```

#### deleteRecipe (FR-015)

Deletes a recipe owned by the authenticated user.

```typescript
// Input
type DeleteRecipeInput = {
  id: string // Recipe ID, required
}

// Action
async function deleteRecipe(input: DeleteRecipeInput): Promise<ActionResult<void>>

// Success Response
{ success: true, data: null }

// Error Codes
'UNAUTHORIZED' | 'NOT_FOUND' | 'FORBIDDEN' | 'SERVER_ERROR'
```

#### publishRecipe (FR-017)

Makes a recipe publicly visible.

```typescript
// Input
type PublishRecipeInput = {
  id: string // Recipe ID, required
}

// Action
async function publishRecipe(input: PublishRecipeInput): Promise<ActionResult<Recipe>>

// Success Response includes published_at timestamp
```

#### unpublishRecipe (FR-018)

Removes a recipe from public visibility.

```typescript
// Input
type UnpublishRecipeInput = {
  id: string // Recipe ID, required
}

// Action
async function unpublishRecipe(input: UnpublishRecipeInput): Promise<ActionResult<Recipe>>
```

---

### Image Actions

#### uploadRecipeImage (FR-011)

Uploads an image for a recipe.

```typescript
// Input
type UploadImageInput = {
  recipe_id: string
  file: File // Max 5MB, JPEG or PNG
  position?: number
}

// Action
async function uploadRecipeImage(formData: FormData): Promise<ActionResult<RecipeImage>>

// Success Response
{
  success: true,
  data: {
    id: string,
    storage_path: string,
    url: string, // Public URL
    position: number
  }
}

// Error Codes
'VALIDATION_ERROR' | 'UNAUTHORIZED' | 'FORBIDDEN' | 'FILE_TOO_LARGE' | 'INVALID_FILE_TYPE' | 'MAX_IMAGES_EXCEEDED' | 'SERVER_ERROR'
```

#### deleteRecipeImage

Deletes an image from a recipe.

```typescript
// Input
type DeleteImageInput = {
  id: string // Image ID
}

// Action
async function deleteRecipeImage(input: DeleteImageInput): Promise<ActionResult<void>>
```

---

### Saved Recipe Actions

#### saveRecipe (FR-024)

Saves a public recipe to the user's collection.

```typescript
// Input
type SaveRecipeInput = {
  recipe_id: string
}

// Action
async function saveRecipe(input: SaveRecipeInput): Promise<ActionResult<SavedRecipe>>

// Error Codes
'UNAUTHORIZED' | 'NOT_FOUND' | 'ALREADY_SAVED' | 'CANNOT_SAVE_OWN' | 'SERVER_ERROR'
```

#### unsaveRecipe (FR-025)

Removes a recipe from the user's saved collection.

```typescript
// Input
type UnsaveRecipeInput = {
  recipe_id: string
}

// Action
async function unsaveRecipe(input: UnsaveRecipeInput): Promise<ActionResult<void>>
```

---

## Queries (Supabase Client)

### getPublicRecipes (FR-019)

Fetches published recipes for the public feed.

```typescript
// Query
const { data, error } = await supabase
  .from('recipes')
  .select(`
    id,
    title,
    description,
    prep_time_minutes,
    cook_time_minutes,
    servings,
    category,
    published_at,
    author:profiles(id, display_name),
    images:recipe_images(id, storage_path, position)
  `)
  .eq('published', true)
  .order('published_at', { ascending: false })
  .range(offset, offset + limit - 1)

// Response Type
type PublicRecipe = {
  id: string
  title: string
  description: string | null
  prep_time_minutes: number | null
  cook_time_minutes: number | null
  servings: number | null
  category: string | null
  published_at: string
  author: {
    id: string
    display_name: string
  }
  images: {
    id: string
    storage_path: string
    position: number
  }[]
}
```

### searchRecipes (FR-020, FR-021)

Searches recipes by title/ingredients with optional category filter.

```typescript
// Query Parameters
type SearchParams = {
  query?: string    // Search term
  category?: string // Category filter
  limit?: number    // Default 20
  offset?: number   // Default 0
}

// Uses PostgreSQL full-text search
const { data, error } = await supabase
  .rpc('search_recipes', {
    search_query: query,
    category_filter: category,
    result_limit: limit,
    result_offset: offset
  })
```

### getRecipeById (FR-023)

Fetches a single recipe with full details.

```typescript
// Query
const { data, error } = await supabase
  .from('recipes')
  .select(`
    *,
    author:profiles(id, display_name),
    ingredients(*),
    instructions(*),
    images:recipe_images(*)
  `)
  .eq('id', recipeId)
  .single()

// Response includes all related data
// RLS ensures only published or owned recipes are returned
```

### getMyRecipes

Fetches recipes owned by the authenticated user.

```typescript
const { data, error } = await supabase
  .from('recipes')
  .select(`
    id,
    title,
    published,
    created_at,
    updated_at,
    images:recipe_images(id, storage_path, position)
  `)
  .eq('author_id', userId)
  .order('updated_at', { ascending: false })
```

### getSavedRecipes (FR-024)

Fetches recipes saved by the authenticated user.

```typescript
const { data, error } = await supabase
  .from('saved_recipes')
  .select(`
    id,
    created_at,
    recipe:recipes(
      id,
      title,
      author:profiles(id, display_name),
      images:recipe_images(id, storage_path, position)
    )
  `)
  .eq('user_id', userId)
  .order('created_at', { ascending: false })
```

### checkIfSaved (FR-026)

Checks if a recipe is saved by the current user.

```typescript
const { data, error } = await supabase
  .from('saved_recipes')
  .select('id')
  .eq('user_id', userId)
  .eq('recipe_id', recipeId)
  .maybeSingle()

// Returns null if not saved, object with id if saved
```

---

## Type Definitions

```typescript
// Shared types for the application

type ActionResult<T> =
  | { success: true; data: T }
  | { success: false; error: ActionError }

type ActionError = {
  code: string
  message: string
  details?: Record<string, string[]>
}

type Category =
  | 'Breakfast'
  | 'Lunch'
  | 'Dinner'
  | 'Appetizer'
  | 'Side Dish'
  | 'Dessert'
  | 'Beverage'
  | 'Snack'
  | 'Soup'
  | 'Salad'
  | 'Sauce'
  | 'Other'

type Profile = {
  id: string
  display_name: string
  created_at: string
  updated_at: string
}

type Recipe = {
  id: string
  author_id: string
  title: string
  description: string | null
  prep_time_minutes: number | null
  cook_time_minutes: number | null
  servings: number | null
  category: Category | null
  published: boolean
  published_at: string | null
  created_at: string
  updated_at: string
}

type Ingredient = {
  id: string
  recipe_id: string
  position: number
  quantity: string | null
  unit: string | null
  name: string
  created_at: string
}

type Instruction = {
  id: string
  recipe_id: string
  step_number: number
  content: string
  created_at: string
}

type RecipeImage = {
  id: string
  recipe_id: string
  storage_path: string
  position: number
  created_at: string
}

type SavedRecipe = {
  id: string
  user_id: string
  recipe_id: string
  created_at: string
}
```

---

## Error Codes Reference

| Code | HTTP Equivalent | Description |
|------|-----------------|-------------|
| VALIDATION_ERROR | 400 | Input validation failed |
| UNAUTHORIZED | 401 | User not authenticated |
| FORBIDDEN | 403 | User lacks permission |
| NOT_FOUND | 404 | Resource not found |
| ALREADY_SAVED | 409 | Recipe already in saved collection |
| CANNOT_SAVE_OWN | 409 | Cannot save own recipe |
| FILE_TOO_LARGE | 413 | Image exceeds 5MB limit |
| INVALID_FILE_TYPE | 415 | File is not JPEG or PNG |
| MAX_IMAGES_EXCEEDED | 422 | Recipe already has 10 images |
| SERVER_ERROR | 500 | Internal server error |
