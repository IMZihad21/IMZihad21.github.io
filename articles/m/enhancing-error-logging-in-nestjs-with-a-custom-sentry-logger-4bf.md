# Enhancing Error Logging in NestJS with Sentry

- Canonical URL: https://imzihad21.github.io/articles/a/enhancing-error-logging-in-nestjs-with-a-custom-sentry-logger-4bf/
- Source URL: https://dev.to/imzihad21/enhancing-error-logging-in-nestjs-with-a-custom-sentry-logger-4bf
- Web View: https://imzihad21.github.io/articles/a/enhancing-error-logging-in-nestjs-with-a-custom-sentry-logger-4bf/
- Published: 2024-11-03T17:31:51.000Z
- Modified: 2024-11-03T17:31:51.000Z
- Reading time: 5 minutes
- Tags: nestjs, sentry, logger, exception

## Enhancing error logging in NestJS with Sentry

Error logs without contextual metadata make diagnosing failures in distributed production environments difficult and time-consuming. In high-concurrency NestJS services, plain console output fails to capture active request payloads, authenticated user identifiers, or execution traces needed to reproduce edge-case crashes.

Integrating Sentry using a custom NestJS logger alongside a global exception filter establishes automated error capture with rich HTTP execution context while preserving standard console streaming for container diagnostics.

### The problem and production context

Standard backend logging typically writes unstructured error messages to stdout. When uncaught exceptions happen in production, operations teams lack the context necessary to reproduce the incident.

- **Failure scenario**: An unhandled exception occurs inside a nested controller handler processing a complex request. The service writes a generic error string to stdout without HTTP method, URL, headers, request body, or authenticated user identity. Engineers cannot determine which client payload triggered the failure, delaying incident mitigation.
- **Why default approaches fall short**: NestJS `BaseExceptionFilter` catches unhandled exceptions and formats standard HTTP responses, but does not forward diagnostics to external telemetry backends. Manual `try/catch` blocks inside controllers introduce boilerplate and risk omissions.
- **Production impact**: Unhandled errors fail silently or clutter container logs without stack traces, increasing Mean Time to Resolution (MTTR) and obscuring recurring bugs.

### Mental model and core concepts

A production-ready error monitoring pipeline separates structured runtime logging from uncaught exception filtering.

#### 1. Dual-pipeline observability model

The architecture partitions telemetry into two distinct paths:
- Operational logging: Handled by extending `ConsoleLogger` to capture explicit `logger.error()` and `logger.verbose()` calls across application code.
- Exception interception: Handled by a global NestJS exception filter implementing `BaseExceptionFilter` to intercept unhandled application crashes.

#### 2. Custom logger extension mechanics

Extending `ConsoleLogger` preserves local console logging for container stdout while packaging context parameters (such as execution stack and component names) into isolated Sentry scopes via `Sentry.withScope()` and `Sentry.captureMessage()`.

#### 3. Exception filter and HTTP context enrichment

The global filter intercepts unhandled exceptions, extracts the active `Request` object from `ArgumentsHost`, and enriches Sentry event scopes with:
- HTTP method and request URL.
- Request headers, route parameters, query strings, and body payloads.
- Authenticated user identity (`userId`, `userEmail`, `userName`, `userRole`).
- Nested exception response structures.

#### 4. Global filter registration via APP_FILTER

Binding the exception filter using the `APP_FILTER` dependency injection token ensures that the filter participates in the NestJS dependency injection lifecycle and intercepts exceptions across all controllers globally.

#### 5. SDK initialization and environment sampling

Initializing the Sentry SDK during bootstrap configures environment boundaries (`development`, `staging`, `production`), sampling rates, normalization depth, and profiling integrations before application modules begin accepting traffic.

#### 6. File structure layout

The components are organized cleanly within the application source tree:

```text
src/
  utility/
    logger/
      sentry.logger.ts
      sentry-exception.filter.ts
  app.module.ts
  main.ts
```

### Production implementation

Install the required Sentry dependency:

```bash
npm install @sentry/node @sentry/profiling-node
```

Implement the custom logger, global exception filter, module registration, and bootstrap initialization.

