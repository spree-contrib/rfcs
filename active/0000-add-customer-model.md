- Start Date: 2015-06-29
- RFC PR: (leave this empty)
- Spree Issue: (leave this empty)

# Abstract

Currently the Spree::User model represents a customer, in addition to it's primary responsibility of representing a login account. This unnecessarily complicates the Spree::User model, makes the domain less extensible and prevents some real-world use cases from being handled by Spree. The addition of a Spree::Customer model should be able to simplify the system.

# Motivation

The current Spree system ties together orders, payments and addresses via the user model. Even though it is possible to create an order without a user model, associating information between these "anonymous customers" is not. One real-world situation we have encountered is with people being unwilling or unable to provide an email address for phone orders. The email address is required in the admin order customer details screen; but it can be made optional by monkey-patching Spree::Order#require_email to return false. This works and allows us to process the orders, but we have no way to associate information such as customer notes to this person.

Moving the associations to a Spree::Customer model will allow us to make sure every single order has a valid customer record. This will also allow the Spree::User to become simpler since it will have less responsibility. Some database constraints can be be tightened since there will be less need for nullable foreign key between spree_orders and spree_customers. This will also allow us to (internally or in a plugin) add notes to each customer.

Another benefit is that this will open up the doors to a better customer search process. The current admin user search is quite limited and we would like to contribute a better search process. The customer search when editing an order could use the same system -- we've had to disable/hide search in our own installation because it was not optimized to work with our size database.

# Detailed Design

## Schema changes

* Create a spree_customers table with an id, ship_address_id, bill_address_id and not-nullable timestamp columns.
* Copy the id, ship_address_id, bill_address_id and timestamp columns from spree_users to the spree_customers table.
* Rename the user_id column to customer_id in the spree_credit_cards, spree_promotion_rules, spree_promotion_rules_users and spree_state_changes tables.
* Add a not-nullable customer_id column to spree_users and set the value the same as the id.
* Add a spree_customers record for every spree_orders record with a null customer_id. Copy the bill_address_id, bill_address_id and timestamps to the customer.
* Change spree_orders.customer_id to be not-nullable.
* Remove the ship_address_id and bill_address_id from spree_users.
* Rename spree_promotion_rules_users to spree_promotion_rules_customers
* Add foreign keys pointing to spree_customers if rails version is gte 4.2

## Model Changes

* Add Spree::Customer model
  * Add associations for billing and shipping addresses
  * Add an association for credit cards

* Create Spree::Core::UserBase module
  * Move Spree::Core::UserAddress and Spree::Core::UserPaymentSource functionality into the module
  * Change associations to proxy through Spree::Customer
  * Add an orders association that proxies through Spree::Customer
  * Add before_destroy :check_completed_orders test (from Spree::LegacyUser)

* Change Spree::LegacyUser (in spree)
  * Remove check_completed_orders and before_destroy hook
  * Remove orders association
  * Replace Spree::Core::User* modules with Spree::Core::UserBase

* Change Spree::User (in spree_auth_devise)
  * Remove orders association
  * Replace Spree::Core::User* modules with Spree::Core::UserBase

* Remove code in core/config/initializers/user_class_extensions.rb that is redundant with Spree::Core::UserBase

* Change order creation logic to use the Spree::Customer for the user, and if one does not exist, create it and assign it to the user and order.

## Controller Changes

* Replace usage of Spree::User with Spree::Customer where applicable
* Change the admin user CRUD system to work primarily with customers

The majority of changes will probably need to be performed within controllers, helpers and especially specs. Lots of specs use a user record to represent a customer in a specific context. If the proxying logic is done correctly, much of the code will continue to work, but it will still require manual verification that all instances are properly replaced. No doubt other things will be found that need touching, as is usually the case when refactoring a "god object".

# Drawbacks

Overall the code complexity will be reduced, but there is a risk of missing something and introducing bugs. Lots and lots of code in the specs will need to be updated.

# Alternatives

I've considered making email address optional within Spree::User. This weakens the semantics of Spree::User and will increase the number of code paths.

Without this change we won't have a way to search for customers (the built-in search is too slow), and we won't have a central record we can associate customer notes to. Either way, we will probably need to implement this feature for our customers, and while we would prefer to contribute it upstream if that's not an option we will need to fork since the change is too invasive to try to keep it in sync with the upstream repo.

# Unresolved Questions

The exact impact of the controller and non-model changes are not clear yet. Also the degree of the spec changes aren't clear. These are probably things that cannot be known until implementation is attempted.
