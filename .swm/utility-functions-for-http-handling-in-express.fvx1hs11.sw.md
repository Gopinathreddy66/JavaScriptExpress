---
title: Utility Functions for HTTP Handling in Express
---
# introduction

This document explains key utility functions in <SwmPath>[lib/utils.js](lib/utils.js)</SwmPath> that support HTTP handling in Express. These utilities handle HTTP methods, <SwmToken path="lib/utils.js" pos="32:7:7" line-data=" * Return strong ETag for `body`.">`ETag`</SwmToken> generation, content type normalization, query parsing, and proxy trust configuration.

We will cover:

1. How HTTP methods are normalized and exposed.
2. How <SwmToken path="lib/utils.js" pos="32:7:7" line-data=" * Return strong ETag for `body`.">`ETag`</SwmToken> generation is implemented and configured.
3. How content types are normalized.
4. How query parsers are compiled.
5. How proxy trust settings are compiled.

# HTTP methods normalization

<SwmSnippet path="/lib/utils.js" line="25">

---

Express needs a consistent list of HTTP methods in lowercase to handle routing and method checks uniformly. This is done by mapping <SwmToken path="lib/utils.js" pos="26:23:25" line-data=" * A list of lowercased HTTP methods that are supported by Node.js.">`Node.js`</SwmToken>'s METHODS array to lowercase strings. This list is exposed as <SwmToken path="lib/utils.js" pos="29:0:2" line-data="exports.methods = METHODS.map((method) =&gt; method.toLowerCase());">`exports.methods`</SwmToken> for use throughout Express.

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

# <SwmToken path="lib/utils.js" pos="32:7:7" line-data=" * Return strong ETag for `body`.">`ETag`</SwmToken> generation and configuration

<SwmToken path="lib/utils.js" pos="241:16:16" line-data=" * Create an ETag generator function, generating ETags with">`ETags`</SwmToken> help with HTTP caching by providing a hash of response bodies. Express supports both strong and weak <SwmToken path="lib/utils.js" pos="241:16:16" line-data=" * Create an ETag generator function, generating ETags with">`ETags`</SwmToken>. The utility creates two <SwmToken path="lib/utils.js" pos="32:7:7" line-data=" * Return strong ETag for `body`.">`ETag`</SwmToken> generator functions—one for strong <SwmToken path="lib/utils.js" pos="241:16:16" line-data=" * Create an ETag generator function, generating ETags with">`ETags`</SwmToken> and one for weak <SwmToken path="lib/utils.js" pos="241:16:16" line-data=" * Create an ETag generator function, generating ETags with">`ETags`</SwmToken>—using a shared factory function that converts the response body to a buffer and applies the etag module.

<SwmSnippet path="/lib/utils.js" line="40">

---

The <SwmToken path="lib/utils.js" pos="130:2:2" line-data="exports.compileETag = function(val) {">`compileETag`</SwmToken> function converts various user configurations (boolean, string, or function) into the appropriate <SwmToken path="lib/utils.js" pos="43:7:7" line-data=" * Return weak ETag for `body`.">`ETag`</SwmToken> generator function. This allows flexible <SwmToken path="lib/utils.js" pos="43:7:7" line-data=" * Return weak ETag for `body`.">`ETag`</SwmToken> handling based on app settings.

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

Express often needs to normalize content types, for example converting "html" to <SwmToken path="lib/utils.js" pos="54:24:26" line-data=" * Normalize the given `type`, for example &quot;html&quot; becomes &quot;text/html&quot;.">`text/html`</SwmToken>. The <SwmToken path="lib/utils.js" pos="61:2:2" line-data="exports.normalizeType = function(type){">`normalizeType`</SwmToken> function checks if the input contains a slash (indicating a full MIME type) and parses it accordingly. Otherwise, it looks up the MIME type using the <SwmToken path="lib/utils.js" pos="18:9:11" line-data="var mime = require(&#39;mime-types&#39;)">`mime-types`</SwmToken> module, defaulting to <SwmToken path="lib/utils.js" pos="64:19:23" line-data="    : { value: (mime.lookup(type) || &#39;application/octet-stream&#39;), params: {} }">`application/octet-stream`</SwmToken> if unknown.

<SwmSnippet path="/lib/utils.js" line="61">

---

<SwmToken path="lib/utils.js" pos="75:2:2" line-data="exports.normalizeTypes = function(types) {">`normalizeTypes`</SwmToken> applies this normalization to an array of types. This standardizes content type handling across the framework.

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

Express supports different query string parsing modes: simple (using Node's querystring), extended (using qs), or a custom function. The <SwmToken path="lib/utils.js" pos="162:2:2" line-data="exports.compileQueryParser = function compileQueryParser(val) {">`compileQueryParser`</SwmToken> utility converts these options into the actual parsing function used by Express.

<SwmSnippet path="/lib/utils.js" line="154">

---

This abstraction allows users to choose their preferred query parsing strategy without changing internal code.

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

Express can be configured to trust certain proxies for client IP address resolution. The <SwmToken path="lib/utils.js" pos="194:2:2" line-data="exports.compileTrust = function(val) {">`compileTrust`</SwmToken> function converts various user inputs (boolean, number, string, array, or function) into a function that determines if a given proxy address is trusted.

<SwmSnippet path="/lib/utils.js" line="186">

---

This supports flexible configurations like trusting all proxies, a fixed number of hops, or specific IP addresses.

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
