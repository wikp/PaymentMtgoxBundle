# PaymentMtgoxBundle

This bundle utilizes [MtGox][mtgox]'s IPN (Instant Payment Notification)
providing ability to create for example donation system, or simple shop.
The only thing you are forced to do is to create `Order` class
implementing `OrderInterface` and its doctrine repository (implementing
`OrderRepositoryInterface`) and redirect user to MtGox Payment Page
(implementation of obtaining address to redirect is also included in
this bundle).

Full documentation and big refactor soon, so use only for testing
purposes (I'm using this on production and everything works well, but my
refactor implies your refactor ;) ).

[mtgox]: https://mtgox.com/
