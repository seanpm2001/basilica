# Basilica

A basilica is like a forum, but for a few ill-defined differences. For more detail please consult the table below, adapted from a crude sketch I made while drunk.

Forum | Basilica
----: | :-------
PHP | Haskell
90s | 2010s
trolls | friends
"rich formatting" | markdown
paging | lazy tree
threads ↑ comments ↓ | uniform hierarchy
`<form>` | HTTP API
inline CSS | bots, webhooks, extensions
F5 | websockets

# Status

Basilica is still *remarkably unfinished*.

Small bits and pieces might happen to work here and there, but such behavior should be considered unusual.

# API

Basilica exposes a simple CRUD API, and is designed to be easy for computers to speak to. There are a few models that it speaks, always in JSON:

## Authentication

There's a goofy hand-rolled auth scheme.

There are no passwords. Authentication is done purely through email. The process looks this:

- request a code (see `POST /codes`)
- Basilica emails it to you
- you trade in the code for a token (see `POST /tokens`)
- you use that token to authenticate all future requests (details to follow)

This is similar to the "forgot my password" flow found in most apps, except that you don't have to pretend to remember anything.

## Models

### Post

```json
{ "id": 49
, "idParent": 14
, "by": "ian"
, "at": "2014-08-17T01:19:15.139Z"
, "count": 0
, "content": "any string"
, "children": []
}
```

- `id` is a monotonically increasing identifier, and *it is the only field that should be used for sorting posts*.
- `idParent` *might be `null`*. Root posts have no parents.
- `by` is currently just a string. Later it may be something else.
- `at` is a string representing the date that the post was created, in ISO 8601 format. This field exists to be displayed to the user; it should not be used for sorting or paging. Use `id` for that.
- `count` is the *total number of children that this post has*, regardless of the number of children returned in any response.
- `children` is a list of posts whose `idParent` is equal to this post's `id`. This is *not necessarily an exhaustive list*. Comparing the number of elements in this field to the `count` field can tell you if there are more children to load.
    - `children` will *always* be sorted by `id`, with newer posts (larger `id`s) in the front of the list

### User

```json
{ "id": 32
, "email": "name@example.com"
, "face": {}
}
```

- `face` is an object that indicates how to render a thumbnail of the user. Currently the only valid options are:
    - `{ "gravatar": "a130ced3f36ffd4604f4dae04b2b3bcd" }`
    - `{ "string": "☃" }`
    - **not implemented**
- there'll be more fields later, probably

### Token

```json
{ "id": 91
, "token": "a long string"
, "idUser": 32
}
```

## Routes

### `POST /posts/:idParent`

- for: creating a new post as a child of the specified `idParent`
- `idParent` is optional. If ommitted, this will create a post with `idParent` set to `null`.
- arguments: an `x-www-form-urlencoded` body is expected with
    - `by` (any string)
        - required
        - when accounts are implemented, this will be restricted
    - `content` (any string)
        - required
        - must not be the empty string
- response: the newly created post, JSON-encoded
    - if the post has a `count` other than `0`, that's a bug
    - the post will not have `children`
- note: eventually this will require a token and not take a `by` field, but that is **not implemented**

### `GET /posts/:id`

- for: loading posts and post children
- arguments: query parameters
    - `depth`: how deeply to recursively load `children`
        - **not implemented**
        - default: `1`
        - if `0`, the response will not include `children` at all
        - valid values: just `0` and `1` right now
    - `after`: the `id` of a post
        - **not implemented**
        - optional
        - ignored if `depth` is `0`
        - the response will not include any posts created before this in the `children` list (recursively, if multiple depths are ever supported)
    - `limit`: the maximum number of `children` to load
        - **not implemented**
        - default: `50`
        - ignored if `depth` is `0`
        - valid values: `1` to `500`
        - applies recursively, if multiple depths are ever supported
- response: a JSON-encoded post
    - if `depth` is greater than `0`, it will include `children`
    - remember that `count` is always the *total* number of children, regardless of the `limit`

### `GET /posts`

- for: loading every single post in the entire database, catching up after a disconnect (with `after`)
- arguments: query parameters
    - `after`: the `id` of a post
        - optional
        - the response will only contain posts created after the specified post
    - `limit`: the maximum number of posts to return
        - **not implemented**
        - default: `50`
        - valid values: `1` to `500`
- response:
    - if `after` is specified, and there were more than `limit` posts to return, this returns... some error code. I'm not sure what though. `410`, maybe?
        - **not implemented**
    - otherwise, a JSON array of posts with no `children` fields, sorted by `id` from newest to oldest

### `POST /codes`

- for: creating a new code, which can be used to obtain a `token`
- arguments:
    - `email`: the email address of the user for which you would like to create a code
- response: this route will always return an empty response body with a `200` status code, regardless of whether `email` corresponds to a valid email address
    - a timing attack can absolutely be used to determine if the email corresponds to a valid account or not; knock yourself out
- note: if the `email` corresponds to a `user`, this route has the side effect of emailing the user a `code` that can be used to redeem a `token`
- **not implemented**

### `DELETE /codes/:code`

- for: revoking a code, in case it was sent in error
- **not implemented**
- or documented

### `POST /tokens`

- for: creating a new token
- arguments:
    - `code`: a code obtained from a call to `POST /codes`
        - required
- note: auth tokens don't do anything yet
- response:
    - if the code is valid, a JSON-encoded token
    - otherwise, `401`
- side effect: invalidates the `code` specified
- **not implemented**

### `GET /tokens`

- for: listing tokens
- response: an array of JSON-encoded token objects with only `id` specified
    - probably other stuff later
- **not implemented**

### `DELETE /tokens/:id`

- for: revoking a token ("logging out")
- arguments:
    - `id`: the `id` of the token to revoke
        - required
- response: `200`, `404`, or `401`
- **not implemented**

## Known clients

- [browser client](https://github.com/ianthehenry/basilica-client), nowhere near finished

# Development

Right now it uses SQLite. You need to create the database.

    $ sqlite3 basilica.db ".read schema.sql"

And install the dependencies.

    $ cabal install --only-dependencies -j

Then you can run it.

    $ cabal run
