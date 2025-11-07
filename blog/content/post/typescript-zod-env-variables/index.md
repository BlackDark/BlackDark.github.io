+++
author = "Eduard Marbach"
title = "Type-Safe Environment Variables with TypeScript and Zod"
date = "2025-11-07"
description = "A comprehensive guide to loading and validating environment variables with full TypeScript support and IDE autocomplete"
tags = [
    "typescript",
    "zod",
    "environment-variables",
    "type-safety",
]
categories = [
    "development",
]
+++

Managing environment variables in a type-safe way is crucial for building robust applications. In this guide, I'll show you how to leverage TypeScript and Zod to create a fully type-checked environment configuration system with excellent IDE support.

<!--more-->

## The Problem

Traditional approaches to environment variables in Node.js applications often look like this:

```typescript
const apiKey = process.env.API_KEY;
const port = process.env.PORT || 3000;
```

This approach has several issues:
- **No type safety**: Everything is a `string | undefined`
- **No validation**: Invalid values only cause runtime errors
- **No IDE autocomplete**: You have to remember variable names
- **No documentation**: It's unclear what variables are required

## The Solution: Zod + TypeScript

Zod is a TypeScript-first schema validation library that provides runtime validation and automatic type inference. Let's build a type-safe environment configuration system!

## Basic Setup

First, install the required dependencies:

```bash
npm install zod
# or
pnpm add zod
```

### Simple Example

Let's start with a basic example:

```typescript
import { z } from 'zod';

// Define your schema
const envSchema = z.object({
  NODE_ENV: z.enum(['production', 'development', 'test']),
  PORT: z.string().transform(Number).pipe(z.number().positive()),
  API_KEY: z.string().min(1),
  DATABASE_URL: z.string().url(),
});

// Parse and validate
const env = envSchema.parse(process.env);

// Now `env` is fully typed!
console.log(env.PORT); // TypeScript knows this is a number
console.log(env.NODE_ENV); // TypeScript knows this is 'production' | 'development' | 'test'
```

The beauty here is that TypeScript automatically infers the types from your Zod schema. Your IDE will provide perfect autocomplete!

## Advanced Patterns

### Boolean Environment Variables

Environment variables are always strings, but we often want booleans:

```typescript
// Helper for boolean conversion
const zBoolIsTrue = () =>
  z
    .string()
    .toLowerCase()
    .transform((x) => x === 'true')
    .pipe(z.boolean());

const envSchema = z.object({
  ENABLE_FEATURE_X: zBoolIsTrue().default('false'),
  DEBUG_MODE: zBoolIsTrue().default('false'),
});

const env = envSchema.parse(process.env);
// env.ENABLE_FEATURE_X is a boolean, not a string!
```

### Default Values and Optional Fields

```typescript
const envSchema = z.object({
  // Required field
  API_KEY: z.string(),
  
  // Optional field
  OPTIONAL_SERVICE_URL: z.string().optional(),
  
  // Field with default value
  LOG_LEVEL: z.enum(['error', 'warn', 'info', 'debug']).default('info'),
  
  // Default from another env variable (fallback)
  CACHE_TTL: z.string().default(process.env.DEFAULT_TTL || '3600'),
});
```

### Derived Values and Transformations

Sometimes you need to compute values based on other environment variables:

```typescript
const envSchema = z.object({
  AWS_REGION: z.string().default('us-east-1'),
  AWS_ACCOUNT_ID: z.string().default('123456789012'),
  S3_BUCKET_NAME: z.string().default('my-app-storage'),
  S3_BUCKET_REGION: z.string().optional(),
}).transform((data) => ({
  ...data,
  // Use AWS_REGION as default for S3_BUCKET_REGION
  S3_BUCKET_REGION: data.S3_BUCKET_REGION ?? data.AWS_REGION,
  // Construct full S3 bucket ARN
  S3_BUCKET_ARN: `arn:aws:s3:::${data.S3_BUCKET_NAME}`,
}));

const env = envSchema.parse(process.env);
// env.S3_BUCKET_ARN is automatically computed!
```

### Cross-Field Validation

Zod's `.refine()` method allows you to validate relationships between fields:

