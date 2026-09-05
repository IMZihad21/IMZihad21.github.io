# Custom Role-Based Access Control in NestJS Using Custom Guards

- Canonical URL: https://imzihad21.github.io/articles/a/custom-role-based-access-control-in-nestjs-using-custom-guards-jol/
- Source URL: https://dev.to/imzihad21/custom-role-based-access-control-in-nestjs-using-custom-guards-jol
- Web View: https://imzihad21.github.io/articles/a/custom-role-based-access-control-in-nestjs-using-custom-guards-jol/
- Published: 2024-11-11T20:04:34.000Z
- Modified: 2024-11-11T20:04:34.000Z
- Reading time: 3 minutes
- Tags: nestjs, rbac, authorization, security

Role checks scattered across controllers become fragile very fast. A custom guard with metadata-based role requirements gives centralized, reusable authorization behavior.

This guide shows a clean RBAC flow in NestJS using an enum, a decorator, and a global guard.

### Why It Matters

- Keeps role authorization logic in one guard instead of many endpoints.
- Makes access rules explicit and easy to review.
- Works naturally with JWT-based authentication pipelines.
- Supports future scaling to advanced policy models.

### Core Concepts

#### 1. Role Enumeration

Define a strict role set for compile-time safety.

```typescript
export enum ApplicationUserRoleEnum {
  ADMIN = "ADMIN",
  OWNER = "OWNER",
  USER = "USER",
}
```

#### 2. Required Roles Decorator

Use metadata to declare which roles can access an endpoint.

```typescript
import { SetMetadata } from "@nestjs/common";
import { ApplicationUserRoleEnum } from "../enum/application-user-role.enum";

export const ROLES_METADATA_KEY = "roles";

export const RequiredRoles = (...roles: ApplicationUserRoleEnum[]) =>
  SetMetadata(ROLES_METADATA_KEY, roles);
```

#### 3. Authorization Guard

Read metadata from handler/class and compare with authenticated user role.

```typescript
import {
  CanActivate,
  ExecutionContext,
  ForbiddenException,
  Injectable,
  Logger,
} from "@nestjs/common";
import { Reflector } from "@nestjs/core";
import { ApplicationUserRoleEnum } from "../enum/application-user-role.enum";
import { ROLES_METADATA_KEY } from "../decorator/roles.decorator";

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

    const request = context.switchToHttp().getRequest();
    const user = request.user as { role?: ApplicationUserRoleEnum } | undefined;

    if (!user?.role || !requiredRoles.includes(user.role)) {
      this.logger.error(`Unauthorized role. Required: ${requiredRoles.join(", ")}`);
      throw new ForbiddenException("You do not have access to perform this action");
    }

    return true;
  }
}
```

#### 4. Global Guard Registration

Register once with `APP_GUARD` for application-wide enforcement.

```typescript
import { Module } from "@nestjs/common";
import { APP_GUARD } from "@nestjs/core";
import { ApplicationUserRolesGuard } from "./guards/application-user-roles.guard";
import { ApplicationUserController } from "./controllers/application-user.controller";
import { ApplicationUserService } from "./services/application-user.service";

@Module({
  controllers: [ApplicationUserController],
  providers: [
    {
      provide: APP_GUARD,
      useClass: ApplicationUserRolesGuard,
    },
    ApplicationUserService,
  ],
})
export class ApplicationUserModule {}
```

#### 5. Endpoint-Level Role Rules

Apply `@RequiredRoles(...)` where needed.

```typescript
import { Body, Controller, Delete, Get, Param, Patch } from "@nestjs/common";
import { RequiredRoles } from "../decorator/roles.decorator";
import { ApplicationUserRoleEnum } from "../enum/application-user-role.enum";
import { UpdateUserDto } from "../dto/update-user.dto";

@Controller("users")
export class ApplicationUserController {
  @Get(":id")
  findOne(@Param("id") id: string) {
    return `User ${id} details`;
  }

  @Patch(":id")
  @RequiredRoles(ApplicationUserRoleEnum.ADMIN, ApplicationUserRoleEnum.OWNER)
  update(@Param("id") id: string, @Body() updateUserDto: UpdateUserDto) {
    return `Updated user ${id} with data ${JSON.stringify(updateUserDto)}`;
  }

  @Delete(":id")
  @RequiredRoles(ApplicationUserRoleEnum.ADMIN)
  remove(@Param("id") id: string) {
    return `Deleted user ${id}`;
  }
}
```

#### 6. Authorization Workflow

The runtime flow is:

- Guard intercepts request.
- Reflector reads required roles metadata.
- Guard compares user role against allowed roles.
- Access is granted or `ForbiddenException` is thrown.

### Practical Example

Typical auth stack composition:

```text
JwtAuthGuard -> ApplicationUserRolesGuard -> Controller Handler
```

This sequence ensures identity is verified first, then role permission is checked. If someone reaches an admin endpoint without admin role, the guard says no before business logic even wakes up.

### Common Mistakes

- Using decorator function reference instead of metadata key string in reflector lookup.
- Registering role guard without authentication guard in the pipeline.
- Assuming role data exists in `request.user` without JWT claim mapping.
- Duplicating role checks manually inside controller methods.
- Forgetting class-level metadata fallback when using shared controller rules.

### Quick Recap

- RBAC in NestJS is clean with enum + metadata decorator + guard.
- `APP_GUARD` centralizes enforcement.
- Reflector-based role lookup keeps endpoint policies declarative.
- Forbidden responses remain consistent and auditable.
- This pattern is extensible for larger authorization models.

### Next Steps

1. Add hierarchical roles with inheritance mapping.
2. Add resource-based checks for owner-specific records.
3. Add metrics for denied requests by endpoint and role.
4. Add e2e tests for authorized and forbidden role scenarios.