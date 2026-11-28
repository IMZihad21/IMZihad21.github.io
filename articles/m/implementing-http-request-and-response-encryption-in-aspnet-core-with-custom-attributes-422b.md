# Implementing HTTP Request and Response Encryption in ASP.NET Core with Custom Attributes

- Canonical URL: https://imzihad21.github.io/articles/a/implementing-http-request-and-response-encryption-in-aspnet-core-with-custom-attributes-422b/
- Source URL: https://dev.to/imzihad21/implementing-http-request-and-response-encryption-in-aspnet-core-with-custom-attributes-422b
- Web View: https://imzihad21.github.io/articles/a/implementing-http-request-and-response-encryption-in-aspnet-core-with-custom-attributes-422b/
- Published: 2024-11-07T06:15:22.000Z
- Modified: 2024-11-07T06:15:22.000Z
- Reading time: 7 minutes
- Tags: dotnet, cryptography, security, webdev

## Implementing HTTP request and response encryption in ASP.NET Core with custom attributes

Sensitive APIs require HTTPS by default, but specific applications demand payload encryption at the application layer. This approach encrypts request bodies and query parameters beyond the transport layer, protecting payloads from intermediate logging proxies, internal service meshes, or compromised network infrastructure.

This implementation creates an opt-in encryption pipeline in ASP.NET Core using a custom attribute that implements `IFilterFactory`, an asynchronous resource filter that wraps request and response streams, and a matching client-side Axios interceptor. The architecture decrypts incoming data before model binding runs, encrypts outgoing payloads before network dispatch, and centralizes cryptographic operations without altering controller action logic.

### The problem and production context

Transport Layer Security (TLS) terminates at the perimeter when applications sit behind reverse proxies, load balancers, or API gateways. In regulated environments, unencrypted payloads pass across internal networks where inspection tools, log aggregators, or internal network taps can read them. Standard ASP.NET Core pipelines process request bodies as plain text, forcing developers to manually handle decryption and encryption in individual actions if they need payload isolation.

- **Failure scenario**: A controller action receives sensitive data through query strings or request bodies while intermediate proxies capture access logs containing plain text parameters. Alternatively, manual controller-level decryption leads to inconsistent implementations where developers forget to decrypt incoming models or leak raw cryptographic exceptions directly back to API consumers.
- **Why default approaches fall short**: ASP.NET Core model binding executes before action filters run. If manual decryption is placed inside the controller action body, model binding fails or binds garbage ciphertext to typed parameters. Furthermore, scattering cryptographic operations across individual controllers leads to mismatched key derivations, code duplication, and heightened risk of omitted encryption on newly added sensitive routes.
- **Production impact**: Unencrypted sensitive payloads are captured in network telemetry, crash dumps, and proxy logs. Inconsistent cryptographic implementations lead to runtime serialization failures, unhandled exceptions that reveal internal infrastructure details to clients, and administrative overhead during secret rotation.

Production considerations addressed by this design:
- Centralizes encryption logic in reusable middleware components rather than scattered controller code.
- Allows endpoints to opt in declaratively without altering controller actions or business logic.
- Applies consistent cryptographic handling across request bodies, response bodies, and query strings.
- Reduces the risk of omitted encryption on sensitive routes through standardized attribute decoration.

### Mental model and core concepts

#### 1. Attribute-driven activation

The `EncryptedTransportAttribute` implements `IFilterFactory` to enable declarative activation per controller or per action method while supporting dependency injection. `IFilterFactory.CreateInstance` resolves configuration options (`ApiEncryptionOptions`) from the DI container and instantiates the resource filter with configured shared secrets.

```csharp
using Microsoft.AspNetCore.Mvc.Filters;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Options;

[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method)]
public sealed class EncryptedTransportAttribute : Attribute, IFilterFactory
{
    public bool IsReusable => false;

    public IFilterMetadata CreateInstance(IServiceProvider serviceProvider)
    {
        var encryptionOptions = serviceProvider
            .GetRequiredService<IOptions<ApiEncryptionOptions>>();

        return new EncryptedTransportFilter(encryptionOptions.Value);
    }
}
```

#### 2. Resource filter pipeline and stream interception

The ASP.NET Core resource filter (`IAsyncResourceFilter`) executes immediately after authorization filters and before model binding. By intercepting `ResourceExecutingContext`, the filter wraps `context.HttpContext.Request.Body` with a decrypting `CryptoStream` and replaces `context.HttpContext.Response.Body` with an encrypting `CryptoStream`. Any encrypted query string is intercepted, decrypted, and rebound to `context.HttpContext.Request.QueryString` prior to model binding.

