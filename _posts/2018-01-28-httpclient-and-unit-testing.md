---
layout: post
title: HttpClient and Unit Testing
date: 2018-01-28 14:32
categories: [c#, Programming, testing]
---
If you've written C# which uses `HttpClient` and tried to unit test it, you probably noticed that it's not the easiest thing in the world to mock out. If [this github issue](https://github.com/dotnet/corefx/issues/1624){:target="_blank"} is any indication, many developers don't find it particularly easy either. It's certainly not impossible, but it requires learning about some of the internals of `HttpClient`, such as `HttpMessageHandler` as the `HttpClient` is designed as just a wrapper around these things.

## Using Moq
Over time I've personally solved this problem again and again for different projects by implementing a mock `HttpMessageHandler`, either explicitly or using a framework like [Moq](https://github.com/Moq/moq4){:target="_blank"} to setup the method calls and return values. The problem is that with these solutions, I found that it was either too loose in terms of ability to validate the code, or too complicated to get the level of strictness I wanted.

Here's an example from the github issue above for how to mock a request using Moq:

```cs
var requestUri = new Uri("http://google.com");
var expectedResponse = "Response text";

var mockResponse = new HttpResponseMessage(HttpStatusCode.OK) { Content = new StringContent(expectedResponse) };
var mockHandler = new Mock<HttpClientHandler>();
mockHandler
    .Protected()
    .Setup<Task<HttpResponseMessage>>(
        "SendAsync",
        It.Is<HttpRequestMessage>(message => message.RequestUri == requestUri),
        It.IsAny<CancellationToken>())
    .Returns(Task.FromResult(mockResponse));

var httpClient = new HttpClient(mockHandler.Object);
var result = await httpClient.GetStringAsync(requestUri).ConfigureAwait(false);
Assert.AreEqual(expectedResponse, result);
```

That sets up a mock which matches a call to a specific url and returns a mock response. It works well enough, but surely there's a better way.

## Patterns in other languages/frameworks

While working on some tests for an [Angular](https://angular.io/){:target="_blank"} project, I realized how much nicer their http testing library is and how easy it was to expect specific requests, do additional validations on the details of the request, and respond. Here's an example:

```ts
let logInSuccessful = false;
let error: Error;

let httpMock = TestBed.get(HttpTestingController) as HttpTestingController;
let authenticationService = TestBed.get(AuthenticationService) as AuthenticationService;
authenticationService.logInWithPassword("someUsername", "somePassword")
  .then(() => logInSuccessful = true)
  .catch(e => error = e);

let request = httpMock.expectOne({ method: "post", url: "/api/auth/token" });
expect(request.request.body).toEqual("grant_type=password&username=someUsername&password=somePassword&scope=openid%20offline_access");

let response = createMockResponse();
request.flush(response);
tick();

expect(logInSuccessful).toEqual(true);
expect(error).toBeUndefined();

httpMock.verify();
```

## The Solution

So with that, I decided to build a .NET library to implement a similar pattern as what I was using in the Angular test. It's on [Nuget.org as Testing.HttpClient](https://www.nuget.org/packages/Testing.HttpClient){:target="_blank"} ([Github](https://github.com/dfederm/Testing.HttpClient){:target="_blank"}).

Here's an example of a unit test using the library:

```cs
[TestMethod]
public async Task ExampleTest()
{
    using (var http = new HttpClientTestingFactory())
    {
        var worker = new Worker(http.HttpClient);

        // Make the call, but don't await the task
        var resultTask = worker.FetchDataAsync();

        // Expect the request and respond to it
        var request = http.Expect("http://some-website.com/some-path");
        request.Respond(HttpStatusCode.OK, "123");

        // Await the result and assert on it
        var result = await resultTask;
        Assert.AreEqual(123, result);
    }
}
```

## Design tradeoffs
Because I based this library heavily off Angular's http testing, the `Expect` call must be made after the request is made or else it won't match a request and fail the test. This means that your test needs to separate the calls that start the http call and the one that awaits it. This design decision makes it easy to inspect the actual full request and simplifies the mocking logic.

However, this admittedly may be a little awkward for C# developers who haven't seen this pattern which is heavily used in Angular testing. I describe this pattern as "synchronously testing asynchronous code". For those familiar with Angular, this is the difference between using `async` and `fakeAsync`.

In a future post I plan on diving into the pros and cons of both testing approaches in more depth.
