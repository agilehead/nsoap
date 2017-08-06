# NSOAP: Native Syntax Object Access Protocol

NSOAP is a Remote Procedure Call (RPC) and URL convention that uses familiar JavaScript syntax for method invocation and parameter passing. In addition to web services, NSOAP conventions can also be used for client-side routing in React, Angular etc. The NSOAP project provides routers for Express, Koa and React. Contributions invited for other frameworks and languages.  

Attempting to explain it without code is futile. Let's go straight to the examples.

Choose your framework

<ul class="selector">
  <li><a href="/">Express JS</a></li>
  <li><a href="/koa.html">Koa JS</a></li>
  <li><a href="/react.html">React JS</a></li>
</ul>

## Initializing your App: The App Object

The App Object (myApp in the following example) contains all the "routes" the application will respond to.

```javascript
// This example assumes Express JS
const express = require("express");
const nsoap = require("nsoap-express");

const app = express();

const myApp = {
  addTwoNumbers(x, y) {
    return x + y;
  },
  sayHello(name) {
    return `Hello ${name}!`
  }
}

app.use(nsoap(myApp));
```

## Invoking Functions

Invoke a function that adds two numbers looks like plain JavaScript

```bash
curl "http://www.example.com/addTwoNumbers(10,20)"
```

Arguments can be strings, numbers or booleans. If they are strings, they must still be valid JavaScript identifiers.

```bash
# numeric argument
curl "http://www.example.com/addTwoNumbers(10,20)"
# boolean argument
curl "http://www.example.com/findAll(true)"
# string argument
curl "http://www.example.com/search(thomas)"
```

Use parameters.

```bash
# numeric
curl "http://www.example.com/addTwoNumbers(x,y)?x=10&y=20"
# string
curl "http://www.example.com/search(x)?x=thomas"
```

If the argument is a string and it contains spaces or other characters, you will need to quote and encode them. They can only be passed via parameter variables.

```bash
# Have spaces? Must quote and encode.
# %22 is double quote, %20 is space
# x = "thomas jacob"
curl "http://www.example.com/search(x)?x=%22thomas%20jacob%22"
```

You may pass full JSON objects via parameter variables.

```bash
# x = { "title": "bring milk", "assignee": "me" })
# encodeURIComponent(x)
curl "http://www.example.com/findTodo(x)?x=
%7B%20%22title%22%3A%20%22bring%20milk%22%2C%20%22assignee%22%3A%20%22me%22%20%7D"
```

## Returning a response

Applications are expected to return the response to be sent to the client.

```javascript
const myApp = {
  addTwoNumbers(x, y, { request, response }) {
    return x + y;
  },
}
```

In case there is an error, throw an exception.

```javascript
const myApp = {
  addTwoNumbers(x, y, { request, response }) {
    if (typeof  x === "undefined" || typeof y === "undefined") {
      throw new Error("Invalid value.")
    } else {
      return x + y;
    }
  },
}
```

## On the server, use GET, POST, PUT whatever.

Arguments passed via the query string need to be URI encoded as seen in examples above. Arguments passed via HTTP method body are parsed with JSON.parse; so they need to be valid. For examples, see the documentation for NSOAP-Express or NSOAP-Koa.

```bash
# Using POST with JSON content type
curl -H "Content-Type: application/json" -X POST -d '{"x":10,"y":20}' "http://www.example.com/addTwoNumbers(x,y)"
# Using POST with url encoding.
curl --data "x=10&y=20" "http://www.example.com/addTwoNumbers(x,y)"
```

## Organizing code with Namespaces

Invoke a function defined on an object. This allows organizing the code into namespaces similar to directories.

```bash
curl "http://www.example.com/math.square(20)"
# OR
curl "http://www.example.com/math.square(x)?x=20"
# returns 400
```

## Parenthesis

Parenthesis may be omitted if the function can be called without arguments. If the url refers to an object instead of a function, its value is simply returned.

```bash
curl "http://www.example.com/default"
# is the same as
curl http://www.example.com/default()
```

## Default Functions

Routers will provide Applications an option to specify the default function to be called when the url omits the function name. Routers should use "index" if no default function is specified.

If the url points to the root (in other words, if the path = "/"), the default function is invoked on the App object.

```bash
# Assuming that the default function name has not been changed
curl "http://www.example.com/"
# is the same as
curl http://www.example.com/index()
```

## Function Chaining

Chained function calls work the same way you expect it to work. The following url invokes the getAccounts function on the result of the customer function. If the function returns a Promise (or a Future), the Promise is resolved and the subsequent function or object is accessed on the resolved value.

```bash
curl "http://www.example.com/customer(100).getAccounts(2017)"
#OR
curl "http://www.example.com/customer(custId).getAccounts(year)?custId=100&year=2017"
```

## Parameter Type Inference

NSOAP supports parameter type inference for strings, numbers and booleans. In the following example, the function parameters are inferred as string, number, boolean and number.

```bash
curl "http://www.example.com/search(Jeswin,20,true,x)?x=100"
```

## Case-sensitivity

NSOAP is case-sensitive. So the following will not assign 100 to the parameter 'x'.

```bash
# Error. 'x' is not the same as 'X'
curl "http://www.example.com/squareRoot(x)?X=100"
```

## Advanced Default Functions

If the return value is an object, and it contains a property with the same name as the default function name, and if the value of the property is a function, it is invoked.

That may sound confusing, let's look at an example. Assume that the default function name is unchanged, or "index". The addTwoNumbers() function returns an object with a property named "index" which is a function. Since it matches the default function name, it is invoked.

```javascript
// The app
const myApp = {
  addTwoNumbers(x, y) {
    return {
      index() {
        return x + y;
      }
    }
  },
}
```

