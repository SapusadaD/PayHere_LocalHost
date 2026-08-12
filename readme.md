# PayHere on localhost

A working PayHere checkout on a local development machine, with no tunnel and
no public URL.

---

## The problem

PayHere confirms a payment by sending a POST request to your `notify_url`. That
address has to be reachable from the internet. `http://localhost:8080` is not,
so the confirmation never arrives and the order stays on PENDING forever.

## The solution

PayHere's JavaScript SDK tells the browser the moment a payment succeeds. The
browser passes that to your server, which completes the order.

```
Customer pays
  -> PayHere reports success to the browser
  -> browser calls POST /api/payments/confirm?orderId=N
  -> server checks the order belongs to this user and is still pending
  -> order completes
```

---

## Step 1: Register localhost

Sign in at <https://sandbox.payhere.lk>

1. Open **Integrations** from the side menu
2. Click **Add Domain/App**
3. Select **Domain** and enter `localhost`
4. Click **Request to Allow** and wait until the row shows **Active**

Copy two values from that page:

| Value | Where |
|---|---|
| Merchant ID | Top of the Integrations page |
| Merchant Secret | The `localhost` row |

Merchant secrets are per domain. Use the one on the `localhost` row.

## Step 2: Configure the application

In `src/main/resources/app.properties`:

```properties
app.url=http://localhost:8080/yourapp

payhere.merchant.id=<your merchant id>
payhere.merchant.secret=<secret from the localhost row>
payhere.currency=LKR
payhere.country=Sri Lanka
payhere.sandbox=true

# Business App credentials, from Settings > API Keys.
# Leave blank on localhost. See the note below.
payhere.app.id=
payhere.app.secret=

# Lets the browser confirm the order. Required on localhost.
payhere.trust.client=true
```

`payhere.trust.client=true` is what makes this work. Without it the server
refuses the confirmation and the order stays pending.

### What about payhere.app.id and payhere.app.secret?

Those are OAuth credentials for a **Business App**, created under
**Settings > API Keys**. They authenticate the Merchant APIs: Retrieval,
Charging, Subscription, Refund.

The Retrieval API in particular looks like the obvious answer here, because
your server calls PayHere rather than waiting to be called:

```
GET https://sandbox.payhere.lk/merchant/v1/payment/search?order_id=123
```

It does not work from a development machine. PayHere matches the *caller's*
address against the allowed-domains list on the API key, and your server calls
out over your internet connection's public IP, not from `localhost`. The
request comes back as:

```json
{"status":-1,"msg":"Access denied for the domain","data":null}
```

No value in that allowed-domains box changes this, so leave both keys blank.
They become useful once the application is deployed on a server with a fixed
IP address that PayHere can whitelist.

## Step 3: Build the payment on the server

Create the order first, then hand the browser a signed payment object. The hash
proves the request came from you, and PayHere rejects anything without it.

```java
String merchantId = Env.require("payhere.merchant.id");
String secret     = Env.require("payhere.merchant.secret");
String orderId    = String.valueOf(order.getId());
String amount     = String.format(Locale.US, "%.2f", totalLkr);

// MD5(merchant_id + order_id + amount + currency + MD5(secret).toUpperCase())
String secretHash = md5(secret).toUpperCase();
String hash = md5(merchantId + orderId + amount + "LKR" + secretHash).toUpperCase();
```

The amount inside the hash must match the amount you send, formatted to exactly
two decimal places.

## Step 4: Open the payment popup

Load the SDK:

```html
<script src="https://www.payhere.lk/lib/payhere.js"></script>
```

Start the payment and confirm on success:

```javascript
payhere.onCompleted = async function (orderId) {
    // PayHere cannot reach localhost, so tell the server from here
    await fetch(`/yourapp/api/payments/confirm?orderId=${orderId}`, {
        method: 'POST',
        credentials: 'include'
    });
    window.location.href = 'invoice.html?orderId=' + orderId;
};

payhere.onDismissed = function () {
    console.log('Payment cancelled');
};

payhere.onError = function (error) {
    console.error('PayHere error:', error);
};

payhere.startPayment({
    sandbox:     true,
    merchant_id: ph.merchantId,
    return_url:  undefined,
    cancel_url:  undefined,
    notify_url:  ph.notifyUrl,
    order_id:    ph.orderId,
    items:       ph.itemsDescription,
    amount:      ph.amount,
    currency:    'LKR',
    hash:        ph.hash,
    first_name:  ph.firstName,
    last_name:   ph.lastName,
    email:       ph.email,
    phone:       ph.phone,
    address:     ph.address,
    city:        ph.city,
    country:     'Sri Lanka'
});
```

