---
title: Utility Functions for HTTP Handling in Express
---
# introduction

This document explains key utility functions in <SwmPath>[lib/utils.js](lib/utils.js)</SwmPath> that support HTTP handling in Express. These utilities handle HTTP methods, <SwmToken path="lib/utils.js" pos="32:7:7" line-data=" * Return strong ETag for `body`.">`ETag`</SwmToken> generation, content type normalization, query parsing, and proxy trust configuration.

We will cover:

1. How HTTP methods are normalized and exposed.
2. How <SwmToken path="lib/utils.js" pos="241:16:16" line-data=" * Create an ETag generator function, generating ETags with">`ETags`</SwmToken> are generated and configured.
3. How content types are normalized.
4. How query string parsers are compiled.
5. How proxy trust settings are compiled.

# http methods normalization

<SwmSnippet path="/lib/utils.js" line="25">

---

Express needs a consistent list of HTTP methods in lowercase to handle routing and method checks uniformly. This is done by mapping <SwmToken path="lib/utils.js" pos="26:23:25" line-data=" * A list of lowercased HTTP methods that are supported by Node.js.">`Node.js`</SwmToken>'s METHODS array to lowercase strings, which are then exported for use elsewhere in the framework.

```javascript
/**
 * A list of lowercased HTTP methods that are supported by Node.js.
 * @api private
 */
exports.methods = METHODS.map((method) => method.toLowerCase());

/**
 * Return strong ETag for `body`.
 *
 * @param {String|Buffer} body
 * @param {String} [encoding]
 * @return {String}
 * @api private
 */
```

---

</SwmSnippet>

# etag generation and configuration

<SwmToken path="lib/utils.js" pos="241:16:16" line-data=" * Create an ETag generator function, generating ETags with">`ETags`</SwmToken> are used for cache validation. Express provides two <SwmToken path="lib/utils.js" pos="32:7:7" line-data=" * Return strong ETag for `body`.">`ETag`</SwmToken> generators: strong and weak. These are created by a factory function that takes options specifying the <SwmToken path="lib/utils.js" pos="32:7:7" line-data=" * Return strong ETag for `body`.">`ETag`</SwmToken> type. The factory converts the response body into a buffer and generates the <SwmToken path="lib/utils.js" pos="32:7:7" line-data=" * Return strong ETag for `body`.">`ETag`</SwmToken> accordingly.

The <SwmToken path="lib/utils.js" pos="40:0:2" line-data="exports.etag = createETagGenerator({ weak: false })">`exports.etag`</SwmToken> and <SwmToken path="lib/utils.js" pos="51:0:2" line-data="exports.wetag = createETagGenerator({ weak: true })">`exports.wetag`</SwmToken> functions are created this way, representing strong and weak <SwmToken path="lib/utils.js" pos="241:16:16" line-data=" * Create an ETag generator function, generating ETags with">`ETags`</SwmToken> respectively.

<SwmSnippet path="/lib/utils.js" line="40">

---

To allow flexible configuration, Express compiles an <SwmToken path="lib/utils.js" pos="43:7:7" line-data=" * Return weak ETag for `body`.">`ETag`</SwmToken> function based on user input. The <SwmToken path="lib/utils.js" pos="130:2:2" line-data="exports.compileETag = function(val) {">`compileETag`</SwmToken> function accepts booleans, strings, or functions and returns the appropriate <SwmToken path="lib/utils.js" pos="43:7:7" line-data=" * Return weak ETag for `body`.">`ETag`</SwmToken> generator or disables it.

```javascript
exports.etag = createETagGenerator({ weak: false })

/**
 * Return weak ETag for `body`.
 *
 * @param {String|Buffer} body
 * @param {String} [encoding]
 * @return {String}
 * @api private
 */

exports.wetag = createETagGenerator({ weak: true })

/**
 * Normalize the given `type`, for example "html" becomes "text/html".
 *
 * @param {String} type
 * @return {Object}
 * @api private
 */
```

---

</SwmSnippet>

<SwmSnippet path="/lib/utils.js" line="122">

---

&nbsp;

```javascript
/**
 * Compile "etag" value to function.
 *
 * @param  {Boolean|String|Function} val
 * @return {Function}
 * @api private
 */

exports.compileETag = function(val) {
  var fn;

  if (typeof val === 'function') {
    return val;
  }

  switch (val) {
    case true:
    case 'weak':
      fn = exports.wetag;
      break;
    case false:
      break;
    case 'strong':
      fn = exports.etag;
      break;
    default:
      throw new TypeError('unknown value for etag function: ' + val);
  }

  return fn;
}
```

---

</SwmSnippet>

<SwmSnippet path="/lib/utils.js" line="240">

---

&nbsp;

```javascript
/**
 * Create an ETag generator function, generating ETags with
 * the given options.
 *
 * @param {object} options
 * @return {function}
 * @private
 */

function createETagGenerator (options) {
  return function generateETag (body, encoding) {
    var buf = !Buffer.isBuffer(body)
      ? Buffer.from(body, encoding)
      : body

    return etag(buf, options)
  }
}
```

---

</SwmSnippet>

# content type normalization