```csharp
using System.Security.Cryptography;
using System.Text;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc.Filters;

public sealed class EncryptedTransportFilter : IAsyncResourceFilter
{
    private readonly Aes _aesProvider;

    public EncryptedTransportFilter(ApiEncryptionOptions options)
    {
        _aesProvider = CreateAesProvider(options.SharedSecret);
    }

    public async Task OnResourceExecutionAsync(
        ResourceExecutingContext context,
        ResourceExecutionDelegate next)
    {
        var originalRequestBody = context.HttpContext.Request.Body;
        var originalResponseBody = context.HttpContext.Response.Body;

        await using var decryptedRequestBody = CreateDecryptionStream(originalRequestBody);
        await using var encryptedResponseBody = CreateEncryptionStream(originalResponseBody);

        context.HttpContext.Request.Body = decryptedRequestBody;
        context.HttpContext.Response.Body = encryptedResponseBody;

        if (context.HttpContext.Request.QueryString.HasValue)
        {
            var encryptedQuery = context.HttpContext.Request.QueryString.Value![1..];
            var decryptedQuery = DecryptText(encryptedQuery);
            context.HttpContext.Request.QueryString = new QueryString($"?{decryptedQuery}");
        }

        await next();

        await encryptedResponseBody.FlushAsync();
        context.HttpContext.Request.Body = originalRequestBody;
        context.HttpContext.Response.Body = originalResponseBody;
    }

    private CryptoStream CreateEncryptionStream(Stream responseStream)
    {
        var encryptor = _aesProvider.CreateEncryptor();
        var base64Encoder = new ToBase64Transform();
        var base64Stream = new CryptoStream(responseStream, base64Encoder, CryptoStreamMode.Write);

        return new CryptoStream(base64Stream, encryptor, CryptoStreamMode.Write);
    }

    private CryptoStream CreateDecryptionStream(Stream requestStream)
    {
        var decryptor = _aesProvider.CreateDecryptor();
        var base64Decoder = new FromBase64Transform(FromBase64TransformMode.IgnoreWhiteSpaces);
        var decodedStream = new CryptoStream(requestStream, base64Decoder, CryptoStreamMode.Read);

        return new CryptoStream(decodedStream, decryptor, CryptoStreamMode.Read);
    }

    private string DecryptText(string encryptedText)
    {
        using var cipherBuffer = new MemoryStream(Convert.FromBase64String(encryptedText));
        using var cryptoStream = new CryptoStream(cipherBuffer, _aesProvider.CreateDecryptor(), CryptoStreamMode.Read);
        using var textReader = new StreamReader(cryptoStream);

        return textReader.ReadToEnd();
    }

    private static Aes CreateAesProvider(string sharedSecret)
    {
        var normalizedSecret = sharedSecret.PadRight(32, '0');

        var aes = Aes.Create();
        aes.Key = Encoding.UTF8.GetBytes(normalizedSecret[..32]);
        aes.IV = Encoding.UTF8.GetBytes(normalizedSecret[..16]);
        aes.Mode = CipherMode.CBC;
        aes.Padding = PaddingMode.PKCS7;

        return aes;
    }
}
```

#### 3. Controller or action scope application

The attribute can be applied globally across an entire controller class or scoped selectively to individual action methods, enabling granular control over which routes enforce payload encryption.

Applying at the controller level:

```csharp
[EncryptedTransport]
[Route("api/[controller]")]
public sealed class SecurePayloadController : ControllerBase
{
    [HttpPost("submit")]
    public IActionResult Submit([FromBody] SensitivePayload request)
    {
        return Ok(request.Process());
    }
}
```

Applying at the action level:

```csharp
[Route("api/[controller]")]
public sealed class SecurePayloadController : ControllerBase
{
    [EncryptedTransport]
    [HttpPost("submit")]
    public IActionResult Submit([FromBody] SensitivePayload request)
    {
        return Ok(request.Process());
    }
}
```

#### 4. Client-side interceptor contract

The client must follow the exact same transformation rules for request bodies and query parameters as the server. An Axios request interceptor detects outgoing query strings and request bodies, transforms them using the shared encryption routine, sets `Content-Type: application/json`, and registers a response transformer to decrypt incoming payloads.

```javascript
import axios from "axios";
import { API_ROOT } from "../../constants/NetworkConfig";
import {
  encryptTransportPayload,
  decryptTransportPayload,
} from "../../utilities/transportCrypto";

const encryptedApiClient = axios.create({ baseURL: API_ROOT });

encryptedApiClient.interceptors.request.use((requestConfig) => {
  const [basePath, rawQuery] = requestConfig.url ? requestConfig.url.split("?") : [];

  if (rawQuery) {
    requestConfig.url = `${basePath}?${encryptTransportPayload(rawQuery)}`;
  }

  if (requestConfig.data) {
    requestConfig.headers["Content-Type"] = "application/json";
    requestConfig.transformRequest = [encryptTransportPayload];
  }

  requestConfig.transformResponse = [decryptTransportPayload];

  return requestConfig;
});
```

#### 5. Shared cryptographic utility rules

Client and server must share identical key derivations, cipher modes (such as AES-CBC), and padding schemes (such as PKCS7). Mismatched configuration blocks prevent the underlying streams from completing decryption.

#### 6. Security boundaries

