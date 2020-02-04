---
layout: post
title: "Testing asynchronous code: async vs fake async"
date: 2018-02-04 10:12
categories: [angular, typescript]
tags: [angular, async, .NET, testing, typescript]
---
In the [last post]({% post_url 2018-01-28-httpclient-and-unit-testing %}) I explored implementing a mock which tested asynchronous code in a "fake" asynchronous way, and I promised to dive a little deeper into that concept and compare it with testing in an asynchronous way.

I say "fake" here because it's still using async/await, but the way of testing is more of a step by step approach where the unit test ends up effectively waiting on each task or promise to finish before moving on.

Angular has really clear examples of each pattern, so let's see the differences and compare the pros and cons for each.

## Async

This is the patterns I think C# developers are more familiar with, although I've seen it used in TypeScript as well. It generally involves the three traditional sections: . Here's an example of a test using this pattern:

```ts
it("should display some links when the user is not logged in", async(() => {
    // Arrange (some arranging was already done in beforeEach as well)
    let authenticationService = TestBed.get(AuthenticationService) as AuthenticationService;
    spyOn(authenticationService, "userInfo").and.returnValue(new BehaviorSubject(notLoggedInUser));

    // Act (detectChanges ends up triggering ngOnInit which actually does the work we're testing)
    fixture.detectChanges();
    fixture.whenStable().then(() => {
        fixture.detectChanges();

        // Assert
        let expectedLinks: { text: string, url: string }[] = [ /* Omitted for brevity */ ];
        let navItems = fixture.debugElement.queryAll(By.css(".nav-item"));
        expect(navItems).not.toBeNull();
        expect(navItems.length).toEqual(expectedLinks.length);

        for (let i = 0; i < navItems.length; i++) {
            let link = navItems[i].query(By.css(".nav-link"));
            expect(link).not.toBeNull();

            let expectations = expectedLinks[i];
            expect(link.nativeElement.innerText).toEqual(expectations.text);
            expect(link.attributes.href).toEqual(expectations.url);
        }

        expect(authenticationService.userInfo).toHaveBeenCalled();
    });
}));
```

I won't go into detail about the test as it was pulled from a real example, but the pattern should look familiar. You start in the Arrange section mocking calls and setting up the test, then you trigger the code that's actually being tested, then finally you assert your expectations on any results or side-effects.

A minor note for those unfamiliar with the use of `async` here, it's equivalent to using the `done` construct and calling `done` after the last expectation. Or for any C# developers, it's equivalent to returning a `Task` from your test. The test framework just waits for all async work to finish before the test ends.

## Fake Async

This pattern is used pretty widely in Angular apps, especially when mocking http calls. It's also how I implemented my [Testing.HttpClient](https://www.nuget.org/packages/Testing.HttpClient){:target="_blank"} C# library. It differs from the usual "Arrange, Act, and Assert" and instead interleaves and combines all three. Here's an example:

```ts
it("should return some data", fakeAsync(() => {
    // Act
    let response: IUser;
    let error: HttpErrorResponse;
    userService.getUser(userName)
        .then((r: IUser) => response = r)
        .catch((e: HttpErrorResponse) => error = e);

    // Arrange, with some implicit assertions
    let expectedResponse: IUser = { name: "someName" };
    let request = httpMock.expectOne({ method: "get", url: `/api/users/${userName}` });
    request.flush(expectedResponse);
    tick();

    // Assert
    expect(response).toEqual(expectedResponse, "should return the expected response");
    expect(error).toBeUndefined();
    expect(httpErrorHandlerService.logError).not.toHaveBeenCalled();
    expect(httpErrorHandlerService.getValidationErrors).not.toHaveBeenCalled();
}));
```

In this example, the test being called is made immediately (although there is some setup in a `beforeEach` not shown). It returns a promise, but the promise is unresolved at that point. Next there is an implicit assertion about what a request that the `userService.getUser` looked like and a mock response is provided. Then the promises are flushed, which is why these are _fake_ async tests. Finally, assertions are made.

## Comparing the two

Ironically, the Async pattern more closely follows how you should test synchronous code. Both tend to follow the "Arrange, Act, and Assert" structure. This pattern set up a bunch of mocks and rules beforehand to "wind up" the test, then lets the code under test run to completion, the expectations are checked. This pattern is good for testing how code behaves to inputs and the results it produces.

The fake async pattern is a lot more granular as you essentially interleaves the test code with the code being tested. It's like stepping through a debugger and providing mocks and asserting as you go, line by line. This pattern is better at testing code that creates side-effects, or code that needs to be done in a specific order. Anecdotally though, the granularity can come at a cost of brittleness. Refactoring can easily break tests even though the code remained functionally correct, so you can end up wasting time fixing tests.

One downside of the fake async pattern is that it tends to require extra code to get the async parts of the code flushed. In Angular tests, the `tick` function does this magic for you, as Angular is able to wrap all Promises and so `tick` can wait for their completion for you. In C#, this is fairly cumbersome, as there isn't a way to provide a replacement to the default `TaskFactory`, so you end up having to expose `TaskCompletionSource` objects from all your mocks to provide the flushing functionality.

Additionally, given the right tools, any fake sync test could be made into an async test. In Angular instead of using the `HttpTestingController`, you could provide your own mock `HttpClient` object and use spies to both match specific request patterns and provide responses.

## Verdict

When writing up these examples, I actually had attempted to use the same test written in each pattern, but one would always feel a little awkward. There is definitely something to be said about using the right tool for the job, so in Angular tests if you find yourself testing code that makes http calls or uses timers, feel free to use the fake async pattern. The magic is provided for you, so you might as well use it.

However, I also feel that usage of fake async is fairly niche. It doesn't work well in all scenarios, and it's not even a very common pattern in some languages like C#. The async pattern on the other hand works well in almost all cases, and so it's the pattern I'd suggest any well-rounded software engineer to master. It's a commonly used, widely applicable, well-known, and well-understood pattern.

tl;dr if I was stuck on a desert island with only one unit testing pattern, I'd pick the async pattern over the fake async one.
