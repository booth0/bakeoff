# Data Model: Recipe Sharing Web Application

**Feature**: 001-recipe-sharing
**Date**: 2026-02-03
**Database**: Supabase PostgreSQL

## Entity Relationship Diagram

```text
┌─────────────────┐       ┌─────────────────────┐
│     profiles    │       │       recipes       │
├─────────────────┤       ├─────────────────────┤
│ id (PK, FK)     │───┐   │ id (PK)             │
│ display_name    │   │   │ author_id (FK)      │──┐
│ created_at      │   │   │ title               │  │
│ updated_at      │   └──►│ description         │  │
└─────────────────┘       │ prep_time_minutes   │  │
                          │ cook_time_minutes   │  │
                          │ servings            │  │
                          │ category            │  │
                          │ published           │  │
                          │ published_at        │  │
                          │ created_at          │  │
                          │ updated_at          │  │
                          └─────────────────────┘  │
                                    │              │
                    ┌───────────────┼──────────────┘
                    │               │
                    ▼               ▼
        ┌───────────────────┐   ┌───────────────────┐
        │    ingredients    │   │   instructions    │
        ├───────────────────┤   ├───────────────────┤
        │ id (PK)           │   │ id (PK)           │
        │ recipe_id (FK)    │   │ recipe_id (FK)    │
        │ position          │   │ step_number       │
        │ quantity          │   │ content           │
        │ unit              │   │ created_at        │
        │ name              │   └───────────────────┘
        │ created_at        │
        └───────────────────┘

        ┌───────────────────┐   ┌───────────────────┐
        │   recipe_images   │   │   saved_recipes   │
        ├───────────────────┤   ├───────────────────┤
        │ id (PK)           │   │ id (PK)           │
        │ recipe_id (FK)    │   │ user_id (FK)      │
        │ storage_path      │   │ recipe_id (FK)    │
        │ position          │   │ created_at        │
        │ created_at        │   └───────────────────┘
        └───────────────────┘
```

## Tables

### profiles

Extends Supabase auth.users with public profile information.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | uuid | PK, FK → auth.users(id) | User ID from Supabase Auth |
| display_name | varchar(100) | NOT NULL | Public display name (FR-029, FR-030) |
| created_at | timestamptz | NOT NULL, DEFAULT now() | Profile creation timestamp |
| updated_at | timestamptz | NOT NULL, DEFAULT now() | Last update timestamp |

**Indexes**:
- PRIMARY KEY (id)

**RLS Policies**:
- SELECT: `true` (display names are public)
- UPDATE: `id = auth.uid()` (users can only update their own profile)

---

### recipes

Main recipe entity containing core recipe information.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | uuid | PK, DEFAULT gen_random_uuid() | Unique recipe identifier |
| author_id | uuid | NOT NULL, FK → profiles(id) | Recipe owner (FR-009) |
| title | varchar(200) | NOT NULL | Recipe title |
| description | text | NULL | Optional description (FR-010) |
| prep_time_minutes | integer | NULL, CHECK >= 0 | Preparation time (FR-010) |
| cook_time_minutes | integer | NULL, CHECK >= 0 | Cooking time (FR-010) |
| servings | integer | NULL, CHECK > 0 | Number of servings (FR-010) |
| category | varchar(50) | NULL | Recipe category (FR-010, FR-021) |
| published | boolean | NOT NULL, DEFAULT false | Public visibility (FR-017, FR-028) |
| published_at | timestamptz | NULL | When recipe was published (FR-019) |
| created_at | timestamptz | NOT NULL, DEFAULT now() | Creation timestamp |
| updated_at | timestamptz | NOT NULL, DEFAULT now() | Last modified (FR-016) |

**Indexes**:
- PRIMARY KEY (id)
- INDEX idx_recipes_author (author_id)
- INDEX idx_recipes_published_date (published, published_at DESC) WHERE published = true
- INDEX idx_recipes_category (category) WHERE published = true
- GIN INDEX idx_recipes_search ON to_tsvector('english', title)

**RLS Policies**:
- SELECT: `published = true OR author_id = auth.uid()` (FR-028)
- INSERT: `auth.uid() IS NOT NULL` (FR-009)
- UPDATE: `author_id = auth.uid()` (FR-014)
- DELETE: `author_id = auth.uid()` (FR-015)

**Triggers**:
- `update_updated_at`: Set updated_at = now() on UPDATE
- `set_published_at`: Set published_at = now() when published changes to true

---

### ingredients

Individual ingredients belonging to a recipe.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | uuid | PK, DEFAULT gen_random_uuid() | Unique ingredient identifier |
| recipe_id | uuid | NOT NULL, FK → recipes(id) ON DELETE CASCADE | Parent recipe |
| position | integer | NOT NULL, CHECK >= 0 | Order in ingredient list |
| quantity | varchar(50) | NULL | Amount (e.g., "2", "1/2") (FR-012) |
| unit | varchar(50) | NULL | Unit of measure (e.g., "cups") (FR-012) |
| name | varchar(200) | NOT NULL | Ingredient name (FR-012) |
| created_at | timestamptz | NOT NULL, DEFAULT now() | Creation timestamp |

