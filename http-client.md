# HTTP Client

- [Introduction](#introduction)
- [Making Requests](#making-requests)
    - [Request Data](#request-data)
    - [Headers](#headers)
    - [Authentication](#authentication)
    - [Retries](#retries)
    - [Error Handling](#error-handling)
- [Testing](#testing)
    - [Faking Responses](#faking-responses)
    - [Inspecting Requests](#inspecting-requests)

<a name="introduction"></a>
## Introduction

Laravel provides an expressive, minimal API around the [Guzzle HTTP client](http://docs.guzzlephp.org/en/stable/), allowing you to quickly make outgoing HTTP requests to communicate with other web applications. Laravel's wrapper around Guzzle is focused on its most common use cases and a wonderful developer experience.

Before getting started, you should ensure that you have installed the Guzzle package as a dependency of your application. By default, Laravel automatically includes this dependency:

    composer require guzzlehttp/guzzle

<a name="making-requests"></a>
## Making Requests

To make requests, you may use the `get`, `post`, `put`, `patch`, and `delete` methods. First, let's examine how to make a basic `GET` request:

    use Illuminate\Support\Facades\Http;

    $response = Http::get('http://test.com');

The `get` method returns an instance of `Illuminate\Http\Client\Response`, which provides a variety of methods that may be used to inspect the response:

    $response->body() : string;
    $response->json() : array;
    $response->status() : int;
    $response->ok() : bool;
    $response->successful() : bool;
    $response->serverError() : bool;
    $response->clientError() : bool;
    $response->header($header) : string;
    $response->headers() : array;

The `Illuminate\Http\Client\Response` object also implements the PHP `ArrayAccess` interface, allowing you to access JSON response data directly on the response:

    return Http::get('http://test.com/users/1')['name'];

<a name="request-data"></a>
### Request Data

Of course, it is common when using `POST`, `PUT`, and `PATCH` to send additional data with your request. So, these methods accept an array of data as their second argument. By default, data will be sent using the `application/json` content type:

    $response = Http::post('http://test.com/users', [
        'name' => 'Steve',
        'role' => 'Network Administrator',
    ]);

#### Sending Form URL Encoded Requests

If you would like to send data using the `application/x-www-form-urlencoded` content type, you should call the `asForm` method before making your request:

    $response = Http::asForm()->post('http://test.com/users', [
        'name' => 'Sara',
        'role' => 'Privacy Consultant',
    ]);

#### Multi-Part Requests

If you would like to send files as multi-part requests, you should call the `attach` method before making your request. This method accepts the name of the file and its contents. Optionally, you may provide a third argument which will be considered the file's filename:

    $response = Http::attach(
        'attachment', file_get_contents('photo.jpg'), 'photo.jpg'
    )->post('http://test.com/attachments');

Instead of passing the raw contents of a file, you may also pass a stream resource:

    $photo = fopen('photo.jpg', 'r');

    $response = Http::attach(
        'attachment', $photo, 'photo.jpg'
    )->post('http://test.com/attachments');

<a name="headers"></a>
### Headers

Headers may be added to requests using the `withHeaders` method. This `withHeaders` method accepts an array of key / value pairs:

    $response = Http::withHeaders([
        'X-First' => 'foo',
        'X-Second' => 'bar'
    ])->post('http://test.com/users', [
        'name' => 'Taylor',
    ]);

<a name="authentication"></a>
### Authentication

You may specify basic and digest authentication credentials using the `withBasicAuth` and `withDigestAuth` methods, respectively:

    // Basic authentication...
    $response = Http::withBasicAuth('taylor@laravel.com', 'secret')->post(...);

    // Digest authentication...
    $response = Http::withDigestAuth('taylor@laravel.com', 'secret')->post(...);

#### Bearer Tokens

If you would like to quickly add an `Authorization` bearer token header to the request, you may use the `withToken` method:

    $response = Http::withToken('token')->post(...);

<a name="retries"></a>
### Retries

If you would like HTTP client to automatically retry the request if a client or server error occurs, you may use the `retry` method. The `retry` method accepts two arguments: the number of times the request should be attempted and the number of milliseconds that Laravel should wait in between attempts:

    $response = Http::retry(3, 100)->post(...);

If all of the requests fail, an instance of `Illuminate\Http\Client\RequestException` will be thrown.

<a name="error-handling"></a>
### Error Handling

Unlike Guzzle's default behavior, Laravel's HTTP client wrapper does not throw exceptions on client or server errors (`400` and `500` level responses from servers). You may determine if one of these errors was returned using the `successful`, `clientError`, or `serverError` methods:

    // Determine if the status code was >= 200 and < 300...
    $response->successful();

    // Determine if the response has a 400 level status code...
    $response->clientError();

    // Determine if the response has a 500 level status code...
    $response->serverError();

#### Throwing Exceptions

If you have a response instance and would like to throw an instance of `Illuminate\Http\Client\RequestException` if the response is a client or server error, you may use the `throw` method:

    $response = Http::post(...);

    // Throw an exception if a client or server error occurred...
    $response->throw();

    return $response['user']['id'];

The `Illuminate\Http\Client\RequestException` instance has a public `$response` property which will allow you to inspect the returned response.

The `throw` method returns the response instance if no error occurred, allowing you to chain other operations onto the `throw` method:

    return Http::post(...)->throw()->json();

<a name="testing"></a>
## Testing

Many Laravel services provide functionality to help you easily and expressively write tests, and Laravel's HTTP wrapper is no exception. The `Http` facade's `fake` method allows you to instruct the HTTP client to return stubbed / dummy responses when requests are made.

<a name="faking-responses"></a>
### Faking Responses

For example, to instruct the HTTP client to return empty, `200` status code responses for every request, you may call the `fake` method with no arguments:

    use Illuminate\Support\Facades\Http;

    Http::fake();

    $response = Http::post(...);

#### Faking Specific URLs

Alternatively, you may pass an array to the `fake` method. The array's keys should represent URL patterns that you wish to fake and their associated responses. The `*` character may be used as a wildcard character. Any requests made to URLs that have not been faked will actually be executed. You may use the `response` method to construct stub / fake responses for these endpoints:

    Http::fake([
        // Stub a JSON response for GitHub endpoints...
        'github.com/*' => Http::response(['foo' => 'bar'], 200, ['Headers']),

        // Stub a string response for Google endpoints...
        'google.com/*' => Http::response('Hello World', 200, ['Headers']),
    ]);

If you would like to specify a fallback URL pattern that will stub all unmatched URLs, you may use a single `*` character:

    Http::fake([
        // Stub a JSON response for GitHub endpoints...
        'github.com/*' => Http::response(['foo' => 'bar'], 200, ['Headers']),

        // Stub a string response for all other endpoints...
        '*' => Http::response('Hello World', 200, ['Headers']),
    ]);

#### Faking Response Sequences

Sometimes you may need to specify that a single URL should return a series of fake responses in a specific order. You may accomplish this using the `Http::sequence` method to build the responses:

    Http::fake([
        // Stub a series of responses for GitHub endpoints...
        'github.com/*' => Http::sequence()
                                ->push('Hello World', 200)
                                ->push(['foo' => 'bar'], 200)
                                ->pushStatus(404),
    ]);

When all of the responses in a response sequence have been consumed, any further requests will cause the response sequence to throw an exception. If you would like to specify a default response that should be returned when a sequence is empty, you may use the `whenEmpty` method:

    Http::fake([
        // Stub a series of responses for GitHub endpoints...
        'github.com/*' => Http::sequence()
                                ->push('Hello World', 200)
                                ->push(['foo' => 'bar'], 200)
                                ->whenEmpty(Http::response()),
    ]);

If you would like to fake a sequence of responses but do not need to specify a specific URL pattern that should be faked, you may use the `Http::fakeSequence` method:

    Http::fakeSequence()
            ->push('Hello World', 200)
            ->whenEmpty(Http::response());

#### Fake Callback

If you require more complicated logic to determine what responses to return for certain endpoints, you may pass a callback to the `fake` method. This callback will receive an instance of `Illuminate\Http\Client\Request` and should return a response instance:

    Http::fake(function ($request) {
        return Http::response('Hello World', 200);
    });

<a name="inspecting-requests"></a>
### Inspecting Requests

When faking responses, you may occasionally wish to inspect the requests the client receives in order to make sure your application is sending the correct data or headers. You may accomplish this by calling the `Http::assertSent` method after calling `Http::fake`.

The `assertSent` method accepts a callback which will be given an `Illuminate\Http\Client\Request` instance and should return a boolean value indicating if the request matches your expectations. In order for the test to pass, at least one request must have been issued matching the given expectations:

    Http::fake();

    Http::withHeaders([
        'X-First' => 'foo',
    ])->post('http://test.com/users', [
        'name' => 'Taylor',
        'role' => 'Developer',
    ]);

    Http::assertSent(function ($request) {
        return $request->hasHeader('X-First', 'foo') &&
               $request->url() == 'http://test.com/users' &&
               $request['name'] == 'Taylor' &&
               $request['role'] == 'Developer';
    });
