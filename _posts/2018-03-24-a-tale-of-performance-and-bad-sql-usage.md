---
layout: post
title: A Tale of Performance and Bad SQL Usage
date: 2018-03-24 08:51
categories: [.NET]
tags: [.NET, application insights, performance, sql, telemetry]
comments: true
---
The other day I was looking at telemetry for one of my [websites](https://clickerheroestracker.azurewebsites.net/){:target="_blank"} to try and finally figure out why I was running into my telemetry cap, which I had been putting off for a while. I'm on the free [Application Insights](https://azure.microsoft.com/en-us/services/application-insights/){:target="_blank"} tier, so I only get 1 GB a month and was needing to sample to only 4% of traffic to remain under the cap, which didn't seem right considering the traffic that site gets.

What I found is that the `dependencies` was just being spammed with similar SQL calls:

[![Similar SQL calls](/assets/SqlCalls.png)](/assets/SqlCalls.png)

And based on the behavior of the website, I knew that all of those calls listed were part of a single web request and [SQL transaction](https://docs.microsoft.com/en-us/sql/t-sql/language-elements/transactions-transact-sql){:target="_blank"}. At the time, I originally implemented this, it seems that I was lazy and just put a bunch of separate `INSERT INTO` statements in one transaction. Note that I manually craft the SQL queries in my app (I know, the horror!) because [Entity Framework](https://docs.microsoft.com/en-us/aspnet/entity-framework){:target="_blank"} tends to have pretty bad performance despite being a pretty good programming interface, and [stored procedures](https://docs.microsoft.com/en-us/sql/relational-databases/stored-procedures/stored-procedures-database-engine){:target="_blank"} felt a bit [awkward to use when the inputs are tables](https://docs.microsoft.com/en-us/sql/relational-databases/tables/use-table-valued-parameters-database-engine){:target="_blank"}. Still, judgement may still be warranted this practice, but it's what I have right now.

In any case, I ended up [combining the `INSERT INTO` statements](https://github.com/dfederm/ClickerHeroesTracker/commit/da320be6c60ab8422a54842fc3eac996e77ec1cb){:target="_blank"} for similar tables, which cut the number of SQL calls from 40 to 7 for this sort of request. The results were a drastic decrease in the number of SQL calls and thus telemetry entries:

[![SQL calls after](/assets/SqlCallsAfter.png)](/assets/SqlCallsAfter.png)

This actually was enough of a cut for me to increase my sampling to about 25%. That's still lower than I'd like, but it's a great improvement. The majority of the remaining excess is the SQL calls made by [Entity Framework](https://docs.microsoft.com/en-us/aspnet/entity-framework){:target="_blank"} since I'm using it for [Identity](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity?tabs=visual-studio%2Caspnetcore2x){:target="_blank"} and [Authentication](https://github.com/openiddict/openiddict-core){:target="_blank"}.

One very interesting and initially unexpected side-effect of this change is that each of the individual SQL calls seems to take approximately the same amount of time as before, about 40 or so milliseconds per call (with some outliers where I need to use `HOLDLOCK`), so this actually had an additional benefit of **cutting end to end latency for the request to about one third of what it previously was**.

[![SQL latency](/assets/SqlLatency.png)](/assets/SqlLatency.png)

I'm considering taking this even further and composing one massive SQL query, including the transaction statements to just have one single SQL call per request. The moral of the story though is to just combine calls when it's practical to do so. In my case, it was pretty straightforward and good sense to combine a loop of `INSERT INTO` statements into a single statement, as `INSERT INTO` easily allows for that.

[Stored procedures](https://docs.microsoft.com/en-us/sql/relational-databases/stored-procedures/stored-procedures-database-engine){:target="_blank"} are the obvious solution to this problem, and are one of the reasons they're highly recommended by pretty much anyone who knows anything about SQL, including some readers of this article who I'm sure are livid by avoidance of them, but I personally find the awkwardness of use combined with the fact that my business logic ends up being split between my app and database to be unattractive enough to pass up. This investigation showed me however that when choosing that path one needs to be very careful when crafting their SQL queries and be cognizant of the number of individual calls being made to avoid performance pitfalls.
