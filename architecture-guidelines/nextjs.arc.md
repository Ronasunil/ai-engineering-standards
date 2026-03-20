# Copilot Instructions

## Next.js API Route Conventions

### Route Structure — Thin Routes, Fat Services

Route handlers must be minimal. All business logic lives in service files. Routes only: parse input, call a service, return a response.

```
app/api/users/route.ts        → route handler (thin)
lib/services/user.service.ts   → business logic (fat)
lib/validations/user.schema.ts → Zod schemas
lib/types/user.types.ts        → TypeScript type definitions
lib/constants/user.constants.ts→ constants & enums
lib/errors/app-error.ts        → custom error class
lib/errors/error-handler.ts    → withErrorHandler HOF
lib/middleware/                 → middleware functions
```

### Standard Response Format

Every API response **must** use this shape:

```ts
// Success
Response.json(
  { status: 200, data: result, message: "User created successfully" },
  { status: 200 },
);

// Error
Response.json(
  { status: 404, data: null, message: "User not found" },
  { status: 404 },
);
```

Never return raw data or inconsistent shapes. Always include `status`, `data`, and `message`.

### Zod Validation on Every Route Body

Every route that accepts a body **must** validate it with a Zod schema. Validation is applied as middleware (see below), not inline in the handler.

```ts
// lib/validations/user.schema.ts
import { z } from "zod";

export const createUserSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
  role: z.enum(["admin", "user"]),
});

export type CreateUserPayload = z.infer<typeof createUserSchema>;
```

All types must be inferred from Zod schemas using `z.infer<>` — no duplicate manual type definitions.

### Error Handling — `withErrorHandler` HOF

All route handlers must be wrapped with `withErrorHandler`. Never use bare try/catch in routes.

```ts
// lib/errors/app-error.ts
export class AppError extends Error {
  constructor(
    public message: string,
    public statusCode: number,
  ) {
    super(message);
  }
}

// lib/errors/error-handler.ts
import { AppError } from "./app-error";

export function withErrorHandler(handler: Function) {
  return async (req: Request) => {
    try {
      return await handler(req);
    } catch (error) {
      if (error instanceof AppError) {
        return Response.json(
          { status: error.statusCode, data: null, message: error.message },
          { status: error.statusCode },
        );
      }
      console.error(error);
      return Response.json(
        { status: 500, data: null, message: "Internal Server Error" },
        { status: 500 },
      );
    }
  };
}
```

Throw `AppError` from services for known errors (not found, unauthorized, validation). Unknown errors fall through to the 500 handler.

### Middleware — `composeMiddleware` Chaining

Middleware is applied via higher-order function composition, not Express-style `(req, res, next)`.

```ts
// lib/middleware/compose.ts
export function composeMiddleware(
  handler: Function,
  ...middlewares: Function[]
) {
  return middlewares.reduceRight((acc, middleware) => middleware(acc), handler);
}
```

#### Zod Body Validation Middleware

Body validation is a middleware, not inline code. It takes a Zod schema and validates the request body before the handler runs.

```ts
// lib/middleware/validate-body.ts
import { ZodSchema } from "zod";
import { AppError } from "@/lib/errors/app-error";

export function validateBody(schema: ZodSchema) {
  return (handler: Function) => {
    return async (req: Request) => {
      const body = await req.json();
      const result = schema.safeParse(body);
      if (!result.success) {
        throw new AppError(result.error.errors[0].message, 400);
      }
      (req as any).validatedBody = result.data;
      return handler(req);
    };
  };
}
```

### Route Handler Example — Putting It All Together

```ts
// app/api/users/route.ts
import { withErrorHandler } from "@/lib/errors/error-handler";
import { composeMiddleware } from "@/lib/middleware/compose";
import { validateBody } from "@/lib/middleware/validate-body";
import { createUserSchema } from "@/lib/validations/user.schema";
import { UserService } from "@/lib/services/user.service";

const handleCreateUser = async (req: Request) => {
  const payload = (req as any).validatedBody;
  const user = await UserService.create(payload);
  return Response.json(
    { status: 201, data: user, message: "User created successfully" },
    { status: 201 },
  );
};

export const POST = withErrorHandler(
  composeMiddleware(handleCreateUser, validateBody(createUserSchema)),
);

export const GET = withErrorHandler(async () => {
  const users = await UserService.list();
  return Response.json(
    { status: 200, data: users, message: "Users fetched successfully" },
    { status: 200 },
  );
});
```

### Constants — No Magic Values

Never hardcode strings, numbers, or config values in routes or services. Extract to dedicated constant files.

```ts
// lib/constants/user.constants.ts
export const USER_ROLES = ["admin", "user"] as const;
export const MAX_LOGIN_ATTEMPTS = 5;
export const DEFAULT_PAGE_SIZE = 20;
```

### Type Definitions — End-to-End Typing

Every value from request payload to service input to database call to response must be typed. Use Zod inference for request types, explicit interfaces for service return types.

```ts
// lib/types/user.types.ts
export interface UserResponse {
  id: string;
  name: string;
  email: string;
  role: string;
  createdAt: Date;
}

export interface ApiResponse<T> {
  status: number;
  data: T;
  message: string;
}
```

### Summary of Rules

1. **Thin routes, fat services** — route files only parse, delegate, and respond
2. **Zod on every body** — validated via `validateBody` middleware, types inferred with `z.infer<>`
3. **Standard response shape** — `{ status, data, message }` on every response, success or error
4. **`withErrorHandler` on every route export** — no bare try/catch in routes
5. **`composeMiddleware` for chaining** — auth, validation, logging all compose as HOFs
6. **Constants in separate files** — no magic values in business logic
7. **Full type coverage** — payload → service → DB → response, all typed
8. **Modular file organization** — services, validations, types, constants, middleware each get their own directory under `lib/`
