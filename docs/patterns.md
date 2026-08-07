# Patterns

An API facade can take a few shapes, depending on where the mess lives.

## Aggregating facade

Combines several services behind one call. Client calls one method, the facade fans out and merges the results.

## Normalizing facade

Translates vendor-shaped responses into your domain model. The client never sees `snake_case` or version quirks.

## Caching facade

Adds a cache in front of slow or rate-limited endpoints. The rest of the app keeps the same interface.

## Choosing

Start with a normalizing facade, add aggregation when call sites repeat the same fan-out, add caching last, when you have measured the cost.
