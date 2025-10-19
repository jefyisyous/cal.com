# GitHub Copilot Instructions for Cal.com

This document provides context and guidelines for GitHub Copilot when working on the Cal.com codebase.

## Project Overview

Cal.com is an open-source scheduling platform built as a monorepo using Yarn workspaces and Turbo for build orchestration. The main application is a Next.js application with a PostgreSQL database using Prisma ORM.

## Repository Structure

- `apps/web/` - Main Next.js application (work primarily in this folder)
- `packages/prisma/` - Database schema and migrations
- `packages/trpc/` - API layer using tRPC
- `packages/ui/` - Shared UI components
- `packages/features/` - Feature-specific code
- `packages/app-store/` - Third-party app integrations

## Quick Command Reference

### Development
- `yarn dev` - Start development server
- `yarn dx` - Start development with database setup
- `yarn build` - Build all packages
- `yarn lint:fix` - Lint and fix code
- `yarn type-check` - TypeScript checking

### Testing
- `yarn test <filename>` - Run unit tests
- `yarn test <filename> -- --integrationTestsOnly` - Run integration tests
- `yarn e2e <filename> --grep "<testName>"` - Run specific E2E test
- Always run tests with `TZ=UTC` for consistent timezone handling

### Database
- `yarn workspace @calcom/prisma db-migrate` - Development migrations
- `yarn workspace @calcom/prisma db-deploy` - Production deployment
- `yarn prisma generate` - Update TypeScript types after schema changes

## Coding Standards

### Import Guidelines

**Type Imports:**
```typescript
// ✅ Good - Use type imports
import type { User } from "@prisma/client";
import type { NextApiRequest, NextApiResponse } from "next";

// ❌ Bad - Regular import for types
import { User } from "@prisma/client";
```

**Avoid Barrel Imports:**
```typescript
// ❌ Bad - Avoid importing from index.ts barrel files
import { Button } from "@calcom/ui";

// ✅ Good - Import directly from source files
import { Button } from "@calcom/ui/components/button";
```

### File Naming Conventions

- **Components**: PascalCase (e.g., `BookingForm.tsx`)
- **Utilities**: kebab-case (e.g., `date-utils.ts`)
- **Types**: PascalCase with `.types.ts` suffix (e.g., `Booking.types.ts`)
- **Tests**: Same as source file + `.test.ts` or `.spec.ts`
- **Repositories**: Must include `Repository` suffix (e.g., `PrismaBookingRepository.ts`)
- **Services**: Must include `Service` suffix (e.g., `MembershipService.ts`)

### Code Structure

**Early Returns:**
```typescript
// ✅ Good - Use early returns to reduce nesting
if (!booking) return null;
if (!user.isAdmin) throw new Error("Unauthorized");
```

**Composition Over Prop Drilling:**
- Use React children and context instead of passing props through multiple components

### Error Handling

```typescript
// ✅ Good - Descriptive error with context
throw new Error(`Unable to create booking: User ${userId} has no available time slots for ${date}`);

// ❌ Bad - Generic error
throw new Error("Booking failed");

// ✅ Good - Use proper error classes
import { TRPCError } from "@trpc/server";

throw new TRPCError({
  code: "BAD_REQUEST",
  message: "Invalid booking time slot",
});
```

## Database Guidelines

### Prefer Select Over Include

```typescript
// ✅ Good - Use select for performance and security
const booking = await prisma.booking.findFirst({
  select: {
    id: true,
    title: true,
    user: {
      select: {
        id: true,
        name: true,
        email: true,
      }
    }
  }
});

// ❌ Bad - Include fetches all fields
const booking = await prisma.booking.findFirst({
  include: {
    user: true, // This gets ALL user fields
  }
});
```

### Security Rules

```typescript
// ❌ NEVER expose credential keys
const user = await prisma.user.findFirst({
  select: {
    credentials: {
      select: {
        key: true, // ❌ NEVER do this
      }
    }
  }
});

// ✅ Good - Never select credential.key field
const user = await prisma.user.findFirst({
  select: {
    credentials: {
      select: {
        id: true,
        type: true,
        // key field is excluded for security
      }
    }
  }
});
```

### Schema Changes

