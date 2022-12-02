# XRTC API

## Table of contents

- [Registration](#registration)
- [Login](#login)
- [Set item](#set-item)
- [Get item](#get-item)
- [API conventions](#conventions)
- [Example with Curl](#use-with-curl)
- [Example payloads](#example-payloads)

## Registration

A two-step e-mail registration is required to get an account and an API key.

**Step 1**: Registration code will be sent to the provided email

Format specification:

 - Request URL: `https://api.xrtc.org/v1/auth/registrationstart`
 - Request body raw:
   - First request, sends code by email: `{"email":"yourname@validemail.com"}`
   - Re-send code: `{"email":"yourname@validemail.com", "registrationtoken":"abcdefghijklmopqrstu-1234567890"}`
 - Response: `{"registrationtoken":"abcdefghijklmopqrstu-1234567890"}`
 - Error reponse: `{"error":{"errorgroup":4,"errorcode":32,"errormessage":"RegistrationStart.processOperations: RegistrationStart.newRegistration: Invalid email address name.surname@baddomain.com"}}`
 - Typical errors:
   - INVALID_ARGUMENTS	= 30 (wrong argument name or type or value)
   - INVALID_EMAIL = 32 (non-existent email address)
   - MAX_CONCURRENT_REGISTRATIONS = 40 (registration tried too many times over a period of time, try again later)
   - MAX_CODE_SEND	= 41 (code has been re-sent too many times or too frequently, try again later)

**Step 2**: The generated registration token and the email code have to be provided within 5 minutes to obtain an account and an API key

Format specification:

 - Request URL: `https://api.xrtc.org/v1/auth/registrationcode`
 - Request body raw: `{"registrationtoken":"abcdefghijklmopqrstu-1234567890","emailcode":"123456"}`
 - Response: `{"accountid":"AC0987654321012345","apikey":"xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx","apikeyname":"exampleapikeyname"}`
 - Typical errors:
   - INVALID_ARGUMENTS	= 30 (wrong argument name or type or value)
   - INVALID_TOKEN	= 33 (wrong registration token given, no match to Step 1)
   - INVALID_CODE = 34 (user typed wrong code, try again up to 3 times)
   - MAX_CODE_RETRY = 42 (wrong code typed 3 times, start registration from scratch)
   - TIMEOUT_CODE = 43 (user was too slow to type the code, it expired. Start registration from scratch)

A new account is created together with the first API key. For an exisiting account, a new API key is added. There can be up to the maximum of API_KEY_COUNT_MAX used in parallel. Beyond this limit, the oldest API is overwitten by a new one. Key time validity is indefinite, but if the key is not used for longer than API_KEY_AGE_MAX, it is disabled. Save API keys securely.

 - Account id format (string): AC followed by a long number, e.g. AC0987654321012345
 - API key format (string): ASCII alphanumeric strings 32 characters long generated per FIPS-140-2 standard (~200 bit entropy)

## Login

Against `accountid` and `apikey`, successful login returns `JSESSIONID` cookie that has to be used in all subsequent requests (along with AWSALBAPP-0 cookie). If the cookie is not used for longer than 1 minute, login has to be repeated. The cookies cannot be re-used/shared across clients with different IPs. Too frequent logins (fater than once in LOGIN_TIMEOUT) for a single `apikey` are rejected for security reasons. This means that parallel set/get requests from one client should rely on a single `JSESSIONID` cookie per end client application. For a massively distributed application, contact support to generate bulk API keys and to lift the safety limits.

Format specification:

 - Request URL: `https://api.xrtc.org/v1/auth/login`
 - Request body raw: `{"accountid":"AC0987654321012345","apikey":"xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"}`
 - Response: `{"servertimestamp":1659716661000}`
 - Typical errors:
   - INVALID_ARGUMENTS	= 30 (wrong argument name or type or value)
   - INVALID_CREDENTIALS	= 35 (wrong account id and/or API key or disabled API key)
   - TOO_FREQUENT_LOGIN	= 45 (login called too often for this API key)

## Set item

Input: array of items `{"portalid":"abc", "payload":"xxx"}`.

Output: returns `{"servertimestamp":2134688"}`, where `servertimestamp` is the time stamp when the item arrived to the server in the server time frame.

`portalid` is an ASCII alphanumeric user-defined string of up to 32 characters.

`payload` is a UTF-8 string (must be escaped or URL-encoded) or a Base64 string (must be escaped) [see example paylod encodings](#example-payloads).

The maximum overall size of serialized json (including possible escaping) is limited to PAYLOAD_SIZE_MAX (otherwise request is rejected). Only the account id that created the portal can set items to and get items from it.

Portals and their content are ephimerial up to ITEM_AGE_MAX and PORTALS_COUNT_MAX. Internally the portal is organized as a ring buffer. If the get rate is systematically lower than the set rate, the oldest items will be overwritten.

Format specification:

 - Request URL: `https://api.xrtc.org/v1/item/set`
 - Request body raw: `{items:[{"portalid":"abc", "payload":"xxx"},{...}]}`
 - Response: `{"servertimestamp":2134688}`
 - Typical errors:
   - INPUT_BINARYJSON_PARSE_FAIL = 20 (wrong argument names, unescaped strings, brackets mismatch, MAX_PAYLOAD_SIZE exceeded)
   - INVALID_ARGUMENTS = 30 (wrong argument values)
   - SESSION_INVALIDATED	= 10011 (HTTP status code 401, "Server session denied" meaning wrong or expired [JSESSIONID cookie](#conventions))

## Get item

Input: array of portals with the reference timestamp `{portals:[{"portalid":"1234567890"}]}`

Output: returns array of items `{"portalid":"abc", "payload":"xxx", "servertimestamp":2134688"}`, where `servertimestamp` is the time stamp when the item arrived to the server in the server time frame.

The input logic what to get:

 - `portalid`:
   - if you provide an empty list `{portals:[]}` or null `{portals:null}`, it returns the latest data for _all_ portals of this `accountid`.
     - *This is useful to list of portals of this account and to check the lastest data in each of them for monitoring*
     - *If specific portals are not provided,* **none** *of the additional parameters are effective if specific portals are not provided*
   - if you provide one or several portals `{portals:[{"portalid":"abc111},{...}]}`, it returns the latest data for _the specified_ portals, since the last get item request within this login session.
     - *If specific portals are provided, additional parameters `mode`, `schedule`, `cutoff` can be used to fine tune the behavior*

  - `mode`:
    - `probe` (default) checks the latest data and returns imediately. If the data is not available, it returns an empty json.
      - *This mode is useful for occasional monitoring if there is anything new in a portal without waiting*
    - `watch` checks the latest data, and if it is not available, waits for it (in any of the portals) and then immediately returns. If the data is not available after GET_ITEM_TIMEOUT, it returns an empty json.
      - *This mode is useful for geting new data from a portal without complexities of streaming*
    - `stream` same as `watch`, but after returning one data it does not close the request and will keep waiting for new data and returning it continuisly, separated by `\n`, until GET_ITEM_TIMEOUT.
      - *This mode is useful for continuous data streaming*
      - *The use of an SDK is recommended (e.g. Python) because streaming functionality is technically not possible to use with Curl or Postman*

  - `schedule`:  
    - LIFO (default), defines delivery of the latest data first. Some older data can be skipped if newer data arrives, but both the newer and the older data do not fit the maximum overal size of the serialized json (see below).
    - FIFO, defines delivery of the oldest data first. The desire is to bring all data without ommitions, but pay attention that if the set item rate exceeds the get item rate, the circular buffer will overrun and thus start overwriting the oldest items.

  - `cutoff`:
    - defines time in ms for the maximum relative age of the items, default -1 (no effect).
      - *This is useful to discard old items*

The output logic what to get: if any of the following limits are exceeded, the exceeding items will be ignored (the oldest items are ignored for LIFO schedule, the latest items are ignored for FIFO schedule):
 - PAYLOAD_SIZE_MAX limits the maximum total size of serialized json including possible escaping
 - ITEM_AGE_MAX limits the maximum age of an item
 - ITEM_COUNT_MAX limits the maximum number of items

Format specification:

 - Request URL: `https://api.xrtc.org/v1/item/get`
 - Request body raw:
   - Get latest items from all portals whatever is there `{portals:[]}` or `{portals:null}` (empty list of portals or null)
   - Get the latest items from a portal whatever is there `{portals:[{"portalid":"abc111},{...}]}`
   - Stream FIFO from a portal: `{portals:[{"portalid":"abc111}],"mode":"stream","schedule":"FIFO"}`

 - Response: `{items:[{"portalid":"abc","payload":"xxx","servertimestamp":2134688},{...}]}` or `{}` (empty json) when there is no data
 - Typical errors:
   - INPUT_BINARYJSON_PARSE_FAIL = 20 (wrong argument names, unescaped strings, brackets mismatch, MAX_PAYLOAD_SIZE exceeded)
   - INVALID_ARGUMENTS = 30 (wrong argument values)
   - SESSION_INVALIDATED	= 10011 (HTTP status code 401, "Server session denied" meaning wrong or expired [JSESSIONID cookie](#conventions))

## Conventions
 - Encoding: `portalid` ASCII alphanumeric, `payload` UTF-8 (binary can be transmitted as Base64)
 - Request type: all HTTP requests are POST type
 - Timestamps: UNIX time in milliseconds UTC
 - URL format: `https://api.xrtc.org/v1/...`
 - Ping URL (icmp): `ping.xrtc.org`
 - Versioning: only major (not backward compatible) version is used as `.../v1/...`
 - Cookies: `JSESSIONID` and `AWSALBAPP-0` cookies must be supplied to each request other than registration or login
 - Endpoint limits:
   - API_KEY_COUNT_MAX = 10
   - API_KEY_AGE_MAX = 3 months
   - PORTALS_COUNT_MAX = 10
   - ITEM_AGE_MAX = 1h
   - PAYLOAD_SIZE_MAX = 1kB
   - GET_ITEM_TIMEOUT = 5s
   - LOGIN_TIMEOUT = 5s
 - Account usage limits:
   - 1 million requests per month (account-level limit)
   - 20 requests per second (API-key-level limit)
 - Errors:
   - HTTP status codes: all erorrs are HTTP 400 "Bad request", except HTTP 401 "Anauthorized" when server session/cookie is denied/invalidated/expired
   - XRTC error response syntax: `{"error":{"errorgroup":6,"errorcode":1,"errormessage":"Example error message"}}`
   - XRTC error groups: 2 - system errors, 3 - registration, 4 - login, 6 - application
   - XRTC error codes: see typical errors and their codes in each endpoint

## Use with CURL

Login (note the escaped quotes inside json):

`curl -c cookies.txt -d "{\"accountid\":"AC****",\"apikey\":\"****\"}" https://api.xrtc.org/v1/auth/login`

Set item (to load content from file, insted of -d option, use -F option):

Get item:

`curl -b cookies.txt -d "{portals:[{\"portalid\":\"send\",\"servertimestamp\":1664548042272}]}" https://api.xrtc.org/v1/item/get`

Set item:

`curl -b cookies.txt -d "{items:[{\"portalid\":\"send\",\"payload\":\"23658abc\"}]}" https://api.xrtc.org/v1/item/set`

or by uploading a file:

`curl -b cookies.txt -d @test.txt https://api.xrtc.org/v1/item/set`


Useful options:
 - `-v` verbose output for diagnostics
 - `--trace-time` event time stamps for disagnostics

## Example payloads

Original example payload: `{"myapplicationdata":{"sensor":"0001","temperature":"9000"}}`

Escaped example payload: `{\"myapplicationdata\":{\"sensor\":\"0001\",\"temperature\":\"9000\"}}`

URL-encoded example payload: `%7B%22myapplicationdata%22%3A%7B%22sensor%22%3A%220001%22%2C%22temperature%22%3A%229000%22%7D%7D`

Base64-encoded example payload: `eyJteWFwcGxpY2F0aW9uZGF0YSI6eyJzZW5zb3IiOiIwMDAxIiwidGVtcGVyYXR1cmUiOiI5MDAwIn19`

 - *Note on Base64 transport compatibility:* json serializer (e.g. gson) may replace padding (=) with unicode escaped "\u003d" (in fact, by [RFC4627](https://www.rfc-editor.org/rfc/rfc4627) section 2.5 "any character may be escaped" within a string). Hence, the returned payload string has to be first properly unescaped. Additionally, the client-side encoder may replace + and / with "URL and Filename safe" - and _ per [RFC4648](https://www.rfc-editor.org/rfc/rfc4648), to prevent json escaping / with "\\/" (for this e.g. Python `base64.urlsafe_b64decode` can be used).
