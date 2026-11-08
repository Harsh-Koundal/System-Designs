# Bloom Filter System Design
<img width="2188" height="1395" alt="BloomFilter_systemDesign" src="https://github.com/user-attachments/assets/8f72938c-d11e-47ed-a81f-c1447614b8c4" />

 
## 1. Overview

This document describes a username-existence checking service built with
**Node.js**, **Express.js**, and **RedisBloom**.

The service uses a Redis Bloom Filter to quickly determine whether a
username **might exist** in the filter or is **definitely absent**. It
exposes two HTTP endpoints:

-   `POST /api/bloom-filter` --- add a username to the Bloom Filter.
-   `GET /api/bloom-filter/:username` --- check a username against the
    Bloom Filter.

The main goals are:

-   **Fast membership checks:** use RedisBloom for efficient
    probabilistic lookups.
-   **Memory efficiency:** represent membership using a bit array rather
    than storing a retrievable list of usernames in the filter.
-   **Simple integration:** expose the Bloom Filter through REST APIs.
-   **Consistent input handling:** trim whitespace and normalize
    usernames to lowercase before processing.

> A Bloom Filter is a probabilistic data structure. A negative result
> means the item is definitely not present in the filter; a positive
> result means it might be present and can be a false positive.

## 2. Architecture at a glance

``` text
                    User / Postman / HTTP Client
                              |
                +-------------+-------------+
                |                           |
                v                           v
       POST /api/bloom-filter     GET /api/bloom-filter/:username
          (add username)              (check username)
                |                           |
                +-------------+-------------+
                              |
                              v
                   Node.js + Express Server
                              |
                       Express Router
                              |
                     express.json() middleware
                              |
                   +----------+----------+
                   |                     |
                   v                     v
          AddUsername Controller   CheckUsername Controller
                   |                     |
          Validate request body   Validate route parameter
          Normalize username      Normalize username
                   |                     |
          Check filter key        BF.EXISTS
          BF.RESERVE if absent          |
          BF.ADD                        |
                   |                     |
                   +----------+----------+
                              |
                              v
                       RedisBloom
                    Key: users:bloomfilter
                              |
                              v
                    JSON response to client
```

The Express application receives requests, routes them to the
appropriate controller, and the controller executes RedisBloom commands
against the configured Redis key.

## 3. Main components

### 3.1 Client

The client can be Postman, a frontend application, or another HTTP
client.

It can:

-   Submit a username for insertion.
-   Check whether a username might exist.
-   Read the JSON response returned by the API.

### 3.2 Node.js and Express server

The backend runs on Node.js and uses Express.js to handle HTTP requests.

Responsibilities:

1.  Initialize the Express application.
2.  Configure JSON request-body parsing with `express.json()`.
3.  Connect to Redis before accepting requests.
4.  Register the Bloom Filter routes.
5.  Pass requests to the relevant controller.

The Redis connection is managed through the Redis configuration module.
The application should establish the connection during server startup.

### 3.3 Express routes and middleware

The router maps the HTTP method and URL to a controller function.

  -------------------------------------------------------------------------------------
  Method            Endpoint                        Controller        Purpose
  ----------------- ------------------------------- ----------------- -----------------
  `POST`            `/api/bloom-filter`             `addUsername`     Add a username to
                                                                      the filter

  `GET`             `/api/bloom-filter/:username`   `checkUsername`   Check possible
                                                                      membership
  -------------------------------------------------------------------------------------

The `express.json()` middleware parses JSON request bodies so that the
POST controller can read `req.body`.

### 3.4 AddUsername controller

The `addUsername(req, res)` controller adds a username to the Bloom
Filter.

Responsibilities:

1.  Read `username` from `req.body`.
2.  Validate that the username is a non-empty string.
3.  Trim whitespace and convert the username to lowercase.
4.  Check whether the Redis key `users:bloomfilter` exists.
5.  If the key is absent, create the filter using `BF.RESERVE`.
6.  Insert the normalized username using `BF.ADD`.
7.  Return a JSON response describing the insertion result.

Redis commands used:

``` redis
EXISTS users:bloomfilter
BF.RESERVE users:bloomfilter 0.01 100000
BF.ADD users:bloomfilter harsh
```

`BF.RESERVE` creates a Bloom Filter with a configured false-positive
rate and expected capacity. `BF.ADD` adds an item to the filter.

### 3.5 CheckUsername controller

The `checkUsername(req, res)` controller checks whether a username might
be present in the filter.

Responsibilities:

1.  Read `username` from `req.params`.
2.  Validate that the parameter is present and non-empty.
3.  Trim whitespace and convert the username to lowercase.
4.  Execute `BF.EXISTS` against `users:bloomfilter`.
5.  Convert the Redis result into a boolean.
6.  Return a JSON response containing `mightExist`.

Redis command used:

``` redis
BF.EXISTS users:bloomfilter harsh
```

Interpretation:

