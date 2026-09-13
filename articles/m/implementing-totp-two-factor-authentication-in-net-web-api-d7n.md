# Implementing TOTP-Based Two-Factor Authentication in .NET Web API

- Canonical URL: https://imzihad21.github.io/articles/a/implementing-totp-two-factor-authentication-in-net-web-api-d7n/
- Source URL: https://dev.to/imzihad21/implementing-totp-two-factor-authentication-in-net-web-api-d7n
- Web View: https://imzihad21.github.io/articles/a/implementing-totp-two-factor-authentication-in-net-web-api-d7n/
- Published: 2025-02-16T07:46:40.000Z
- Modified: 2025-02-16T07:46:40.000Z
- Reading time: 7 minutes
- Tags: dotnet, security, twofactorauth, qrcode

## Implementing TOTP two-factor authentication in .NET Web API

Password authentication alone does not protect users against modern credential attacks. Time-based one-time password (TOTP) two-factor authentication adds a second verification layer that mitigates the risk of account takeover from credential stuffing, password reuse, and phishing campaigns.

This implementation establishes an RFC 6238 compliant TOTP authentication system in .NET Web API featuring QR code enrollment, out-of-band verification, encrypted ephemeral challenge tokens, and a two-stage authentication pipeline. The architecture guarantees that authenticator secrets are confirmed before activation, challenge tokens prevent intermediate session leaks, and temporal verification windows accommodate device clock drift without compromising security.

### The problem and production context

Single-factor password authentication exposes web applications to account takeover when credentials leak across third-party breaches or phishing attacks. While standard authenticator applications provide multi-factor security, naive server implementations introduce operational vulnerabilities. Storing shared secrets in plain text, enabling multi-factor flags before verifying device synchronization, or issuing premature session tokens during login challenges defeats the purpose of second-factor defense.

- **Failure scenario**: An application enables two-factor authentication on a user account immediately upon generating a QR code. If the user closes the enrollment screen without successfully scanning the barcode, they are locked out of their account on subsequent logins. Alternatively, an attacker intercepts a step-one login response that exposes user identifiers in plain text, enabling targeted OTP brute-forcing.
- **Why default approaches fall short**: Naive login workflows return temporary full-privilege tokens or expose predictable user database primary keys during authentication challenges. Furthermore, systems that validate OTP codes using strict single-timestamp comparisons fail intermittently in production due to minor NTP clock skew between mobile devices and server clusters.
- **Production impact**: Preventable lockouts overwhelm support desks, plain text secret compromises lead to complete bypass of two-factor protections, and unmitigated OTP brute-forcing endpoints allow automated tools to exhaust six-digit keyspaces within minutes.

Operational capabilities provided by this implementation:
- Protects accounts beyond single-factor passwords against automated credential stuffing.
- Integrates with standard authenticator apps like Google Authenticator and Microsoft Authenticator.
- Satisfies industry and regulatory security compliance requirements.
- Mitigates the fallout from credential stuffing and phishing attacks across distributed user bases.

### Mental model and core concepts

#### 1. Two-factor authentication enrollment with QR code generation

The enrollment workflow generates a 160-bit (20-byte) cryptographically random secret key encoded in Base32. It formats the secret into a standard `otpauth://` URI with the service issuer name, user email, SHA-1 algorithm, 6-digit output, and a 30-second rotation period. To prevent client-side tampering during enrollment confirmation, the generated secret key is encrypted using the user's unique security stamp before dispatching the payload with the Base64 QR code image.

```csharp
[HttpGet("CreateProtectedQr")]
public async Task<IActionResult> CreateProtectedQr()
{
    var user = await GetCurrentUserAsync();

    if (user == null || string.IsNullOrWhiteSpace(user.UniqueId))
        return BadRequest("User not valid.");

    var secretKey = Base32Encoding.ToString(KeyGeneration.GenerateRandomKey(20));

    var serviceName = Uri.EscapeDataString(_config["Totp:ServiceIssuer"] ?? "MyApp");
    var totpUri = $"otpauth://totp/{serviceName}:{Uri.EscapeDataString(user.Email)}" +
                  $"?secret={secretKey}&issuer={serviceName}&algorithm=SHA1&digits=6&period=30";

    var qrBase64 = GenerateQrCodeBase64(totpUri);
    var encryptedKey = CryptoHelper.Encrypt(secretKey, user.UniqueId);

    return Ok(new ProtectedQrResponse
    {
        QrCodeBase64 = qrBase64,
        EncryptedKey = encryptedKey
    });
}
```

#### 2. Confirm and store authenticator secret