Express normalizes content types to ensure consistent handling of MIME types. The <SwmToken path="lib/utils.js" pos="61:2:2" line-data="exports.normalizeType = function(type){">`normalizeType`</SwmToken> function converts shorthand types like "html" into full MIME types like <SwmToken path="lib/utils.js" pos="54:24:26" line-data=" * Normalize the given `type`, for example &quot;html&quot; becomes &quot;text/html&quot;.">`text/html`</SwmToken> using the <SwmToken path="lib/utils.js" pos="18:9:11" line-data="var mime = require(&#39;mime-types&#39;)">`mime-types`</SwmToken> library. It also parses parameters if present.

<SwmSnippet path="/lib/utils.js" line="61">

---

For arrays of types, <SwmToken path="lib/utils.js" pos="75:2:2" line-data="exports.normalizeTypes = function(types) {">`normalizeTypes`</SwmToken> applies <SwmToken path="lib/utils.js" pos="61:2:2" line-data="exports.normalizeType = function(type){">`normalizeType`</SwmToken> to each element.

```javascript
exports.normalizeType = function(type){
  return ~type.indexOf('/')
    ? acceptParams(type)
    : { value: (mime.lookup(type) || 'application/octet-stream'), params: {} }
};

/**
 * Normalize `types`, for example "html" becomes "text/html".
 *
 * @param {Array} types
 * @return {Array}
 * @api private
 */

exports.normalizeTypes = function(types) {
  return types.map(exports.normalizeType);
};


/**
 * Parse accept params `str` returning an
 * object with `.value`, `.quality` and `.params`.
 *
 * @param {String} str
 * @return {Object}
 * @api private
 */
```

---

</SwmSnippet>

# query parser compilation

Express supports different query string parsing strategies. The <SwmToken path="lib/utils.js" pos="162:2:2" line-data="exports.compileQueryParser = function compileQueryParser(val) {">`compileQueryParser`</SwmToken> function converts a string or function input into a query parser function. It supports:

- 'simple' (default <SwmToken path="lib/utils.js" pos="26:23:25" line-data=" * A list of lowercased HTTP methods that are supported by Node.js.">`Node.js`</SwmToken> <SwmToken path="lib/utils.js" pos="172:5:7" line-data="      fn = querystring.parse;">`querystring.parse`</SwmToken>)
- 'extended' (using <SwmToken path="lib/utils.js" pos="268:3:5" line-data="  return qs.parse(str, {">`qs.parse`</SwmToken> for richer parsing)
- disabling parsing

<SwmSnippet path="/lib/utils.js" line="154">

---

This allows users to choose how query strings are parsed based on their needs.

```javascript
/**
 * Compile "query parser" value to function.
 *
 * @param  {String|Function} val
 * @return {Function}
 * @api private
 */

exports.compileQueryParser = function compileQueryParser(val) {
  var fn;

  if (typeof val === 'function') {
    return val;
  }

  switch (val) {
    case true:
    case 'simple':
      fn = querystring.parse;
      break;
    case false:
      break;
    case 'extended':
      fn = parseExtendedQueryString;
      break;
    default:
      throw new TypeError('unknown value for query parser function: ' + val);
  }

  return fn;
}
```

---

</SwmSnippet>

<SwmSnippet path="/lib/utils.js" line="259">

---

&nbsp;

```javascript
/**
 * Parse an extended query string with qs.
 *
 * @param {String} str
 * @return {Object}
 * @private
 */

function parseExtendedQueryString(str) {
  return qs.parse(str, {
    allowPrototypes: true
  });
}
```

---

</SwmSnippet>

# proxy trust compilation

Express needs to determine which proxies it trusts to correctly handle client IP addresses. The <SwmToken path="lib/utils.js" pos="194:2:2" line-data="exports.compileTrust = function(val) {">`compileTrust`</SwmToken> function converts various inputs into a function that returns whether a given proxy is trusted.

It supports:

- functions (used as-is)
- booleans (trust all or none)
- numbers (trust based on hop count)
- strings (<SwmToken path="lib/utils.js" pos="208:5:7" line-data="    // Support comma-separated values">`comma-separated`</SwmToken> IPs or subnets)
- arrays (list of trusted proxies)

<SwmSnippet path="/lib/utils.js" line="186">

---

This flexibility lets Express adapt to different deployment environments.

```javascript
/**
 * Compile "proxy trust" value to function.
 *
 * @param  {Boolean|String|Number|Array|Function} val
 * @return {Function}
 * @api private
 */

exports.compileTrust = function(val) {
  if (typeof val === 'function') return val;

  if (val === true) {
    // Support plain true/false
    return function(){ return true };
  }

  if (typeof val === 'number') {
    // Support trusting hop count
    return function(a, i){ return i < val };
  }

  if (typeof val === 'string') {
    // Support comma-separated values
    val = val.split(',')
      .map(function (v) { return v.trim() })
  }

  return proxyaddr.compile(val || []);
}
```

---

</SwmSnippet>

&nbsp;

*This is an auto-generated document by Swimm 🌊 and has not yet been verified by a human*

<SwmMeta version="3.0.0" repo-id="Z2l0aHViJTNBJTNBSmF2YVNjcmlwdEV4cHJlc3MlM0ElM0FHb3BpbmF0aHJlZGR5NjY=" repo-name="JavaScriptExpress"><sup>Powered by [Swimm](https://app.swimm.io/)</sup></SwmMeta>
