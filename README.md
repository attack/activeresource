# Active Resource

Active Resource (ARes) connects business objects and Representational State Transfer (REST)
web services.  It implements object-relational mapping for REST web services to provide transparent
proxying capabilities between a client (ActiveResource) and a RESTful service (which is provided by Simply RESTful routing
in ActionController::Resources).

## Philosophy

Active Resource attempts to provide a coherent wrapper object-relational mapping for REST
web services. It follows the same philosophy as Active Record, in that one of its prime aims
is to reduce the amount of code needed to map to these resources.  This is made possible
by relying on a number of code- and protocol-based conventions that make it easy for Active Resource
to infer complex relations and structures.  These conventions are outlined in detail in the documentation
for ActiveResource::Base.

## Overview

Model classes are mapped to remote REST resources by Active Resource much the same way Active Record maps model classes to database
tables.  When a request is made to a remote resource, a REST JSON request is generated, transmitted, and the result
received and serialized into a usable Ruby object.

## Download and installation

The latest version of Active Resource can be installed with RubyGems:

  % [sudo] gem install activeresource

Or added to a Gemfile:

  gem 'activeresource'

Source code can be downloaded on GitHub

* https://github.com/rails/activeresource/tree/master

### Configuration and Usage

Putting Active Resource to use is very similar to Active Record.  It's as simple as creating a model class
that inherits from ActiveResource::Base and providing a <tt>site</tt> class variable to it:

```
class Person < ActiveResource::Base
  self.site = "http://api.people.com:3000"
end
```

Now the Person class is REST enabled and can invoke REST services very similarly to how Active Record invokes
life cycle methods that operate against a persistent store.

```
# Find a person with id = 1
tyler = Person.find(1)
Person.exists?(1)  # => true
```

As you can see, the methods are quite similar to Active Record's methods for dealing with database
records.  But rather than dealing directly with a database record, you're dealing with HTTP resources (which may or may not be database records).

Connection settings (`site`, `headers`, `user`, `password`, `bearer_token`, `proxy`) and the connections themselves are store in
thread-local variables to make them thread-safe, so you can also set these dynamically, even in a multi-threaded environment, for instance:

    ActiveResource::Base.site = api_site_for(request)

### Authentication

Active Resource supports the token based authentication provided by Rails through the <tt>ActionController::HttpAuthentication::Token</tt> class using custom headers.

```
class Person < ActiveResource::Base
  self.headers['Authorization'] = 'Token token="abcd"'
end
```

You can also set any specific HTTP header using the same way.  As mentioned above, headers are thread-safe, so you can set headers dynamically, even in a multi-threaded environment:

```
ActiveResource::Base.headers['Authorization'] = current_session_api_token
```

ActiveResource supports 2 options for HTTP authentication today.

1. Basic
```
class Person < ActiveResource::Base
  self.user = 'my@email.com'
  self.password = '123'
end
# username: my@email.com password: 123
```

2. Bearer Token
```
class Person < ActiveResource::Base
  self.auth_type = :bearer
  self.bearer_token = 'my-token123'
end
# Bearer my-token123
```

### Protocol

Active Resource is built on a standard JSON or XML format for requesting and submitting resources over HTTP.  It mirrors the RESTful routing
built into Action Controller but will also work with any other REST service that properly implements the protocol.
REST uses HTTP, but unlike "typical" web applications, it makes use of all the verbs available in the HTTP specification:

* GET requests are used for finding and retrieving resources.
* POST requests are used to create new resources.
* PUT requests are used to update existing resources.
* DELETE requests are used to delete resources.

