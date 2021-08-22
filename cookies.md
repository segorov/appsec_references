## Cookie Attributes
### Path
- Think of this as the "minimum prefix" that needs to match the request URI for the cookie to be sent with a request. Examples:
    - Cookie path `/` matches requests for all paths.
    - Cookie path `/foo` matches requests for `/foo` and `/foo/bar`, `/foo/bar/baz`.
- Cookies can be set with a different path than the path of the current page.
- Multiple cookies with the same name for the same domain but different paths can co-exist in the cookie jar, and will all be sent in requests from Chrome and Firefox.

#### Example
```
document.cookie = "ACOOKIE=a;expires=Fri, 31 Dec 9999 23:59:59 GMT;path=/;SameSite=None;Secure";
document.cookie = "ACOOKIE=foo;expires=Fri, 31 Dec 9999 23:59:59 GMT;path=/foo;SameSite=None;Secure";
document.cookie = "ACOOKIE=foobar;expires=Fri, 31 Dec 9999 23:59:59 GMT;path=/foo/bar;SameSite=None;Secure";
```

##### Request
```
GET /foo/bar HTTP/1.1
...
Cookie: ACOOKIE=foobar; ACOOKIE=foo; ACOOKIE=a
```

Inside the request, if multiple cookies are sent, they are generally ordered by the specificity of their path:
> The user agent SHOULD sort the cookie-list in the following order: <br>
Cookies with longer paths are listed before cookies with shorter paths.

Both Chrome and Firefox appear to respect this ordering when sending requests.
How the ordering of the cookie values is handled server-side is up to the receiving server's user agent. For example, Python's `SimpleCookie` will return the last value from the request header:

```
// Header sent with request
Cookie: ACOOKIE=foobar; ACOOKIE=foo; ACOOKIE=a

// Python code
...
cookies = SimpleCookie(self.headers.get('Cookie'))
if 'ACOOKIE' in cookies:
    print 'ACOOKIE=' + cookies['ACOOKIE'].value
...

// output
ACOOKIE=a

```


### Domain
- The Domain attribute specifies which hosts a user agent may send the cookie to.
- If Domain is unspecified, it defaults to the same host that set the cookie, excluding subdomains. This is called a "host-only" cookie.
- If Domain is specified, then subdomains are always included. Example:
    - `Domain: foo.com` cookies may be sent with requests to `bar.foo.com`
- Cookies cannot be set for domains that don't match the current domain, including sub-domains of the current domain. They _can_ be set for "super-domains" of the current domain. Examples:
    - Response to request `sub.a.com/foo` cannot set a cookie with Domain `b.com` or `sub.sub.a.com`, but it can set a cookie with Domain `a.com`.
- Cookies do not provide isolation by port. If a domain has 2 HTTP services listening on different ports, cookies for that domain will be sent to both services. Example:
    - Cookie is set for Domain `a.com`. It will also be sent along with requests to `a.com:8080`. Note: if cookie `Secure` attribute is set, it will only be sent over HTTPS connections, so it would be sent to `https://a.com:8080/` but not `http://a.com:8080/`.
- Sometimes the domain will start with a dot: `.foo.com`. This used to mean the cookie was also valid for sub-domains of `foo.com`, but RFC 6265 changed this rule so modern browsers ignore the leading dot and cookies are valid for subdomains without it.

### Secure
- If the Secure attribute of a cookie is set, the user-agent should only send that cookie over HTTPS and never over cleartext HTTP.
  - Note: localhost can an exception. When connecting to localhost, `Secure` cookies will be sent over HTTP as well in Firefox.
- In practice, the Secure attribute should pretty much always be set.

### HttpOnly
- The HttpOnly attribute of a cookie tells the user agent that this cookie cannot be accessed through client side scripts.
Example:
```
// Server-side response directs user agent to create HttpOnly cookie
Set-Cookie:HOC=foo;expires=Fri, 31 Dec 9999 23:59:59 GMT;path=/;Secure;HttpOnly

// Javascript on page
<script> console.log(document.cookie); <script>
> ""
```
- An important note about `HttpOnly` is that requests initiated from client side scripts via XHR _will_ still send `HttpOnly` cookies with requests. They just can't be directly accessed and read by the scripts. The name of this flag is misleading in this respect.
Example:
```
// https://a.com/ contains the following script:
<script>
    const Http = new XMLHttpRequest();
    const url='https://a.com/script';
    Http.open("GET", url);
    Http.send();
</script>

// request to `https://a.com/script contains header:
Cookie: HOC=foo
```
- (?) Any `HttpOnly` cookies set via a response to an XHR request will also be unavailable to client side scripts through the response object. Example:
```