-   `1` --- the username might exist in the Bloom Filter.
-   `0` --- the username is definitely absent from the Bloom Filter.

A positive result is not proof that the username exists in a primary
database.

### 3.6 Redis and RedisBloom

Redis stores the Bloom Filter data structure under this key:

``` text
users:bloomfilter
```

The filter is a probabilistic bit array. It does not retain usernames in
a form that can be listed or retrieved.

The implementation uses these RedisBloom commands:

  -----------------------------------------------------------------------
  Command                             Purpose
  ----------------------------------- -----------------------------------
  `BF.RESERVE`                        Create a Bloom Filter with the
                                      chosen error rate and capacity

  `BF.ADD`                            Add an item to the Bloom Filter

  `BF.EXISTS`                         Check whether an item might exist
                                      in the filter
  -----------------------------------------------------------------------

The Redis server must have the **RedisBloom module** available for these
commands.

## 4. Bloom Filter configuration

The filter is configured with the following values:

  ------------------------------------------------------------------------
  Setting                                      Value Meaning
  --------------------- ---------------------------- ---------------------
  Redis key                      `users:bloomfilter` Key used to store the
                                                     filter

  False-positive rate                    `0.01` (1%) Target probability of
                                                     a false positive

  Expected capacity                        `100,000` Expected number of
                                                     inserted items
  ------------------------------------------------------------------------

These are configuration targets, not a guarantee that every lookup will
be correct. A Bloom Filter can return false positives, but under normal
operation it does not return false negatives for items successfully
added to the same filter.

## 5. API documentation

### 5.1 Add username

**Endpoint**

``` http
POST /api/bloom-filter
Content-Type: application/json
```

**Request body**

``` json
{
  "username": "Harsh"
}
```

The controller normalizes the value to `harsh` before inserting it.

**Example success response**

``` json
{
  "success": true,
  "username": "harsh",
  "added": true,
  "message": "Username added to Bloom Filter"
}
```

If RedisBloom reports that the item may already be present, `added` can
be `false`. This does not prove that the username was previously
inserted because Bloom Filters can produce false positives.

**Validation response**

``` json
{
  "success": false,
  "message": "Username is required"
}
```

The API should return HTTP `400` for invalid or empty usernames and HTTP
`500` when an unexpected server-side error prevents processing.

### 5.2 Check username

**Endpoint**

``` http
GET /api/bloom-filter/:username
```

**Example request**

``` http
GET /api/bloom-filter/Harsh
```

**Example positive response**

``` json
{
  "success": true,
  "username": "harsh",
  "mightExist": true,
  "message": "Username might exist"
}
```

**Example negative response**

``` json
{
  "success": true,
  "username": "unknown",
  "mightExist": false,
  "message": "Username definitely does not exist"
}
```

The `mightExist` field represents Bloom Filter membership, not verified
existence in an authoritative user database.

## 6. End-to-end request flows

### 6.1 Add username flow

``` text
Client
  |
  v
POST /api/bloom-filter
  |
  v
Express JSON middleware
  |
  v
AddUsername controller
  |
  v
Read and validate req.body.username
  |
  v
Trim and lowercase username
  |
  v
Check whether users:bloomfilter exists
  |
  +---- Key absent? ---- Yes ----> BF.RESERVE
  |                                  |
  +----------------------------------+
  |
  v
BF.ADD users:bloomfilter <username>
  |
  v
Return JSON insertion result
  |
  v
Client
```

### 6.2 Check username flow

``` text
Client
  |
  v
GET /api/bloom-filter/:username
  |
  v
Express router
  |
  v
CheckUsername controller
  |
  v
Read and validate req.params.username
  |
  v
Trim and lowercase username
  |
  v
BF.EXISTS users:bloomfilter <username>
  |
  v
Interpret Redis result
  |
  +---- 1 ----> mightExist: true
  |
  +---- 0 ----> mightExist: false
  |
  v
Return JSON response
  |
  v
Client
```

## 7. Docker deployment and persistence

The supplied architecture shows Redis running as a Docker Compose
service, with a named volume for Redis data.

The logical deployment components are:

-   **Docker Compose:** starts and manages the Redis service.
-   **Redis with RedisBloom:** provides Redis storage and Bloom Filter
    commands.
-   **Named volume (`redis-data`):** persists Redis data outside the
    container's writable layer.

A representative Compose configuration is:

``` yaml
services:
  redis:
    image: <redis-image-with-redis-bloom>
    ports:
      - "6379:6379"
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis-data:/data

volumes:
  redis-data:
```

Use an image that actually includes or loads the RedisBloom module. The
placeholder above must be replaced with the image selected by the
project. A plain Redis image without RedisBloom will not recognize
`BF.RESERVE`, `BF.ADD`, or `BF.EXISTS`.

Append-only persistence helps Redis recover writes after a restart. The
named volume must be retained if the filter data should survive
container replacement.