After making changes to `packages/prisma/schema.prisma`:
1. Run `yarn prisma generate` to update TypeScript types
2. Create migration: `npx prisma migrate dev --name migration_name`
3. Always squash migrations as per [Prisma docs](https://www.prisma.io/docs/orm/prisma-migrate/workflows/squashing-migrations)

## Repository Pattern

### Method Naming Rules

**1. Don't include the repository's entity name in method names:**
```typescript
// ✅ Good - Concise method names
class BookingRepository {
  findById(id: string) { ... }
  findByUserId(userId: string) { ... }
  create(data: BookingCreateInput) { ... }
}

// ❌ Bad - Redundant entity name
class BookingRepository {
  findBookingById(id: string) { ... }
  createBooking(data: BookingCreateInput) { ... }
}
```

**2. Use `include` or similar keywords for methods that fetch relational data:**
```typescript
// ✅ Good - Clear indication of included relations
class EventTypeRepository {
  findById(id: string) { ... }
  findByIdIncludeHosts(id: string) { ... }
  findByIdIncludeHostsAndSchedule(id: string) { ... }
}
```

**3. Keep methods generic and reusable:**
```typescript
// ✅ Good - Generic, reusable
class BookingRepository {
  findByUserIdIncludeAttendees(userId: string) { ... }
}

// ❌ Bad - Use-case-specific
class BookingRepository {
  findBookingsForReporting(userId: string) { ... }
}
```

**4. No business logic in repositories:**
- Repositories handle data access only
- Business logic belongs in the Service layer

## Performance Guidelines

### Basic Rules
- Aim for O(n) or O(n log n) complexity, avoid O(n²)
- Use database-level filtering instead of JavaScript filtering
- Consider pagination for large datasets
- Use database transactions for related operations

### Day.js Usage

```typescript
// ⚠️ Slow in performance-critical code (loops)
dates.map((date) => dayjs(date).add(1, "day").format());

// ✅ Better - Use .utc() for performance
dates.map((date) => dayjs.utc(date).add(1, "day").format());

// ✅ Best - Use native Date when possible
dates.map((date) => new Date(date.valueOf() + 24 * 60 * 60 * 1000));
```

**Prefer date-fns over Day.js when timezone awareness isn't critical:**
- Use `startOfMonth(dateObj)` / `endOfDay(dateObj)` instead of `dayjs.startOf(...)`
- Use built-in `Intl` API with `i18n.language` for locale-dependent formatting
- Day.js's plugin system is heavy and preloads all plugins including locale handling

## Next.js App Directory Guidelines

### Authorization Checks in Pages

**TL;DR:** Always put permission checks directly in `page.tsx` or relevant server components, not in `layout.tsx`.

```tsx
// ✅ Good - Permission check in page.tsx
// app/admin/page.tsx
import { redirect } from "next/navigation";
import { getUserSession } from "@/lib/auth";

export default async function AdminPage() {
  const session = await getUserSession();

  if (!session || session.user.role !== "admin") {
    redirect("/");
  }

  // Protected content here
  return <div>Welcome, Admin!</div>;
}
```

**Why not in layouts:**
- Layouts don't intercept all requests
- APIs and server actions bypass layouts
- Risk of data leaks

## Testing Strategy

### Priority Order When Fixing Issues

1. Run `yarn type-check:ci --force` to identify TypeScript errors
2. Run `yarn test` to identify failing unit tests
3. Fix type errors first (they often cause test failures)
4. Address failing tests incrementally, one file at a time

### Running Tests

```bash
# Unit test specific file
yarn vitest run packages/lib/some-file.test.ts

# Integration test specific file
yarn test routing-form-response.integration-test.ts -- --integrationTestsOnly

# E2E test specific file
PLAYWRIGHT_HEADLESS=1 yarn e2e tests/booking-flow.e2e.ts

# Run specific test by name
yarn e2e tests/booking-flow.e2e.ts --grep "should create booking"
```

### Test Development Guidelines

- Always ensure Playwright tests pass locally before pushing
- Take an incremental approach: fix one file at a time
- Use `TZ=UTC` for consistent timezone handling
- For E2E tests, use: `PLAYWRIGHT_HEADLESS=1 yarn e2e [test-file.e2e.ts]`

## PR Requirements

### Title Format
- Use conventional commits: `feat:`, `fix:`, `refactor:`
- Be specific: `fix: handle timezone edge case in booking creation`
- Not generic: `fix: booking bug`

### Size Limits
- Large PRs (>500 lines or >10 files) should be split
- Pattern: Database → Backend → Frontend → Tests

### Checklist
- PR title must follow Conventional Commits specification
- For most PRs, only linting and type checking are required
- E2E tests only run if PR has "ready-for-e2e" label
- Create PRs in draft mode by default

## Development Setup

1. Install dependencies: `yarn`
2. Set up environment:
   - Copy `.env.example` to `.env`
   - Generate keys: `openssl rand -base64 32` for `NEXTAUTH_SECRET` and `CALENDSO_ENCRYPTION_KEY`
   - Configure Postgres database URL in `.env`
   - Set `DATABASE_DIRECT_URL` to the same value as `DATABASE_URL`
3. Database setup: `yarn workspace @calcom/prisma db-migrate`

**Test Users:** When setting up local development database, default users are created with passwords matching usernames (e.g., 'free:free' and 'pro:pro')

## Internationalization

All UI strings must be properly translated using the i18n system:
- Labels for new UI elements
- Option values displayed to users
- Any text that appears in the interface

Add new strings to the translation system even if related strings already exist.

## Type Safety Rules

**Type casting with "as any" is strictly forbidden.** Use proper type-safe solutions:
- Prisma extensions system
- Type parameter constraints
- Repository pattern isolation
- Explicit type definitions
- Extension composition patterns

## Common Patterns

### Workflow Triggers
- Use `scheduleWorkflowReminders` function to trigger workflows
- Examine existing implementations before adding new triggers
- Function filters workflows by trigger type and processes each step

### Calendar Events
- Cal.com events in Google Calendar are identified by iCalUID ending with "@Cal.com"
- Example: "2GBXSdEixretciJfKVmYN8@Cal.com"

### Generated Files
- App-store integrations use `*.generated.ts` files
- These are created by the app-store-cli tool
- Typically don't manually edit generated files
- Static imports are preferred over dynamic imports for performance

## CI/CD Guidelines

When reviewing CI check failures:
1. E2E tests can be flaky and may fail intermittently
2. Focus only on CI failures directly related to your code changes
3. Infrastructure-related failures can be disregarded if code-specific checks pass

Always run type checks locally (`yarn type-check:ci`) before assuming CI failures are unrelated.

## Additional Resources

For more detailed information, see:
- `.agents/README.md` - Complete development guide
- `.agents/commands.md` - All build, test & dev commands
- `.agents/knowledge-base.md` - Comprehensive knowledge base & best practices
- `.agents/coding-standards.md` - Detailed coding standards

## Logging

Control logging verbosity by setting `NEXT_PUBLIC_LOGGER_LEVEL` in `.env`:
- 0: silly
- 1: trace
- 2: debug
- 3: info
- 4: warn
- 5: error
- 6: fatal
