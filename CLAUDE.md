# CLAUDE.md - Next.js 15 + SQLite SaaS Template

> Opinionated, production-ready conventions for building SaaS applications with Next.js 15 App Router and SQLite (better-sqlite3).

## Stack & Versions

- **Framework:** Next.js 15 (App Router, not Pages Router)
- **Runtime:** Node.js 20+ (LTS)
- **Database:** SQLite via `better-sqlite3` (synchronous, fast, embedded)
- **Styling:** Tailwind CSS 4
- **Components:** shadcn/ui or Radix primitives
- **Auth:** Lucia Auth or NextAuth.js v5
- **Validation:** Zod (schemas everywhere)
- **Types:** TypeScript 5 (strict mode)

## Folder Structure

```
my-saas/
├── app/                    # Next.js App Router
│   ├── (auth)/            # Route groups for auth pages
│   │   ├── login/
│   │   ├── register/
│   │   └── layout.tsx     # Auth layout (no nav)
│   ├── (dashboard)/       # Route group for app
│   │   ├── dashboard/
│   │   ├── settings/
│   │   └── layout.tsx     # Dashboard layout (with nav)
│   ├── api/               # API routes
│   │   ├── auth/
│   │   └── webhook/
│   ├── layout.tsx         # Root layout
│   └── page.tsx           # Landing page
├── components/
│   ├── ui/                # Reusable UI primitives
│   ├── forms/             # Form-specific components
│   └── features/          # Domain-specific components
├── lib/
│   ├── db/                # Database layer
│   │   ├── index.ts       # Connection singleton
│   │   ├── schema.ts      # Table definitions
│   │   └── migrations/    # .sql migration files
│   ├── auth/              # Auth utilities
│   ├── validation/        # Zod schemas
│   └── utils.ts           # General utilities
├── types/
│   └── index.ts           # Shared TypeScript types
├── public/
└── scripts/
    ├── migrate.ts         # Run migrations
    └── seed.ts            # Development seed data
```

## SQL / Migration Conventions

### Migration File Naming
```
YYYYMMDDHHMMSS_descriptive_name.sql
```

Example: `20240325143000_create_users_table.sql`

### Schema Rules

1. **Always use INTEGER PRIMARY KEY for IDs**
   ```sql
   -- ✅ Good
   id INTEGER PRIMARY KEY AUTOINCREMENT,
   
   -- ❌ Bad
   id TEXT PRIMARY KEY,  -- Don't use UUIDs in SQLite
   ```

2. **Timestamps as Unix seconds (INTEGER)**
   ```sql
   -- ✅ Good
   created_at INTEGER DEFAULT (unixepoch()),
   updated_at INTEGER DEFAULT (unixepoch()),
   
   -- ❌ Bad
   created_at DATETIME DEFAULT CURRENT_TIMESTAMP,  -- Timezone issues
   ```

3. **Foreign keys with ON DELETE**
   ```sql
   user_id INTEGER NOT NULL,
   FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
   ```

4. **Indexes on foreign keys and search fields**
   ```sql
   CREATE INDEX idx_posts_user_id ON posts(user_id);
   CREATE INDEX idx_posts_created_at ON posts(created_at);
   ```

### Running Migrations

```bash
npm run migrate        # Run pending migrations
npm run migrate:fresh  # Drop, recreate, migrate, seed
```

## Component Patterns

### Server Components (Default)

```tsx
// app/dashboard/page.tsx
import { getUser } from '@/lib/auth';
import { db } from '@/lib/db';

export default async function DashboardPage() {
  const user = await getUser();
  if (!user) return redirect('/login');
  
  const posts = db.prepare('SELECT * FROM posts WHERE user_id = ?').all(user.id);
  
  return (
    <div>
      <h1>Welcome, {user.name}</h1>
      <PostList posts={posts} />
    </div>
  );
}
```

### Client Components (Explicit)

```tsx
// components/forms/CreatePostForm.tsx
'use client';

import { useState } from 'react';
import { createPost } from './actions';

export function CreatePostForm() {
  const [isPending, setIsPending] = useState(false);
  
  async function handleSubmit(formData: FormData) {
    setIsPending(true);
    await createPost(formData);
    setIsPending(false);
  }
  
  return (
    <form action={handleSubmit}>
      {/* Form fields */}
    </form>
  );
}
```

### Server Actions

```tsx
// components/forms/actions.ts
'use server';

import { revalidatePath } from 'next/cache';
import { db } from '@/lib/db';
import { createPostSchema } from '@/lib/validation';

export async function createPost(formData: FormData) {
  const data = Object.fromEntries(formData);
  const parsed = createPostSchema.parse(data);
  
  const result = db.prepare(`
    INSERT INTO posts (title, content, user_id)
    VALUES (?, ?, ?)
  `).run(parsed.title, parsed.content, parsed.userId);
  
  revalidatePath('/dashboard');
  return { id: result.lastInsertRowid };
}
```

