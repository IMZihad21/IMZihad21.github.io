# JWT with OIDC Authentication in Distributed Systems: Building Trust at Scale

- Canonical URL: https://imzihad21.github.io/articles/a/jwt-with-oidc-authentication-in-distributed-systems-building-trust-at-scale-2nno/
- Source URL: https://dev.to/imzihad21/jwt-with-oidc-authentication-in-distributed-systems-building-trust-at-scale-2nno
- Web View: https://imzihad21.github.io/articles/a/jwt-with-oidc-authentication-in-distributed-systems-building-trust-at-scale-2nno/
- Published: 2025-09-17T15:31:13.000Z
- Modified: 2025-09-17T15:31:13.000Z
- Reading time: 3 minutes
- Tags: dotnet, authentication, oidc, jwt

In distributed systems, services should validate tokens independently without sharing signing secrets. That is exactly where asymmetric JWT + OIDC discovery works beautifully.

This guide shows a trust model where one authority signs tokens and all services validate them using public keys from standard discovery endpoints.

### Why It Matters

- Removes shared-secret sprawl across services.
- Enables independent token validation per service.
- Supports safe key rotation with minimal friction.
- Aligns with OIDC/JWKS standards used across modern ecosystems.

### Core Concepts

#### 1. Central Signing Authority

One token service signs JWTs with private RSA key. Other services never see that private key.

#### 2. Distributed Validation

Consumer services validate signature using public keys loaded from authority metadata/JWKS.

#### 3. OIDC Discovery Endpoints

Expose `.well-known/openid-configuration` and `jwks` endpoints.

#### 4. Key Identification and Rotation

Set `kid` in signing key and JWT header so validators pick correct public key.

#### 5. Middleware Trust Configuration

Use `Authority`, issuer/audience validation, and lifetime checks.

#### 6. Security Baselines

Use HTTPS metadata in production and protect private key lifecycle.

### Practical Example

#### Token Service (Signer)

```csharp
public sealed class AsymmetricTokenProvider
{
    private readonly RsaSecurityKey _signatureKey;
    private readonly JwtSecurityTokenHandler _tokenHandler = new();
    private readonly TokenConfiguration _config;
    private readonly JsonWebKey _publicJwk;

    public AsymmetricTokenProvider(TokenConfiguration config)
    {
        _config = config;

        var rsa = RSA.Create();
        rsa.ImportFromPem(config.PrivateKey);

        _signatureKey = new RsaSecurityKey(rsa)
        {
            KeyId = GenerateKeyFingerprint(rsa)
        };

        _publicJwk = GeneratePublicJwk(_signatureKey);
    }

    public string CreateToken(IEnumerable<Claim> assertions)
    {
        var descriptor = new SecurityTokenDescriptor
        {
            Issuer = _config.Issuer,
            Audience = _config.Audience,
            Subject = new ClaimsIdentity(assertions),
            Expires = DateTime.UtcNow.Add(_config.DefaultDuration),
            SigningCredentials = new SigningCredentials(_signatureKey, SecurityAlgorithms.RsaSha256)
        };

        var token = _tokenHandler.CreateToken(descriptor);
        return _tokenHandler.WriteToken(token);
    }

    public JsonWebKey GetPublicJwk() => _publicJwk;

    private static string GenerateKeyFingerprint(RSA rsa)
    {
        var parameters = rsa.ExportParameters(false);
        using var hasher = SHA256.Create();
        var hash = hasher.ComputeHash(parameters.Modulus!);
        return Base64UrlEncoder.Encode(hash.AsSpan(0, 16));
    }

    private static JsonWebKey GeneratePublicJwk(RsaSecurityKey signatureKey)
    {
        var publicParams = signatureKey.Rsa!.ExportParameters(false);
        var publicKey = RSA.Create();
        publicKey.ImportParameters(publicParams);

        var jwk = JsonWebKeyConverter.ConvertFromRSASecurityKey(
            new RsaSecurityKey(publicKey) { KeyId = signatureKey.KeyId }
        );

        jwk.Alg = SecurityAlgorithms.RsaSha256;
        jwk.Use = "sig";
        return jwk;
    }
}
```

#### JWT Bearer Validation Middleware

```csharp
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = configuration["Jwt:AuthorityUrl"];
        options.RequireHttpsMetadata = true;

        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            ValidateIssuer = true,
            ValidIssuer = configuration["Jwt:IssuingAuthority"],
            ValidateAudience = true,
            ValidAudience = configuration["Jwt:Audience"],
            ValidateLifetime = true,
            ClockSkew = TimeSpan.FromMinutes(2)
        };
    });
```

#### OIDC Discovery + JWKS Endpoints

```csharp
[ApiController]
[Route(".well-known")]
[AllowAnonymous]
public sealed class DiscoveryEndpoints : ControllerBase
{
    private readonly AsymmetricTokenProvider _tokenProvider;
    private readonly LinkGenerator _urlGenerator;

    public DiscoveryEndpoints(AsymmetricTokenProvider tokenProvider, LinkGenerator urlGenerator)
    {
        _tokenProvider = tokenProvider;
        _urlGenerator = urlGenerator;
    }

    [HttpGet("openid-configuration")]
    public IActionResult GetOpenIdConfiguration()
    {
        var keysEndpoint = _urlGenerator.GetUriByAction(
            HttpContext,
            nameof(GetJsonWebKeys),
            controller: "DiscoveryEndpoints"
        );

        return Ok(new
        {
            issuer = "https://api.example.com",
            jwks_uri = keysEndpoint
        });
    }

    [HttpGet("jwks")]
    public IActionResult GetJsonWebKeys()
    {
        var jwk = _tokenProvider.GetPublicJwk();
        return Ok(new { keys = new[] { jwk } });
    }
}
```

This pattern keeps trust centralized while validation remains distributed. High scale, low secret chaos.

### Common Mistakes

- Sharing private signing keys with downstream services.
- Setting `RequireHttpsMetadata = false` in production.
- Mismatching token `issuer` and discovery `issuer` values.
- Rotating keys without stable `kid` strategy.
- Overly large clock skew windows that weaken validation strictness.

### Quick Recap

- One authority signs JWTs with private key.
- Services validate via public keys from JWKS.
- OIDC discovery standardizes trust metadata distribution.
- `kid` enables safe key rotation.
- HTTPS + strict token validation keep system trustworthy.

### Next Steps

1. Add automated key rotation with overlap window for old/new keys.
2. Add token revocation strategy for high-risk scenarios.
3. Add distributed audit logging for auth failures.
4. Add integration tests for discovery and JWKS rollover behavior.