Application-layer encryption provides defense in depth against compromised intermediate infrastructure and unauthorized log inspection. It does not replace HTTPS or Transport Layer Security (TLS), which remains mandatory for channel confidentiality, certificate verification, and protection against man-in-the-middle attacks.

### Production implementation

The following implementation contains the client-side cryptographic utility module that mirrors the server AES-CBC and Base64 format.

```javascript
import CryptoJS from "crypto-js";
import { TRANSPORT_SHARED_SECRET } from "../constants/appSettings";

const normalizedSecret = TRANSPORT_SHARED_SECRET.padEnd(32, "0");
const aesKey = CryptoJS.enc.Utf8.parse(normalizedSecret.substring(0, 32));
const aesIv = CryptoJS.enc.Utf8.parse(normalizedSecret.substring(0, 16));

const aesTransportConfig = {
  iv: aesIv,
  mode: CryptoJS.mode.CBC,
  padding: CryptoJS.pad.Pkcs7,
};

export const encryptTransportPayload = (payload) => {
  if (payload === null || payload === undefined) {
    return payload;
  }

  const serializedPayload = CryptoJS.enc.Utf8.parse(
    typeof payload === "string" ? payload : JSON.stringify(payload)
  );

  const encryptedPayload = CryptoJS.AES.encrypt(serializedPayload, aesKey, aesTransportConfig);
  return CryptoJS.enc.Base64.stringify(encryptedPayload.ciphertext);
};

export const decryptTransportPayload = (encodedPayload) => {
  if (!encodedPayload) {
    return encodedPayload;
  }

  try {
    const cipherBytes = CryptoJS.enc.Base64.parse(encodedPayload);

    const decryptedText = CryptoJS.AES.decrypt(
      { ciphertext: cipherBytes },
      aesKey,
      aesTransportConfig
    ).toString(CryptoJS.enc.Utf8);

    try {
      return JSON.parse(decryptedText);
    } catch {
      return decryptedText;
    }
  } catch {
    return encodedPayload;
  }
};
```

This keeps encryption and decryption logic centralized instead of scattering cryptographic operations across individual API calls.

### Architectural trade-offs and edge cases

* **Latency versus consistency**: Application-layer cryptographic operations introduce CPU overhead and stream buffering latency. For large payloads, Base64 encoding expands payload size by approximately 33 percent over the network. In return, the architecture guarantees end-to-end payload confidentiality through internal gateways and intermediate loggers.
* **Failure recovery**: Corrupted payloads or mismatched cipher keys cause `CryptographicException` during stream reads. The filter must catch stream decoding failures gracefully and return structured HTTP 400 Bad Request responses rather than allowing unhandled 500 exceptions to bubble up and terminate connections.
* **Scale limitations**: Synchronous or high-volume encryption on multi-megabyte payloads can increase memory pressure and garbage collection frequency due to intermediate byte arrays and stream buffers. For high-throughput file streaming endpoints, native transport encryption (TLS) or chunked hardware-accelerated envelopes should be preferred.
* **Defense in depth vs replacement**: Application-layer encryption must operate alongside TLS rather than as a substitute. Without TLS, the initial key exchange and transport headers remain vulnerable to interception and tampering.

### Common anti-patterns and gotchas

* **Treating application-layer encryption as an alternative to HTTPS**: Developers disable TLS thinking payload encryption is sufficient. This exposes HTTP headers, authentication tokens, and metadata to interception; HTTPS must always remain enabled.
* **Mismatched key or IV derivation between client and server**: Client and server codebases evolve independently, leading to divergent padding rules, encoding formats, or string truncation. Standardize derivation algorithms and verify them with cross-platform unit tests.
* **Omitting query-string decryption prior to MVC model binding**: Developers attempt to decrypt query parameters in an action filter or inside the controller action. Because model binding occurs after the resource filter but before action filters, failing to rewrite `QueryString` in the resource filter causes model binding to fail.
* **Leaking raw cryptographic exceptions back to client callers**: Cryptographic padding or authentication failures throw internal exceptions. Exposing stack traces or specific cryptographic errors in API responses provides oracle attack vectors to malicious actors. Return generic bad request errors instead.
* **Hardcoding secrets and omitting secret rotation procedures**: Embedding static shared secrets directly in source repositories creates severe vulnerability if repositories are compromised. Maintain keys in secure key management vaults and support versioned key rotation.

### Implementation checklist

1. Replace static IV derivation with per-request cryptographically random IVs prepended to ciphertexts.
2. Upgrade to authenticated encryption (such as AES-GCM) to verify payload integrity and prevent padding oracle attacks.
3. Add a version header or payload prefix to support backwards-compatible algorithm updates during secret rotations.
4. Define structured, sanitized error responses for decryption failures that do not expose internal cryptographic exceptions.
5. Register `ApiEncryptionOptions` with strong secrets retrieved from environment variables or a key vault.
6. Verify that query string and payload decoding execute before ASP.NET Core model binding triggers.