```typescript
import {
  ArgumentsHost,
  Catch,
  ConsoleLogger,
  Controller,
  Get,
  Injectable,
  Logger,
  Module,
  Provider,
} from "@nestjs/common";
import { ConfigModule, ConfigService } from "@nestjs/config";
import { APP_FILTER, BaseExceptionFilter, NestFactory } from "@nestjs/core";
import { NestExpressApplication } from "@nestjs/platform-express";
import * as Sentry from "@sentry/node";
import { nodeProfilingIntegration } from "@sentry/profiling-node";

interface AuthenticatedUser {
  userId: string;
  userEmail: string;
  userName: string;
  userRole: string;
}

interface RequestWithContext {
  url: string;
  method: string;
  headers: Record<string, string>;
  params: Record<string, string>;
  query: Record<string, string>;
  body?: Record<string, unknown>;
  user?: AuthenticatedUser;
}

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

@Catch()
export class SentryExceptionFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost): void {
    const http = host.switchToHttp();
    const request = http.getRequest<RequestWithContext>();

    Sentry.withScope((scope) => {
      if (request) {
        scope.setTag("url", request.url);
        scope.setTag("method", request.method);
        scope.setTag("environment", process.env.NODE_ENV ?? "development");

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

@Injectable()
export class AppService {
  getHello(): string {
    return "Hello World";
  }
}

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }
}

@Module({
  imports: [ConfigModule.forRoot({ isGlobal: true })],
  controllers: [AppController],
  providers: [AppService, SentryExceptionFilterProvider],
})
export class AppModule {}

const applicationLogger = new Logger("Bootstrap");

async function bootstrap(): Promise<void> {
  const app = await NestFactory.create<NestExpressApplication>(AppModule);
  const configService = app.get(ConfigService);

  const dsn = configService.get<string>("SENTRY_DSN", "");
  const environment = configService.get<string>("NODE_ENV", "development");

  Sentry.init({
    dsn: dsn,
    environment: environment,
    tracesSampleRate: environment === "production" ? 0.1 : 1.0,
    profilesSampleRate: environment === "production" ? 0.1 : 1.0,
    normalizeDepth: 5,
    integrations: [nodeProfilingIntegration()],
  });

  app.useLogger(new SentryLogger());

  await app.listen(3000);
}

bootstrap()
  .then(() => applicationLogger.log("Server is running"))
  .catch((error: unknown) => applicationLogger.error("Bootstrap failed", error));
```

### Architectural trade-offs and edge cases

Integrating third-party error trackers requires balancing diagnostic completeness with network latency and data security.

* **Latency versus consistency**: Capturing exceptions asynchronously via Sentry's Node SDK offloads event transmission to background workers, keeping HTTP response latency minimal. However, unhandled process-level crashes (such as `process.on('uncaughtException')`) require flushing pending events with `Sentry.flush(2000)` before process exit to prevent telemetry loss.
* **Failure recovery**: If Sentry endpoints experience downtime or network partitions, the SDK queues events in a bounded in-memory buffer, dropping older events when capacity is exceeded without blocking or failing active HTTP requests.
* **Scale limitations**: Sending full request bodies and 100% trace sampling (`tracesSampleRate: 1.0`) on high-traffic production endpoints quickly saturates network bandwidth and Sentry ingestion quotas. Production systems must tune sampling rates and implement payload size limits.

### Common anti-patterns and gotchas

* **Environment variable misconfiguration**: Configuring `SENTRY_DNS` instead of the expected `SENTRY_DSN` environment variable causes silent initialization failure without error reporting.
* **Unscrubbed sensitive data leakage**: Forwarding raw request bodies containing passwords, credit card numbers, or authorization bearer tokens to external Sentry dashboards violates data security compliance. Implement data scrubbing before capturing events.
* **Missing filter provider registration**: Creating `SentryExceptionFilter` but omitting `SentryExceptionFilterProvider` from `AppModule` providers results in unhandled exceptions bypassing Sentry entirely.
* **Capturing message strings without stack traces**: Calling `Sentry.captureMessage()` with simple text rather than `Sentry.captureException()` strips stack trace frames, rendering debugging difficult.
* **100% trace sampling in production**: Leaving `tracesSampleRate: 1.0` enabled in high-throughput environments exhausts quota limits and incurs unnecessary infrastructure costs.

### Implementation checklist

1. Install `@sentry/node` and `@sentry/profiling-node` dependencies.
2. Verify `SENTRY_DSN` and `NODE_ENV` environment variables in application configuration.
3. Implement `SentryLogger` extending `ConsoleLogger` with scoped error forwarding.
4. Implement `SentryExceptionFilter` extending `BaseExceptionFilter` with request and user context tagging.
5. Register `SentryExceptionFilterProvider` under `APP_FILTER` in `AppModule`.
6. Initialize Sentry during bootstrap before attaching `SentryLogger`.
7. Configure payload sanitization to redact authentication tokens and sensitive fields.
8. Adjust `tracesSampleRate` and `profilesSampleRate` according to environment traffic volume.
9. Configure Sentry alert thresholds for error rate anomalies and verify alert delivery.