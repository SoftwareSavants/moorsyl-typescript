# Moorsyl TypeScript SDK

Official TypeScript and JavaScript client for the [Moorsyl API](https://docs.moorsyl.com) — send SMS, verify phone numbers, and react to events in Mauritania.

- Documentation — [docs.moorsyl.com](https://docs.moorsyl.com)
- Source — [github.com/SoftwareSavants/moorsyl-typescript](https://github.com/SoftwareSavants/moorsyl-typescript)
- npm — [@moorsyl/sdk](https://www.npmjs.com/package/@moorsyl/sdk)

## Installation

```bash
npm install @moorsyl/sdk
```

## Initialization

```typescript
import { Configuration, SMSApi, VerifyApi } from "@moorsyl/sdk";

const config = new Configuration({
  apiKey: "sk_live_...",
});

const sms = new SMSApi(config);
const verify = new VerifyApi(config);
```

The SDK defaults to `https://api.moorsyl.com/api`. No additional configuration is required for production use.

## Send an SMS

Requires a secret key (`sk_…`). See [API Keys](https://docs.moorsyl.com/api-keys).

```typescript
import { Configuration, SMSApi } from "@moorsyl/sdk";

const config = new Configuration({ apiKey: "sk_live_..." });
const sms = new SMSApi(config);

const response = await sms.smsSend({
  smsSendRequest: {
    to: "+22236551999",
    from: "MyBrand",
    body: "Your order #1042 has been shipped.",
  },
});

console.log("Accepted:", response.accepted);
console.log("Idempotency key:", response.idempotencyKey);
```

Supply `idempotencyKey` to prevent duplicate sends on retries:

```typescript
await sms.smsSend({
  smsSendRequest: {
    to: "+22236551999",
    from: "MyBrand",
    body: "Your code is 847291",
    idempotencyKey: "otp-user42-20240101T120000",
  },
});
```

## Phone Verification

Use a publishable key (`pk_…`) so this code can run safely in a browser or mobile app. See [API Keys](https://docs.moorsyl.com/api-keys).

```typescript
import { Configuration, VerifyApi } from "@moorsyl/sdk";

const config = new Configuration({ apiKey: "pk_live_..." });
const verify = new VerifyApi(config);
```

### Send a verification code

```typescript
const { verificationId } = await verify.verifySend({
  verifySendRequest: { to: "+22236551999" },
});
```

### Check the code

```typescript
const { status } = await verify.verifyCheck({
  verifyCheckRequest: { verificationId, code: userEnteredCode },
});

if (status === "approved") {
  // phone is verified — proceed
}
```

### Full flow

```typescript
import { Configuration, VerifyApi } from "@moorsyl/sdk";

async function verifyPhoneNumber(phoneNumber: string, userCode: string) {
  const config = new Configuration({ apiKey: "pk_live_..." });
  const verify = new VerifyApi(config);

  // Step 1 — send the code
  const { verificationId } = await verify.verifySend({
    verifySendRequest: { to: phoneNumber },
  });

  // Step 2 — check the code the user entered
  const { status } = await verify.verifyCheck({
    verifyCheckRequest: { verificationId, code: userCode },
  });

  if (status === "approved") {
    console.log("Phone verified!");
  } else {
    console.log("Invalid or expired code.");
  }
}
```

## Error handling

API methods throw `ResponseError` on non-2xx responses:

```typescript
import { ResponseError } from "@moorsyl/sdk";

try {
  await sms.smsSend({ smsSendRequest: { to: "+22236551999", from: "MyBrand", body: "Hello!" } });
} catch (e) {
  if (e instanceof ResponseError) {
    console.log("Status:", e.response.status);
    console.log("Body:", await e.response.json());
  }
}
```

Common status codes are documented on the [SMS](https://docs.moorsyl.com/sms) and [Verify](https://docs.moorsyl.com/verify) pages.

## Webhook signature verification

Verify incoming webhook signatures to ensure requests are genuine. See [Webhooks](https://docs.moorsyl.com/webhooks).

```typescript
import { createHmac, timingSafeEqual } from "node:crypto";

function verifyWebhook(rawBody: string, header: string, secret: string): boolean {
  const parts = Object.fromEntries(
    header.split(",").map((p) => p.split("=") as [string, string])
  );
  const timestamp = parts["t"];
  const signature = parts["v1"];

  if (!timestamp || !signature) return false;

  const age = Math.floor(Date.now() / 1000) - Number(timestamp);
  if (age > 300) return false;

  const keyBytes = Buffer.from(secret.replace("whsec_", ""), "base64");
  const expected = createHmac("sha256", keyBytes)
    .update(`${timestamp}.${rawBody}`)
    .digest("hex");

  return timingSafeEqual(Buffer.from(expected), Buffer.from(signature));
}
```

## Requirements

- Node.js 18+ (or any environment with a global `fetch`)
- TypeScript 4.x or 5.x (optional)

## License

MIT