```
- (?) Note: if the XHR request is cross-domain, cookies may or may not be sent depending on the cookie's `SameSite` attribute. See `SameSite` section.

### SameSite
- The SameSite attribute declares if a cookie should be restricted to a first-party/same-site context.
- What constitutes "first-party"? Example:
    - When loading a page on domain a.com, assets on the page from b.com are considered third-party. Assets from sub-domains such as sub.a.com are considered first-party.
- Valid values for SameSite are `None`, `Lax`, and `Strict`. The general meaning of the 3 settings:
    - `None`: Cookies will be sent in all contexts. Same behavior as before SameSite attribute existed.
    - `Lax`: Cookies will only be sent when navigating to a site (ex. clicking a link to b.com on a page hosted on a.com, but not loading an image from b.com on a page hosted on a.com).
    - `Strict`: Cookies will only be sent in a first-party context. Ex. a page hosted on a.com will only send cookies for other assets on a.com or subdomains of a.com.
    - A deep dive into specific types of embedded/fetched/linked content and page interactions how the SameSite settings apply follows.
- What about different ports? Does SameSite apply to cookies set for a.com to content from a.com:8080? This is apparently a grey area and behavior may be browser-specific. Example in Firefox:
    - Cookie with domain a.com is also sent with requests to a.com:8080. This also applies in a third-party context: b.com domain cookies are sent with requests for b.com:8080 content embedded on a page on a.com assuming SameSite checks pass.
- In recent browser versions, if a SameSite attribute is not explicitly specified, the default values is "Lax".
- If a cookie SameSite setting is 'None', it must also have the Secure flag (this is currently not strictly enforced, but will be in the future).

#### SameSite Test Scenarios
Here is a matrix describing how browsers (tested on Firefox and Chrome) will apply SameSite policy to different types of content/user interactions.
Test scenario is as follows:
- Basic HTTP server listening on localhost port 443 with self-signed TLS certificate installed
- The following entries in `/etc/hosts`:
```
127.0.0.1 a.com
127.0.0.1 sub.a.com
127.0.0.1 b.com
```

- Cookies set manually for each domain with different SameSite settings for each test scenario (Note: domain _must_ be specified explicitly for cookies to be sent to sub-domains of the current domain; otherwise they are restricted to the exact domain only.)
```
document.cookie = "ACOOKIE=a;Domain=a.com;expires=Fri, 31 Dec 9999 23:59:59 GMT;path=/;SameSite=None;Secure
document.cookie = "SUBCOOKIE=sub;Domain=sub.a.com;expires=Fri, 31 Dec 9999 23:59:59 GMT;path=/;SameSite=None;Secure
document.cookie = "BCOOKIE=b;Domain=b.com;expires=Fri, 31 Dec 9999 23:59:59 GMT;path=/;SameSite=None;Secure
```

Each test scenario is captured in its own HTML file (see tests directory).
To run each test, load the page via its https://a.com URL and observe whether a cookie is sent with requests for assets on b.com and sub.a.com..
Note: it's recommended to disable browser cache when running the tests. A simple way to do this is to reload a page using Ctrl/Cmd + Shift + R.

#### SameSite Test Results Matrix

Each test scenario was ran with each of the 3 SameSite settings for the `b.com` cookie. The following table summarizes whether the `b.com` cookie is sent with each type of third-party `b.com` content embedded on an `a.com` page or action taken with a `b.com` destination (ex. submitting a form to `b.com` from `a.com`).

|                         | None | Lax | Strict |
| ----------------------- | ---  | --- | ---    |
| `form` submit           | Yes  | Yes^^  | No  |
| `img` tag               | Yes  | No | No |
| `iframe` source         | Yes  | No | No |
| clicking an `href` link | Yes  | Yes | No |
| XHR request             | No^ | No | No |
| `meta` refresh          | Yes  | Yes | No |
| `window.location.href`   | Yes | Yes | No |
| `window.location.replace`| Yes | Yes | No |

^The request is made, but reading the response is not allowed: Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource at https://b.com/image-js.jpg. (Reason: CORS header ‘Access-Control-Allow-Origin’ missing).
^^ Cookie is sent with GET requests, but not wit POST requests.

Subdomain results: subdomains are treated as first-party as far as cookie SameSite restrictions go. This means that all requests for `sub.a.com` and `a.com` content on `sub.a.com` pages contained both `ACOOKIE` and `SUBCOOKIE` and all embedded content/actions taken with a `sub.a.com` source/destination on `a.com` pages contained `SUBCOOKIE`.