To prevent account lockouts, the system enables two-factor authentication only after the user proves successful configuration by submitting a valid code. The action decrypts the protected key using the user's security identifier, validates the submitted OTP against the decrypted key, and only then marks `IsTwoFactorEnabled = true` while persisting the authenticator secret.

```csharp
[HttpPut("UpdateUserAuthenticator")]
public async Task<IActionResult> UpdateUserAuthenticator(AuthenticatorUpdateRequest input)
{
    var currentUser = await GetCurrentUserAsync();

    if (currentUser == null || string.IsNullOrWhiteSpace(currentUser.UniqueId))
        return BadRequest("User not valid.");

    var decryptedKey = CryptoHelper.Decrypt(input.EncryptedKey, currentUser.UniqueId);

    if (!IsOtpValid(decryptedKey, input.OTP))
        return BadRequest("OTP verification failed.");

    currentUser.AuthKey = decryptedKey;
    currentUser.IsTwoFactorEnabled = true;

    _unitOfWork.Users.Update(currentUser);
    await _unitOfWork.SaveChangesAsync();

    return Ok(true);
}
```

#### 3. Step 1 login: primary credential validation and challenge issuance

The initial login endpoint verifies primary password credentials. If two-factor authentication is disabled for the account, the endpoint issues a standard access token immediately. If two-factor authentication is active, the endpoint halts full authentication and issues an encrypted ephemeral OTP identifier signed with an internal challenge signing key.

```csharp
[AllowAnonymous]
[HttpPost("LoginUser")]
public async Task<IActionResult> LoginUser(UserLoginRequest input)
{
    var user = await GetUserByCredentialsAsync(input);

    if (user == null)
        return Unauthorized("Credentials are invalid.");

    if (!user.IsTwoFactorEnabled)
    {
        var (token, expiry) = GenerateAccessToken(user);
        return Ok(new UserSessionOutput
        {
            AccessToken = token,
            AccessTokenExpiry = expiry
        });
    }

    var secureSigningKey = _config["Security:OtpChallengeKey"]
        ?? throw new InvalidOperationException("Missing OTP challenge signing key.");

    var encryptedIdentifier = CryptoHelper.Encrypt(user.Id.ToString(), secureSigningKey);

    return Ok(new UserLoginResponse
    {
        EncryptedOtpIdentifier = encryptedIdentifier
    });
}
```

#### 4. Step 2 login: OTP verification and session token issuance

The secondary login endpoint accepts the encrypted challenge identifier and the user's 6-digit TOTP code. It decrypts the challenge payload using the internal server key, parses the underlying user ID, loads the user record, and validates the OTP code. Upon successful verification, the system generates and returns the final access token and session expiration metadata.

```csharp
[AllowAnonymous]
[HttpPost("ValidateOtpLogin")]
public async Task<IActionResult> ValidateOtpLogin(OtpLoginInput input)
{
    var secureSigningKey = _config["Security:OtpChallengeKey"]
        ?? throw new InvalidOperationException("Missing OTP challenge signing key.");

    var decryptedId = CryptoHelper.Decrypt(input.EncryptedOtpIdentifier, secureSigningKey);

    if (!long.TryParse(decryptedId, out var userId))
        return BadRequest("Invalid user identifier.");

    var account = await _unitOfWork.Users.GetById(userId);

    if (account == null)
        return Unauthorized("User not found.");

    if (!IsOtpValid(account.AuthKey, input.OTP))
        return BadRequest("Invalid OTP.");

    var (accessToken, accessExpiry) = GenerateAccessToken(account);

    return Ok(new UserSessionOutput
    {
        AccessToken = accessToken,
        AccessTokenExpiry = accessExpiry
    });
}
```

#### 5. Temporal verification window and clock drift accommodation

Mobile devices and server clocks experience slight drift over time. A strict 30-second timestamp comparison causes validation failures when clocks diverge by even a few seconds. The verification mechanism configures a `VerificationWindow` allowing matching codes across the current window, two previous intervals (-60s), and two future intervals (+60s).

```csharp
private static bool IsOtpValid(string authKey, string code)
{
    if (string.IsNullOrWhiteSpace(authKey) || string.IsNullOrWhiteSpace(code))
        return false;

    var totp = new Totp(Base32Encoding.ToBytes(authKey));
    return totp.VerifyTotp(code, out _, new VerificationWindow(previous: 2, future: 2));
}
```

#### 6. Security guardrails and operational boundaries

To protect the authentication perimeter against compromised keys and automated attacks:
- Store challenge signing and encryption keys in environment configuration or a hardware key vault.
- Encrypt TOTP secrets at rest in the database using envelope encryption.
- Apply strict rate limiting and account lockout policies to all OTP submission endpoints.

