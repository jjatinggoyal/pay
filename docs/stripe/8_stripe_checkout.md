# Stripe Checkout

[Stripe Checkout](https://stripe.com/docs/payments/checkout) allows you to simply redirect to Stripe for handling payments. The main benefit is that it's super fast to setup payments in your application, they're SCA compatible, and they will get improved automatically by Stripe.

> [!WARNING]
> You need to configure webhooks before using Stripe Checkout otherwise your application won't be updated with the correct data.
>
> See [Webhooks](/docs/stripe/5_webhooks.md) section on how to do that.

![stripe checkout example](https://i.imgur.com/nFsCBCK.gif)

### How to use Stripe Checkout with Pay

Choose the checkout button mode you need and pass any required arguments. Read the [Stripe Checkout Session API docs](https://stripe.com/docs/api/checkout/sessions/create) to see what options are available. For instance:

```ruby
class SubscriptionsController < ApplicationController
  def checkout
    # Make sure the user's payment processor is Stripe
    current_user.set_payment_processor :stripe

    # One-time payments (https://stripe.com/docs/payments/accept-a-payment)
    @checkout_session = current_user.payment_processor.checkout(mode: "payment", line_items: "price_1ILVZaKXBGcbgpbZQ26kgXWG")

    # Or Subscriptions (https://stripe.com/docs/billing/subscriptions/build-subscription)
    @checkout_session = current_user.payment_processor.checkout(
      mode: 'subscription',
      locale: I18n.locale,
      line_items: [{
        price: 'price_1ILVZaKXBGcbgpbZQ26kgXWG',
        quantity: 4
      }],
      subscription_data: {
        trial_period_days: 15,
        metadata: {
          pay_name: "base" # Optional. Overrides the Pay::Subscription name attribute
        },
      },
      success_url: root_url,
      cancel_url: root_url
    )

    # Or Setup a new card for future use (https://stripe.com/docs/payments/save-and-reuse)
    @checkout_session = current_user.payment_processor.checkout(mode: "setup")

    # If you want to redirect directly to checkout
    # redirect_to @checkout_session.url, allow_other_host: true, status: :see_other
  end
end
```

Then link to it in your view:

```erb
<%= link_to "Checkout", @checkout_session.url %>
```

> [!NOTE]
> Due to a [bug](https://github.com/hotwired/turbo/issues/211#issuecomment-966570923) in the browser's `fetch` implementation, you will need to disable Turbo if redirecting to Stripe checkout server-side.
>
> ```erb
> <%= link_to "Checkout", checkout_path, data: { turbo: false } %>
> ```

The `stripe_checkout_session_id` param will be included on success and cancel URLs automatically. This allows you to lookup the checkout session on your success page and confirm the payment was successful before fulfilling the customer's purchase.

https://stripe.com/docs/payments/checkout/custom-success-page

## Stripe Customer Billing Portal

Customers will want to update their payment method, subscription, etc. This can be done with the [Customer Billing Portal](https://stripe.com/docs/billing/subscriptions/integrating-customer-portal). It works the same as the other Stripe Checkout pages.

First, create a session in your controller:

```ruby
class SubscriptionsController < ApplicationController
  def index
    @portal_session = current_user.payment_processor.billing_portal

    # You can customize the billing_portal return_url (default is root_url):
    # @portal_session = current_user.payment_processor.billing_portal(return_url: your_url)
  end
end
```

Then link to it in your view:

```erb
<%= link_to "Billing Portal", @portal_session.url %>
```

Or redirect to it in your controller:

```ruby
redirect_to @portal_session.url, allow_other_host: true, status: :see_other
```

## Fulfilling orders after Checkout completed

For one-time payments, you'll need to add a webhook listener for the Checkout `stripe.checkout.session.completed` and `stripe.checkout.session.async_payment_succeeded` events. Some payment methods are delayed so you need to verify the `payment_status == "paid"`. The async payment succeeded event fires when delayed payments are complete.

For subscriptions, Pay will automatically create the `Pay::Subscription` record for you.

To create custom webhook listeners for specific events, you can create your custom webhook listener classes under a folder like `app/webhooks`, like this:
```ruby
# app/webhooks/fulfill_checkout.rb

class FulfillCheckout
  def call(event)
    object = event.data.object

    return if object.payment_status != "paid"

    # Handle fulfillment
  end
end
```

And then subscribe your custom webhook listener class to specific Stripe events on `config/initializers/pay.rb`:
```ruby
ActiveSupport.on_load(:pay) do
  Pay::Webhooks.delegator.subscribe "stripe.checkout.session.completed", FulfillCheckout.new
  Pay::Webhooks.delegator.subscribe "stripe.checkout.session.async_payment_succeeded", FulfillCheckout.new
end
```

That's it!
