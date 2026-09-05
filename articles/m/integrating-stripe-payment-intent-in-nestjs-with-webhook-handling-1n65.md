# Integrating Stripe Payment Intent in NestJS with Webhook Handling

- Canonical URL: https://imzihad21.github.io/articles/a/integrating-stripe-payment-intent-in-nestjs-with-webhook-handling-1n65/
- Source URL: https://dev.to/imzihad21/integrating-stripe-payment-intent-in-nestjs-with-webhook-handling-1n65
- Web View: https://imzihad21.github.io/articles/a/integrating-stripe-payment-intent-in-nestjs-with-webhook-handling-1n65/
- Published: 2024-11-11T20:54:41.000Z
- Modified: 2024-11-11T20:54:41.000Z
- Reading time: 3 minutes
- Tags: nestjs, stripe, payment, webhook

Stripe integration in backend APIs is mostly about two things: create payment intent correctly and trust webhook events safely. If either side is wrong, payment state can drift from your booking state.

This guide shows a practical NestJS setup for PaymentIntent creation and Stripe webhook processing.

### Why It Matters

- Keeps booking/payment state consistent with Stripe lifecycle.
- Uses webhook signature verification to prevent spoofed events.
- Centralizes payment logic in service layer.
- Supports clean API contracts for frontend checkout flow.

### Core Concepts

#### 1. Enable Raw Body in Nest Bootstrap

Stripe webhook verification needs the raw request body.

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

#### 2. Controller Routes

Expose one endpoint for payment intent retrieval/creation and one public webhook endpoint.

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

#### 3. Stripe Service Initialization

Initialize Stripe using configuration values.

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

#### 4. Payment Intent Workflow

Find existing intent or create a new one, then return client-facing data.

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

#### 5. Webhook Verification and Event Handling

Verify Stripe signature and handle key payment events.

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

#### 6. Module Composition

Keep payment controller, service, and repository together in one module.

```typescript
import { Module } from "@nestjs/common";

@Module({
  controllers: [PaymentReceiveController],
  providers: [PaymentReceiveService, PaymentReceiveRepository],
})
export class PaymentReceiveModule {}
```

### Practical Example

Recommended event-to-booking status mapping:

- `payment_intent.created` -> `PaymentCreated`
- `payment_intent.succeeded` -> `PaymentCompleted`
- `payment_intent.payment_failed` -> `PaymentFailed`

This mapping keeps booking status synchronized with Stripe without asking customer support to decode payment mystery stories.

### Common Mistakes

- Forgetting `rawBody: true` and breaking signature verification.
- Trusting webhook payload without signature validation.
- Updating payment state from client callback only instead of webhook truth.
- Mixing booking update logic directly inside controller.
- Ignoring idempotency in webhook processing.

### Quick Recap

- Configure Nest app for raw webhook body.
- Create payment intents with metadata for business correlation.
- Verify webhook signature before processing events.
- Map event types to internal booking/payment statuses.
- Keep payment logic in service layer for maintainability.

### Next Steps

1. Add idempotency-key handling for payment intent creation.
2. Add webhook replay protection with event ID persistence.
3. Add transactional update flow for booking and payment records.
4. Add integration tests with Stripe CLI webhook forwarding.