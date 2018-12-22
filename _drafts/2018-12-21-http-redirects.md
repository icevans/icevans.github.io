# Redirect Status Codes, Sinatra, and Rack-Test

I encountered an oddity in a Sinatra application today, and it turned out to be rather interesting to sort out: I got to read through some Rack and Sinatra source code, and learned a few things about the HTTP spec.

So suppose you've got a little Sinatra application with a route like the following:

```ruby
post '/new_post' do
  # ...
	redirect '/'
end
```

A quick check with `curl` reveals that if you send a `POST` request to this location, you'll get a response with a `303 See Other` status. So you decide to write a quick test for this redirect using minitest and rack-test:

```ruby
	# ... test boilerplate ommitted
	
	def test_new_post
		post '/new_post'
		assert_equal(303, last_response.status)
	end
```

Nice. Now you run your test and... It fails. Specifically, minitest will tell you that it expected `303` but actually got `302`. This is odd. When your app is running normally (say, by running `rackup`), this route returns a 303 response, but when rack-test runs it, the same route returns a 302.

To sort this out, let's remember that your Sinatra app is a Rack application. This means that after it receives a request from Rack, it must return an array of the following form:

```ruby
[status, headers, body]
```

This array represents the HTTP response that your application would like the server to send back to the client. The first element is an integer representing the status code, the second is a hash of headers, and the third is an object representing the body of the response (the only constraint on this object is that it [respond to `each` and return only strings](https://www.rubydoc.info/github/rack/rack/master/file/SPEC#label-The+Body)).

Your application passes this array to Rack, and Rack handles the details of  commuicating your desires to the web server. Now one nice thing about this architecture is that it makes it easy to write _Rack middleware_. Before talking to the server, Rack will pass the response array to any bit of middleware you tell it to use. That middleware can modify the response as it sees fit, and then sends it back to Rack to finally hand off to the server.

So Rack (and whatever stack of middleware you've told it to use) is the final determiner of what your response looks like. So let's return to our problem: in testing, when we run our app with rack-test, we get a different status code than when we run it normally. Now that we know how Rack works, a natural first thought is that Rack (or some middleware injected by rack-test) is modifying our response code.

What's the easiest way to check this hypothesis? Read [the source code!](https://github.com/rack-test/rack-test) (It's not that hard, and besides, it builds character.) If we look at rack-test's [mock_session.rb](https://github.com/rack-test/rack-test/blob/master/lib/rack/mock_session.rb), we can see how it handles requests starting on line 26:

```
def request(uri, env)
	env['HTTP_COOKIE'] ||= cookie_jar.for(uri)
	@last_request = Rack::Request.new(env)
	status, headers, body = @app.call(@last_request.env)

	@last_response = MockResponse.new(status, headers, body, env['rack.errors'].flush)
	body.close if body.respond_to?(:close)

	cookie_jar.merge(last_response.headers['Set-Cookie'], uri)

	@after_request.each(&:call)

	if @last_response.respond_to?(:finish)
	  @last_response.finish
	else
	  @last_response
	end
end
```

We see on the third and fourth lines that rack-test calls our app, sets a `status` variable to whatever status code our app returned, and then uses that to instantiate a new `MockResponse` object and assign that to the `@last_response` instance variable. And this instance variable is exactly what we're interacting with in our test when we write `assert_equal(303, last_response.status)`. So so far, rack-test has not interfered with our response at all. 

But perhaps `MockResponse` mucks with it. `MockResponse` is a Rack module. Let's check the source for its `instantiate` method (line 166):

```ruby
def initialize(status, headers, body, errors = StringIO.new(""))
  @original_headers = headers
  @errors           = errors.string if errors.respond_to?(:string)
  @cookies = parse_cookies_from_header

  super(body, status, headers)
end
```

Nothing here interferes with our status, it just gets passed up to `MockResponse`'s `Rack::Response`. We already know nothing here changes our response because this is the same class that gets instantiated when we run our app outside of testing. But if you don't believe me, scan through it yourself -- you'll see that nothing in the initialize method alters the status code.

So far then, it doesn't look like anything in the chain is altering our status. To be sure, we've only just checked the most obvious places. But before diving deeper into rack-test and/or Rack, it would probably be a good idea to check the [source for the `Sinatra::Redirect` method](https://github.com/sinatra/sinatra/blob/master/lib/sinatra/base.rb) and make sure it's doing what we think!

```ruby
def redirect(uri, *args)
  if env['HTTP_VERSION'] == 'HTTP/1.1' and env["REQUEST_METHOD"] != 'GET'
    status 303
  else
    status 302
  end

  # According to RFC 2616 section 14.30, "the field value consists of a
  # single absolute URI"
  response['Location'] = uri(uri.to_s, settings.absolute_redirects?, settings.prefixed_redirects?)
  halt(*args)
end
```

Aha! Apparently, Sinatra sets the status to 303 only if the Rack environment says the HTTP version is 1.1 and the request method was not `GET`. In our case, the request method is `POST`, but what about the HTTP version? Indeed, if you throw a `binding.pry` in your test and then inspect `env['HTTP_VERSION]`, you'll see that it is `nil` in the testing environment. But if you inspect this value in your normally running applicaiton, you'll see that it is `HTTP/1.1`. Because the HTTP version is different between testing and production, Sinatra is generating different status codes.

That's the crux of our mystery solved, but a few questions remain: (a) why does Sinatra do this?, (b) why is `env[HTTP_VERSION]` `nil` in testing, and (c) what should we do about this to fix our test?

## Why Does Sinatra Do This?

To understand the behavior of Sinatra's redirect method, we need to look into the history of the HTTP spec. In the old days of HTTP 1.0, there was no 303 status: just 302. [According to that spec](https://tools.ietf.org/html/rfc1945#section-9.3), the correct behavior for the client receiving a 302 redirect was to issue another request, _using the same method_ as the initial request. Of course, it eventually became a common pattern to redirect to a page after a POST request, and expect the client to send a GET request for that page. Since there was no better status code to express this intent, application developers used 302 and browsers played along (even though this was technically incorrect, as the spec points out).

To rectify this situation, HTTP/1.1 introduced a new status code: 303. This status code is intended to tell the client that the server carried out the initial request and that the client should now issue a GET request for the new resource (that's a bit oversimplified -- [read the spec](https://tools.ietf.org/html/rfc7231#section-6.4.3) for details).

Sinatra is doing the right thing. Modern clients using HTTP/1.1 get the semantically correct 303 response code to redirect them after a POST. But older HTTP 1.0 clients don't know what 303 means, so Sinatra sends them the 302 code that they're expecting.

## Why is the HTTP Version `nil` in Testing?

Since we're just testing, rack-test avoids the overhead of firing up a real server and sending full HTTP requests to your application. All your app needs is a Rack compliant request object: rack-test constructs one of these for your test, and then gives it to Rack to give to your application. Your app constructs a response and rack-test generats a `@last_response` instance variable for you to write your minitest assertions against. Since there is no actual server or HTTP in this process, rack-test does not set the `env['HTTP_VERSION']`.

## How Should we Fix Our Test

Given this architecture, it makes sense that rack-test doesn't set an HTTP version. We could change our redirect tests to expect 302. But it seems weird to divorce our tests from reality in this way. There are better options. We could write some middleware to add the `HTTP_VERSION` to the Rack environment before our app gets called. The easiest way to inject that into our test when we define our `app` metod:

```ruby
require_relative 'your_middleware.rb'

class AppTest < MiniTest::Test
	include Rack::Test::Methods
	
	def app
		app = Rack::Builder.new do
			use YourMiddleware
			run Sinatra::Application
		end
		app
	end
	
	# ...
end
```

There are many good articles that introduce you to writing middleware. Basically, all your `YourMiddlware` class would need to do is `env['HTTP_VERSION'] = 'HTTP/1.1`, and then call your Sinatra app.

 An even better solution would be to use rack-test's handy `env` method. This method allows us to modify the environment rack-test will use right from our tests:

```ruby
	def test_new_post
		env('HTTP_VERSION', 'HTTP/1.1')
		post '/new_post'
		assert_equal(303, last_response.status)
	end
	```
	
The test now passes! Sweet. If we wanted to, we could even write our test to check for the correct behavior on both versions of HTTP:

```ruby
	def test_new_post
		env('HTTP_VERSION', 'HTTP/1.1')
		post '/new_post'
		assert_equal(303, last_response.status)
		
		env('HTTP_VERSION', 'HTTP/1.0')
		post '/new_post'
		assert_equal(302, last_response.status)
	end
```

🎉

Of course, at that point, we're sorting of testing Sinatra more than our application. Determining the HTTP status code is Sinatra's job. Our application's job is to determine the _location_ of the redirect. So another option would be to test that the response has the correct `Location` header -- this should be the same regardless of the status code:

```
def test_new_post
	post '/new_post'
	assert_equal('http://example.org', last_response.headers['Location'])
end
```

That's a clean solution and sidesteps the whole issue.

## Wrapping Up

We saw that our app generates different status codes in testing and development/production. We spent some time rooting around in rack-test and Rack, only to discover that the issue was with Sinatra's `redirect` method all along. And in the end, the best solution may be to not test for status codes at all. But it was a good opportunity freshen up our understanding of Rack, and, more importantly, it was a very good exercise in reading the source code from some big, serious gems to see how they work. It can be intimidating to do this, but it's an invaluable skill to develop, and isn't as hard as you might think -- it's just Ruby after all!