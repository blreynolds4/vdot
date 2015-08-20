# go-rest-api-template

*WORK IN PROGRESS*

Reusable template for building REST Web Services in Golang. Uses gorilla/mux as a router/dispatcher and Negroni as a middleware handler.

## Introduction

### Why?

After writing many REST APIs with Java Dropwizard, Node.js/Express and Go, I wanted to distill my lessons learned into a reusable template for writing REST APIs, in the Go language (my favourite).

It's mainly for myself. I don't want to keep on reinventing the wheel and just want to get the foundation of my REST API 'ready to go' so I can focus on the business logic and integration with other systems and data stores.

Just to be clear: this is not a freameowrk, library, package or anything like that. This tries to use a couple of very good Go packages and libraries that I like and cobbled together.

The main ones are:
* [gorilla/mux](http://www.gorillatoolkit.org/pkg/mux) for routing
* [negroni](https://github.com/codegangsta/negroni) as a middleware handler
* [testify](https://github.com/stretchr/testify) for writing easier test assertions
* [godep](https://github.com/tools/godep) for dependency management
* [render](https://github.com/unrolled/render) for HTTP response rendering

Whilst working on this, I've tried to write up as much as my thought process as possible, everything from the design of the API and routes, some details of the Go code like JSON formatting in structs and my thoughts on testing. However, if you feel that there is something missing, send a PR, raise an issue or contact me on twitter [@leeprovoost](https://twitter.com/leeprovoost).

### Knowledge of Go

If you're new to programming in Go, I would highly recommend you to read the following two resources:

* [A Tour of Go](https://tour.golang.org/welcome/1)
* [Effective Go](https://golang.org/doc/effective_go.html)

You can work your way through those in two to three days. I wouldn't advise buying any books right now. I unfortunately did and I have yet to open them. If you really want to get some books, there is a [github](https://github.com/dariubs/GoBooks) repo that tries to list the main ones.

### Development tools

TO DO SublimeText, GoSublime, etc.

### Manage dependencies

In order to manage package dependencies, we're using the [Godep](https://github.com/tools/godep) package tool.

Install Godep on your system:

```
go get github.com/tools/godep
```

When you add new packages, just use the standard `go get package` first, edit your code, then run the `godep save`. It will add an extra entry to `Godeps/Godeps.json`.

TO DO add some thoughts on Go 1.5 vendoring

### Live Code Reloading

TO DO

### How to run

`go run main.go` works fine if you have a single file you're working on, but once you have multiple files you'll have to start using the proper go build tool and run the compiled executable.

```
go build && ./go-rest-api-template
```

## Code deep dive

### Code Structure

Main server file (bootstrapping of http server and router):

```
main.go
```

Route handlers:

```
handlers.go
```

Data model descriptions:

```
passport.go
user.go
```

Mock database with operations:

```
database.go
```

Tests:

```
database_test.go
```

Third-party packages:

```
/Godeps
```

### main.go

TO DO

### Data model

We are going to use a travel Passport for our example. I've chosen Id as the unique key for the passport because (in the UK), passport book numbers these days have a unique 9 character field length (e.g. 012345678). A passport belongs to a user and a user can have one or more passports.

```
type User struct {
  Id              int    `json:"id"`
  FirstName       string `json:"first_name"`
  LastName        string `json:"last_name"`
  DateOfBirth     string `json:"date_of_birth"`
  LocationOfBirth string `json:"location_of_birth"`
}

type Passport struct {
  Id           string `json:"id"`
  DateOfIssue  string `json:"date_of_issue"`
  DateOfExpiry string `json:"date_of_expiry"`
  Authority    string `json:"authority"`
  CustomerId   int    `json:"customer_id"`
}
```

The first time you create a struct, you may not be aware that uppercasing and lowercasing your field names have a meaning in Go. It's similar to public and private members in Java. Uppercase = public, lowercase = private. There are some good discussions on Stackoverflow about [this](http://stackoverflow.com/questions/21825322/why-golang-cannot-generate-json-from-struct-with-front-lowercase-character). The gist is that if field names with a lowercase won't be visible to json.Marshal.

You may not want to expose your data to the consumer of your web service in this format, so you can override the way your fields are marshalled by adding ``json:"first_name"`` to each field with the desired name.

### Operations on our (mock) data

I wanted to create a template REST API that didn't depend on a database, so started with a simple in-memory database that we can work with.

We're first creating a Database struct that will hold the data:

```
type Database struct {
  UserList  map[int]User
  MaxUserId int
}
```
The UserList will hold a list of User structs and the MaxUserId holds the latest used integer. MaxUserId mimicks the behaviour of an autogenerated ID in conventional databases.

We will now create a global database variable so that it's accessible across our whole API:

```
var db *Database
```

In order to make it a bit more useful, we will initialise it with some user objects.Luckily, we can make use of the `init` function that gets automatically called when you start the application:

```
func init() {
  list := make(map[int]User)
  list[0] = User{0, "John", "Doe", "31-12-1985", "London"}
  list[1] = User{1, "Jane", "Doe", "01-01-1992", "Milton Keynes"}
  db = &Database{list, 1}
}
```

Now, returning a list of users is quite easy, it's just showing the UserList:

```
func ListUsersHandler(w http.ResponseWriter, req *http.Request) {
  Render.JSON(w, http.StatusOK, db.List())
}
```

This will return the following to the client:

```
{
    "users": [
        {
            "date_of_birth": "31-12-1985",
            "first_name": "John",
            "id": 0,
            "last_name": "Doe",
            "location_of_birth": "London"
        },
        {
            "date_of_birth": "01-01-1992",
            "first_name": "Jane",
            "id": 1,
            "last_name": "Doe",
            "location_of_birth": "Milton Keynes"
        }
    ]
}
```

Notice the `Render.JSON`? That's part of `"github.com/unrolled/render"` and allows us to render JSON output when we send data back to the client.

Another example is the retrieval of a specific object:

```
func GetUserHandler(w http.ResponseWriter, req *http.Request) {
  vars := mux.Vars(req)
  uid, _ := strconv.Atoi(vars["uid"])
  user, err := db.Get(uid)
  if err == nil {
    Render.JSON(w, http.StatusOK, user)
  } else {
    Render.JSON(w, http.StatusNotFound, err)
  }
}
```

This reads the uid variable from the route (`/users/{uid}`), converts the string to an integer and then looks up the user in our UserList by ID. If the user does not exit, we return a 404 and an error object. If the user exists, we return a 200 and a JSON object with the user.

Example:

```
{
    "date_of_birth": "31-12-1985",
    "first_name": "John",
    "id": 0,
    "last_name": "Doe",
    "location_of_birth": "London"
}
```

### API routes and route handlers

Now that we have defined the data model, we need to translate that to a REST interface:

* Retrieve a list of all users: `GET /users` -> The `GET` just refers to the HTTP action you would use. If you want to test this in the command line, then you can use curl: `curl -X GET http://localhost:3009/users` or `curl -X POST http://localhost:3009/users`
* Retrieve the details of an individual user: `GET /users/{uid}` -> {uid} allows us to create a variable, named uid, that we can use in our code. An example of this url would be `GET /users/1`
* Create a new user: `POST /users`
* Update a user: `PUT /users/{uid}`
* Delete a user: `DELETE /users/{uid}`

We now need to do the same for handling passports. Don't forget that a passport belongs to a user, so to retrieve a list of all passports for a given user, we would use `GET /users/{uid}/passports`.

When we want to retrieve an specific passport, we don't need to prefix the route with `/users/{uid}` anymore because we know exactly which passport we want to retrieve. So, instead of `GET /users/{uid}/passports/{pid}`, we can just use `GET /passports/{pid}`.

Once you have the API design sorted, it's just a matter of creating the code that gets called when a specific route is hit. We implement those with Handlers.

```golang
  router.HandleFunc("/users", UsersHandler).Methods("GET")
  router.HandleFunc("/users/{uid}", UsersHandler).Methods("GET")
  router.HandleFunc("/users", UsersHandler).Methods("POST")
  router.HandleFunc("/users/{uid}", UsersHandler).Methods("PUT")
  router.HandleFunc("/users/{uid}", UsersHandler).Methods("DELETE")

  router.HandleFunc("/users/{uid}/passports", PassportsHandler).Methods("GET")
  router.HandleFunc("/passports/{pid}", PassportsHandler).Methods("GET")
  router.HandleFunc("/users/{uid}/passports", PassportsHandler).Methods("POST")
  router.HandleFunc("/passports/{pid}", PassportsHandler).Methods("PUT")
  router.HandleFunc("/passports/{pid}", PassportsHandler).Methods("DELETE")
```

Last but not least, we want to handle two special cases:

```golang
  router.HandleFunc("/", HomeHandler)
  router.HandleFunc("/healthcheck", HealthcheckHandler).Methods("GET")
```

When someone hits our API, without a specified route, then we can handle that with either a standard 404 (not found), or any other type of feedback.

We also want to set up a health check that monitoring tools like [Sensu](https://sensuapp.org/) can call: `GET /healthcheck`. The health check route can return a 204 OK when the serivce is up and running, including some extra stats. A 204 means "Hey, I got your request, all is fine and I have nothing else to say". It essentially tells your client that there is no body content.

```
func HealthcheckHandler(w http.ResponseWriter, req *http.Request) {
  Render.Text(w, http.StatusNoContent, "")
}
```

This health check is very simple. It just checks whether the service is up and running, which can be useful in a build and deployment pipelines where you can check whether your newly deployed API is running (as part of a smoke test). More advanced health checks will also check whether it can reach the database, message queue or anything else you'd like to check. Trust me, your DevOps colleagues will be very grateful for this. (Don't forget to change your HTTP status code to 200 if you want to report on the various components that your health check is checking.)

### Testing your routes with curl commands

Let's start with some simple curl tests. Open your terminal and try the following curl commands.

Retrieve a list of users:

```
curl -X GET http://localhost:3009/users | python -mjson.tool
```

That should result in the following result:

```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   228  100   228    0     0  41996      0 --:--:-- --:--:-- --:--:-- 45600
{
    "users": [
        {
            "date_of_birth": "31-12-1985",
            "first_name": "John",
            "id": 0,
            "last_name": "Doe",
            "location_of_birth": "London"
        },
        {
            "date_of_birth": "01-01-1992",
            "first_name": "Jane",
            "id": 1,
            "last_name": "Doe",
            "location_of_birth": "Milton Keynes"
        }
    ]
}
```

Get a specific user:

```
curl -X GET http://localhost:3009/users/0 | python -mjson.tool
```

Results in:

```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   104  100   104    0     0   6625      0 --:--:-- --:--:-- --:--:--  6933
{
    "date_of_birth": "31-12-1985",
    "first_name": "John",
    "id": 0,
    "last_name": "Doe",
    "location_of_birth": "London"
}
```

Adding a user:

```
TO DO
```

Deleting a user:

```
TO DO
```

Updating an existing user:

```
TO DO
```

### Testing

There are lots of opinions on testing, how much you should be testing, which layers of your applications, etc. When I'm working with micro services, I tend to focus on two types of tests to start with: testing the data access layer and testing the actual HTTP service.

In this example, we want to test the List, Add, Get, Update and Delete operations on our in-memory document database. The data access code is stored in the `database.go` file, so following Go convention we will create a new file called `database_test.go`.

In the `database_test.go` file, we have two sections:

First, we're going to create a common initialiser, this code sets up our database, inserts a couple of test records and then fires off the tests:

```
func TestMain(m *testing.M) {
  list := make(map[int]User)
  list[0] = User{0, "John", "Doe", "31-12-1985", "London"}
  list[1] = User{1, "Jane", "Doe", "01-01-1992", "Milton Keynes"}
  db = &Database{list, 1}
  retCode := m.Run()
  os.Exit(retCode)
}
```
Once this is ready, we can start writing tests. Let's have a look at the easiest one where we list the elements in our database:

```
func TestList(t *testing.T) {
  list := db.List()
  count := len(list["users"])
  assert.Equal(t, 2, count, "There should be 2 items in the list.")
}
```

This first calls the `db.List()` function, which returns a list of users. We then count the number of elements and last but not least we then check whether that count equals 2.

In standard Go, you would actually write something like:

```
if 2 != count {
  t.Errorf("Expected 2 elements in the list, instead got %v", count)
}
```

However there is a neat Go package called [testify](https://github.com/stretchr/testify) that gives you assertions like Java and that's why we can write cleaner test code like:

```
assert.Equal(t, 2, count, "There should be 2 items in the list.")
```

The `TestList` is only testing for a positive result, but we really need to test for failures as well.

This is our test code for the Delete functionality:

```
func TestDeleteSuccess(t *testing.T) {
  ok, err := db.Delete(1)
  assert.Equal(t, true, ok, "they should be equal")
  assert.Nil(t, err)
}

func TestDeleteFail(t *testing.T) {
  ok, err := db.Delete(10)
  assert.Equal(t, false, ok, "they should be equal")
  assert.NotNil(t, err)
}
```

The first test function `TestDeleteSuccess` tries to delete a known existing user, with Id 1. We're expecting that the error object is Nil. The second test function `TestDeleteFail` tries to look up a non-existing user with Id 10, and as expected, this should return an actual Error object.

How do we run the tests?

Simple:

```
go test
```

If you want it more verbose, then:

```
go test -v
```

Which will give you:

```
=== RUN TestList
--- PASS: TestList (0.00s)
=== RUN TestGetSuccess
--- PASS: TestGetSuccess (0.00s)
=== RUN TestGetFail
--- PASS: TestGetFail (0.00s)
=== RUN TestAdd
--- PASS: TestAdd (0.00s)
=== RUN TestUpdateSuccess
--- PASS: TestUpdateSuccess (0.00s)
=== RUN TestUpdateFail
--- PASS: TestUpdateFail (0.00s)
=== RUN TestDeleteSuccess
--- PASS: TestDeleteSuccess (0.00s)
=== RUN TestDeleteFail
--- PASS: TestDeleteFail (0.00s)
PASS
ok    github.com/leeprovoost/go-rest-api-template 0.008s
```

Do you want to get some more info on your code coverage? No worries, Go has you covered (no pun intended):

```
go test -cover
```

This will give you:

```
PASS
coverage: 34.9% of statements
ok    github.com/leeprovoost/go-rest-api-template 0.009s
```

TO DO Testing the HTTP service

### Environment Variables

TO DO

## Useful references

* Structs and JSON formatting: http://stackoverflow.com/questions/21825322/why-golang-cannot-generate-json-from-struct-with-front-lowercase-character
* Undertanding method receivers and pointers: http://nathanleclaire.com/blog/2014/08/09/dont-get-bitten-by-pointer-vs-non-pointer-method-receivers-in-golang/
* Read JSON POST body: http://stackoverflow.com/questions/15672556/handling-json-post-request-in-go
* Writing modular GO REST APIs: http://thenewstack.io/make-a-restful-json-api-go/
* Use render for generating JSON, see https://github.com/unrolled/render/issues/7 for use of global variable
* Testing techniques: https://talks.golang.org/2014/testing.slide#1
* Testing Go HTTP API: http://dennissuratna.com/testing-in-go/
* Great overview of HTTP response codes: http://stackoverflow.com/a/2342631
