# Implementing TOTP-Based Two-Factor Authentication in .NET Web API

- Canonical URL: https://imzihad21.github.io/articles/a/implementing-totp-two-factor-authentication-in-net-web-api-d7n/
- Source URL: https://dev.to/imzihad21/implementing-totp-two-factor-authentication-in-net-web-api-d7n
- Web View: https://imzihad21.github.io/articles/a/implementing-totp-two-factor-authentication-in-net-web-api-d7n/
- Published: 2025-02-16T07:46:40.000Z
- Modified: 2025-02-16T07:46:40.000Z
- Reading time: 3 minutes
- Tags: dotnet, security, twofactorauth, qrcode

Password authentication alone is not enough for modern security requirements. TOTP-based 2FA adds a second factor, which significantly reduces account takeover risk.

This guide shows a practical .NET Web API implementation: QR enrollment, OTP verification, and two-step login.

### Why It Matters

- Adds strong protection beyond password-only auth.
- Works with standard authenticator apps.
- Helps align with modern security compliance expectations.
- Reduces phishing and credential stuffing impact.

### Core Concepts

#### 1. 2FA Enrollment with QR Code

Generate a TOTP secret, build OTP URI, create QR image, and return a protected enrollment payload.

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

#### 2. Confirm and Store Authenticator Secret

Validate OTP from user, then persist verified authenticator key.

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

#### 3. Step 1 Login (Password Check)

Validate credentials first, then return temporary challenge token for OTP step.

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

#### 4. Step 2 Login (OTP Verification)

Decrypt challenge token, load user, verify OTP, and issue final access token.

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

#### 5. OTP Validation Window

Allow small clock skew using verification window.

```csharp
private static bool IsOtpValid(string authKey, string code)
{
    if (string.IsNullOrWhiteSpace(authKey) || string.IsNullOrWhiteSpace(code))
        return false;

    var totp = new Totp(Base32Encoding.ToBytes(authKey));
    return totp.VerifyTotp(code, out _, new VerificationWindow(previous: 2, future: 2));
}
```

#### 6. Security Guardrails

- Keep signing/encryption keys in configuration or secret manager.
- Encrypt TOTP secrets at rest.
- Add rate limiting for OTP attempts.

### Practical Example

Authentication flow:

1. User submits email/password.
2. API validates credentials.
3. If 2FA enabled, API returns OTP challenge token.
4. User submits OTP + challenge token.
5. API validates OTP and issues access token.

When implemented cleanly, users get stronger security without painful login UX.

### Common Mistakes

- Hardcoding challenge encryption/signing keys.
- Enabling 2FA before first OTP verification succeeds.
- Storing authenticator secret in plain text.
- Skipping OTP rate limits and lockout policies.
- Ignoring clock drift during OTP verification.

### Quick Recap

- TOTP adds a second verification factor.
- Enrollment requires secret generation and QR provisioning.
- Login should be two-step when 2FA is enabled.
- OTP verification should tolerate small time drift.
- Secrets and challenge keys must be managed securely.

### Next Steps

1. Add backup recovery codes for account recovery.
2. Add brute-force protection for OTP endpoint.
3. Add audit logs for 2FA setup and verification failures.
4. Move secret storage to dedicated vault-backed encryption.