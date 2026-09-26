# Integrating Stripe Payment Intent in NestJS with Webhook Handling

- Canonical URL: https://imzihad21.github.io/articles/a/integrating-stripe-payment-intent-in-nestjs-with-webhook-handling-1n65/
- Source URL: https://dev.to/imzihad21/integrating-stripe-payment-intent-in-nestjs-with-webhook-handling-1n65
- Web View: https://imzihad21.github.io/articles/a/integrating-stripe-payment-intent-in-nestjs-with-webhook-handling-1n65/
- Published: 2024-11-11T20:54:41.000Z
- Modified: 2024-11-11T20:54:41.000Z
- Reading time: 6 minutes
- Tags: nestjs, stripe, payment, webhook

## Integrating Stripe PaymentIntent in NestJS with webhook handling

Integrating Stripe into backend services centers on two fundamental tasks: creating payment intents reliably and validating incoming webhook events securely. Failing to verify webhooks or handle state transitions properly causes internal booking records to diverge from payment realities, leading to unfulfilled customer orders or duplicate fulfillment.

This implementation establishes an asynchronous payment processing pipeline in NestJS using Stripe PaymentIntents, unparsed raw body buffering, and cryptographic HMAC-SHA256 signature verification. The architecture guarantees that payments are authorized through secure client secrets, incoming webhook events are cryptographically authenticated before processing, and state transitions reconcile reliably between Stripe and local database records.

### The problem and production context

E-commerce and reservation systems fail when they rely exclusively on frontend redirects to confirm financial transactions. When a customer completes checkout on Stripe's hosted UI or mobile elements, network disconnects, closed browser tabs, or client tampering can prevent the success callback from reaching the backend server. Furthermore, public webhook endpoints accept incoming HTTP requests from any source, allowing malicious actors to forge payment success payloads if signatures are not verified.

- **Failure scenario**: An application relies on a frontend client redirect callback to transition an order from `Pending` to `Paid`. A customer loses connectivity immediately after card authorization, causing the frontend script never to execute. The customer's credit card is charged by Stripe, but the order remains unfulfilled in the merchant database. Alternatively, an attacker sends fabricated JSON payloads to an unverified webhook endpoint, marking orders as completed without paying.
- **Why default approaches fall short**: Express-based frameworks like NestJS parse incoming JSON payloads into JavaScript objects by default. During this parsing, whitespace differences, property reordering, or character re-encoding alter the raw byte sequence. When Stripe's cryptographic verification utility calculates HMAC against a re-serialized JSON string, the signature comparison fails.
- **Production impact**: Unverified webhooks introduce severe financial fraud vulnerabilities. Unhandled raw body requirements lead to signature verification exceptions that reject valid Stripe webhooks, causing order status divergence and customer escalation.

Operational requirements addressed by this design:
- Keeps internal booking and order states synchronized with Stripe payment transitions.
- Validates cryptographic webhook signatures to block spoofed requests.
- Consolidates payment provider operations inside dedicated service classes.
- Delivers a structured API contract to frontend checkout clients.

### Mental model and core concepts

#### 1. Raw body preservation in NestJS bootstrap

Stripe webhook signature validation requires the exact, unparsed byte sequence transmitted across the network. If the body is modified or re-serialized by JSON parsers, the cryptographic HMAC signature will not match. The NestJS bootstrap configuration must explicitly set `rawBody: true` on the `NestExpressApplication` instance to attach `req.rawBody` as a raw Node.js Buffer.

```typescript
import { NestFactory } from "@nestjs/core";
import { NestExpressApplication } from "@nestjs/platform-express";
import { AppModule } from "./app.module";

async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(AppModule, {
    rawBody: true,
  });

  await app.listen(4000);
}

bootstrap();
```

#### 2. Controller routing and boundary isolation

The controller exposes an authenticated endpoint for client-side checkout initialization and a public, unauthenticated webhook endpoint that receives raw body payloads and Stripe signature headers.