`return_url` and `cancel_url` are ignored by the JavaScript SDK. The
`onCompleted`, `onDismissed` and `onError` callbacks replace them.

## Step 5: Accept the confirmation on the server

Keep this endpoint narrow. It must require a session, check the order belongs
to that user, and only act on an order that is still pending.

```java
@POST
@Path("/confirm")
@IsUser
public Response confirmFromClient(@QueryParam("orderId") int orderId) {
    JsonObject res = new JsonObject();

    if (!Env.getBoolean("payhere.trust.client", false)) {
        res.addProperty("status", false);
        res.addProperty("message", "Client confirmation is disabled");
        return Response.ok(AppUtil.GSON.toJson(res)).build();
    }

    User user = (User) request.getSession(false).getAttribute("user");

    try (Session s = HibernateUtil.getSessionFactory().openSession()) {
        Order order = s.find(Order.class, orderId);

        if (order == null || order.getUser().getId() != user.getId()) {
            res.addProperty("status", false);
            res.addProperty("message", "Order not found");
            return Response.ok(AppUtil.GSON.toJson(res)).build();
        }
        if (order.getStatus().getValue().equals("COMPLETED")) {
            res.addProperty("status", true);
            res.addProperty("message", "Already confirmed");
            return Response.ok(AppUtil.GSON.toJson(res)).build();
        }
    }

    orderService.completeOrder(orderId);

    res.addProperty("status", true);
    res.addProperty("message", "Payment confirmed");
    return Response.ok(AppUtil.GSON.toJson(res)).build();
}
```

With those three checks in place, the worst a forged call can do is complete the
caller's own unpaid order. It cannot touch anyone else's.

## Step 6: Confirm from the invoice page too

The browser callback can be missed if the customer closes the tab too early.
Have the invoice page call the same endpoint when it loads, so the order still
completes.

```javascript
const res = await fetch(`/yourapp/api/payments/confirm?orderId=${orderId}`, {
    method: 'POST',
    credentials: 'include'
});
const data = await res.json();
if (data.status) {
    loadInvoice(orderId);
}
```

The endpoint is idempotent, so calling it twice is harmless.

## Step 7: Build and deploy

```bash
mvn clean package
```

Then redeploy. Editing `app.properties` under `src` changes nothing until Maven
copies it into the build, so always rebuild after a configuration change.

## Step 8: Test

Add a paid item to the cart and check out. Use a sandbox test card:

| Card | Number |
|---|---|
| Visa | 4916 2175 0161 1292 |
| MasterCard | 5307 7321 2553 1191 |
| AMEX | 3467 8100 5510 225 |

Expiry: any future date. CVV: any 3 digits, 4 for AMEX.

The order should complete and the invoice should render.

---

## Moving to production

Set `payhere.trust.client=false` and point `app.public.url` at your real HTTPS
address. Then implement the `notify_url` callback, which is the correct
mechanism once the server is publicly reachable:

```java
@POST
@Path("/notify")
@Consumes(MediaType.APPLICATION_FORM_URLENCODED)
public Response notifyPayment(MultivaluedMap<String, String> form) {
    // MD5(merchant_id + order_id + amount + currency + status_code + MD5(secret))
    if (!PayHereUtil.isSignatureValid(form)) return Response.ok().build();
    if (PayHereUtil.statusCode(form) != 2)   return Response.ok().build();
    if (!PayHereUtil.amountMatches(form, order.getTotalAmount())) return Response.ok().build();

    orderService.completeOrder(orderId);
    return Response.ok().build();
}
```

Always answer 200, even when rejecting, or PayHere will keep retrying. Verify
the signature, check the status code is 2, and re-check the amount against your
own record before releasing anything.

Client confirmation is a development convenience. Server-side callback
verification is the real mechanism.