```bash
curl "http://www.example.com/addTwoNumbers(10,20)"
# returns 30
```

This allows for more powerful chained functions. The following script can now respond to "www.example.com/getCustomer(1)" and "www.example.com/getCustomer(1).getTotalPurchases()"

```javascript
// The app
const myApp = {
  getCustomer(id) {
    return {
      index() {
        return id === 1 ? { name: "Jeswin" } : { name: "Thomas" }
      },
      totals() {
        return id === 1 ? 100 : 200;
      }
    }
  },
}
```

```bash
# will return { name: "Jeswin" }
curl "http://www.example.com/getCustomer(1)"

# will return 100
curl "http://www.example.com/getCustomer(1).totals"
```

## Raw Request and Response Handling

Sometimes, you might want to access the request and response objects directly. You could do that by returning a function as the result. In the following example, the handler for the url /getCustomer(100) has full access to the request and response objects.  

```javascript
const myApp = {
  getCustomerName(id) {
    return (req, res) => {
      res.send(id === 1 ? "Jeswin" : "Thomas)
    }
  },
}
```

```bash
# will return "Jeswin"
curl "http://www.example.com/getCustomerName(1)"
```

## HTTP Headers and Cookies

Values defined in HTTP Headers and Cookies are given the same treatment as values passed via querystring or the body. They key is taken as the parameter name and the corresponding value is assigned.

```bash
# returns 400
curl --header "x:20" "http://www.example.com/math.square(x)"
```

## Security

NSOAP-Express is HTTP method agnostic. This means that a method can be called via GET, POST or whatever. If you wish to allow only specific HTTP methods, you need to use Raw Request and Response Handling.

```javascript
const myApp = {
  greeting(id) {
    return (req, res) => {
      if (req.method === "POST") {
        res.send("Hello")
      } else {
        res.status(404).send("Sorry, nothing to see here.")
      }
    }
  },
}
```

Be careful with Cookies, ensure that you are protected against CSRF vulnerabilities. Our recommendation is that you use HTTP Headers and don't use Cookies; if you're building a Single Page App or using AJAX to make calls.

```bash
curl --header "session-token:AD332DA12323AAA" "http://www.example.com/placeOrder(itemId, quantity, sessionToken)?itemId=200&quantity=3"
```

## Order of searching parameters in a Request

NSOAP server-side routers must attempt to find parameter values in the following order

- Headers
- Querystring
- Body
- Cookies

This means that if a parameter say "x" is defined in the Headers and in the Body, the value found in Headers will take precedence.

## Status Codes

Router will set the status code to 200 when the function executes without errors and 500 if an exception was thrown. If special status codes are needed (say HTTP 301 Permanent Redirect), applications are expected to use Raw Request and Response Handling as in the following example.

```javascript
const myApp = {
  getCustomer(id) {
    return (req, res) => {
      res.status(200).send("Hello")
    }
  },
}
```

## Reading and Writing Streams

To read or write streams, you may use Raw Request and Response Handling.

```javascript
const myApp = {
  streamData() {
    return (req, res) => {
      const interval = setInterval(() => {
        res.write(JSON.stringify({ foo: Math.random() * 100, count: ++c }) + '\n');
        if (c === 10) {
          clearInterval(interval);
          res.end();
        }
      }, 1000);
    }
  },
}
```

## Advanced Options

Additional options may be pass in via the options object while initializing NSOAP-Express.
You would probably not need any of these.

### urlPrefix

Makes the router handle only urls prefixed with the specified path.

```javascript
//...
const options = { urlPrefix = "/home" };
app.use(nsoap(myApp, options));
```

### getBody(req: Request) : Object

Allows an app to alter the body parameter dictionary passed into NSOAP-Express.
If getBody is not defined, the body is assumed to be req.body; which is what middleware like "bodyparser" do.

The following example adds additional parameters to the body dictionary which was already created by bodyparser.

```javascript
function getBody(req) {
  return { ...req.body, newParam: 1, anotherParam: "TWO" }
}

const options = { getBody };
app.use(nsoap(myApp, options));
```

### getCookies(req: Request) : Object

Allows an app to alter the Cookies parameter dictionary passed into NSOAP-Express.
If getCookies is not defined, the body is assumed to be req.cookies; which is what middleware like "cookie-parser" do.

The following example adds additional parameters to the cookies dictionary which was already created by cookie-parser.

```javascript
function getCookies(req) {
  return { ...req.cookies, newParam: 1, anotherParam: "TWO" }
}

const options = { getCookies };
app.use(nsoap(myApp, options));
```

### parseQuery(query: Object) : Object

Allows an app to alter the querystring dictionary passed into NSOAP.

```javascript
function parseQuery(query) {
  return { ...query, id: 100 }
}

const options = { parseQuery };
app.use(nsoap(myApp, options));
```

### parseBody(body: Object) : Object

Allows an app to alter the body dictionary passed into NSOAP.
This is the same dictionary that was previously constructed via getBody() or in its absense req.body.

```javascript
function parseBody(body) {
  return { ...body, id: 100 }
}

const options = { parseBody };
app.use(nsoap(myApp, options));
```

### parseHeaders(headers: Object) : Object

Allows an app to alter the headers dictionary passed into NSOAP.

```javascript
function parseHeaders(headers) {
  return { ...headers, id: 100 }
}

const options = { parseHeaders };
app.use(nsoap(myApp, options));
```

### parseCookies(cookies: Object) : Object

Allows an app to alter the headers dictionary passed into NSOAP.

```javascript
function parseCookies(cookies) {
  return { ...cookies, id: 100 }
}

const options = { parseCookies };
app.use(nsoap(myApp, options));
```