## Database Layer

### Connection Singleton

```typescript
// lib/db/index.ts
import Database from 'better-sqlite3';

const db = new Database(process.env.DATABASE_URL || './data/app.db');
db.pragma('journal_mode = WAL');
db.pragma('foreign_keys = ON');

export { db };
```

### Type-Safe Queries

```typescript
// lib/db/types.ts
export interface User {
  id: number;
  email: string;
  name: string;
  createdAt: number;
  updatedAt: number;
}

// lib/db/queries.ts
import { db } from './index';
import type { User } from './types';

export function getUserByEmail(email: string): User | undefined {
  return db.prepare('SELECT * FROM users WHERE email = ?').get(email) as User | undefined;
}

export function createUser(data: Omit<User, 'id' | 'createdAt' | 'updatedAt'>): User {
  const result = db.prepare(`
    INSERT INTO users (email, name, password_hash)
    VALUES (?, ?, ?)
  `).run(data.email, data.name, data.passwordHash);
  
  return getUserById(Number(result.lastInsertRowid))!;
}
```

## Validation Patterns

### Zod Schemas

```typescript
// lib/validation/index.ts
import { z } from 'zod';

export const createPostSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(10).max(10000),
  published: z.boolean().default(false),
});

export const updateUserSchema = z.object({
  name: z.string().min(2).max(100).optional(),
  email: z.string().email().optional(),
}).refine(data => data.name || data.email, {
  message: "At least one field must be provided"
});
```

## Environment Variables

```bash
# .env.local (never commit)
DATABASE_URL="./data/app.db"
NEXTAUTH_SECRET="generate-with-openssl-rand-base64-32"
NEXTAUTH_URL="http://localhost:3000"
GITHUB_CLIENT_ID=""
GITHUB_CLIENT_SECRET=""
```

## Dev Commands

```bash
# Development
npm run dev              # Start dev server with turbopack

# Database
npm run migrate          # Run pending migrations
npm run migrate:fresh    # Reset database
npm run seed             # Add development data
npm run db:studio        # Open Drizzle Studio (if using Drizzle)

# Code quality
npm run lint             # ESLint
npm run typecheck        # TypeScript check
npm run format           # Prettier format
npm run test             # Run tests
npm run test:watch       # Watch mode
```

## What We DON'T Do (And Why)

| Anti-Pattern | Why We Avoid It | What We Do Instead |
|--------------|-----------------|-------------------|
| Pages Router | App Router is the future | Use App Router with async components |
| Prisma | Adds complexity, slower | Use better-sqlite3 directly |
| MongoDB | Overkill for SaaS | SQLite scales to millions of rows |
| Redux/Zustand | Server components reduce need | Server state + URL state + React Query |
| CSS Modules | Tailwind is faster | Tailwind with custom config |
| UUIDs as PK | Wasted space, slower | INTEGER PRIMARY KEY |
| Soft deletes | Complexity without benefit | Hard deletes with audit logs |
| Monorepo | Premature optimization | Single repo, separate when needed |

## Security Checklist

- [ ] Auth middleware on protected routes
- [ ] CSRF protection on all mutations
- [ ] Rate limiting on auth endpoints
- [ ] Input validation with Zod (never trust client)
- [ ] SQL injection prevention (use prepared statements)
- [ ] XSS protection (escape output, CSP headers)
- [ ] Secure session cookies (httpOnly, secure, sameSite)

## Performance Rules

1. **Use Server Components by default** — zero JS bundle impact
2. **Stream where possible** — use `loading.tsx` boundaries
3. **Database queries in parallel** — `Promise.all([...])`
4. **Cache aggressively** — `unstable_cache` for expensive queries
5. **Image optimization** — always use `next/image`

## Testing

```typescript
// __tests__/posts.test.ts
import { createPost } from '@/lib/db/queries';
import { db } from '@/lib/db';

beforeEach(() => {
  db.prepare('DELETE FROM posts').run();
});

test('createPost inserts and returns post', () => {
  const post = createPost({
    title: 'Test',
    content: 'Content',
    userId: 1,
  });
  
  expect(post.id).toBeDefined();
  expect(post.title).toBe('Test');
});
```

## Deployment

### Vercel (Recommended)

```bash
# vercel.json
{
  "buildCommand": "npm run migrate && npm run build"
}
```

Use Vercel Postgres for production (still SQLite-compatible Drizzle schema).

### Self-Hosted

```dockerfile
# Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run migrate
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

---

**This CLAUDE.md is a living document.** Update it as the project evolves. Every rule exists for a reason — if you break one, document why in a comment.
