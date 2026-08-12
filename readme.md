# PayHere on localhost

PayHere normally confirms a payment by POSTing to your `notify_url`. On a
development machine there is no public address for it to reach, so orders sit
on PENDING forever and the invoice page eventually gives up.

This is how to make checkout work end to end without a tunnel.

---

## Why the obvious fixes do not work

Worth knowing before you spend an evening on them.

**A tunnel (ngrok, Cloudflare) works but is fragile.** The free address changes
every restart, so `app.public.url` needs updating each session, and a demo dies
if the tunnel drops.

**The Retrieval API looks ideal but is blocked.** Your server calls PayHere
instead of waiting to be called, which sounds perfect for localhost. It fails
with `Access denied for the domain`. PayHere matches the *caller's* address
against the allowed-domains list on your API key, and your server calls from
your internet connection's public IP, not from `localhost`. Nothing you type in
that box can satisfy it from a home connection.

**Importing certificates does not help either.** You may hit
`SSLHandshakeException: unable to find valid certification path` first. That is
a separate JVM truststore problem sitting in front of the domain problem. Fix it
and you still land on `Access denied`.

The approach below skips all of that.

---

## How it works

PayHere's JavaScript SDK tells the *browser* the moment a payment succeeds. The
browser passes that to your server, which completes the order.

```
Customer pays
  -> PayHere popup reports success to the browser
  -> browser calls POST /api/payments/confirm?orderId=N
  -> server checks the order belongs to you and is still pending
  -> order completes, files unlock
```

The invoice page also calls `confirm` on its own, so it still works if the
browser callback does not fire.

---

## Setup

### 1. Register localhost with PayHere

Sign in at https://sandbox.payhere.lk

Go to **Integrations**, click **Add Domain/App**, and enter `localhost` as the
domain. Wait for it to show **Active**.

Copy two values:

- **Merchant ID**, shown at the top of the Integrations page
- **Merchant Secret**, on the `localhost` row

The secret is per domain. The one on a different row will not work.

### 2. Fill in app.properties

In `src/main/resources/app.properties`:

```properties
payhere.merchant.id=1224610
payhere.merchant.secret=<the secret from the localhost row>
payhere.sandbox=true

# Lets the browser confirm the order, since the gateway cannot reach localhost
payhere.trust.client=true
```

`payhere.trust.client=true` is the line that makes this work. Without it the
server refuses the confirmation and the order stays pending.

You do **not** need `payhere.app.id`, `payhere.app.secret`, or
`payhere.tls.insecure`. Those are for the Retrieval API, which does not work
from localhost. Leave them blank.

### 3. Rebuild and redeploy

```
mvn clean package
```

Then redeploy in your IDE. Editing `app.properties` under `src` changes nothing
until Maven copies it into the build. This is the single most common reason it
still fails after a "fix".

### 4. Test

Add a paid file to the cart and check out. Use PayHere's sandbox card:

```
Card    4916 2175 0000 0000
Expiry  any future date
CVV     any 3 digits
```

You should land on a completed invoice with the file ready to download.

---

## If it still fails

The invoice page prints the reason under "Payment Not Confirmed". Read that
line, it names the cause.

| What it says | What to do |
|---|---|
| `payhere.trust.client is not true ... (value read: <blank>)` | The deployed build has the old properties file. Rebuild and redeploy. |
| `Order not found` | The session belongs to a different account from the one that placed the order. Sign in as the buyer. |
| `Unauthorized` in the PayHere popup | `localhost` is not registered, or you used the wrong merchant secret. |
| Popup never opens | The hash is wrong. Check `payhere.merchant.secret` matches the `localhost` row exactly. |

---

## Before deploying anywhere real

Set `payhere.trust.client=false` and set `app.public.url` to the real HTTPS
address. The notify callback is already implemented, verifies PayHere's
signature, and re-checks the amount against the stored order, so it takes over
automatically.

Client confirmation is a development convenience. It is deliberately narrow:
the endpoint requires a signed-in session, the order must belong to that user,
and it must still be pending, so the worst a forged call can do is complete the
caller's own unpaid order. That is acceptable on a local machine and not
acceptable in production.
