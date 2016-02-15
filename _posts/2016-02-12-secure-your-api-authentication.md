---
layout: post
title: Secure Your API Authentication
---

Judging from numerous different APIs I've used, many of them lacked good authentication principles along with not so great assumptions about security, and often about url formats and versioning. Mine included.

[Hypermedia, REST and HTTP](http://blog.steveklabnik.com/posts/2011-07-03-nobody-understands-rest-or-http) are still bit misunderstood and many developers out there apply their own conclusions about how the authentication and versioning should look like, e.g. `/api/v1/posts.json?api_key=KEY`. But that's a whole another topic. Let's focus on authentication for now.

In most cases when building API, you want to build out some sort of authentication to control who have access to API and who doesn't. Since authentication is usually driven by need to control amount of users and managing traffic, or managing paid consumers for the API, you want it to be as secure as possible. Therefore, you eventually don't want to end up with this &ndash; `/api/posts.json?api_key=KEY`, but you should rather embrace HTTP headers instead. Here's why.

## API key as a parameter

Passing API key along with request parameters mixes the key with parameters for the resource actions and altogether just brings confusion, since it derives its function from authentication. You may argue that it doesn't seem very odd, because many APIs use this approach, but that doesn't justify it. Secondly, passing an API key as a parameter usually involes server-side lookup for the key in the database and based on it's existence, server allows/denies access for the request. Much like this.

{% highlight ruby %}
# app/controllers/api/application_controller.rb

def authenticate
  unless ApiKey.exists?(value: params[:api_key])
    head 401
  end
end
{% endhighlight %}

*Rather than rendering unauthorized response, you should consider raising an exception. Excuse my brewity.*

At the first sight, this approach doesn't seem that bad. But once you consider [timing attacks](https://codahale.com/a-lesson-in-timing-attacks/), you can quickly realize, that you're approach is faulty from beginning. Comparison in database might be a subject to timining attack as well, unless you hash the key in database. But that's not a great solution either, since you're just covering the approach itself.

Instead of passing the key as a parameter, you should rather use HTTP headers. HTTP header `Authorization` comes in hand. So you refactor the `authenticate` method accordingly.

{% highlight ruby %}
# app/controllers/api/application_controller.rb

# Request Headers:
#   Authorization: Token token="API_KEY"

def authenticate
  key, _ = ActionController::HttpAuthentication::Token.token_and_options(request)

  unless ApiKey.exists?(value: key)
    head 401
  end
end
{% endhighlight %}

That looks a lot better. Requesting the API with `curl -H 'Authorization: Token token="API_KEY"' http://api.site.com/api/posts` works just like before. But it only looks better, since the authentication logic is still flawed. We're still exposing our database to timing attacks. So how can we fix it?

## Token authentication with other metadata

The trick is to pass an additional attribute to validate key without exposing its comparison. In order to do that, we pass in other metadata such as email. We lookup account by the email, fetch stored API key and afterwards we compare the keys, securely.

{% highlight ruby %}
# app/services/api/authentication_service.rb

# Request Headers:
#   Authorization: Token token="API_KEY", email="user@example.com"

module Api
  class AuthenticationService
    class MissingEmailError < ArgumentError; end

    def self.authenticate(request)
      token, options = ActionController::HttpAuthentication::Token.token_and_options(request)

      raise MissingEmailError unless options[:email].present?

      user = User.find_by(email: options[:email])

      if user && ActiveSupport::SecurityUtils.secure_compare(user.authentication_token, token)
        user
      end
    end
  end
end
{% endhighlight %}

The key step is the `secure_compare`, otherwise you would expose the comparison to another timing attack. So if all goes well &ndash; user is found and the key matches user's key, you receive `user` object from `Api::AuthenticationService` and continue with the request or in case of `nil` or exception, you halt and render unauthrorized response. 

But as you can see, this adds new authrorization level for a consumer of the API, since he has to make sure to pass in the email on every request and it seems like bit of a drag. But that's the price for secure staleless authentication.

This is actually the easier solution to securing this API. Other option is to introduce more secure approach to generating API keys, e.g. *JWT*.

## JWT &ndash; JSON Web Tokens

JWT is way more robust and standardized [RFC7519](https://tools.ietf.org/html/rfc7519) solution for authentication, since server is responsible for generating and validating tokens. Client only consumes generated token and passes it along in HTTP Authorization header, just like before, but without additional metadata. JWT is very exhaustive for the purpose of this blog, so if you want to have thorough knowledge of how it works, look into RFC or [check out its very consise introduction](https://jwt.io/introduction).

For purpose of minimal solution, let's look at rolling you own authentication with JWT with Ruby. First, let me explain the basics.

{% highlight ruby %}
"aaa.bbb.ccc"
# or
"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ"
{% endhighlight %}

JWT contains of three parts. The first part &ndash; `aaa` is a header, base64 encoded JSON consisting of token type and encryption/hashing algorithm.

{% highlight json %}
{
  "alg": "HS256",
  "typ": "JWT"
}
{% endhighlight %}

The `bbb` part is a payload. Payload contains information about issuer of token, expiration and so on. But payload is the place to store our identification of user, let's say user's id. 

{% highlight json %}
{
  "iss": "Identity Issuer",
  ...
  "user_id": 12
}
{% endhighlight %}

And the last part is signature. Signuture guaranties that data were not in any way altered on the way to the server, e.g. `user_id` is maliciously changed. To ensure that, server encrypts it with *secret* like this.

{% highlight javascript %}
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
{% endhighlight %}

Security is satisfied by signing the payload and header with server's secret and that way you cannot issue you own token. JWT implementation provides cryptographic hashing via HMAC or PKI with RSA. Take your pick.

Let's authenticate user.

## Authentication with JWT

Install [JWT gem for Ruby](https://github.com/jwt/ruby-jwt) and create `Api::SessionsController` for obtaining the token.

{% highlight ruby %}
# app/controllers/api/sessions_controller.rb

def create
  # Authenticate user by login and password, e.g. has_secure_password, or render authentication error

  payload = { iss: 'Our Secure API', user_id: user.id }
  token = JWT.encode(payload, 'our little secret', 'HS256') # Use HMACSH256

  render json: token
end
{% endhighlight %}

That's it. Our user has a token. Let's authenticate him.

{% highlight ruby %}
# app/controllers/api/application_controller.rb

# Request Headers:
#   Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJPdXIgU2VjdXJlIEFQSSIsInVzZXJfaWQiOjF9.n_I9tHYNAyEibhDLs0D_mSNV41owFt4GkACg5vg-DI0

def authenticate
  token = request.headers['Authorization'].match(/Bearer (.*)\z/)[1]
  payload, _ = JWT.decode(token, 'our little secret', 'HS256') # This raises an error if signature validation fails
  @current_user = User.find(payload[:user_id])
end
{% endhighlight %}

And that's it. No API key is stored, only credentials for user.

You should extract `encode` and `decode` into separate class or service object, e.g. `AuthenticationToken`, which wraps JWT and know how to extract request headers and lookup user. Good idea is to setup expiration by `exp` payload attribute as well. Other than that, your simple secure authentication is pretty much done.

There're also other robust solutions to authentication, for instance [OAuth2](http://oauth.net/2/). So before jumping into JWT, make sure to look into other alternatives as well.

Resources:

* <https://labs.kollegorna.se/blog/2015/04/build-an-api-now/>
* <https://jwt.io/introduction/>
* <https://github.com/jwt/ruby-jwt>
* <https://codahale.com/a-lesson-in-timing-attacks/>
