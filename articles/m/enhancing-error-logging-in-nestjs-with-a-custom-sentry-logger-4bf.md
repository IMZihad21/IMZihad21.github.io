# Enhancing Error Logging in NestJS with Sentry

- Canonical URL: https://imzihad21.github.io/articles/a/enhancing-error-logging-in-nestjs-with-a-custom-sentry-logger-4bf/
- Source URL: https://dev.to/imzihad21/enhancing-error-logging-in-nestjs-with-a-custom-sentry-logger-4bf
- Web View: https://imzihad21.github.io/articles/a/enhancing-error-logging-in-nestjs-with-a-custom-sentry-logger-4bf/
- Published: 2024-11-03T17:31:51.000Z
- Modified: 2024-11-03T17:31:51.000Z
- Reading time: 3 minutes
- Tags: nestjs, sentry, logger, exception

Error logs without context are just expensive noise. In a real NestJS system, we want logs that show what failed, where it failed, and who was affected.

This setup uses Sentry with a custom NestJS logger and a global exception filter, while keeping default console logging behavior.

### Why It Matters

- Centralized error tracking reduces debugging time.
- Request and user context make incidents easier to reproduce.
- Global exception capture prevents silent failures.
- Console logging remains available for local and container logs.

### Core Concepts

#### 1. Install Sentry SDK

Install required package:

```bash
npm install @sentry/node
```

#### 2. Custom Logger Extension

Extend `ConsoleLogger` and forward error/verbose logs to Sentry.

```typescript
import { ConsoleLogger } from "@nestjs/common";
import * as Sentry from "@sentry/node";

export class SentryLogger extends ConsoleLogger {
  error(message: unknown, ...optionalParams: unknown[]): void {
    const errorMessage = String(message ?? "");
    let stack: unknown = "";
    let context = "";

    if (optionalParams.length === 1) {
      context = String(optionalParams[0] ?? "");
    }

    if (optionalParams.length >= 2) {
      stack = optionalParams[0];
      context = String(optionalParams[1] ?? "");
    }

    const formattedMessage = context ? `${context}: ${errorMessage}` : errorMessage;

    Sentry.withScope((scope) => {
      scope.setExtra("message", errorMessage);
      scope.setExtra("context", context);
      scope.setExtra("stack", stack);
      Sentry.captureMessage(formattedMessage, "error");
    });

    super.error(errorMessage, ...(optionalParams as []));
  }

  verbose(message: unknown, ...optionalParams: unknown[]): void {
    const verboseMessage = String(message ?? "");
    const context = String(optionalParams[0] ?? "");
    const extra = optionalParams.slice(1);
    const formattedMessage = context ? `${context}: ${verboseMessage}` : verboseMessage;

    Sentry.withScope((scope) => {
      scope.setExtra("message", verboseMessage);
      scope.setExtra("context", context);
      scope.setExtra("extra", extra);
      Sentry.captureMessage(formattedMessage, "info");
    });

    super.verbose(verboseMessage, ...(extra as []));
  }
}
```

#### 3. Global Exception Filter

Catch all unhandled exceptions and send enriched request context to Sentry.

```typescript
import { Catch, type ArgumentsHost, type Provider } from "@nestjs/common";
import { APP_FILTER, BaseExceptionFilter } from "@nestjs/core";
import * as Sentry from "@sentry/node";

@Catch()
class SentryExceptionFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const http = host.switchToHttp();
    const request = http.getRequest();

    Sentry.withScope((scope) => {
      if (request) {
        scope.setTag("url", request.url);
        scope.setTag("method", request.method);
        scope.setTag("environment", process.env.NODE_ENV || "development");

        scope.setExtra("request", {
          url: request.url,
          method: request.method,
          headers: request.headers,
          params: request.params,
          query: request.query,
          body: request.body ?? {},
        });

        if (request.user) {
          scope.setUser({
            id: request.user.userId,
            email: request.user.userEmail,
            username: request.user.userName,
          });

          scope.setExtra("userRole", request.user.userRole);
        }
      }

      if (typeof exception === "object" && exception !== null && "response" in exception) {
        const exceptionResponse = (exception as { response?: unknown }).response;
        scope.setExtra("response", exceptionResponse);
      }

      Sentry.captureException(exception);
    });

    super.catch(exception as never, host);
  }
}

export const SentryExceptionFilterProvider: Provider = {
  provide: APP_FILTER,
  useClass: SentryExceptionFilter,
};
```

#### 4. Sentry Initialization in Bootstrap

Initialize Sentry once during app startup and register custom logger.

```typescript
import { Logger } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import { NestFactory } from "@nestjs/core";
import { NestExpressApplication } from "@nestjs/platform-express";
import * as Sentry from "@sentry/node";
import { nodeProfilingIntegration } from "@sentry/profiling-node";
import { AppModule } from "./app.module";
import { SentryLogger } from "./utility/logger/sentry.logger";

const logger = new Logger("MyApp");

async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(AppModule);
  const configService = app.get(ConfigService);

  Sentry.init({
    dsn: configService.get<string>("SENTRY_DSN", ""),
    environment: configService.get<string>("NODE_ENV", "development"),
    tracesSampleRate: 1.0,
    profilesSampleRate: 1.0,
    normalizeDepth: 5,
    integrations: [nodeProfilingIntegration()],
  });

  app.useLogger(new SentryLogger());

  await app.listen(3000);
}

bootstrap()
  .then(() => logger.log("Server is running"))
  .catch((error) => logger.error("Bootstrap failed", error));
```

#### 5. Register Filter in App Module

Register the global filter provider in `AppModule`.

```typescript
import { Module } from "@nestjs/common";
import { AppController } from "./app.controller";
import { AppService } from "./app.service";
import { ValidationProvider } from "./validation.provider";
import { SentryExceptionFilterProvider } from "./utility/logger/sentry-exception.filter";

@Module({
  imports: [],
  controllers: [AppController],
  providers: [AppService, ValidationProvider, SentryExceptionFilterProvider],
})
export class AppModule {}
```

#### 6. Logging Strategy

Use this layered strategy:

- Logger captures operational messages and error signals.
- Exception filter captures unhandled failures with request context.
- Sentry groups and tracks incidents across environments.

### Practical Example

Typical project structure for this setup:

```text
src/
  utility/
    logger/
      sentry.logger.ts
      sentry-exception.filter.ts
  app.module.ts
  main.ts
```

When a controller throws an unhandled error, the filter sends request/user metadata to Sentry, and the logger still writes to console output. Two observability channels, one less panic.

### Common Mistakes

- Using `SENTRY_DNS` instead of correct `SENTRY_DSN` environment key.
- Sending sensitive request data to Sentry without sanitization.
- Forgetting to register the global exception filter provider.
- Capturing only message strings and losing stack/context details.
- Running `tracesSampleRate` at `1.0` in high-traffic production without budget planning.

### Quick Recap

- Extend NestJS logger to forward important logs to Sentry.
- Add global exception filter for unhandled errors.
- Initialize Sentry at bootstrap with environment-based config.
- Register filter provider in module DI container.
- Keep logs contextual, structured, and privacy-aware.

### Next Steps

1. Add data scrubbing for secrets and personal data before sending events.
2. Tune sampling rates by environment (`dev`, `staging`, `prod`).
3. Add alert rules in Sentry for critical error thresholds.
4. Add release tracking to map errors to deployments.