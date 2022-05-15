# WhiteSource Home Assignment

This is a CRUD app for movies.

## Launching and making requests

Launching the project requires **Docker** to be installed.
Also, requests in the example section are made using [curlie](https://github.com/rs/curlie),
so you may want to install it too (or use [httpie](https://httpie.io/), they have a similar syntax)

Run

```shell
docker compose up
```

to build and run the app and databases.
You can select with which database to run by setting `MYAPP_DB`
variable in `.env` file; valid values are `mongo` and `postgres`.

To run tests:

```shell
go test ./...
```

### Examples

#### Ping

```shell
$ curlie -k https://0.0.0.0:443/ping 
HTTP/2 200 
content-type: application/json; charset=utf-8
content-length: 16
date: Sat, 14 May 2022 22:06:47 GMT

{
    "message": "OK"
}
```

#### Insert a movie

```shell
$ curlie -k POST https://0.0.0.0:443/v1/movie title=Titanic year:=1997 country=USA 
HTTP/2 201 
content-type: application/json; charset=utf-8
content-length: 125
date: Sat, 14 May 2022 23:05:59 GMT

{
    "title": "Titanic",
    "year": 1997,
    "country": "USA",
    "id": "628035d74fb8f0fa0bf7b2f7",
    "created_at": "2022-05-14T23:05:59.954Z"
}
```

#### Find a movie

```shell
$ curlie -k GET https://0.0.0.0:443/v1/movie/628035d74fb8f0fa0bf7b2f7             
HTTP/2 200
content-type: application/json; charset=utf-8
content-length: 119
date: Sat, 14 May 2022 23:06:19 GMT

{
"title": "Titanic",
"year": 1997,
"country": "USA",
"id": "628035d74fb8f0fa0bf7b2f7",
"created_at": "2022-05-14T23:05:59.954Z"
}

```

#### Update a movie

```shell
$ curlie -k PUT https://0.0.0.0:443/v1/movie/628035d74fb8f0fa0bf7b2f7 title=Amelie year:=2001 country=France 
HTTP/2 200 
content-type: application/json; charset=utf-8
content-length: 16
date: Sat, 14 May 2022 23:32:32 GMT

{
    "message": "OK"
}
```

```shell
$ curlie -k GET https://0.0.0.0:443/v1/movie/628035d74fb8f0fa0bf7b2f7 
HTTP/2 200 
content-type: application/json; charset=utf-8
content-length: 121
date: Sat, 14 May 2022 23:33:28 GMT

{
    "title": "Amelie",
    "year": 2001,
    "country": "France",
    "id": "628035d74fb8f0fa0bf7b2f7",
    "created_at": "2022-05-14T23:05:59.954Z"
}
```

#### Delete a movie

```shell
$ curlie -k DELETE https://0.0.0.0:443/v1/movie/628035d74fb8f0fa0bf7b2f7 
HTTP/2 204 
content-type: application/json; charset=utf-8
date: Sat, 14 May 2022 23:34:35 GMT

```

```shell
$ curlie -k GET https://0.0.0.0:443/v1/movie/628035d74fb8f0fa0bf7b2f7 
HTTP/2 404 
content-type: application/json; charset=utf-8
content-length: 27
date: Sat, 14 May 2022 23:34:51 GMT

{
    "message": "no such movie"
}
```

## Project structure

The application is designed to follow the principles of DDD and inversion of
control, i.e. we have our domain layer in `movie` and infrastructure layers
in other packages, which depend on entities from `movie` and not vice versa.

```shell
├── cmd
├── internal
│   ├── movie # business logic/domain layer
│   ├── rest # HTTP layer
│   └── storage # DB layer
│       ├── mongo
│       └── postgres 
└── pkg 
    └── must # just some utility functions 
```

## Some decisions I've made

1. `PUT` doesn't create new objects and can only update. It could've been
   implemented as an `upsert`, but I thought that this was
   an unnecessary over-complication, especially given that
   IDs for objects are expected to be generated by the database
   and not the client.
2. Every layer contains its own model, which is an overkill
   for this particular case as the application contains
   almost no business logic, but I believe this approach
   works well with services that are even a bit larger,
   because it allows for more decoupling.
   This is essentially a DTO pattern (but I think that
   names like `movieJSON` are more idiomatic to Go than `movieDTO`,
   which some people use)
3. In a real application I would store config files/values
   in a separate directory and source them using Viper,
   or at read them as CLI arguments or environment variables, but here
   all the values (ports, URLs, etc.) are just hardcoded.
   In production, I wouldn't do that, but here it's OK, I guess :)
4. I have an `ErrWrongIDFormat` which is returned when an ID
   does not conform to Mongo `ObjectID` or `UUID` for Postgres
   format, but when this error occurs, the user only receives
   a `404 Not Found` instead of some other error.
   I believe it's a good decision, because ID format is
   an implementation detail that doesn't matter to the user,
   but having an error that tells exactly what happened may
   come very useful when debugging or inspecting logs.
5. I've used [moq](https://github.com/matryer/moq) for generating
   the movie repository mock, because it makes writing tests
   easier (you could read more about it in the
   [author's post](https://medium.com/@matryer/meet-moq-easily-mock-interfaces-in-go-476444187d10#.uy9qkloty))