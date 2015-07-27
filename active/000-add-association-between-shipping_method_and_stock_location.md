- Start Date: 2015-07-27
- RFC PR: (leave this empty)
- Spree Issue: 6600

# Abstract

In ecommerce business, available shipping methods for an order are decided by
these three factors:

- Place delivered from
- Place delivered to
- Items delivering

Spree currently considers last two factors but the first factor.
This Request for Comments asks about considering where the item is delivered from
when choosing available shipping methods for an order.

# Motivation

Spree allows us to manage stock locations and they are where the items are
shipped. However stock locations are not associated with shipping methods
directly at this moment.

For users who have a simple supply chain strategy, it wouldn't matter.
For examples,

- it has only one stock location
- the stock location is determined by shipping address
- the stock location is determined by line items
- all stock locations support same shipping methods

For those scenarios, you can ignore the stock location on determining shipping
methods.

However the current implementation doesn't work well if you have a bit more
complicated supply chain strategy.
For examples,

- stocks of an item are distributed to multiple stock locations around the world
- each stock location supports different shipping methods
- stock location is determined with various factors such as cost, shipping
  duration, stock, leading time for backorder etc.

To solve these business requirement, Spree needs to know which shipping methods
are supported by each stock location.

# Detailed Design

## Data Model

Add many to many association between stock location and shipping method.

- A Spree::StockLocation model has many Spree::ShippingMethod models
- A Spree::ShippingMethod model has many Spree::StockLocation models likewise

Also make schema changes that represent the models will be required.

## Admin Pages

On "Configuration - Shipping Methods - Editing Shipping Methods", let admin
add/remove stock locations for the shipping method.

On "Configuration - Stock Locations - Editing Stock Location", let admin
add/remove shipping methods.

## Available Shipping Methods for Order

Filter out shipping methods that are not supported by the selected stock
location.


# Further Discussion

## Migration

We need to discuss how we migrate existing users to this structure. If we make
those changes without migrating strategy, no shipping methods can be selected
for their ecommerce site.

## Select All

It might be useful to have an ability to select all options. For example, if you
enable the option on stock location, it will support a shipping method that is
newly added.

It might be useful for managing other associations such as zones etc. too.


