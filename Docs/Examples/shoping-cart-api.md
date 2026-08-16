# Example: Shopping Cart API

Describes a shopping cart and order service (add item / checkout / payment / shipping) using DocScript.
This example exercises additional grammar rules from [`Spec`](../SPEC.md) not covered in the [Auth API example](./auth-api.md): loops over collections, chained composition across multiple objects, multi-level nesting with the indent form, and repeated error scenarios on the same method.

```
c(Cart)(user:User)-creates a new empty cart for a user
c(Cart)p(user)->User-detials of user

c(Cart)f(add_item)(product:Product, quantity:int)->bool-adds a product to the cart
c(Cart)f(add_item)!-raises OutOfStock if the requested quantity is unavailable

c(Cart)f(remove_item)(product_id:str)->bool-removes a product from the cart
c(Cart)f(remove_item)!-returns false without effect if the product is not in the cart

c(Cart)v(items)->list(CartItem)-all items currently in the cart
c(Cart)v(total)->float-the current total price of the cart

for item in v(items) -> if use c(CartItem)v(is_available) -> f(recalculate_total)

c(Order)(cart:Cart, address:Address)-creates a new order from a checked-out cart
c(Order)f(checkout)()->Payment-finalizes the cart and starts the payment process
c(Order)f(checkout)!-raises EmptyCartError if the cart has no items
c(Order)f(checkout)!-raises InvalidAddressError if the shipping address is incomplete

c(Order)f(checkout)v(payment)->Payment-the payment object created once checkout succeeds

if use c(Order)f(checkout) ->
    if use c(Payment)f(charge) ->
        if use f(send_confirmation_email) ->
            return v(order_id)
        if use f(notify_warehouse) ->
            return v(shipment_id)

async c(Payment)f(charge)(amount:float, method:PaymentMethod)->bool-charges the customer asynchronously
c(Payment)f(charge)!-raises PaymentDeclined if the payment method is rejected
c(Payment)f(charge)!-raises PaymentTimeout if the payment gateway does not respond in time

c(Shipment)v(status)->str-current shipment status (e.g. "pending", "shipped", "delivered")
c(Shipment)f(track)()->TrackingInfo-fetches the latest tracking information

while use f(order_pending) -> c(Shipment)f(track)-poll tracking status until delivery
```

## Notes

- The block starting with `if use c(Order)f(checkout) ->` uses the **indent form** (Spec Section 7.2) because it has more than two nesting levels and two sibling branches at the innermost level.
- `c(Order)f(checkout)v(payment)` is another example of the composition/output relationship in a chain (Spec Section 3.1): after `checkout` succeeds, a `payment` value becomes available.
- `c(Payment)f(charge)` has two separate `!-` scenarios, each written as its own statement, per the "one `!-` per statement" constraint (Spec Section 8.1).
- `c(Order)f(checkout)()` shows an empty parameter list — the method takes no arguments but is still marked as the final target of the chain (Spec Section 10.1).
- `c(Cart)p(user)->User-detials of user` about of `p(...)` in [Spec](../SPEC.md), ['Referring to a Specific Parameter'](../SPEC.md#51-referring-to-a-specific-parameter) and ['Constraint: `p()` Cannot Take Its Own Parameters'](../SPEC.md#52-constraint-p-cannot-take-its-own-parameters) section.