## 8. Project structure

The key files shown in the project design are:

``` text
BloomFilter/
├── config/
│   └── redisConfig.js
├── controllers/
│   └── bloomFilterController.js
├── routes/
│   └── usernameRoute.js
├── server.js
├── docker-compose.yml
└── package.json
```

  ----------------------------------------------------------------------------
  File                                     Responsibility
  ---------------------------------------- -----------------------------------
  `config/redisConfig.js`                  Creates and exports the Redis
                                           client and connection function

  `controllers/bloomFilterController.js`   Implements username insertion and
                                           membership-check logic

  `routes/usernameRoute.js`                Defines the Bloom Filter API routes

  `server.js`                              Configures Express, connects Redis,
                                           registers routes, and starts the
                                           server

  `docker-compose.yml`                     Defines the Redis container and
                                           persistent volume

  `package.json`                           Stores project dependencies and
                                           scripts
  ----------------------------------------------------------------------------

## 9. Error handling and operational considerations

The controllers should handle expected and unexpected errors
consistently.

  -----------------------------------------------------------------------
  Scenario                            Expected behavior
  ----------------------------------- -----------------------------------
  Username missing from POST body     Return HTTP `400`

  Username is not a string            Return HTTP `400`

  Username contains only whitespace   Return HTTP `400`

  Redis connection is unavailable     Return an appropriate server error
                                      and log the failure

  RedisBloom command fails            Return an appropriate server error
                                      and log the failure

  Bloom Filter reports `0`            Return `mightExist: false`

  Bloom Filter reports `1`            Return `mightExist: true`
  -----------------------------------------------------------------------

Operational considerations:

-   Connect to Redis before starting the HTTP server.
-   Ensure the Redis deployment has RedisBloom enabled.
-   Initialize the filter safely. Checking for a key and then reserving
    it in every POST request can create a race if multiple requests
    arrive at the same time. Prefer controlled initialization during
    startup or handle the already-created-key case.
-   Do not treat the Bloom Filter as the source of truth for account
    registration.
-   If the application has a user database, use a positive Bloom Filter
    result as a reason to perform an authoritative database lookup.
-   Enforce a unique constraint in the user database when usernames must
    be unique.

## 10. Testing strategy

The API can be tested manually with Postman and automatically with Jest
and Supertest.

Suggested test cases:

### Add username API

-   Add a valid username.
-   Add a username containing uppercase letters and verify
    normalization.
-   Reject a missing username.
-   Reject a non-string username.
-   Reject a whitespace-only username.
-   Handle Redis errors.

### Check username API

-   Check a username that was added.
-   Check a username that was not added.
-   Check usernames with different letter casing.
-   Reject an empty or invalid username parameter.
-   Handle Redis errors.
-   Verify that a positive result is described as "might exist," not as
    confirmed existence.

For predictable tests, use an isolated Redis instance or a test filter
key and clean up test data between test cases. Mock Redis commands when
testing controller behavior independently of Redis.

## 11. Limitations and trade-offs

### Probabilistic results

A Bloom Filter can return a false positive: it may report that an item
might exist even though that item was never added.

A negative result means the item is definitely absent from the filter,
assuming the filter has not been reset or corrupted.

### No username retrieval

A Bloom Filter does not provide a way to list or retrieve the original
usernames. It supports membership checks, not reverse lookup.

### No standard deletion

A standard Bloom Filter does not support safe removal of individual
items. Deletion requires a different structure, such as a Counting Bloom
Filter, or rebuilding the filter.

### Capacity and false-positive rate

The expected capacity and false-positive target affect memory usage. If
the number of inserted items grows beyond the planned capacity, the
actual false-positive rate can increase. RedisBloom can use scalable
filters when configured for that purpose, but the current design uses
the `BF.RESERVE` configuration shown above.

## 12. Recommended implementation sequence

1.  Configure Redis with the RedisBloom module.
2.  Create the Redis client and connection configuration.
3.  Initialize the Express application and JSON middleware.
4.  Create the POST and GET routes.
5.  Implement username validation and normalization.
6.  Implement filter initialization and `BF.ADD`.
7.  Implement `BF.EXISTS` and the membership response.
8.  Test valid, invalid, positive, and negative cases.
9.  Test Redis connection failures and command errors.
10. Verify Docker persistence across container restarts.

## 13. Summary

The Bloom Filter service exposes a small REST API for adding usernames
and checking possible membership. Express handles HTTP routing and
validation, controllers contain the application logic, and RedisBloom
maintains the probabilistic data structure under the key
`users:bloomfilter`.

The design uses a target false-positive rate of 1% and an expected
capacity of 100,000 items. It is useful as a fast membership pre-check,
but it should not replace an authoritative user database when a
confirmed username-existence decision is required.

**Key principle:** A Bloom Filter can tell you that an item is
definitely absent or might be present; it cannot prove a positive result
or retrieve the original item.