For more information on how this protocol works with Active Resource, see the ActiveResource::Base documentation;
for more general information on REST web services, see the article here[http://en.wikipedia.org/wiki/Representational_State_Transfer].

### Find

Find requests use the GET method and expect the JSON form of whatever resource/resources is/are being requested.  So,
for a request for a single element, the JSON of that item is expected in response:

```
# Expects a response of
#
# {"id":1,"first":"Tyler","last":"Durden"}
#
# for GET http://api.people.com:3000/people/1.json
#
tyler = Person.find(1)
```

The JSON document that is received is used to build a new object of type Person, with each
JSON element becoming an attribute on the object.

```
tyler.is_a? Person  # => true
tyler.last  # => 'Durden'
```

Any complex element (one that contains other elements) becomes its own object:

```
# With this response:
# {"id":1,"first":"Tyler","address":{"street":"Paper St.","state":"CA"}}
#
# for GET http://api.people.com:3000/people/1.json
#
tyler = Person.find(1)
tyler.address  # => <Person::Address::xxxxx>
tyler.address.street  # => 'Paper St.'
```

Collections can also be requested in a similar fashion

```
# Expects a response of
#
# [
#   {"id":1,"first":"Tyler","last":"Durden"},
#   {"id":2,"first":"Tony","last":"Stark",}
# ]
#
# for GET http://api.people.com:3000/people.json
#
people = Person.all
people.first  # => <Person::xxx 'first' => 'Tyler' ...>
people.last  # => <Person::xxx 'first' => 'Tony' ...>
```

### Create

Creating a new resource submits the JSON form of the resource as the body of the request and expects
a 'Location' header in the response with the RESTful URL location of the newly created resource.  The
id of the newly created resource is parsed out of the Location response header and automatically set
as the id of the ARes object.

```
# {"first":"Tyler","last":"Durden"}
#
# is submitted as the body on
#
# if include_root_in_json is not set or set to false => {"first":"Tyler"}
# if include_root_in_json is set to true => {"person":{"first":"Tyler"}}
#
# POST http://api.people.com:3000/people.json
#
# when save is called on a new Person object.  An empty response is
# is expected with a 'Location' header value:
#
# Response (201): Location: http://api.people.com:3000/people/2
#
tyler = Person.new(:first => 'Tyler')
tyler.new?  # => true
tyler.save  # => true
tyler.new?  # => false
tyler.id    # => 2
```

### Update

'save' is also used to update an existing resource and follows the same protocol as creating a resource
with the exception that no response headers are needed -- just an empty response when the update on the
server side was successful.

```
# {"first":"Tyler"}
#
# is submitted as the body on
#
# if include_root_in_json is not set or set to false => {"first":"Tyler"}
# if include_root_in_json is set to true => {"person":{"first":"Tyler"}}
#
# PUT http://api.people.com:3000/people/1.json
#
# when save is called on an existing Person object.  An empty response is
# is expected with code (204)
#
tyler = Person.find(1)
tyler.first # => 'Tyler'
tyler.first = 'Tyson'
tyler.save  # => true
```

### Delete

Destruction of a resource can be invoked as a class and instance method of the resource.

```
# A request is made to
#
# DELETE http://api.people.com:3000/people/1.json
#
# for both of these forms.  An empty response with
# is expected with response code (200)
#
tyler = Person.find(1)
tyler.destroy  # => true
tyler.exists?  # => false
Person.delete(2)  # => true
Person.exists?(2) # => false
```

### Associations

Relationships between resources can be declared using the standard association syntax
that should be familiar to anyone who uses activerecord. For example, using the
class definition below:

```
class Post < ActiveResource::Base
  self.site = "http://blog.io"
  has_many :comments
end

post = Post.find(1)      # issues GET http://blog.io/posts/1.json
comments = post.comments # issues GET http://blog.io/comments.json?post_id=1
```

In this case, the `Comment` model would have to be implemented as ActiveResource, too.

If you control the server, you may wish to include nested resources thus avoiding a
second network request. Given the resource above, if the response includes comments
in the response, they will be automatically loaded into the activeresource object.
The server-side model can be adjusted as follows to include comments in the response.

```
class Post < ActiveRecord::Base
  has_many :comments

  def as_json(options)
    super.merge(:include=>[:comments])
  end
end
```

### Logging

Active Resource instruments the event `request.active_resource` when doing a request
to the remote service. You can subscribe to it by doing:

```
ActiveSupport::Notifications.subscribe('request.active_resource')  do |name, start, finish, id, payload|
```

The `payload` is a `Hash` with the following keys:

* `method` as a `Symbol`
* `request_uri` as a `String`
* `result` as an `Net::HTTPResponse`

## License

Active Resource is released under the MIT license:

* http://www.opensource.org/licenses/MIT

## Contributing to Active Resource

Active Resource is work of many contributors. You're encouraged to submit pull requests, propose
features and discuss issues.

See {CONTRIBUTING}[https://github.com/rails/activeresource/blob/master/CONTRIBUTING.md].

## Support

Full API documentation is available at

* http://rubydoc.info/gems/activeresource

Bug reports and feature requests can be filed with the rest for the Ruby on Rails project here:

* https://github.com/rails/activeresource/issues

You can find more usage information in the ActiveResource::Base documentation.