### Production implementation

The following authentication workflow coordinates the two-stage login lifecycle across client and server components.

```csharp
public async Task<UserSessionOutput> ExecuteFullAuthenticationPipelineAsync(
    UserLoginRequest credentials,
    Func<string, Task<string>> promptForOtpCodeCallback)
{
    var initialLoginResult = await LoginUser(credentials);

    if (initialLoginResult is OkObjectResult okResult)
    {
        if (okResult.Value is UserSessionOutput standardSession)
        {
            return standardSession;
        }

        if (okResult.Value is UserLoginResponse challengeResponse)
        {
            var otpCode = await promptForOtpCodeCallback(challengeResponse.EncryptedOtpIdentifier);
            var validationInput = new OtpLoginInput
            {
                EncryptedOtpIdentifier = challengeResponse.EncryptedOtpIdentifier,
                OTP = otpCode
            };

            var finalResult = await ValidateOtpLogin(validationInput);

            if (finalResult is OkObjectResult validatedResult && validatedResult.Value is UserSessionOutput finalSession)
            {
                return finalSession;
            }
        }
    }

    throw new UnauthorizedAccessException("Authentication failed during credential or two-factor validation.");
}
```

The flow operates in five deterministic stages:
1. The user submits their email and password.
2. The API validates credentials against the database.
3. If two-factor authentication is enabled, the API returns an encrypted challenge identifier.
4. The client submits the TOTP code alongside the encrypted challenge identifier.
5. The API verifies the code and issues the access token.

This design enforces multi-factor security without complicating the normal login path for accounts where two-factor authentication is disabled.

### Architectural trade-offs and edge cases

* **Latency versus consistency**: Verifying cryptographic hashes and decrypting encrypted identifiers introduces minor processing overhead during authentication requests. In return, the architecture ensures that no user database IDs or preliminary unauthenticated tokens are exposed to the client.
* **Failure recovery**: If a user loses their authenticator device or synchronizes their phone clock improperly beyond the configured drift window, standard TOTP checks fail. Backup recovery codes or an authenticated administrator reset pipeline must be provided to prevent permanent account lockout.
* **Scale limitations**: In multi-server web farms, all API nodes must synchronize system clocks via Network Time Protocol (NTP). Server clock deviations greater than 30 seconds disrupt the entire verification window across different cluster instances.
* **Secret encryption at rest**: Encrypting TOTP secrets in the database introduces a dependency on the application's data protection keys. When keys rotate, existing TOTP secrets must remain decryptable or be migrated to new key rings.

### Common anti-patterns and gotchas

* **Hardcoding challenge encryption or signing keys**: Developers place encryption keys directly in application source code or source control. If the repository is exposed, attackers can forge challenge tokens for any user account. Store keys securely in cloud key vaults or environment variables.
* **Enabling two-factor authentication before verification**: Marking `IsTwoFactorEnabled = true` when generating the QR code locks out users who encounter network disruptions or fail to scan the barcode. Always verify a submitted code before committing the enabled state.
* **Storing TOTP secrets in plain text**: Persisting raw Base32 secret keys directly in database columns allows anyone with database read access to generate valid authentication codes indefinitely. Always encrypt secrets at rest.
* **Omitting rate limits and account lockout policies**: Because TOTP codes consist of only six digits (1,000,000 combinations), an unthrottled endpoint can be brute-forced within a single 30-second window. Enforce rate limiting by IP and username.
* **Ignoring clock drift between client devices and servers**: Enforcing an exact zero-drift comparison causes frequent intermittent authentication failures due to minor device clock variations. Configure a controlled verification window.

### Implementation checklist

1. Register `Totp:ServiceIssuer` and `Security:OtpChallengeKey` in environment variables or key vault configuration.
2. Implement cryptographic helper utilities to encrypt TOTP secrets at rest in database storage.
3. Build the QR code generation endpoint with Base32 secret encoding and encrypted enrollment state.
4. Verify user-submitted OTP codes before marking `IsTwoFactorEnabled = true` in the database.
5. Structure the login endpoint to return encrypted challenge tokens rather than session tokens when 2FA is active.
6. Configure the `VerificationWindow` with balanced past and future tolerances to handle minor device clock skew.
7. Apply rate limiting and lockout thresholds to the OTP validation endpoint to prevent automated brute-force attacks.
8. Implement single-use recovery codes to allow self-service account recovery if authenticator devices are lost.
9. Verify that NTP clock synchronization is active across all application server nodes.
10. Record audit logs for two-factor setup and failed verification attempts.