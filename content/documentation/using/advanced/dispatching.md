---
draft: false
title: "Custom dispatching rules"
date: 2020-03-03
publishdate: 2020-03-03
lastmod: 2021-02-09
menu:
  docs:
    parent: using
    name: Custom dispatching rules
    weight: 130
toc: true
weight: 30 #rem
---

## Introduction

As explained into [Using exposed mocks](../mocks), Microcks is using `Dispatcher` and `Dispatching Rules` for finding an appropriate response to return when receiving a mock request. When importing a new Service or API, Microcks is indeed looking at the variable parts between the different samples of a same operation to infer those two elements.

However in some cases, this could not be enough and it will be useful to override these deduced values. Think about the case that you are dealing with an API operation that creates resources and that depending on the payload content you want specific different responses. The default dispatching rules would not allow to do that as Microcks does not analyse imported requests body content, nor is able to infer your business rules ;-) But you can realize that by overriding the `Dispatching Rules` after the first import of the API into the repository.


## The need for custom dispatching rules

Let's illustrate this with an example! Say we've got this API allowing to record a new beer in our own catalog. We have this `POST` method that allows to create new beer resources and we want to make the difference between 2 cases: the `Accepted` and the `Not accepted` responses. So we have to start describing the 2 samples into our API contract. You'll notice in the capture below that:

* `Dispatcher` and `Dispatching Rules` are empty. That means that you'll get the first found response when invoking the mock, no matter the request.
* We have used [Templating features](../templates/) to make the response content more dynamic. So the `{{ }}` notation within response body.

![dispatcher-case](/images/dispatcher-case.png)

## Dispatching rules override

Now just select `Edit Properties` of the operation from the 3-dots menu on the right of the opration name. You should be logged as a repository `manager` to have this option (see [Users management](../administrating/users/) section of documentation if needed). The next screen allows you to override default operation properties, one of them being the `Dispatching Rules` like shown in the screen below.

![dispatcher-edit](/images/dispatcher-edit.png)

> The above rule can be simply read like this *"From the incoming request, look for the value of the country field. If this field has the value 'Belgium' then return the response called 'Accepted' otherwise return the response called 'Not accepted'."*

### Inferred dispatchers

A reminder on default, inferred dispatchers: you may find `URI_PARTS`, `URI_PARAMS`, `URI_ELEMENTS`, `QUERY_MATCH` or `SCRIPT`. The first three are usually found when using Postman or OpenAPI as a contract artifact ; they are deduced from the paths and contract elements. The last two are usually found when using SoapUI as a contraxct artifact.

### JSON BODY dispatcher

The `JSON_BODY` dispatching strategy allows specifying a dispatching rule that will analyse the request payload in order to find a matching response. In order to specify such an expression you can use the help vertical right secion of the page that will provide examples and copy/paste shortcuts.

The dispatching rules of `JSON_BODY` dispatcher are always expressed using a JSON payload with 3 properties:

* `exp` is the expression to evaluate against the request body. It is indeed a [JSON Pointer](https://tools.ietf.org/html/rfc6901) expression. We already use this expressiong language in [Templating features](../templates/#json-body-pointer-expression). From the evaluation of this expression, we'll get a value. Here `/country` denotes the `country` field of incoming request.
* `op` is the operator to apply. Different operators are available like `equals`, `range`, `regexp`, `size` and `presence`,
* `cases` are a number of cases where keys are values to compare to extracted value from incoming request.

Depending on the operator applied, the `cases` may have different specification formats.

| Operator | Cases syntax | Comments |
| ---------- | ----------------- | ------- |
| `equals` | `"<value>": "<response>"` | A case named `default` is used as default option |
| `range` | `[<min>;<max>]: "<response>"` | Bracket side matter: `[` means incluse, `]` means exclusive for a left bracket. A case named `default` is used as default option | 
| `size` | `"[<min>;<max>]": "<response>"` | Size of an array property. Brackets must be inclusive. A case named `default` is used as default option |
| `regexp` | `"<posix regexp>": "<response>"` | Regular expression applied to value. A case named `default` is used as default option |
| `regexp` | `"found": "<response>"` | Check the presence/absence of a property. 2 mandatory cases: `found` and `default` |

Now that we have edited or dipsatching rule and save, we can check this rule is applied to our operation. This override of rule will be persisted into Microcks and will survive future discoveries and refreshed of this API version.

![dispatcher-updated](/images/dispatcher-updated.png)

> We recommend having an in-depth look at the exemple provided on the page to fully understand the power of different options.

#### Test our new rule!

Given the templated responses and the above dispatching rule evaluting the body of incoming requests, we can now test our mock.

Let start by creating a new beer coming from Belgium:

```sh
$ curl -X POST http://microcks.example.com/rest/Beer+Catalog+API/1.0/beer \
    -H 'Content-type: application/json' \
    -d '{"name": "Abbey Brune", "country": "Belgium", "type": "Brown ale", "rating": 4.2, "references": [ { "referenceId": 1234 }, { "referenceId": 5678 } ]}'  
{
  "name": "Abbey Brune",
  "country": "Belgium",
  "type": "Brown ale",
  "rating": 4.2,
  "references": [
    { "referenceId": 1234 },
    { "referenceId": 5678 }
  ]
}
```

It is a success as the `country` has the `Belgium` value and the `Accepted` response is returned. Templates in this response are evaluated regarding request content.

Now let's try with a German beer... You'll see that the `Not accepted` response is matched (look also at the return code) and adapted regarding incoming request:

```sh
$ curl -X POST http://microcks.example.com/rest/Beer+Catalog+API/1.0/beer \
    -H 'Content-type: application/json' \
    -d '{"name": "Spaten Oktoberfiest", "country": "Germany", "type": "Amber", "rating": 2.8, "references": []}'
< HTTP/1.1 406 
{
  "error": "Not accepted",
  "message": "Germany origin country is forbiden"
}
```

### FALLBACK dispatcher

Another useful advanced dispatching strategy introduced in the [Advanced Dispatching and Constraints for mocks](../../../../blog/advanced-dispatching-constraints/) blog post, is the `FALLBACK` strategy. As you may have guessed by its name, it behaves like a `try-catch` wrapping block in programming: it will try applying a first dispatcher with its own rule and if it find nothings it will default to a `fallback` response. This will allow you to define a default response event of the incoming requests does not match any dispatching criteria.

The dispatching rules of `FALLBACK` dispatcher are expressed using a JSON payload with 3 properties:

* `dispatcher` is the original dispatching strategy you want to be applied at first. Valid values are all the other dispatching strategies,
* `dispatcherRules` are the rules you want the original dispatcher to apply when looking for a response,
* `fallback` is simply the name of the response to use as the fallback if nothing is found on first try.

Here's below the sample that was introduced in afore mentioned blog post. In case of unknown region requested as a query parameters on a Weather Forecast API, we'll fallback to an `unknown` response providing meaningful error message:

![fallback-dispatcher](/images/blog/advanced-dispatching-constraints-final.png)

#### Test our new rule!

Just issue a Http request with an unmanaged region like below:

```sh
$ curl 'https://microcks.apps.example.com/rest/WeatherForecast+API/1.0.0/forecast?region=center&apiKey=qwertyuiop' -k
Region is unknown. Choose in north, west, east or south.%
```