```typescript
const envSchema = z.object({
  ENABLE_LOGGING: zBoolIsTrue().default('false'),
  LOG_SERVICE_URL: z.string().optional(),
  LOG_API_KEY: z.string().optional(),
}).refine(
  (data) => !data.ENABLE_LOGGING || (data.LOG_SERVICE_URL && data.LOG_API_KEY),
  {
    message: 'LOG_SERVICE_URL and LOG_API_KEY are required when ENABLE_LOGGING is true',
    path: ['LOG_SERVICE_URL'],
  }
);
```

## Production-Ready Pattern: Shared Configuration

For larger applications, you'll want to share common configuration across multiple services. Here's a scalable pattern:

### Shared Configuration Base

```typescript
// config/shared.ts
import { z } from 'zod';

const zBoolIsTrue = () =>
  z
    .string()
    .toLowerCase()
    .transform((x) => x === 'true')
    .pipe(z.boolean());

const zLogLevel = z.enum(['error', 'warn', 'info', 'debug', 'verbose'] as const);

// Base shared schema
export const sharedRawSchema = z.object({
  LOG_LEVEL: zLogLevel.default('info'),
  LOG_JSON: zBoolIsTrue().default('false'),
  IS_DEV: zBoolIsTrue().default('false'),
  
  // AWS Configuration
  AWS_REGION: z.string().default('us-east-1'),
  AWS_ACCOUNT_ID: z.string().default('123456789012'),
  
  // Database
  DB_TABLE_NAME: z.string().default('main-table'),
  DB_TABLE_REGION: z.string().optional(),
  DB_TABLE_ENDPOINT: z.string().optional(),
  
  // Redis
  REDIS_HOST: z.string().default('localhost'),
  REDIS_PORT: z.number().default(6379),
  REDIS_PASSWORD: z.string().optional(),
  REDIS_USE_TLS: zBoolIsTrue().default('false'),
  
  // Feature Flags
  ENABLE_TELEMETRY: zBoolIsTrue().default('false'),
  TELEMETRY_ENDPOINT: z.string().optional(),
});

// Apply transformations and validations
export const sharedSchema = sharedRawSchema
  .refine((data) => !data.ENABLE_TELEMETRY || data.TELEMETRY_ENDPOINT, {
    message: 'TELEMETRY_ENDPOINT is required when ENABLE_TELEMETRY is true',
    path: ['TELEMETRY_ENDPOINT'],
  })
  .transform((data) => ({
    ...data,
    // Apply derived defaults
    DB_TABLE_REGION: data.DB_TABLE_REGION ?? data.AWS_REGION,
  }));

export type SharedEnvs = z.infer<typeof sharedSchema>;
```

### Application-Specific Configuration

Now you can extend the shared configuration for specific applications:

```typescript
// apps/api/config.ts
import { z } from 'zod';
import { sharedRawSchema } from '../../config/shared';

// Define app-specific variables
const appSpecificSchema = z.object({
  API_PORT: z.string().transform(Number).pipe(z.number().positive().default(3000)),
  API_KEY: z.string(),
  RATE_LIMIT_MAX: z.string().transform(Number).pipe(z.number().positive().default(100)),
  ENABLE_CORS: zBoolIsTrue().default('true'),
});

// Merge with shared configuration
const apiEnvSchema = sharedRawSchema
  .extend(appSpecificSchema.shape)
  .refine((data) => !data.ENABLE_TELEMETRY || data.TELEMETRY_ENDPOINT, {
    message: 'TELEMETRY_ENDPOINT is required when ENABLE_TELEMETRY is true',
    path: ['TELEMETRY_ENDPOINT'],
  })
  .transform((data) => ({
    ...data,
    DB_TABLE_REGION: data.DB_TABLE_REGION ?? data.AWS_REGION,
  }));

type ApiEnvs = z.infer<typeof apiEnvSchema>;

// Validate once at startup
let envs: ApiEnvs | undefined;

export const getEnvs = (): ApiEnvs => {
  if (envs) return envs;
  
  const parsed = apiEnvSchema.safeParse(process.env);
  
  if (!parsed.success) {
    throw new Error(
      `Invalid environment variables: ${JSON.stringify(parsed.error.flatten().fieldErrors)}`
    );
  }
  
  envs = parsed.data;
  return envs;
};
```

### Reusable Factory Pattern

For even more flexibility, create a factory function:

```typescript
// config/factory.ts
import { z } from 'zod';
import { sharedRawSchema } from './shared';

export function createAppEnv<TAppShape extends z.ZodRawShape>(
  appShape: TAppShape,
  transformFn?: (data: any) => any,
) {
  const mergedSchema = sharedRawSchema.extend(appShape);
  
  const finalSchema = transformFn
    ? mergedSchema.transform(transformFn)
    : mergedSchema;
  
  let appEnvs: any;
  
  return {
    schema: finalSchema,
    getEnvs() {
      if (appEnvs) return appEnvs;
      
      const parsed = finalSchema.safeParse(process.env);
      
      if (!parsed.success) {
        throw new Error(
          `Invalid environment variables: ${JSON.stringify(parsed.error.flatten().fieldErrors)}`
        );
      }
      
      appEnvs = parsed.data;
      return appEnvs;
    },
  };
}
```

Usage:

```typescript
// apps/worker/config.ts
import { z } from 'zod';
import { createAppEnv } from '../../config/factory';

const { getEnvs } = createAppEnv(
  {
    WORKER_CONCURRENCY: z.string().transform(Number).pipe(z.number().positive().default(5)),
    QUEUE_NAME: z.string().default('default-queue'),
    QUEUE_REGION: z.string().optional(),
  },
  (data) => ({
    ...data,
    // Apply transformations
    QUEUE_REGION: data.QUEUE_REGION ?? data.AWS_REGION,
    QUEUE_URL: `https://sqs.${data.QUEUE_REGION}.amazonaws.com/${data.AWS_ACCOUNT_ID}/${data.QUEUE_NAME}`,
  })
);

export { getEnvs };
```

## IDE Support & Developer Experience

The best part? **Perfect IDE integration!**

```typescript
import { getEnvs } from './config';

const env = getEnvs();

// ✅ Full autocomplete
env. // IDE shows all available properties

// ✅ Type checking
env.API_PORT // TypeScript knows this is a number
env.LOG_LEVEL // TypeScript knows this is 'error' | 'warn' | 'info' | 'debug' | 'verbose'

// ❌ TypeScript error - property doesn't exist
env.TYPO_VARIABLE // Error: Property 'TYPO_VARIABLE' does not exist

// ✅ IntelliSense shows the exact type
const port: number = env.API_PORT; // ✅ Works
const port: string = env.API_PORT; // ❌ Type error
```

## Error Messages

When validation fails, you get clear, actionable error messages:

```typescript
// If API_KEY is missing:
// Error: Invalid environment variables: {
//   "API_KEY": ["Required"]
// }

// If PORT is invalid:
// Error: Invalid environment variables: {
//   "PORT": ["Expected number, received nan"]
// }
```

## Best Practices

1. **Validate early**: Parse environment variables at application startup
2. **Fail fast**: Crash immediately if validation fails
3. **Use defaults wisely**: Only for truly optional values
4. **Document with types**: The schema serves as documentation
5. **Single source of truth**: Use `getEnvs()` everywhere instead of `process.env`
6. **Cache the result**: Parse once, reuse the validated object

## Testing

Testing becomes easier with validated environment variables:

```typescript
// test/setup.ts
import { z } from 'zod';

export function createTestEnv<T extends z.ZodTypeAny>(
  schema: T,
  overrides: Partial<z.infer<T>> = {}
): z.infer<T> {
  const defaultTestEnv = {
    NODE_ENV: 'test',
    API_KEY: 'test-api-key',
    DATABASE_URL: 'postgresql://localhost:5432/test',
    ...overrides,
  };
  
  return schema.parse(defaultTestEnv);
}
```

## Conclusion

Using Zod for environment variable validation provides:

- ✅ **Full type safety** with automatic TypeScript inference
- ✅ **Runtime validation** that catches errors before they cause problems
- ✅ **Excellent IDE support** with autocomplete and inline documentation
- ✅ **Clear error messages** when something goes wrong
- ✅ **Self-documenting code** - the schema is the documentation
- ✅ **Scalable patterns** for monorepos and microservices

Say goodbye to `process.env` and hello to type-safe, validated configuration! Your IDE will thank you, and your production environment will be more stable.

## Resources

- [Zod Documentation](https://zod.dev/)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
- [12-Factor App: Config](https://12factor.net/config)

---

Have you implemented type-safe environment variables in your projects? What patterns have worked well for you? Let me know in the comments!
