---
title: Utility Functions for Express-like Framework
---
# introduction

This document explains the key utility functions implemented in <SwmPath>[lib/utils.js](lib/utils.js)</SwmPath> that support core features of the Express-like framework. These utilities handle HTTP method normalization, <SwmToken path="lib/utils.js" pos="32:7:7" line-data=" * Return strong ETag for `body`.">`ETag`</SwmToken> generation, content type handling, query parsing, and proxy trust configuration.

We will cover:

1. How HTTP methods are normalized and why.
2. How <SwmToken path="lib/utils.js" pos="32:7:7" line-data=" * Return strong ETag for `body`.">`ETag`</SwmToken> generation is implemented and configured.
3. How content types are normalized.
4. How query parsers are compiled.
5. How proxy trust settings are compiled.

# http methods normalization

<SwmSnippet path="/lib/utils.js" line="25">

---

The file exports a list of HTTP methods supported by <SwmToken path="lib/utils.js" pos="26:23:25" line-data=" * A list of lowercased HTTP methods that are supported by Node.js.">`Node.js`</SwmToken>, all converted to lowercase. This standardizes method names across the framework and avoids case sensitivity issues when matching routes or handling requests.

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

# etag generation and compilation

<SwmToken path="lib/utils.js" pos="241:16:16" line-data=" * Create an ETag generator function, generating ETags with">`ETags`</SwmToken> are used for HTTP caching validation. The utilities provide two <SwmToken path="lib/utils.js" pos="32:7:7" line-data=" * Return strong ETag for `body`.">`ETag`</SwmToken> generators: one for strong <SwmToken path="lib/utils.js" pos="241:16:16" line-data=" * Create an ETag generator function, generating ETags with">`ETags`</SwmToken> and one for weak <SwmToken path="lib/utils.js" pos="241:16:16" line-data=" * Create an ETag generator function, generating ETags with">`ETags`</SwmToken>. These are created by a factory function that takes options specifying the <SwmToken path="lib/utils.js" pos="32:7:7" line-data=" * Return strong ETag for `body`.">`ETag`</SwmToken> strength.

The framework exports these generators separately so it can flexibly use either strong or weak <SwmToken path="lib/utils.js" pos="241:16:16" line-data=" * Create an ETag generator function, generating ETags with">`ETags`</SwmToken> depending on configuration.

<SwmSnippet path="/lib/utils.js" line="40">

---

To allow dynamic configuration, the <SwmToken path="lib/utils.js" pos="130:2:2" line-data="exports.compileETag = function(val) {">`compileETag`</SwmToken> function converts various input types (boolean, string, or function) into a proper <SwmToken path="lib/utils.js" pos="43:7:7" line-data=" * Return weak ETag for `body`.">`ETag`</SwmToken> generator function. This lets users specify "weak", "strong", or custom functions for <SwmToken path="lib/utils.js" pos="43:7:7" line-data=" * Return weak ETag for `body`.">`ETag`</SwmToken> generation.

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

Content types can be specified in shorthand (like "html") or full MIME types. The <SwmToken path="lib/utils.js" pos="61:2:2" line-data="exports.normalizeType = function(type){">`normalizeType`</SwmToken> function converts shorthand types into full MIME types using the <SwmToken path="lib/utils.js" pos="18:9:11" line-data="var mime = require(&#39;mime-types&#39;)">`mime-types`</SwmToken> library. If the input already contains a slash (indicating a full MIME type), it parses accept parameters instead.

This normalization ensures consistent content type handling throughout the framework, especially when matching request Accept headers or setting response <SwmToken path="lib/utils.js" pos="217:15:17" line-data=" * Set the charset in a given Content-Type string.">`Content-Type`</SwmToken> headers.

<SwmSnippet path="/lib/utils.js" line="61">

---

The <SwmToken path="lib/utils.js" pos="75:2:2" line-data="exports.normalizeTypes = function(types) {">`normalizeTypes`</SwmToken> function applies this normalization to arrays of types.

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

The framework supports different query string parsing strategies. The <SwmToken path="lib/utils.js" pos="162:2:2" line-data="exports.compileQueryParser = function compileQueryParser(val) {">`compileQueryParser`</SwmToken> function converts a configuration value into a parsing function. It supports:

- "simple" (default <SwmToken path="lib/utils.js" pos="26:23:25" line-data=" * A list of lowercased HTTP methods that are supported by Node.js.">`Node.js`</SwmToken> <SwmToken path="lib/utils.js" pos="172:5:7" line-data="      fn = querystring.parse;">`querystring.parse`</SwmToken>)
- "extended" (using the qs library for richer parsing)
- custom functions

<SwmSnippet path="/lib/utils.js" line="154">

---

This abstraction allows users to choose how query strings are parsed without changing the core framework code.

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

Proxy trust settings determine which proxies the framework should trust when parsing client IP addresses. The <SwmToken path="lib/utils.js" pos="194:2:2" line-data="exports.compileTrust = function(val) {">`compileTrust`</SwmToken> function converts various input types into a function that checks if a given proxy address is trusted.

It supports:

- boolean true (trust all)
- number (trust based on hop count)
- <SwmToken path="lib/utils.js" pos="208:5:7" line-data="    // Support comma-separated values">`comma-separated`</SwmToken> strings (list of trusted proxies)
- arrays or custom functions

<SwmSnippet path="/lib/utils.js" line="186">

---

This flexible compilation supports common deployment scenarios behind proxies or load balancers.

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
