# Custom Role-Based Access Control in NestJS Using Custom Guards

- Canonical URL: https://imzihad21.github.io/articles/a/custom-role-based-access-control-in-nestjs-using-custom-guards-jol/
- Source URL: https://dev.to/imzihad21/custom-role-based-access-control-in-nestjs-using-custom-guards-jol
- Web View: https://imzihad21.github.io/articles/a/custom-role-based-access-control-in-nestjs-using-custom-guards-jol/
- Published: 2024-11-11T20:04:34.000Z
- Modified: 2024-11-11T20:04:34.000Z
- Reading time: 5 minutes
- Tags: nestjs, rbac, authorization, security

## Custom role-based access control in NestJS using custom guards

Scattering role checks across controller action methods leads to brittle authorization logic and security oversights. Under concurrent API workloads, manual condition checks introduce maintenance overhead, inconsistent security enforcement across routes, and accidental privilege escalation vulnerabilities.

Combining NestJS custom decorators with execution guards provides a declarative, centralized mechanism for enforcing role-based access control (RBAC), guaranteeing uniform security evaluation before request handlers execute.

### The problem and production context

When authorization logic is implemented directly within controller methods or services, access control rules become fragmented and difficult to audit across expanding microservices or monolithic API surfaces.

- **Failure scenario**: A developer introduces an administrative endpoint or updates an existing entity-updating handler but forgets to add an inline role verification check. An authenticated standard user issues a request against the endpoint, executing privileged administrative operations such as user deletion or data updates without authorization.
- **Why default approaches fall short**: Relying on manual inline assertions or ad-hoc middleware leads to duplication. NestJS default controllers do not enforce authorization boundaries out of the box; without an automated reflection pipeline, route security depends entirely on individual developer diligence.
- **Production impact**: Privilege escalation vulnerabilities expose sensitive customer data, invalidate regulatory compliance mandates, and complicate security audit trails due to scattered authorization logging.

### Mental model and core concepts

Implementing role-based access control in NestJS relies on metadata reflection, lifecycle execution ordering, and type-safe role definitions.

#### 1. Type-safe role enumeration

Domain permissions should be bound to immutable string literals or TypeScript enums. Defining an explicit enum prevents typos and guarantees that role assignments across entities, tokens, and guards remain strictly typed:

```typescript
export enum ApplicationUserRoleEnum {
  ADMIN = "ADMIN",
  OWNER = "OWNER",
  USER = "USER",
}
```

#### 2. Custom metadata decorator

NestJS provides `SetMetadata` to attach custom reflection metadata to controller classes or route handlers. Defining a dedicated metadata key guarantees consistent extraction:

```typescript
export const ROLES_METADATA_KEY = "roles";
```

The custom decorator accepts a variable number of allowed roles and attaches them directly to the route handler context:

```typescript
export const RequiredRoles = (...roles: ApplicationUserRoleEnum[]) =>
  SetMetadata(ROLES_METADATA_KEY, roles);
```

#### 3. Execution guard and metadata reflection

The guard implements `CanActivate`, intercepting requests after authentication middleware but before route handler dispatching. Using `Reflector.getAllAndOverride`, the guard extracts role metadata from the handler level, falling back to class-level metadata if method-level rules are omitted.

#### 4. Global guard registration

Registering the guard globally using the `APP_GUARD` token guarantees that every incoming request passes through role verification across all controllers without requiring repetitive manual guard bindings per controller.

#### 5. Authorization execution flow

During request dispatching:

```text
JwtAuthGuard -> ApplicationUserRolesGuard -> Controller Handler
```

1. The authentication guard (`JwtAuthGuard`) validates token signatures and populates `request.user`.
2. `ApplicationUserRolesGuard` intercepts the HTTP execution context.
3. `Reflector` extracts role metadata defined on the handler, falling back to class-level metadata if present.
4. If no role metadata is detected, execution continues.
5. If metadata is present, the guard validates that `request.user.role` matches one of the required roles.
6. If a match exists, access is granted. Otherwise, a `ForbiddenException` (HTTP 403) terminates the request immediately.

### Production implementation

Implement the role enum, decorator, reflection guard, controller integration, and module registration.

