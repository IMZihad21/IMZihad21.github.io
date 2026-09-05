# Comprehensive Encryption and Security Service in NestJS: Argon2 Hashing, Token Generation, and AES Encryption

- Canonical URL: https://imzihad21.github.io/articles/a/comprehensive-encryption-and-security-service-in-nestjs-argon2-hashing-token-generation-and-aes-encryption-595o/
- Source URL: https://dev.to/imzihad21/comprehensive-encryption-and-security-service-in-nestjs-argon2-hashing-token-generation-and-aes-encryption-595o
- Web View: https://imzihad21.github.io/articles/a/comprehensive-encryption-and-security-service-in-nestjs-argon2-hashing-token-generation-and-aes-encryption-595o/
- Published: 2024-11-11T20:25:00.000Z
- Modified: 2024-11-11T20:25:00.000Z
- Reading time: 3 minutes
- Tags: nestjs, argon2, aes, encryption

## Comprehensive Encryption and Security Service in NestJS: Argon2, Token Generation, and AES Encryption

Security features often spread across many files and become inconsistent over time. A dedicated NestJS security service keeps hashing, token generation, and encryption logic centralized.

This guide shows a modular `EncryptionService` that handles password hashing, token generation, and symmetric encryption in one reusable layer.

### Why It Matters

- Centralizes sensitive cryptographic operations in one service.
- Reduces repeated and inconsistent security code.
- Improves maintainability and audit readiness.
- Supports secure auth and sensitive-data workflows.

### Core Concepts

#### 1. Core Service Configuration

Load secrets from environment variables and keep crypto parameters explicit.

```typescript
import {
  BadRequestException,
  Injectable,
  InternalServerErrorException,
  Logger,
} from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import * as argon2 from "argon2";
import { createCipheriv, createDecipheriv, createHash, randomInt } from "crypto";
import { v4 as uuidv4 } from "uuid";
import base64url from "base64url";

@Injectable()
export class EncryptionService {
  private readonly logger = new Logger(EncryptionService.name);
  private readonly algorithm = "aes-256-cbc";
  private readonly ivLength = 16;
  private readonly secretKey: string;

  constructor(private readonly configService: ConfigService) {
    this.secretKey = this.configService.get<string>("ENCRYPTION_SECRET", "default_secret");
  }
}
```

#### 2. Argon2 Password Hashing

Use Argon2 for secure password hashing and verification.

```typescript
async hashPassword(rawPassword: string): Promise<string> {
  if (!rawPassword) {
    this.logger.error("Password is required");
    throw new BadRequestException("Password is required");
  }

  try {
    return await argon2.hash(rawPassword);
  } catch (error) {
    this.logger.error("Failed to hash password", error as Error);
    throw new InternalServerErrorException("Failed to hash password");
  }
}

async verifyPassword(rawPassword: string, hashedPassword: string): Promise<boolean> {
  if (!rawPassword || !hashedPassword) {
    this.logger.error("Password and hash are required");
    throw new BadRequestException("Password and hash are required");
  }

  try {
    return await argon2.verify(hashedPassword, rawPassword);
  } catch (error) {
    this.logger.error("Failed to verify password", error as Error);
    throw new InternalServerErrorException("Failed to verify password");
  }
}
```

#### 3. Temporary Password Generation

Generate temporary passwords using cryptographically secure randomness.

```typescript
generateTemporaryPassword(length = 12): string {
  if (length < 8) {
    throw new BadRequestException("Temporary password length must be at least 8");
  }

  const lowercaseChars = "abcdefghijklmnopqrstuvwxyz";
  const uppercaseChars = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";
  const numericChars = "0123456789";
  const specialChars = "!@#$%^&*()-_+=";
  const allChars = lowercaseChars + uppercaseChars + numericChars + specialChars;

  try {
    let password = "";

    for (let i = 0; i < length; i += 1) {
      password += allChars[randomInt(0, allChars.length)];
    }

    return password;
  } catch (error) {
    this.logger.error("Failed to generate temporary password", error as Error);
    throw new InternalServerErrorException("Failed to generate temporary password");
  }
}
```

#### 4. Unique Token Generation

Create URL-safe tokens from UUID-based random material.

```typescript
generateUniqueToken(length: number = 3): string {
  const mergedUuid = Array.from({ length }, () => uuidv4()).join("");
  const tokenBuffer = Buffer.from(mergedUuid.replace(/-/g, ""), "hex");
  return base64url(tokenBuffer);
}
```

#### 5. AES-256-CBC Encryption

Encrypt plain text and return URL-safe output.

```typescript
encryptString(text: string): string {
  if (!text) {
    throw new BadRequestException("Text is required for encryption");
  }

  try {
    const key = createHash("sha256").update(this.secretKey).digest();
    const iv = Buffer.alloc(this.ivLength, 0);

    const cipher = createCipheriv(this.algorithm, key, iv);
    let encrypted = cipher.update(text, "utf8", "base64");
    encrypted += cipher.final("base64");

    return encrypted.replace(/\+/g, "-").replace(/\//g, "_").replace(/=/g, "");
  } catch (error) {
    this.logger.error("Encryption failed", error as Error);
    throw new InternalServerErrorException("Encryption failed");
  }
}
```

#### 6. AES-256-CBC Decryption

Convert URL-safe payload back to original plain text.

```typescript
decryptString(cipherText: string): string {
  if (!cipherText) {
    throw new BadRequestException("Cipher text is required for decryption");
  }

  try {
    let base64Text = cipherText.replace(/-/g, "+").replace(/_/g, "/");

    while (base64Text.length % 4 !== 0) {
      base64Text += "=";
    }

    const key = createHash("sha256").update(this.secretKey).digest();
    const iv = Buffer.alloc(this.ivLength, 0);

    const decipher = createDecipheriv(this.algorithm, key, iv);
    let decrypted = decipher.update(base64Text, "base64", "utf8");
    decrypted += decipher.final("utf8");

    return decrypted;
  } catch (error) {
    this.logger.error("Decryption failed", error as Error);
    throw new InternalServerErrorException("Decryption failed");
  }
}
```

### Practical Example

Register as a global module for app-wide reuse:

```typescript
import { Global, Module } from "@nestjs/common";
import { EncryptionService } from "./encryption.service";

@Global()
@Module({
  providers: [EncryptionService],
  exports: [EncryptionService],
})
export class EncryptionModule {}
```

One service for hashing, tokens, and encryption means fewer security surprises and fewer copy-paste incidents.

### Common Mistakes

- Using weak or hardcoded encryption secrets.
- Generating passwords with `Math.random()` instead of crypto-safe randomness.
- Reusing insecure encryption settings without documenting trade-offs.
- Returning raw cryptographic errors to API clients.
- Forgetting to rotate secrets in production.

### Quick Recap

- Argon2 handles password hashing and verification.
- Secure temporary passwords should use crypto randomness.
- UUID + Base64URL token generation supports web-safe identifiers.
- AES encryption/decryption belongs in one centralized service.
- Global module registration keeps security behavior consistent.

### Next Steps

1. Replace fixed IV flow with random IV + prefixed payload format.
2. Move from CBC to AES-GCM for authenticated encryption.
3. Add secret rotation versioning to encrypted payloads.
4. Add rate limiting for password verification endpoints.