**Indexes**:
- PRIMARY KEY (id)
- INDEX idx_ingredients_recipe (recipe_id, position)
- GIN INDEX idx_ingredients_search ON to_tsvector('english', name)

**RLS Policies**:
- Inherit from parent recipe (check recipe access via subquery)

**Constraints**:
- UNIQUE (recipe_id, position)

---

### instructions

Step-by-step cooking instructions for a recipe.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | uuid | PK, DEFAULT gen_random_uuid() | Unique instruction identifier |
| recipe_id | uuid | NOT NULL, FK → recipes(id) ON DELETE CASCADE | Parent recipe |
| step_number | integer | NOT NULL, CHECK > 0 | Step order (FR-013) |
| content | text | NOT NULL | Instruction text (FR-013) |
| created_at | timestamptz | NOT NULL, DEFAULT now() | Creation timestamp |

**Indexes**:
- PRIMARY KEY (id)
- INDEX idx_instructions_recipe (recipe_id, step_number)

**RLS Policies**:
- Inherit from parent recipe (check recipe access via subquery)

**Constraints**:
- UNIQUE (recipe_id, step_number)

---

### recipe_images

Photos associated with a recipe, stored in Supabase Storage.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | uuid | PK, DEFAULT gen_random_uuid() | Unique image identifier |
| recipe_id | uuid | NOT NULL, FK → recipes(id) ON DELETE CASCADE | Parent recipe |
| storage_path | varchar(500) | NOT NULL | Path in Supabase Storage (FR-011) |
| position | integer | NOT NULL, DEFAULT 0, CHECK >= 0 | Display order |
| created_at | timestamptz | NOT NULL, DEFAULT now() | Upload timestamp |

**Indexes**:
- PRIMARY KEY (id)
- INDEX idx_recipe_images_recipe (recipe_id, position)

**RLS Policies**:
- Inherit from parent recipe (check recipe access via subquery)

**Constraints**:
- Maximum 10 images per recipe enforced at application level (FR-011)

---

### saved_recipes

Junction table for users saving other users' recipes.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | uuid | PK, DEFAULT gen_random_uuid() | Unique save identifier |
| user_id | uuid | NOT NULL, FK → profiles(id) ON DELETE CASCADE | User who saved (FR-024) |
| recipe_id | uuid | NOT NULL, FK → recipes(id) ON DELETE CASCADE | Saved recipe (FR-027) |
| created_at | timestamptz | NOT NULL, DEFAULT now() | When saved |

**Indexes**:
- PRIMARY KEY (id)
- UNIQUE INDEX idx_saved_unique (user_id, recipe_id)
- INDEX idx_saved_user (user_id, created_at DESC)

**RLS Policies**:
- SELECT: `user_id = auth.uid()` (FR-024)
- INSERT: `user_id = auth.uid()` (FR-024)
- DELETE: `user_id = auth.uid()` (FR-025)

---

## Supabase Storage

### Bucket: recipe-images

| Setting | Value |
|---------|-------|
| Public | true |
| File size limit | 5MB (FR-011) |
| Allowed MIME types | image/jpeg, image/png (FR-011) |

**Path Convention**: `{user_id}/{recipe_id}/{filename}`

**Storage Policies**:
- INSERT: `auth.uid()::text = (storage.foldername(name))[1]`
- DELETE: `auth.uid()::text = (storage.foldername(name))[1]`
- SELECT: Public (all files readable)

---

## Validation Rules

### Recipe
- title: 1-200 characters, required
- description: 0-2000 characters, optional
- prep_time_minutes: 0-1440 (24 hours max), optional
- cook_time_minutes: 0-1440 (24 hours max), optional
- servings: 1-100, optional
- category: One of predefined categories, optional

### Ingredient
- quantity: 0-50 characters, optional
- unit: 0-50 characters, optional
- name: 1-200 characters, required
- Minimum 1 ingredient per recipe (FR-009)

### Instruction
- step_number: Sequential starting from 1
- content: 1-2000 characters, required
- Minimum 1 instruction per recipe (FR-009)

### Profile
- display_name: 1-100 characters, required (FR-030)

### Image
- Maximum 10 images per recipe (FR-011)
- Maximum 5MB per image (FR-011)
- Allowed types: JPEG, PNG (FR-011)

---

## Categories Enum

Predefined recipe categories (FR-021):

```text
- Breakfast
- Lunch
- Dinner
- Appetizer
- Side Dish
- Dessert
- Beverage
- Snack
- Soup
- Salad
- Sauce
- Other
```

---

## Migration Order

1. Create `profiles` table with trigger for new user signup
2. Create `recipes` table
3. Create `ingredients` table
4. Create `instructions` table
5. Create `recipe_images` table
6. Create `saved_recipes` table
7. Create search indexes
8. Apply RLS policies
9. Configure storage bucket