```typescript
import {
  Body,
  CanActivate,
  Controller,
  Delete,
  ExecutionContext,
  ForbiddenException,
  Get,
  Injectable,
  Logger,
  Module,
  Param,
  Patch,
  SetMetadata,
} from "@nestjs/common";
import { APP_GUARD, Reflector } from "@nestjs/core";

export enum ApplicationUserRoleEnum {
  ADMIN = "ADMIN",
  OWNER = "OWNER",
  USER = "USER",
}

export const ROLES_METADATA_KEY = "roles";

export const RequiredRoles = (...roles: ApplicationUserRoleEnum[]) =>
  SetMetadata(ROLES_METADATA_KEY, roles);

interface AuthenticatedUser {
  id: string;
  role: ApplicationUserRoleEnum;
}

interface RequestWithUser {
  user?: AuthenticatedUser;
}

export class UpdateUserDto {
  name?: string;
  email?: string;
}

@Injectable()
export class ApplicationUserRolesGuard implements CanActivate {
  private readonly logger = new Logger(ApplicationUserRolesGuard.name);

  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<ApplicationUserRoleEnum[]>(
      ROLES_METADATA_KEY,
      [context.getHandler(), context.getClass()]
    );

    if (!requiredRoles || requiredRoles.length === 0) {
      return true;
    }

    const request = context.switchToHttp().getRequest<RequestWithUser>();
    const user = request.user;

    if (!user || !user.role || !requiredRoles.includes(user.role)) {
      this.logger.error(`Unauthorized role. Required: ${requiredRoles.join(", ")}`);
      throw new ForbiddenException("You do not have access to perform this action");
    }

    return true;
  }
}

@Injectable()
export class ApplicationUserService {
  getUser(id: string): string {
    return `User ${id} details`;
  }

  updateUser(id: string, dto: UpdateUserDto): string {
    return `Updated user ${id} with data ${JSON.stringify(dto)}`;
  }

  deleteUser(id: string): string {
    return `Deleted user ${id}`;
  }
}

@Controller("users")
export class ApplicationUserController {
  constructor(private readonly userService: ApplicationUserService) {}

  @Get(":id")
  findOne(@Param("id") id: string): string {
    return this.userService.getUser(id);
  }

  @Patch(":id")
  @RequiredRoles(ApplicationUserRoleEnum.ADMIN, ApplicationUserRoleEnum.OWNER)
  update(
    @Param("id") id: string,
    @Body() updateUserDto: UpdateUserDto
  ): string {
    return this.userService.updateUser(id, updateUserDto);
  }

  @Delete(":id")
  @RequiredRoles(ApplicationUserRoleEnum.ADMIN)
  remove(@Param("id") id: string): string {
    return this.userService.deleteUser(id);
  }
}

@Module({
  controllers: [ApplicationUserController],
  providers: [
    ApplicationUserService,
    {
      provide: APP_GUARD,
      useClass: ApplicationUserRolesGuard,
    },
  ],
})
export class ApplicationUserModule {}
```

### Architectural trade-offs and edge cases

Centralizing access control through NestJS guards introduces architectural trade-offs between flexibility, performance, and authorization complexity.

* **Latency versus consistency**: Guard execution occurs completely in-memory using cached reflection metadata and claims already attached to `request.user`. Latency impact is sub-millisecond, which guarantees consistency across every registered route.
* **Failure recovery**: If metadata extraction encounters unexpected null contexts or malformed user objects, the guard defaults to closed-door denial, throwing an explicit HTTP 403 Forbidden rather than allowing unverified execution to proceed.
* **Scale limitations**: Static role enums satisfy standard role-based access control. However, when domain rules require resource ownership checks (such as verifying a user owns the specific document being edited) or attribute-based access control (ABAC), simple static role metadata becomes insufficient. Such systems require extending guards with database repository queries or integrating dynamic policy engines like CASL.
* **Hierarchical roles**: In static enum comparisons, higher roles do not automatically inherit lower permissions unless explicitly specified in the `@RequiredRoles` decorator list or resolved through a role hierarchy lookup table inside the guard.

### Common anti-patterns and gotchas

* **Mismatched metadata keys**: Developers pass the decorator function reference itself into `reflector.get()` rather than using the defined metadata key string (`ROLES_METADATA_KEY`). The reflector fails to find metadata, silently allowing unauthorized requests through. Always use a consistent string constant for metadata keys.
* **Execution order inversion**: Registering the role guard before the authentication guard executes causes `request.user` to be undefined. The role guard rejects legitimate requests or fails unexpectedly. Always ensure authentication guards populate `request.user` prior to authorization guard execution.
* **Missing token claim extraction**: Developers assume `request.user.role` is automatically available without configuring the underlying Passport JWT strategy to extract and map the role claim from the decoded JWT payload.
* **Inline authorization duplication**: Developers manually duplicate role assertions with inline `if` conditions inside controller handlers. This negates the declarative auditability of guards and creates security drift when route signatures change.
* **Ignoring class-level inheritance**: Failing to use `getAllAndOverride` causes class-level security baselines to be ignored when handler-level metadata is absent.

### Implementation checklist

1. Define a centralized `ApplicationUserRoleEnum` covering all system roles.
2. Implement the `@RequiredRoles()` decorator using `SetMetadata` with a dedicated constant key.
3. Build the `ApplicationUserRolesGuard` implementing `CanActivate` with `Reflector.getAllAndOverride`.
4. Validate that Passport JWT strategies populate `request.user.role` during token verification.
5. Register `ApplicationUserRolesGuard` globally via `APP_GUARD` in the core module.
6. Apply `@RequiredRoles` to privileged controller routes and verify that unassigned endpoints remain accessible to authenticated users.
7. Log denied authorization attempts with required role contexts to monitor potential privilege escalation attempts.
8. Write automated end-to-end tests validating both 200 OK and 403 Forbidden responses across public, restricted, and administrative routes.