# mysqlstore 
[![godoc](https://godoc.org/github.com/alexedwards/scs/engine/mysqlstore?status.png)](https://godoc.org/github.com/alexedwards/scs/engine/mysqlstore)

Package mysqlstore is a MySQL-based storage engine for the [SCS session package](https://godoc.org/github.com/alexedwards/scs/session).

## Usage

### Installation

Either:

```
$ go get github.com/alexedwards/scs/engine/mysqlstore
```

Or (recommended) use use [gvt](https://github.com/FiloSottile/gvt) to vendor the `engine/mysqlstore` and `session` sub-packages:

```
$ gvt fetch github.com/alexedwards/scs/engine/mysqlstore
$ gvt fetch github.com/alexedwards/scs/session
```

### Setup

You should have a working MySQL database containing a `sessions` table with the definition:

```sql
CREATE TABLE sessions (
  token CHAR(43) PRIMARY KEY,
  data BLOB NOT NULL,
  expiry TIMESTAMP(6) NOT NULL
);
CREATE INDEX sessions_expiry_idx ON sessions (expiry);
```

### Example

```go
package main

import (
    "database/sql"
    "io"
    "log"
    "net/http"
    "time"

    "github.com/alexedwards/scs/engine/mysqlstore"
    "github.com/alexedwards/scs/session"
)

func main() {
    // Establish a database/sql pool
    db, err := sql.Open("mysql", "user:pass@/db")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // Create a new MySQLStore instance using the existing database/sql pool, 
    // with a cleanup interval of 5 minutes.
    engine := mysqlstore.New(db, 5*time.Minute)

    sessionManager := session.Manage(engine)
    http.HandleFunc("/put", putHandler)
    http.HandleFunc("/get", getHandler)
    http.ListenAndServe(":4000", sessionManager(http.DefaultServeMux))
}

func putHandler(w http.ResponseWriter, r *http.Request) {
    err := session.PutString(r, "message", "Hello world!")
    if err != nil {
        http.Error(w, err.Error(), 500)
    }
}

func getHandler(w http.ResponseWriter, r *http.Request) {
    msg, err := session.GetString(r, "message")
    if err != nil {
        http.Error(w, err.Error(), 500)
    }
    io.WriteString(w, msg)
}
```

### Cleaning up expired session data

The mysqlstore package provides a background 'cleanup' goroutine to delete expired session data. This stops the database table from holding on to invalid sessions indefinitely and growing unnecessarily large.

You can specify how frequently to run the cleanup when creating a new MySQLStore instance:

```go
// Run a cleanup every 30 minutes.
mysqlstore.New(db, 30*time.Minute)

// Setting the cleanup interval to zero prevents the cleanup from being run.
mysqlstore.New(db, 0)
```

#### Terminating the cleanup goroutine

It's rare that the cleanup goroutine for a MySQLStore instance needs to be terminated. It is generally intended to be long-lived and run for the lifetime of your application.

However, there may be occasions when your use of a MySQLStore instance is transient. A common example would be using it in a short-lived test function. In this scenario, the cleanup goroutine (which will run forever) will prevent the MySQLStore object from being garbage collected even after the test function has finished. You can prevent this by manually calling `StopCleanup()`.

For example:

```go
func TestExample(t *testing.T) {
    db, err := sql.Open("mysql", "user:pass@/db")
    if err != nil {
        t.Fatal(err)
    }
    defer db.Close()

    engine := New(db, time.Second)
    defer engine.StopCleanup()

    // Run test...
}
```

## Notes

The mysqlstore package is underpinned by the [go-sql-driver/mysql](https://github.com/go-sql-driver/mysql) driver.

Full godoc documentation: [https://godoc.org/github.com/alexedwards/scs/engine/mysqlstore](https://godoc.org/github.com/alexedwards/scs/engine/mysqlstore).