smoke.sh
========

A minimal smoke testing framework in Bash.

Features:

- Response body checks
- Response code checks
- Response header checks
- GET/POST/OPTIONS on endpoints
- ORIGIN support for testing CORS responses
- CSRF tokens
- Reporting and sane exit codes

![smoke sh](https://f.cloud.github.com/assets/657357/1238166/6f47f56a-29e4-11e3-9e19-394ca12b5fd0.png)

Example
-------

Checking if the Google Search home page works and contains the word "search":

```bash
#!/bin/bash

. smoke.sh

smoke_url_ok "http://google.com/"
    smoke_assert_body "search"
smoke_report
```

Running:

```bash
$ ./smoke-google
> GET http://google.com/
    [ OK ] 2xx Response code
    [ OK ] Body contains "search"
OK (2/2)
```

For a more advanced and complete example, see below.

Setup and usage
--------------

The recommended setup includes copying the `smoke.sh` file in the appropriate
place and creating a new file in the same directory that you will write your
tests in.

```bash
 $ tree -n
 .
 ├── smoke-google
 └── smoke.sh

```

In your file containing the tests, start with sourcing the `smoke.sh` file and
end with calling `smoke_report` if you want a final report + appropriate exit
code.

```bash
#!/bin/bash

. smoke.sh

# your test assertions go here

smoke_report
```

### GET a URL and check the response code

The minimal smoke test will check if a URL returns with a 200 response code:

```bash
smoke_url_ok "http://google.com"
```

It is also possible to check for other response codes explicitly:

```bash
smoke_url "http://google.com/doesnotexist"
smoke_assert_code 404
```

### GET a URL and check for redirects

In order to check for redirects, you must call `smoke_no_follow` before calling `smoke_url`:

```bash
smoke_no_follow
smoke_url "http://google.com"
smoke_assert_code 302
```

You can follow redirects again by calling `smoke_follow`

### POST a URL and check the response code

A more advanced smoke test will POST data to a URL. Such a test can be used to
for example check if the login form is functional:

```bash
smoke_form_ok "http://example.org/login" path/to/postdata
```

And the POST data (`path/to/postdata`):
```
username=smoke&password=test
```

### Checking if the response body contains a certain string

By checking if the response body contains certain strings you can rule out that
your server is serving a `200 OK`, which seems fine, while it is actually
serving the apache default page:

```bash
smoke_assert_body "Password *"
```

### Checking if the response headers contain a certain string

By checking response headers, you can make sure to get the correct content type:

```bash
smoke_assert_headers "Content-Type: text/html; charset=utf-8"
```

### Checking the server is not responding

In order to check a server is not responding, you can use `smoke_assert_no_response` after calling `smoke_url`:

```bash
smoke_url "http://myserver.com:5000/"
smoke_assert_no_response
```

### Configuring a base URL

It is possible to setup a base URL that is prepended for each URL that is
requested.

```bash
smoke_url_prefix "http://example.org"
smoke_url_ok "/"
smoke_url_ok "/login"
```

### Overriding the host

If the server requires a certain host header to be set, override the host from the URL with

```bash
smoke_host "example.org"
```

To un-override, set it empty:

```bash
smoke_host ""
```

### Overriding request headers

It's possible to set additional request headers, like `X-Forwarded-Proto` for local tests.

```
smoke_header "X-Forwarded-Host: orginal.example.org"
smoke_header "X-Forwarded-Proto: https"
```

Existing custom headers can be unset with `remove_smoke_headers`.

### Checking CORS is enabled for a certain Origin

First of all, set the origin header with:

```
smoke_origin "https://acme.corp"
```

Then test for CORS headers using:

```
smoke_url_cors "https://api.com/endpoint"
    smoke_assert_headers "Access-Control-Allow-Credentials: true"
    smoke_assert_headers "Access-Control-Allow-Origin: https://acme.corp"
```

### CSRF tokens

Web applications that are protected with CSRF tokens will need to extract a
CSRF token from the responses. The CSRF token will then be used in each POST
request issued by `smoke.sh`.

Setup an after response callback to extract the token and set it. Example:

```bash
#!/bin/bash

. smoke.sh

_extract_csrf() {
    CSRF=$(smoke_response_body | grep OUR_CSRF_TOKEN | grep -oE "[a-f0-9]{40}")

    if [[ $CSRF != "" ]]; then
        smoke_csrf "$CSRF" # set the new CSRF token
    fi
}

SMOKE_AFTER_RESPONSE="_extract_csrf"
```

When the CSRF token is set, `smoke.sh` will replace the string
`__SMOKE_CSRF_TOKEN__` in your post data with the given token:

```
username=smoke&password=test&csrf=__SMOKE_CSRF_TOKEN__
```

To get data from the last response, three helper functions are available:

```bash
smoke_response_code    # e.g. 200, 201, 400...
smoke_response_body    # raw body (html/json/...)
smoke_response_headers # list of headers
```

### Authentication

If the server requires an authentication (for example : HTTP Basic authentication), you must call `smoke_credentials` before calling `smoke_url`.
If you simply specify the user name, you will be prompted for a password. 

```bash
smoke_credentials "username" "password"
smoke_url "http://secured-website.com"
```

To un-set credentials, call `smoke_no_credentials` :

```bash
smoke_no_credentials
```

### Debugging

In order to debug your requests, call `smoke_debug` before calling `smoke_url`:

```bash
smoke_debug
smoke_url_ok "http://google.com"
```

You can turn off debugging by calling `smoke_no_debug`

Advanced example
----------------

More advanced example showing all features of `smoke.sh`:

```bash
#!/bin/bash

BASE_URL="$1"

if [[ -z "$1" ]]; then
    echo "Usage:" $(basename $0) "<base_url>"
    exit 1
fi

. smoke.sh

_extract_csrf() {
    CSRF=$(smoke_response_body | grep OUR_CSRF_TOKEN | grep -oE "[a-f0-9]{40}")

    if [[ $CSRF != "" ]]; then
        smoke_csrf "$CSRF" # set the new CSRF token
    fi
}

SMOKE_AFTER_RESPONSE="_extract_csrf"

smoke_url_prefix "$BASE_URL"
smoke_host "example.org"

smoke_url_ok "/"
    smoke_assert_body "Welcome"
    smoke_assert_body "Login"
    smoke_assert_body "Password"
    smoke_assert_headers "Content-Type: text/html; charset=utf-8"
smoke_form_ok "/login" postdata/login
    smoke_assert_body "Hi John Doe"
    smoke_assert_headers "Content-Type: text/html; charset=utf-8"
smoke_report
```

API
---

| function                                | description                                                                                    |
|-----------------------------------------|------------------------------------------------------------------------------------------------|
|`smoke_assert_body <string>`             | assert that the body contains `<string>`                                                       |
|`smoke_assert_code <code>`               | assert that there was a `<code>` response code                                                 |
|`smoke_assert_code_ok`                   | assert that there was a `2xx` response code                                                    |
|`smoke_assert_headers <string>`          | assert that the headers contain `<string>`                                                     |
|`smoke_assert_no_response`               | assert that the server is not responding                                                       |
|`smoke_credentials <string> [<string>]`  | set the credentials to use : login (and password). If password is not set, it will be prompted |
|`smoke_csrf <token>`                     | set the csrf token to use in POST requests                                                     |
|`smoke_form <url> <datafile>`            | POST data on url                                                                               |
|`smoke_form_ok <url> <datafile>`         | POST data on url and check for a `2xx` response code                                           |
|`smoke_origin <origin>`                  | set the `Origin` header                                                                        |
|`smoke_proxy <proxy>`                    | set the HTTP proxy to use [protocol://][user:password@]proxyhost[:port]                        |
|`smoke_no_proxy [<no-proxy-list>]`       | Comma-separated  list of hosts which do not use a proxy                                        |
|`smoke_report`                           | prints the report and exits                                                                    |
|`smoke_response_body`                    | body of the last response                                                                      |
|`smoke_response_code`                    | code of the last response                                                                      |
|`smoke_response_headers`                 | headers of the last response                                                                   |
|`smoke_url <url>`                        | GET a url                                                                                      |
|`smoke_url_ok <url>`                     | GET a url and check for a `2xx` response code                                                  |
|`smoke_url_prefix <prefix>`              | set the prefix to use for every url (e.g. domain)                                              |
|`smoke_host <host>`                      | set the host header to use                                                                     |
|`smoke_header <header>`                  | set additional request header (in the form `Key: value`)                                       |
|`remove_smoke_headers`                   | remove all headers                                                                             |
|`smoke_tcp_ok <host> <port>`             | open a tcp connection and check for a `Connected` response                                     |