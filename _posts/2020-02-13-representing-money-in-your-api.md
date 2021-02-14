---
toc: true
layout: post
description: Representing Money in your API response.
categories: [markdown]
title: Representing Money in your API response
---
# Representing Money in your API response
> Representing monetary values in an API might seem straightforward at first,
> however on second thought, there are some subtle complications that can
 > arise if not given enough thought.

## The Problem.
I was trawling through one of my favourite crypto channels when I
came across an interesting conversation.

![]({{site.baseurl }}/images/money/money_api_convo.png "API discord conversation")

This got me thinking. In principle, once an API is published,
it can't really be changed. You can always publish a new version
but then, now you have to manage multiple versions of an API.

So you have two options. Try to think of all the possible use cases
(hint. impossible).
Or you could build your API in a way that allows it to be flexible
and extensible with backwards compatibility. It turns out, monetary values are a
tricky beast. There also doesn't seem to be any standard way to
represent it. So we'll try and see a few ways of tackling the problem.

As an API developer, You want to make consuming your API as convenient
as possible however you need to know where to draw the line, and sometimes
the borders between client/server responsibilities are not very clear.

### The wrong way

```json
{
    "accountBalance" : "$100,000.00"
}
```

The above seems convenient enough, the client has only to pull the value
pre-formatted and display it on screen. Client developers would love it.
Except your Dutch clients. This is because in some parts of the world,
The Netherlands included, monetary values use the dot format. i.e. (.) to
represent thousand separators and (,) to represent decimal.

- Comma format : 2,000,000.00
- Dot format : 2.000.000,00

They would literally have to write code to clear your formatting before
formatting it in a manner that is useful to them.

{% include alert.html text="A good rule of thumb is... _If the value is to be displayed to the
 user in some sort of UI. It's not your problem, send the data in as basic a format as
 possible. Or use established standards where necessary._" %}

```json
{
    "accountBalance" : "1000000",
    "symbol": "$",
    "code": "USD"
}
```

That looks nice for whole numbers, but what about floating points.
The use case described in the image above indicated that they must have been
dealing with some very small numerical values.

I took a look at the API myself.

```json
GET https://ethgasstation.info/api/ethgasAPI.json?

{
    "fast": 2340,
    "fastest": 2340,
    "safeLow": 1170,
    "average": 1360,
    "block_time": 13.8,
    "blockNum": 11850976,
    "speed": 0.9991300618896608,
    "safeLowWait": 13.1,
    "avgWait": 1.5,
    "fastWait": 0.5,
    "fastestWait": 0.5,
    "gasPriceRange": {
        "4": 230,
        "6": 230,
        "8": 230,
        "10": 230,
        ...
        "820": 230,
        "840": 29.8,
        "860": 29.8,
        ...
        "1460": 1.3
    }
}
```

As expected I saw another warning.
![]({{site.baseurl }}/images/money/gwei_warning.png "gwei warning")

It began to seem to me that the reasons for the design of the APi that way
wasn't a client consideration, but something to do with the API backend implementation.

The most obvious question I had was, Why not just do the division yourself while returning
the value???
Should the client care if you return `"4 : 23.0"` than returning `"4 : 230"` with the added
 caveat that the clients should divide by 10. And what the hell is Gwei?

## About units
So it turns out a Gwei is the smallest unit of ETH (essentially the ETH equivalent of a satoshi)
Due to the nature of cryptocurrencies, you an send over really tiny amounts of money,
and while the examples above assume standard whole numbers, it doesn't deal with fractions.

So how do we represent the currency values in a way that is future proof and still allows us
a bit of flexibility to modify the api implementation, if things change in the future.

Here's an attempt.
```json
{
    "accountBalance" : "230",
    "code": "ETH",
    "unit": "Gwei",
    "multiplicationFactor" : 0.1
}
```

So Here I've added a field `multiplicationFactor`. All it really does is provide a value
which you can multiply by to produce base units, if needed. I've also provided the
unit field as well for information purposes. The idea being that the final value displayed
in base units is the `multiplicationFactor * accountBalance`

So how can we represent 230.34 using this format?

```json
{
    "accountBalance" : "230.34",
    "code": "USD",
    "unit": "BASE",
    "multiplicationFactor" : 1
}
```
It outputs `230.34`

This monetary value can also be represented as.
```json
{
    "accountBalance" : "23034",
    "code": "USD",
    "unit": "MINOR",
    "multiplicationFactor" : 0.01
}
```
Same output representation! 

Of course, you would want to define the units up front, so you don't
surprise your clients, however the unit field is purely informational, the core logic only
depends on the two fields described above.

The question of which units to use is one of those that draws a *meh* response. The case being that there isn't really a right, 
or wrong way to do it. Looking at this [question](https://stackoverflow.com/questions/29341337/how-to-define-money-amounts-in-an-api) 
on StackOverflow yields insight into how different systems implement it.