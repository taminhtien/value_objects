# ValueObjects

Serializable and validatable value objects for ActiveRecord

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'value_objects'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install value_objects

## Usage

### Create the value object class

The value object class inherits from `ValueObjects::Base`, and attributes are defined with `attr_accessor`:

```ruby
class AddressValue < ValueObjects::Base
  attr_accessor :street, :postcode, :city
end

address = AddressValue.new(street: '123 Big Street', postcode: '12345', city: 'Metropolis')
address.street # => "123 Big Street"
address.street = '321 Main St' # => "321 Main St"
address.to_hash # => {:street=>"321 Main St", :postcode=>"12345", :city=>"Metropolis"}
```

### Add validations

Validations can be added using the DSL from `ActiveModel::Validations`:

```ruby
class AddressValue < ValueObjects::Base
  attr_accessor :street, :postcode, :city
  validates :postcode, presence: true
end

address = AddressValue.new(street: '123 Big Street', city: 'Metropolis')
address.valid? # => false
address.errors.to_h # => {:postcode=>"can't be blank"}
address.postcode = '12345' # => "12345"
address.valid? # => true
address.errors.to_h # => {}
```

### Serialization with ActiveRecord

For columns of `json` type, the value object class can be used as the coder for the `serialize` method:

```ruby
class Customer < ActiveRecord::Base
  serialize :home_address, AddressValue
end

customer = Customer.new
customer.home_address = AddressValue.new(street: '123 Big Street', postcode: '12345', city: 'Metropolis')
customer.save
customer.reload
customer.home_address # => #<AddressValue:0x00ba9876543210 @street="123 Big Street", @postcode="12345", @city="Metropolis">
```

For columns of `string` or `text` type, wrap the value object class in a `JsonCoder`:

```ruby
class Customer < ActiveRecord::Base
  serialize :home_address, ValueObjects::ActiveRecord::JsonCoder.new(AddressValue)
end
```

### Validation with ActiveRecord

By default, validating the record does not automatically validate the value object.
Use the `ValueObjects::ValidValidator` to make this automatic:

```ruby
class Customer < ActiveRecord::Base
  serialize :home_address, AddressValue
  validates :home_address, 'value_objects/valid': true
  validates :home_address, presence: true # other validations are allowed too
end

customer = Customer.new
customer.home_address = AddressValue.new(street: '123 Big Street', city: 'Metropolis')
customer.valid? # => false
customer.errors.to_h # => {:home_address=>"is invalid"}
customer.home_address.errors.to_h # => {:postcode=>"can't be blank"}
customer = Customer.new
customer.valid? # => false
customer.errors.to_h # => {:home_address=>"can't be blank"}
```

### With `ValueObjects::ActiveRecord`

For easy set up of both serialization and validation, `include ValueObjects::ActiveRecord` and invoke `value_object`:

```ruby
class Customer < ActiveRecord::Base
  include ValueObjects::ActiveRecord
  value_object :home_address, AddressValue
  validates :home_address, presence: true
end
```

This basically works the same way but also defines the `<attribute>_attributes=` method which can be used to assign the value object using a hash:

```ruby
customer.home_address_attributes = { street: '321 Main St', postcode: '54321', city: 'Micropolis' }
customer.home_address # => #<AddressValue:0x00ba9876503210 @street="321 Main St", @postcode="54321", @city="Micropolis">
```

This is functionally similar to what `accepts_nested_attributes_for` does for associations.

Also, `value_object` will use the `JsonCoder` automatically if it detects that the column type is `string` or `text`.

Additional options may be passed in to customize validation:

```ruby
class Customer < ActiveRecord::Base
  include ValueObjects::ActiveRecord
  value_object :home_address, AddressValue, allow_nil: true
end
```

Or, to skip validation entirely:

```ruby
class Customer < ActiveRecord::Base
  include ValueObjects::ActiveRecord
  value_object :home_address, AddressValue, no_validation: true
end
```

### Value object collections

Serialization and validation of value object collections are also supported.

First, create a nested `Collection` class that inherits from `ValueObjects::Base::Collection`:

```ruby
class AddressValue < ValueObjects::Base
  attr_accessor :street, :postcode, :city
  validates :postcode, presence: true

  class Collection < Collection
  end
end
```

Then use the nested `Collection` class as the serialization coder:

```ruby
class Customer < ActiveRecord::Base
  include ValueObjects::ActiveRecord
  value_object :addresses, AddressValue::Collection
  validates :addresses, presence: true
end

customer = Customer.new(addresses: [])
customer.valid? # => false
customer.errors.to_h # => {:addresses=>"can't be blank"}
customer.addresses << AddressValue.new(street: '123 Big Street', postcode: '12345', city: 'Metropolis')
customer.valid? # => true
customer.addresses << AddressValue.new(street: '321 Main St', city: 'Micropolis')
customer.valid? # => false
customer.errors.to_h # => {:addresses=>"is invalid"}
customer.addresses[1].errors.to_h # => {:postcode=>"can't be blank"}
```

The `<attribute>_attributes=` method also functions in much the same way:

```ruby
customer.addresses_attributes = { '0' => { city: 'Micropolis' }, '1' => { city: 'Metropolis' } }
customer.addresses # => [#<AddressValue:0x00ba9876543210 @city="Micropolis">, #<AddressValue:0x00ba9876503210 @city="Metropolis">]
```

Except, items with '-1' keys are considered as dummy items and ignored:

```ruby
customer.addresses_attributes = { '0' => { city: 'Micropolis' }, '-1' => { city: 'Metropolis' } }
customer.addresses # => [#<AddressValue:0x00ba9876543210 @city="Micropolis">]
```

This is useful when data is submitted via standard HTML forms encoded with the 'application/x-www-form-urlencoded' media type (which cannot represent empty collections). To work around this, a dummy item can be added to the collection with it's key set to '-1' and it will conveniently be ignored when assigned to the value object collection.

### Integrate with Cocoon

Put this into a Rails initializer (e.g. `config/initializers/value_objects.rb`):

```ruby
ValueObjects::ActionView.integrate_with :cocoon
```

This will add the `link_to_add_nested_value` & `link_to_remove_nested_value` view helpers.
Use them in place of Cocoon's `link_to_add_association` & `link_to_remove_association` when working with nested value objects:

```ruby
# use the attribute name (:addresses) in place of the association name
# and supply the value object class as the next argument
link_to_add_nested_value 'Add Address', f, :addresses, AddressValue

# the `f` form builder argument is not needed
link_to_remove_nested_value 'Remove Address'
```

## Maintainers

* Matthew Yeow (https://github.com/tbsmatt), Tinkerbox Studios (https://www.tinkerbox.com.sg/)

## Contributing

* Fork the repository.
* Make your feature addition or bug fix.
* Add tests for it. This is important so we don't break it in a future version unintentionally.
* Commit, but do not mess with rakefile or version. (if you want to have your own version, that is fine but bump version in a commit by itself)
* Submit a pull request. Bonus points for topic branches.

## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).

