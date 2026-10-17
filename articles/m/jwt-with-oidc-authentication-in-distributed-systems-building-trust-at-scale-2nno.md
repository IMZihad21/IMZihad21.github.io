# JWT with OIDC Authentication in Distributed Systems: Building Trust at Scale

- Canonical URL: https://imzihad21.github.io/articles/a/jwt-with-oidc-authentication-in-distributed-systems-building-trust-at-scale-2nno/
- Source URL: https://dev.to/imzihad21/jwt-with-oidc-authentication-in-distributed-systems-building-trust-at-scale-2nno
- Web View: https://imzihad21.github.io/articles/a/jwt-with-oidc-authentication-in-distributed-systems-building-trust-at-scale-2nno/
- Published: 2025-09-17T15:31:13.000Z
- Modified: 2025-09-17T15:31:13.000Z
- Reading time: 5 minutes
- Tags: dotnet, authentication, oidc, jwt

## JWT with OIDC authentication in distributed systems: building trust at scale

In distributed architectures, microservices must validate incoming tokens independently without relying on shared symmetric secrets. Sharing symmetric secret keys across dozens of microservices breaches the principle of least privilege, turns every consumer service repository into an attack vector, and creates tight deployment coupling during key rotation.

This architecture establishes an asymmetric trust model where a single authorization service issues signed tokens using a private RSA key, while downstream microservices autonomously validate incoming tokens using public keys retrieved from standard OpenID Connect (OIDC) JSON Web Key Set (JWKS) discovery endpoints. The design eliminates shared secrets, removes synchronous validation bottlenecks, and provides zero-downtime key rotation.

### The problem and production context

Microservice ecosystems often start by sharing a single symmetric HMAC secret (such as HS256) across all backend services. Under this model, every service that verifies authentication tokens possesses the exact secret required to forge valid tokens for any user or role. Furthermore, rotating a shared symmetric secret requires coordinating simultaneous, zero-downtime deployments across every service in the cluster.

- **Failure scenario**: A low-privilege internal microservice repository is compromised or leaks its configuration logs. An attacker extracts the shared symmetric HMAC secret and forges administrative tokens with arbitrary scopes and lifetimes across the entire distributed system. Alternatively, during an emergency key rotation, out-of-sync service deployments reject valid user requests, causing widespread service outages.
- **Why default approaches fall short**: Synchronous token verification via remote introspection calls (such as calling `/oauth/introspect` for every incoming HTTP request) introduces network latency, creates a single point of failure, and degrades overall cluster availability.
- **Production impact**: Shared-secret compromises lead to lateral system takeovers. Coordinated deployments during key rotation cause operational friction and downtime, while synchronous RPC introspection introduces latency spikes that degrade high-throughput distributed microservices.

Operational requirements addressed by this design:
- Eliminates shared-secret distribution across multiple service teams and repositories.
- Enables autonomous token validation without inter-service RPC overhead.
- Simplifies cryptographic key rotation without requiring simultaneous service redeployments.
- Adheres to standard OIDC and JWKS specifications supported across languages and frameworks.

### Mental model and core concepts

#### 1. Central signing authority

A dedicated authorization service holds the private RSA key and signs outgoing JWTs using RS256. Downstream consumer services never have access to this private key and cannot forge tokens.

#### 2. Autonomous distributed validation

Consumer APIs fetch the signing authority's public keys once and cache them in memory. Inbound request tokens are verified locally by calculating the RSA signature against the public key, avoiding network round-trips for each request.

#### 3. OpenID Connect discovery endpoints

The signing authority exposes standard `.well-known/openid-configuration` and `jwks` endpoints to advertise current validation keys, cryptographic algorithms, and issuer identities in standard format.

#### 4. Cryptographic key identification and rotation

Assigning a unique key ID (`kid`) to each key and embedding it into the JWT header ensures validators retrieve the matching public key from the JWKS cache during active rotation periods.

#### 5. Middleware trust configuration

Consumer services configure standard JWT bearer authentication handlers to validate issuer, audience, lifetime, and signing keys against the authority metadata automatically.

#### 6. Security baselines and perimeter isolation

Enforce HTTPS for metadata discovery endpoints in production environments and secure the private key lifecycle with strict hardware security module (HSM) or cloud vault access controls.

### Production implementation

The following complete components implement the asymmetric token issuer, the standard OpenID Connect discovery controller, and the downstream ASP.NET Core JWT bearer authentication configuration.

First, the asymmetric token provider that signs tokens and generates public JWK representations:

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

Next, the discovery controller exposing standard `.well-known` endpoints for client consumption:

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

Finally, the consumer service configuration that establishes trust against the issuing authority:

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

This pattern centralizes signing authority while keeping token validation decoupled across consumer services.

### Architectural trade-offs and edge cases

* **Latency versus consistency**: In-memory JWKS validation eliminates inter-service RPC latency during request processing. However, because tokens are validated statelessly without calling the authorization server, token revocation is delayed until the token expires, unless a distributed token blacklist is checked.
* **Failure recovery**: Downstream services cache JWKS keys. If the discovery endpoint experiences temporary downtime, consumer services continue validating tokens using cached public keys until the cache lifetime expires.
* **Scale limitations**: RSA signature validation is computationally more intensive than symmetric HMAC verification. For high-throughput microservices handling heavy request volumes, caching verified token subjects or offloading signature checks to an API gateway sidecar (such as Envoy) mitigates CPU load.
* **Key rotation overlap**: During key transitions, the authorization server must publish both new and old public keys in `jwks` while issuing tokens with the new key. Consumer services continue accepting existing valid tokens signed with the previous key until they naturally expire.

### Common anti-patterns and gotchas

* **Distributing private signing keys across downstream consumer services**: Exposing private RSA keys outside the central identity authority breaches the security model and gives any service the power to forge arbitrary tokens.
* **Disabling RequireHttpsMetadata in production**: Setting `RequireHttpsMetadata = false` in production exposes JWKS retrieval to man-in-the-middle attacks, allowing malicious actors to substitute rogue public keys.
* **Mismatches between token issuer claims and advertised discovery URLs**: If the `iss` claim in issued JWTs does not match `ValidIssuer` or the discovery metadata issuer string exactly, all tokens are rejected with 401 Unauthorized errors.
* **Rotating keys without a distinct kid header**: Failing to include a unique `kid` in the JWT header forces downstream validators to guess which public key to use or triggers cache validation failures during key transitions.
* **Configuring excessively large clock skew allowances**: Setting `ClockSkew` to large durations (such as 15 or 30 minutes) effectively extends token lifetimes and delays the enforcement of expired sessions.

### Implementation checklist

1. Generate a strong RSA private key pair (2048-bit minimum) and store it securely in a dedicated secret store or key vault.
2. Implement the `AsymmetricTokenProvider` to sign JWTs with RS256 and calculate reproducible `kid` fingerprints.
3. Expose standard `.well-known/openid-configuration` and `.well-known/jwks` discovery routes from the identity provider.
4. Configure downstream consumer APIs with `options.Authority` pointing to the identity provider discovery URL.
5. Set `RequireHttpsMetadata = true` on all production microservices to guarantee transport security for public keys.
6. Implement automated key rotation schedules with an overlap window supporting both current and previous public keys in the JWKS set.
7. Design a token revocation mechanism, such as distributed deny lists in Redis, for immediate revocation during security incidents.
8. Establish centralized security logging for failed signature validations across downstream services.
9. Add automated integration tests to verify JWKS rollover without service downtime.