```typescript
import { Controller, Get, Headers, HttpCode, Param, Post, Req } from "@nestjs/common";
import { ApiExcludeEndpoint, ApiResponse, ApiTags } from "@nestjs/swagger";
import { RawBodyRequest } from "@nestjs/common";
import { Request } from "express";

@ApiTags("Payment Receive")
@Controller("PaymentReceive")
export class PaymentReceiveController {
  constructor(private readonly paymentService: PaymentReceiveService) {}

  @Get("GetIntentByBookingId/:DocId")
  @ApiResponse({ status: 200, type: SuccessResponseDto })
  getPaymentIntent(
    @AuthUserId() { userId }: ITokenPayload,
    @Param() { DocId }: DocIdQueryDto
  ) {
    return this.paymentService.getPaymentIntent(DocId, userId);
  }

  @Post("StripeWebhook")
  @HttpCode(200)
  @IsPublic()
  @ApiExcludeEndpoint()
  async handleStripeWebhook(
    @Req() req: RawBodyRequest<Request>,
    @Headers("stripe-signature") signature: string
  ): Promise<{ received: boolean }> {
    return await this.paymentService.handleStripeWebhook(req, signature);
  }
}
```

#### 3. Stripe service initialization

The service initializes the official Stripe SDK, resolving API credentials, publishable keys, and webhook signing secrets through NestJS `ConfigService`.

```typescript
import { BadRequestException, Injectable, Logger } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import Stripe from "stripe";

@Injectable()
export class StripePaymentService {
  private readonly logger = new Logger(StripePaymentService.name);
  private readonly stripeService: Stripe;
  private readonly stripeWebhookSecret: string;
  private readonly stripePublishableKey: string;

  constructor(private readonly configService: ConfigService) {
    const stripeSecretKey = this.configService.get<string>("STRIPE_SECRET_KEY", "");
    this.stripePublishableKey = this.configService.get<string>("STRIPE_PUBLISHABLE_KEY", "");
    this.stripeWebhookSecret = this.configService.get<string>("STRIPE_WEBHOOK_SECRET", "");

    this.stripeService = new Stripe(stripeSecretKey, {
      apiVersion: "2024-06-20",
    });
  }
}
```

#### 4. PaymentIntent creation workflow

Creating a PaymentIntent reserves the intent to charge and allows embedding custom audit metadata (such as booking identifiers and customer IDs). The service returns the client secret and publishable key to the caller, allowing frontend checkout elements to complete authorization.

```typescript
async getPaymentIntent(bookingId: string, auditUserId: string): Promise<SuccessResponseDto> {
  try {
    const intentMetadata: StripeIntentMetadata = {
      bookingCode: "bookingCode",
      bookingId,
      paymentReceiveId: "paymentReceiveId",
      auditUserId,
    };

    const stripeIntent = await this.stripeService.paymentIntents.create({
      amount: 1000,
      currency: "usd",
      metadata: intentMetadata,
      automatic_payment_methods: { enabled: true },
    });

    const paymentIntentResponse = new PaymentIntentResponseDto();
    paymentIntentResponse.stripeKey = this.stripePublishableKey;
    paymentIntentResponse.stripeSecret = stripeIntent.client_secret ?? "";
    paymentIntentResponse.status = stripeIntent.status;
    paymentIntentResponse.currency = stripeIntent.currency;

    return new SuccessResponseDto(
      "Payment intent retrieved successfully",
      paymentIntentResponse
    );
  } catch (error) {
    this.logger.error("Error in getPaymentIntent", error as Error);
    throw new BadRequestException("Failed to get payment intent");
  }
}
```

#### 5. Webhook signature verification and event handling

Incoming webhook requests pass through `stripeService.webhooks.constructEvent`, which verifies the HMAC-SHA256 signature using `request.rawBody`, the `stripe-signature` header, and the configured webhook signing secret.

```typescript
import { RawBodyRequest } from "@nestjs/common";
import { Request } from "express";

async handleStripeWebhook(
  request: RawBodyRequest<Request>,
  signature: string
): Promise<{ received: boolean }> {
  try {
    if (!this.stripeWebhookSecret) {
      throw new Error("Stripe webhook secret is missing");
    }

    const event = this.stripeService.webhooks.constructEvent(
      request.rawBody as Buffer,
      signature,
      this.stripeWebhookSecret
    );

    switch (event.type) {
      case "payment_intent.created":
        break;
      case "payment_intent.succeeded":
        break;
      case "payment_intent.payment_failed":
        break;
      default:
        this.logger.log(`Unhandled event type: ${event.type}`);
    }

    return { received: true };
  } catch (error) {
    this.logger.error("Error handling Stripe webhook event", error as Error);
    throw new BadRequestException("Error in webhook event processing");
  }
}
```

