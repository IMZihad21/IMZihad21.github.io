# Seamlessly Integrate Swagger with JWT Authentication in NestJS

- Canonical URL: https://imzihad21.github.io/articles/a/seamlessly-integrate-swagger-with-jwt-authentication-in-nestjs-2aol/
- Source URL: https://dev.to/imzihad21/seamlessly-integrate-swagger-with-jwt-authentication-in-nestjs-2aol
- Web View: https://imzihad21.github.io/articles/a/seamlessly-integrate-swagger-with-jwt-authentication-in-nestjs-2aol/
- Published: 2024-11-03T17:43:11.000Z
- Modified: 2024-11-03T17:43:11.000Z
- Reading time: 4 minutes
- Tags: nestjs, jwt, swagger, webdev

## Integrate Swagger with JWT authentication in NestJS

Accurate API documentation and secure authentication work best when configured together. In NestJS services, developers frequently encounter discrepancies between actual guard enforcement and the interactive OpenAPI documentation exposed to client consumers.

NestJS allows you to bind Swagger documentation directly to JWT Bearer authentication within the Swagger UI explorer. By coupling OpenAPI security definitions with route guards and persistent client state, teams maintain accurate interactive documentation without leaking unauthenticated routes.

### The problem and production context

In production NestJS microservices, documentation often drifts from runtime authorization logic. When documentation lacks interactive authorization workflows, engineers resort to third-party tools like Postman or cURL to verify protected routes.

- **Failure scenario**: An engineer attaches `JwtAuthGuard` to a sensitive user profile endpoint but forgets the `@ApiBearerAuth()` decorator. The generated Swagger UI displays the endpoint as an open public route, confusing frontend integration teams. Alternatively, documenting security without attaching guards leaves the actual endpoint exposed to unauthenticated exploitation.
- **Why default approaches fall short**: Out-of-the-box Swagger documentation in NestJS does not declare authentication schemes automatically. Developers must configure security schemes explicitly and coordinate decorators with runtime guard metadata.
- **Production impact**: Frontend teams waste engineering time troubleshooting 401 Unauthorized responses, documentation drifts from code reality, and security audits fail due to undocumented authentication mechanisms.

Integrating Swagger with JWT Bearer support resolves these operational issues:
- Developers can test secured endpoints directly from Swagger UI without external HTTP clients.
- API documentation stays synchronized with controller implementation.
- Persisting authorization state across page refreshes speeds up local testing workflows.
- Security scheme definitions remain centralized and decoupled from controller logic.

### Mental model and core concepts

#### 1. Swagger package installation

Install the official OpenAPI integration package for NestJS:

```bash
npm install @nestjs/swagger
```

#### 2. Swagger document definition with JWT scheme

Configure the document metadata and define the Bearer authentication scheme:

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

#### 3. Application bootstrap integration

Initialize the documentation module in `main.ts` during application startup:

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

#### 4. Route protection via JWT guard and documentation decorators

Configuring Swagger documents the security requirement, but actual endpoint protection requires NestJS guards:

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

#### 5. Application execution and endpoint access

Start the application:

```bash
npm run start
```

Access Swagger UI in your browser:

```text
http://localhost:4000/swagger
```

#### 6. Interactive JWT authorization testing workflow

Testing secured endpoints involves four basic steps:
1. Obtain a token from your authentication endpoint.
2. Click the `Authorize` button at the top of the Swagger interface.
3. Paste the token into the value field and confirm.
4. Execute requests against protected endpoints.

### Production implementation

The following modular project structure isolates OpenAPI configuration from business logic and controllers.

Directory organization:

```text
src/
  swagger.config.ts
  main.ts
  auth/
    jwt-auth.guard.ts
  users/
    user.controller.ts
```

In `src/swagger.config.ts`, encapsulate document generation and Swagger UI setup:

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

In `src/users/user.controller.ts`, bind both the runtime guard and documentation metadata:

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

Controllers remain focused on business logic while documentation and security metadata stay explicit.

### Architectural trade-offs and edge cases

* **Latency versus consistency**: Generating the Swagger AST document occurs at application startup during `bootstrap()`. While this adds negligible startup latency, all route decorators are evaluated ahead of time, ensuring runtime request routing incurs zero ongoing reflection overhead.
* **Failure recovery**: If invalid decorators or circular class references are introduced into DTOs, NestJS will fail to boot during document generation. This fail-fast behavior prevents deploying corrupted API contracts to staging or production.
* **Scale limitations**: Swagger UI executes in the client browser. For expansive microservices containing hundreds of endpoints with large schema models, generating a single monolithic Swagger document can cause browser rendering delays. In large applications, segment documentation using multiple Swagger endpoints grouped by module.

### Common anti-patterns and gotchas

* **Declaring Swagger Bearer auth without route guards**: Adding `@ApiBearerAuth()` without `@UseGuards(JwtAuthGuard)` documents the route as secured in Swagger UI while leaving the actual HTTP handler unprotected against anonymous access.
* **Adding route guards without Swagger decorators**: Attaching `@UseGuards(JwtAuthGuard)` while omitting `@ApiBearerAuth("Bearer")` leaves Swagger UI unaware that a token is required, causing interactive requests to fail with 401 errors.
* **Leaving persistAuthorization disabled**: Omitting `persistAuthorization: true` forces developers to re-enter JWT tokens on every browser reload, slowing down iterative testing.
* **Confusing documentation with access control enforcement**: Swagger annotations only generate OpenAPI metadata. They do not intercept incoming HTTP traffic or enforce security rules.

### Implementation checklist

1. Install `@nestjs/swagger` into project dependencies.
2. Configure `DocumentBuilder` with `.addBearerAuth()` and custom site options.
3. Call `SwaggerModule.createDocument` and `SwaggerModule.setup` inside `main.ts`.
4. Apply `@UseGuards(JwtAuthGuard)` and `@ApiBearerAuth("Bearer")` consistently on protected endpoints.
5. Verify access to Swagger UI at `/swagger` and test token authorization.
6. Add domain tags to group related endpoints, such as `auth`, `users`, and `orders`.
7. Annotate route responses with DTO schemas to produce complete request and response contracts.
8. Add an environment variable check to disable Swagger UI in sensitive environments if necessary.
9. Add end-to-end tests ensuring protected endpoints return `401 Unauthorized` without a valid token.