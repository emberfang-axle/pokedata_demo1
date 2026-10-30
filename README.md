# PokeData Demo – Pokémon Lookup (IT 126, Module 3, Lesson 4)

A Flutter app that retrieves Pokémon information from the public
[PokéAPI](https://pokeapi.co/) (`https://pokeapi.co/api/v2/pokemon/{name}`)
instead of using a JSON map embedded in the source code.

## Features
- Text field for entering a Pokémon name (case/whitespace-insensitive)
- HTTP `GET` using the `http` package; JSON decoded with `dart:convert`
- Displays name, ID, height, weight, abilities, types, base XP, held items, moves and sprite
- Loading indicator while the request is in progress
- Search button and text field are disabled while loading, so repeated
  submissions cannot produce overlapping or confusing results
- Clear error messages instead of crashes or invalid data

## Running
```
flutter pub get
flutter run
```

## How the app handles each outcome

The service (`PokemonService.fetchPokemon`) performs three separate checks and
reports each failure as its own exception type, so the UI can respond
appropriately.

### Successful requests
The server returns `200 OK`. The body is decoded with `jsonDecode`, verified
to be a JSON object, and converted into a `PokemonDetail`. Required fields
(`id`, `name`, `height`, `weight`) are validated; optional fields (abilities,
types, moves, sprite) fall back to safe empty values. The details are then shown.

### Unsuccessful HTTP responses
The request completed, but the server reported a failure. The status code is
checked **before** the body is processed:
- `404 Not Found` → `PokemonNotFoundException`: "No Pokémon named … was found."
  No retry button is shown because repeating the same request cannot succeed.
- Other non-200 codes → `HttpStatusException`: shows the status code. A retry
  button is offered only for `5xx` server errors.

Previously displayed data is cleared, so the error message is never shown
alongside stale or invalid information.

### Network failures
If no HTTP response can be obtained (device offline, DNS failure, connection
dropped, 10-second timeout), the app throws `NetworkException` and shows
"Could not reach the server…" with a **Retry** button. This differs from a 404
because the server never answered, so retrying later may well work.

### Invalid data
A `200` response whose body is not valid JSON, is not a JSON object, or lacks
required fields throws `InvalidDataException`. The app reports that it could
not understand the data and does not display any partially invalid values.

## Proposed offline strategy
(Not implemented in this activity; local storage is out of scope.)

1. **Cache on success.** After every successful lookup, store the decoded
   Pokémon (the JSON or the `PokemonDetail` fields) locally, keyed by name/ID,
   with a `lastFetched` timestamp. Suitable options are `shared_preferences`
   for a handful of entries, or a local database such as `sqflite`, `drift`
   or `hive` for many.
2. **Fall back to the cache on network failure only.** When a request throws a
   `NetworkException`, look the name up in the cache. If found, display it with
   a banner such as "Showing saved data from {date} – may be outdated" and a
   **Refresh** button. If not found, show the normal network error with Retry.
   A `404` should *not* fall back to the cache, since the server definitively
   said the Pokémon does not exist.
3. **Be honest about freshness.** Always label cached data as possibly outdated,
   and refresh the cache silently when connectivity returns.
4. **Don't rely on a connectivity flag alone.** Being on Wi-Fi does not
   guarantee the API is reachable. Treat the actual request outcome as the
   source of truth, using connectivity information only as a hint.
5. **Optionally cache sprites** (e.g. with `cached_network_image`) so images
   also display offline, and offer a "recently viewed" list backed by the cache.
