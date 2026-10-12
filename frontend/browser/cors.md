
CORS stands for **Cross-Origin Resource Sharing**.

Background: **SOP** (Same-Origin Policy) is enforced by browsers. JavaScript on one site can't just freely read responses from a different site. 

An *origin* is a combination of 3 things:

https://myapp.com:443/page

scheme: https://, 
host: myapp.com, 
port:443

If any of these 3 differ, it's a different origin.

Example: 
- Same Origin: https://myapp.com/api/users 
- Different origin: http://myapp.com/api/users (scheme differs)
- Different origin: https://myapp.com:8080/users (port differs)

This **SOP** quite important. Imagine you're logged into your bank at `bank.com`. Without SOP, any random site you visit could run `fetch(bank.com)` using your logged-in cookies and read your balance. 

But sometimes we need exceptions (e.g. a frontend at `myapp.com` often needs to call an api at a server `api.myapp.com`).

## So, what is CORS?

Thus, **CORS** is the mechanism that allows these exceptions to happen. 

Example:
- Frontend at `http://localhost:3000` runs:
  ```javascript
  fetch("https://api.example.com/users")
  .then(res => res.json())
  .then(console.log);
  ```

- The browser sends this request with an `Origin` header:
  ```
	GET /users HTTP/1.1
	Host: api.example.com
	Origin: http://localhost:3000
  ```

- If the server doesn't opt in, it responds normally with no CORS headers and the browser blocks JavaScript from reading it. Console will probably show:
```
Access to fetch at 'https://api.example.com/users' from origin
'http://localhost:3000' has been blocked by CORS policy: No
'Access-Control-Allow-Origin' header is present on the requested resource.
```

- If the server opts in, it should add: `Access-Control-Allow-Origin: http://localhost:3000`


## Preflight Requests

For more complex requests such as: PUT / DELETE, a JSON `Content-Type` , or custom headers like `Authorization`, the browser first sends an automatic **preflight** OPTIONS request asking permission:

```http
OPTIONS /users HTTP/1.1
Origin: http://localhost:3000
Access-Control-Request-Method: DELETE
Access-Control-Request-Headers: authorization, content-type
```

The server must reply with what it allows:
``` http
Access-Control-Allow-Origin: http://localhost:3000
Access-Control-Allow-Methods: GET, POST, DELETE
Access-Control-Allow-Headers: authorization, content-type
Access-Control-Max-Age: 600
```

Only if this checks out, does the browser send the actual request. 

## Other notes:

- CORS is enforced by the browser, not the server. The server still receives and processes our requests (curl, Postman, and server-to-serve calls ignore CORS entirely). 
- CORS is not a security feature for APIs. It just protects a user's browser. Thus, anyone can still call our API directly, so we still need real authentication. 
- In development, many people avoid CORS by using a **proxy** (e.g. Vite's server.proxy or Next.js rewrites). 
	- Your browser talks to `localhost:3000/api/...` and then the dev server forwards it to the real API. This way, the browser only sees one origin and there's no CORS issue.