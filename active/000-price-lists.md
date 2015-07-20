- Start Date: 2015-04-24
- RFC PR:
- Spree Issue:

# Abstract

I'd like to move the current "price belongs to currency" Spree logic to a more
generic "price belongs to a price list" to enable more flexibility.
A price list is a way to differentiate multiple prices of the same item.

# Motivation

A Store may need different prices for the same product/variant, for
different reasons (see below for a non-exhaustive list of examples).

Spree currently supports different prices if they have different currencies,
but in my experience this is not always the case in the real world.
I'm proposing this because I needed this feature on some projects and the
current Spree prices/currency logic was a clear limit.

Here are some use cases (actual sites, 2 on production and 2 on development):

* A fashion store needs a different price for each zone. Actually, some brands
have a specific price list for some countries and they want to follow this.
They like to use the same base currency (EUR) on all lists to keep things simple.
The payment gateway will do all the currency conversions.

* A B2B store has 3 different price lists. Each agent is assigned to a price
list. Currency is the same for all lists because the target is the same country.

* A B2B + B2C store has different prices for public clients and internal
agents. Currency is the same.

* A furniture store wants to use two different prices, one for their home
country and one for the rest of the europe. Currency is always EUR.

# Detailed Design

I've done a Spree extension for this feature, the code is at
https://github.com/freego/spree_price_lists.
It's probably not production ready, it was extracted from a project and it's
missing tests.

Looks like this is similar to https://github.com/spree-contrib/spree_price_books,
but I think my version is more simple and "generic".
I saw some logics on spree_price_books that are probably extracted from a
project with very specific needs.

On Spree, a variant has many prices. This is a good start point, but it's also
very limited in my opinion because of the `currency` stuff.

Inside Spree, `currency` is just a string and is passed as parameter on many
methods, expecially during the checkout process.
I want to change this to pass a price list reference instead.

A price list is a simple model which can just have a `name` and a `currency` in
the most basic form.

This will not change any behavior for simple stores, because you'll just have
a default price list with your preferred currency.

This should enable flexibility to easily add any kind of custom logic for price
list selection.

For example, on the mentioned fashion store I get the user location from the IP
address and I load the right price list on the fly.
On the B2B store, I get the price list from `current_user` object.
This should be just a matter of overriding a method, which by default gets the
default price list.

# Drawbacks

We have to add some logic to determine which list should be used on each
request.
This can degrade performance, but on a basic Spree store this can be just a
single, cachable query like `PriceList.first` or similar.

# Alternatives

I can just keep improving the extension.
I'm totally for keeping the Spree core as simple as possibile and using
extensions, but in this case I think needed changes are too "low level".
I need many decorators to overwrite many important methods, and it's only to
switch from currency to price list.
Those methods also change often: I basically rewrited everything between Spree
2.3 and 2.4, and it needs some more work for 3.0.

Another alternative I tried was to keep Spree as is and to add "fake"
currencies, like "EU1", "EU2", all with the same € symbol.
The main problem here is that fake currencies doesn't work in the real world.
For example, you can't pass them to a payment gateway.
