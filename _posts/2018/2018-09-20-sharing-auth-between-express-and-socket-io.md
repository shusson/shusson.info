# Sharing auth between express and socket IO

__20/09/2018__

![identity](https://imgs.xkcd.com/comics/identity.png)

JWTs can be a pretty [decisive](https://hn.algolia.com/?query=JWT&sort=byPopularity&prefix&page=0&dateRange=all&type=story) subject and there's still a bit of confusion around how/when to use them. We send JWTs in cookies and use them as session tokens. If we add socket.io to our stack we can use the same session token from the client to authenticate. It's pretty straightforward but the parsing was a little confusing first.

```typescript
// first we get the cookies from the socket
const cookie = socket.request.headers.cookie;
const cookies = cookie.split(";");

// then we need to parse it
const tokenCookie = cookies
    .map((c: string) => Cookie.parse(c))
    .filter((c: Cookie) => c)
    .map((c: Cookie) => c.toJSON())
    .find((o: { key: string }) => o.key === "token");

// decode the token
let token = cookieParser.signedCookie(
    `${decodeURIComponent(tokenCookie.value)}`,
    config.secrets.cookie
);

token = token ? token : "";

// finally we can verify the token using a jwt library
try {
    jwt.verify(token, config.secrets.jwtAuth);
} catch (err) {
    return next(new Error("authentication error"));
}

return next();
```