#### 6. Module composition

Encapsulating controllers, payment services, and persistence repositories within a dedicated feature module maintains clean architectural boundaries.

```typescript
import { Module } from "@nestjs/common";

@Module({
  controllers: [PaymentReceiveController],
  providers: [PaymentReceiveService, PaymentReceiveRepository],
})
export class PaymentReceiveModule {}
```

### Production implementation

The following complete state machine maps Stripe webhook event types directly to internal transaction records and domain entities.

```typescript
export class PaymentLifecycleMapper {
  static mapStripeEventToDomainStatus(eventType: string): string {
    switch (eventType) {
      case "payment_intent.created":
        return "PaymentCreated";
      case "payment_intent.succeeded":
        return "PaymentCompleted";
      case "payment_intent.payment_failed":
        return "PaymentFailed";
      default:
        return "PaymentPending";
    }
  }
}
```

This mapping establishes deterministic state synchronization:
- `payment_intent.created` -> `PaymentCreated`
- `payment_intent.succeeded` -> `PaymentCompleted`
- `payment_intent.payment_failed` -> `PaymentFailed`

This keeps booking status synchronized with Stripe lifecycle events in real time.

### Architectural trade-offs and edge cases

* **Latency versus consistency**: Processing asynchronous webhooks introduces a short propagation delay between customer card authorization and internal database updates. However, this decoupling guarantees payment status consistency even when client connections terminate prematurely during payment completion.
* **Failure recovery**: Stripe automatically retries failed webhook deliveries with exponential backoff over a 72-hour window. If the backend service experiences temporary downtime, incoming events replay once connectivity resumes.
* **Scale limitations**: High-frequency webhook traffic during marketing spikes can overwhelm database write pools if webhooks execute heavy synchronous queries. Webhook handlers should validate signatures quickly, place the event onto an internal worker queue (such as BullMQ), and return an immediate HTTP 200 acknowledgment.
* **Idempotent processing**: Network retries mean the same webhook event can be delivered more than once. Webhook consumers must check whether an event ID has already been recorded before executing financial mutations.

### Common anti-patterns and gotchas

* **Omitting rawBody during NestJS bootstrap**: Forgetting `rawBody: true` in `NestFactory.create` causes `req.rawBody` to be undefined. The webhook signature validation fails on every request with an internal exception.
* **Parsing webhook payloads without verifying stripe-signature**: Accepting unverified JSON directly allows anyone on the internet to send fake payment success payloads, compromising transaction integrity.
* **Relying solely on client-side redirect callbacks**: Updating order records based only on frontend callbacks causes abandoned or failed orders when clients close browsers or suffer connection loss. Always treat webhooks as the source of truth for payment status.
* **Placing business logic directly in controllers**: Scattering database transactions and external API calls inside controller route handlers violates separation of concerns and hinders automated testing.
* **Failing to handle duplicate deliveries idempotently**: Processing `payment_intent.succeeded` twice without checking existing state can trigger duplicate fulfillment actions or inventory deductions.

### Implementation checklist

1. Enable `rawBody: true` in `main.ts` within `NestFactory.create<NestExpressApplication>`.
2. Add `STRIPE_SECRET_KEY`, `STRIPE_PUBLISHABLE_KEY`, and `STRIPE_WEBHOOK_SECRET` to environment configuration.
3. Configure the webhook route with `@IsPublic()` and `@HttpCode(200)` to ensure Stripe receives standard success acknowledgments.
4. Verify cryptographic signatures using `stripe.webhooks.constructEvent` with `req.rawBody` and the `stripe-signature` header.
5. Use idempotency keys during PaymentIntent creation to prevent duplicate charges.
6. Store processed Stripe event IDs in the database to guard against webhook replays.
7. Wrap booking state and payment record updates within database transactions.
8. Set up local integration tests using the Stripe CLI to forward real test webhooks during development.