---
title: Service Objects
---

## Service Objects

A **service object** encapsulates complex business logic into a class, providing a single public method as an entry point for executing the logic. Often named as a verb (e.g. `SendInvoices`, `CreateUser`), a service object is not so much an *object* as it is an *action*.

```ruby
action = BuyPony.new(color: 'white', height: 14.2, adorable: true)
action.call
```

As an application grows, simple CRUD actions can morph into a chain of callbacks, validations, and authorization checks. What was once a one-size-fits-all approach gives way to conditional execution peppered throughout the codebase. When that happens, it's time to think about using service objects for some or all of the domain logic.

Since a service object is just a class, all relevant code is either in the class or called directly from the class. Custom behaviors can be derived via subclassing, and particularly long operations can be broken down into multiple service objects.

Service objects are not tied to any particular layer of the application, so the same logic can be triggered from:

* Traditional web controllers
* API endpoints
* Background jobs
* Long-running clients (e.g. Action Cable)
* Rake tasks
* Unit tests

### Consider using service objects if you find that…

* Complex, conditional callbacks are scattered across multiple models and controllers, making it difficult to determine what happens, where it happens, and under what conditions.
* The same chunk of logic is called from multiple, disparate sources, e.g. the web, APIs, WebSockets, and background jobs.
* Business logic requires knowledge of many models and their relationships.
* Business logic deals primarily with non-models, e.g. interacting with third-party APIs.

### Service objects pair well with:

* [Form objects](#form-objects) – encapsulate larger parameter sets having complex requirements into a form object, which can then be passed directly to the service object
* [Policy objects](#policy-objects) - defer authorization to a class that specializes in it
* [Controller proliferation](#controller-proliferation) - reuse service objects across multiple controllers in different contexts

### Use case: creating an invoice

You may already have experience creating service objects without even realizing it. For example, [Active Job](http://edgeguides.rubyonrails.org/active_job_basics.html) utilizes this pattern: `ActiveJob::Base` is a service object, and `#perform` is its entry point. Rake tasks also commonly take the form of a service object.

Here's an example of an action that simutaneously creates an invoice model and fires off some notifications.

```ruby
class CreateInvoice
  attr_reader :error_message

  def initialize(amount, user)
    @amount, @user = amount, user
  end

  def call
    begin
      create_invoice
      send_invoice_email
      notify_admins_of_success
      true
    rescue
      @error_message = 'Could not create invoice'
      notify_admins_of_error
      false
    end
  end

protected

  def create_invoice
    @invoice = Invoice.create!(amount: @amount, user: @user)
  end

  def send_invoice_email
    # Send the @invoice email...
  end

  def notify_admins_of_success
    # Send admin email, post in Slack, etc...
  end

  def notify_admins_of_error
    # We probably want to know about this problem...
  end
end

class QuietlyCreateInvoice < CreateInvoice

  # The public interface is the same, so no changes to #call

protected

  def send_invoice_email; nil; end
  def notify_admins_of_success; nil; end
  def notify_admins_of_error; nil; end

end
```

And here's how it is used:

```ruby
action = CreateInvoice.new(13.37, user)
action.call

quiet_action = QuietlyCreateInvoice.new(42.42, user)
quiet_action.call
```

There are a few takeaways from this example:

* The "create invoice" action is broken down into logical steps, one per protected method.
* These steps are overridden by the subclass `QuietlyCreateInvoice` so that no emails are sent.
* The state of the service object can be maintained via instance variables like `@invoice`, keeping method definitions clean.
* The result of `#call` is a boolean indicating overall success or failure.
* Callers of the service object can get complex error information beyond the return value of `#call` by querying the `error_message` attribute. Alternatively, the operation could re-raise the original error, raise its own error, or return a rich result object from `#call`.

Some ways to flesh out this example include:

* Break out the email and notification operations into their own service objects.
* Add support for multiple invoice line items with an `add_line_item(description, amount)` method, essentially turning the service object into an invoice factory. Create the service object, accumulate line items depending on the specific use case, then finally trigger `#call`.
* Make the operation idempotent; for example, fail if two identical invoices are created within a very short time period, to prevent double-billing in the event of an error.
* Derive from a base `ApplicationOperation` class, which provides a standard range of application-wide functionality, such as authorization.
* Validate and massage parameters before attempting to create the model; for example, ensure that the `@amount` is non-negative, and round it to two decimal places. Even if there are already valiations on the `Invoice` model, they might not be restrictive enough. Or, the underlying ORM may not use validations at the model level. If validations get out of hand or need to be reused, defer them to a [form object](#form-objects) instead.

### Further reading

* [Service objects in Rails will help you design clean and maintainable code. Here's how.](https://www.netguru.co/blog/service-objects-in-rails-will-help) by Tomek Pewiński
* [Using Services to Keep Your Rails Controllers Clean and DRY](https://blog.engineyard.com/2014/keeping-your-rails-controllers-dry-with-services) by Ben Lewis
* [Gourmet Service Objects](http://brewhouse.io/blog/2014/04/30/gourmet-service-objects.html) by Philippe Creux
* [Anatomy of a Rails Service Object](http://multithreaded.stitchfix.com/blog/2015/06/02/anatomy-of-service-objects-in-rails/) by Dave Copeland

### Gems

* [Trailblazer](https://github.com/apotonick/trailblazer) – see documentation on [Trailblazer::Operation](http://trailblazer.to/gems/operation/)
