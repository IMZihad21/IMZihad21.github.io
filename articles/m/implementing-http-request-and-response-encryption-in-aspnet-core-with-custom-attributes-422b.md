# Implementing HTTP Request and Response Encryption in ASP.NET Core with Custom Attributes

- Canonical URL: https://imzihad21.github.io/articles/a/implementing-http-request-and-response-encryption-in-aspnet-core-with-custom-attributes-422b/
- Source URL: https://dev.to/imzihad21/implementing-http-request-and-response-encryption-in-aspnet-core-with-custom-attributes-422b
- Web View: https://imzihad21.github.io/articles/a/implementing-http-request-and-response-encryption-in-aspnet-core-with-custom-attributes-422b/
- Published: 2024-11-07T06:15:22.000Z
- Modified: 2024-11-07T06:15:22.000Z
- Reading time: 4 minutes
- Tags: dotnet, cryptography, security, webdev

For sensitive APIs, HTTPS is mandatory, but some teams also add application-layer encryption for payload-level protection. This approach can help when you need encrypted bodies and query values beyond standard transport security.

This guide shows an opt-in encryption pipeline in ASP.NET Core using a custom attribute, resource filter, and aligned client interceptor.

### Why It Matters

- Keeps encryption logic centralized and reusable.
- Lets endpoints opt in without polluting controller actions.
- Applies consistent request, response, and query handling.
- Reduces chances of missing encryption in sensitive routes.

### Core Concepts

#### 1. Attribute-Driven Activation

Use an attribute as a filter factory so encryption is declarative and route-specific.

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

#### 2. Resource Filter Pipeline

Intercept request and response streams for decryption/encryption around action execution.

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

#### 3. Controller or Action Scope

Apply encryption globally at controller level or selectively at action level.

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

#### 4. Client-Side Interceptor Contract

Client must follow the same encryption rules for body and query payloads.

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

#### 5. Shared Crypto Utility Rules

Client and server must use identical key shaping and cipher mode for interoperability.

#### 6. Security Boundaries

This is an additional protection layer. It does not replace HTTPS/TLS.

### Practical Example

Client utility aligned with server AES-CBC + Base64 flow:

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

This keeps encryption/decryption in one place instead of scattering crypto steps across every API call and future headache.

### Common Mistakes

- Treating application-layer encryption as replacement for HTTPS.
- Using mismatched key/IV derivation logic between client and server.
- Forgetting query-string decryption before model binding.
- Returning raw crypto errors to clients.
- Skipping secret rotation and secure secret storage.

### Quick Recap

- Use `[EncryptedTransport]` for declarative endpoint protection.
- Resource filter handles stream wrapping for request/response.
- Client interceptor applies matching body/query transforms.
- Keep cryptographic configuration centralized and consistent.
- Use this as extra protection on top of TLS.

### Next Steps

1. Replace static IV derivation with per-request random IV strategy.
2. Add authenticated encryption mode (for example AES-GCM) with integrity checks.
3. Add versioned transport format for backward-compatible client updates.
4. Add structured error responses for decryption failures.