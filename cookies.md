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
- If a cookie SameSite setting is 'None', it must also have the Secure flag (this is currently not strictly enforced, but will be in the future).


// Reading images cross-site via JS
Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource at https://b.com/image.jpg. (Reason: CORS header ‘Access-Control-Allow-Origin’ missing).

// on b.com
// document.cookie = "BCOOKIE=b;expires=Fri, 31 Dec 9999 23:59:59 GMT;path=/;SameSite=None;Secure";

Assume the following test conditons:
The following entries in `/etc/hosts`:
```
127.0.0.1 a.com
127.0.0.1 sub.a.com
127.0.0.1 b.com
```

HTTP server listening on localhost (see `http_serve.py`)


### Firefox HTTPS

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

^Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource at https://b.com/image-js.jpg. (Reason: CORS header ‘Access-Control-Allow-Origin’ missing).

^^ Cookie sent for GET, not for POST

