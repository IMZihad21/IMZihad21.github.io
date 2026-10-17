# Comprehensive Encryption and Security Service in NestJS: Argon2 Hashing, Token Generation, and AES Encryption

- Canonical URL: https://imzihad21.github.io/articles/a/comprehensive-encryption-and-security-service-in-nestjs-argon2-hashing-token-generation-and-aes-encryption-595o/
- Source URL: https://dev.to/imzihad21/comprehensive-encryption-and-security-service-in-nestjs-argon2-hashing-token-generation-and-aes-encryption-595o
- Web View: https://imzihad21.github.io/articles/a/comprehensive-encryption-and-security-service-in-nestjs-argon2-hashing-token-generation-and-aes-encryption-595o/
- Published: 2024-11-11T20:25:00.000Z
- Modified: 2024-11-11T20:25:00.000Z
- Reading time: 5 minutes
- Tags: nestjs, argon2, aes, encryption

## Comprehensive encryption and security service in NestJS: Argon2, token generation, and AES encryption

Security mechanisms frequently scatter across controllers and domain services, leading to inconsistent implementations, configuration drift, and weak cryptographic defaults. A dedicated NestJS security service keeps password hashing, token generation, and symmetric encryption logic centralized in one auditable module.

Building a modular `EncryptionService` provides an auditable provider managing Argon2 password hashing, secure token generation, and symmetric encryption across an application.

### The problem and production context

Fragmenting cryptographic routines across service modules introduces implementation flaws and key management vulnerabilities.

- **Failure scenario**: A controller generates reset tokens using `Math.random()` or encrypts sensitive database columns using AES-256-CBC with a static zero-filled initialization vector (IV). An attacker analyzes ciphertext patterns across database dumps to identify recurring plaintext values and predicts pseudo-random reset tokens to take over administrative accounts.
- **Why default approaches fall short**: Individual feature developers frequently default to fast, insecure algorithms or re-implement ad-hoc token generators. Without a centralized provider, application error handlers leak detailed cryptographic stack traces to API clients, and static hardcoded fallback secrets persist into production environments.
- **Production impact**: Predictable token generation leads to account takeover, weak password hashing enables GPU dictionary cracking, and unauthenticated cipher modes allow ciphertext tampering and padding-oracle attacks.

Centralizing cryptographic logic delivers distinct architectural benefits:
- Centralizes sensitive cryptographic routines into a single injectable provider.
- Eliminates duplicate, divergent crypto implementations across feature modules.
- Simplifies security audits and compliance reviews.
- Provides consistent helper utilities for authentication flows and data protection at rest.

### Mental model and core concepts

Centralizing cryptographic operations requires isolating hashing, pseudorandom token generation, and symmetric encryption behind strong contracts.

#### 1. Core service configuration

Retrieve encryption secrets from environment configuration and maintain explicit crypto parameters. The service reads secrets via `ConfigService` and configures cipher parameters such as the 16-byte initialization vector and algorithm identifier.

#### 2. Argon2 password hashing and verification

Argon2 provides memory-hard protection against GPU-accelerated cracking attacks. The service wraps `argon2.hash` and `argon2.verify`, enforcing mandatory parameter validation and logging failures while throwing standardized NestJS HTTP exceptions.

#### 3. Cryptographically secure temporary password generation

Generating temporary credentials requires uniform entropy. Instead of predictable pseudo-random number generators, `randomInt` from the Node.js `crypto` module generates indices across character sets (lowercase, uppercase, numbers, and symbols) to produce secure temporary passwords.

#### 4. Unique URL-safe token generation

Session identifiers, email verification tokens, and password reset links require URL-safe string encoding. Concatenating UUIDv4 identifiers, converting hexadecimal strings into byte buffers, and encoding them via `base64url` generates collision-resistant, URL-safe authentication tokens.

#### 5. Symmetric encryption with AES-256-CBC

The service derives a 256-bit key from the configured secret using SHA-256 and encrypts plaintext strings using AES-256-CBC, outputting a URL-safe Base64 encoded payload.

#### 6. Decryption and payload reconstruction

Decryption reverses the URL-safe character replacements, applies padding modulo 4, and initializes the AES decipher to restore original plaintext utf8 strings.

### Production implementation

The following implementation provides the complete NestJS `EncryptionService` and global `EncryptionModule`.

```typescript
import {
  BadRequestException,
  Global,
  Injectable,
  InternalServerErrorException,
  Logger,
  Module,
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

  generateUniqueToken(length: number = 3): string {
    const mergedUuid = Array.from({ length }, () => uuidv4()).join("");
    const tokenBuffer = Buffer.from(mergedUuid.replace(/-/g, ""), "hex");
    return base64url(tokenBuffer);
  }

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
}

@Global()
@Module({
  providers: [EncryptionService],
  exports: [EncryptionService],
})
export class EncryptionModule {}
```

### Architectural trade-offs and edge cases

Centralizing security primitives inside a shared provider involves specific architectural constraints and risk boundaries.

* **Latency versus consistency**: Argon2 password hashing introduces deliberate CPU and memory utilization delays to counter offline cracking attacks. High-throughput endpoints requiring frequent credential checks should balance Argon2 cost settings against HTTP request timeout budgets and worker availability.
* **Failure recovery**: Cryptographic errors must be caught and logged internally while returning sanitized `InternalServerErrorException` or `BadRequestException` messages to callers. Leaking OpenSSL cipher error traces across HTTP responses exposes internal padding details to attackers.
* **Scale limitations**: In-process symmetric encryption using Node.js `crypto` executes within the Node thread pool for asynchronous methods or on the main event loop for synchronous routines. High-volume streaming encryption workloads should use Node streaming ciphers to prevent blocking event loop execution.

### Common anti-patterns and gotchas

* **Default fallback secrets in production**: Falling back to a hardcoded default string when `ENCRYPTION_SECRET` is missing in production undermines cryptographic confidentiality.
* **Predictable pseudo-random generation**: Using `Math.random()` to generate password reset tokens or temporary passwords allows attackers to predict future sequences.
* **Static initialization vectors**: Reusing a static zero-filled IV across all encryption calls allows attackers to identify duplicate plaintext blocks across records via frequency analysis.
* **Leaking cryptographic error messages**: Exposing raw cryptographic decipher errors to clients facilitates padding-oracle attacks.
* **Unversioned static encryption keys**: Storing encrypted database records without key version prefixes prevents zero-downtime key rotation when credentials expire or leak.

### Implementation checklist

1. Ensure `ENCRYPTION_SECRET` is defined in production environment configurations without fallback secrets.
2. Register `EncryptionModule` as a `@Global()` module in `AppModule`.
3. Refactor encryption routines to generate a cryptographically random IV per operation and prepend it to ciphertext payloads.
4. Upgrade CBC mode to authenticated encryption modes such as AES-256-GCM to prevent ciphertext tampering.
5. Incorporate key version identifiers into cipher payloads to support zero-downtime key rotation.
6. Implement rate limiting on authentication and password verification endpoints to protect against brute-force attacks.