# Seamlessly Integrate Swagger with JWT Authentication in NestJS

- Canonical URL: https://imzihad21.github.io/articles/a/seamlessly-integrate-swagger-with-jwt-authentication-in-nestjs-2aol/
- Source URL: https://dev.to/imzihad21/seamlessly-integrate-swagger-with-jwt-authentication-in-nestjs-2aol
- Web View: https://imzihad21.github.io/articles/a/seamlessly-integrate-swagger-with-jwt-authentication-in-nestjs-2aol/
- Published: 2024-11-03T17:43:11.000Z
- Modified: 2024-11-03T17:43:11.000Z
- Reading time: 2 minutes
- Tags: nestjs, jwt, swagger, webdev

Strong API documentation and secure authentication should ship together, not as separate tasks. With NestJS, Swagger can document endpoints while also supporting JWT Bearer auth for protected routes.

This guide shows a clean setup for Swagger UI with JWT support in a production-friendly NestJS project.

### Why It Matters

- Developers can test secured endpoints directly from Swagger UI.
- API docs stay aligned with implementation and reduce onboarding time.
- Persisted authorization improves dev workflow during testing.
- Security scheme setup is centralized and easy to maintain.

### Core Concepts

#### 1. Install Swagger Package

Install official Swagger integration package for NestJS.

```bash
npm install @nestjs/swagger
```

#### 2. Define Swagger Document with JWT Scheme

Configure title, version, and Bearer authentication schema.

```typescript
import { INestApplication } from "@nestjs/common";
import {
  DocumentBuilder,
  SwaggerCustomOptions,
  SwaggerModule,
} from "@nestjs/swagger";

const swaggerDocumentConfig = new DocumentBuilder()
  .setTitle("Dummy API")
  .setDescription("API documentation with JWT Bearer authentication")
  .setVersion("1.0.0")
  .addBearerAuth(
    {
      type: "http",
      scheme: "bearer",
      bearerFormat: "JWT",
      in: "header",
      name: "Authorization",
      description: "Paste access token without Bearer prefix",
    },
    "Bearer"
  )
  .addSecurityRequirements("Bearer")
  .build();

const swaggerUiOptions: SwaggerCustomOptions = {
  swaggerOptions: {
    persistAuthorization: true,
  },
  customSiteTitle: "Dummy API Documentation",
};

export function configureSwaggerUI(app: INestApplication) {
  const document = SwaggerModule.createDocument(app, swaggerDocumentConfig);
  SwaggerModule.setup("swagger", app, document, swaggerUiOptions);
}
```

#### 3. Attach Swagger in Bootstrap

Call Swagger setup during app startup.

```typescript
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import { configureSwaggerUI } from "./swagger.config";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  configureSwaggerUI(app);
  await app.listen(4000);
}

bootstrap();
```

#### 4. Protect Routes with JWT Guard

Swagger auth config documents security, but real protection comes from guards.

```typescript
import { Controller, Get, UseGuards } from "@nestjs/common";
import { ApiBearerAuth, ApiTags } from "@nestjs/swagger";
import { JwtAuthGuard } from "./auth/jwt-auth.guard";

@ApiTags("users")
@Controller("users")
export class UserController {
  @Get("profile")
  @UseGuards(JwtAuthGuard)
  @ApiBearerAuth("Bearer")
  getProfile() {
    return { message: "Protected profile endpoint" };
  }
}
```

#### 5. Run and Access

Start the app and open Swagger UI.

```bash
npm run start
```

Default URL:

```text
http://localhost:4000/swagger
```

#### 6. JWT Testing Flow in Swagger UI

Basic sequence for protected endpoint testing:

- Generate token from login endpoint.
- Click `Authorize` in Swagger UI.
- Paste token value.
- Execute protected endpoint requests.

### Practical Example

Recommended file layout:

```text
src/
  swagger.config.ts
  main.ts
  auth/
    jwt-auth.guard.ts
  users/
    user.controller.ts
```

This structure keeps docs setup isolated and reusable. Controllers stay focused on business logic while docs and security metadata remain explicit.

### Common Mistakes

- Configuring Swagger Bearer auth but forgetting to apply route guards.
- Missing `@ApiBearerAuth("Bearer")` on protected endpoints.
- Disabling `persistAuthorization` and re-pasting token every request.
- Hardcoding docs URL assumptions without checking actual server port.
- Treating Swagger docs as security itself. Swagger documents security, guards enforce it.

### Quick Recap

- Install `@nestjs/swagger` and configure document metadata.
- Add JWT Bearer auth scheme with a named security key.
- Register Swagger in `main.ts`.
- Protect routes using JWT guards.
- Use Swagger UI `Authorize` to test secured endpoints quickly.

### Next Steps

1. Add grouped tags for domains like `auth`, `users`, and `orders`.
2. Add response DTO decorators for stronger API contracts.
3. Add environment-based toggle to disable Swagger in production if required.
4. Add e2e tests to verify protected routes return `401` without valid token.