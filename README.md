## Testable RDBMS backed Go application


### Things I talk in 10 min

- independent/repeatable tests for RDBMS backed Go app
- fast tests for RDBMS backed Go app
- easy to write tests for RDBMS backed Go app

---

### Things I don't talk

- how to write 'right' tests
- 'tests are necessary/unnecessary' kind of rant

---

### Problems

- 1. No standard way to setup/teardown database and transaction in Go test
    * Unlike other languages with WAF, it is rather unclear.
    * Django ORM/SQLAlchemy/ActiveRecord have standard, or at least well known way to do this.
    * How to handle transaction in tests is also unclear.
- 2. No standard way to easily create test data
    * Python has factory-boy, Ruby has factory-girl, but Go...

---

### Solutions

The following is just one way to tackle the problems previously stated. I really, really want to hear how other Gophers are doing.

- 1. No standard way to setup/teardown database and transaction in Go test
    - 1-1. Use `TestMain` to setup/teardown database(or schema)
    - 1-2. Use single transaction database driver to make test independent and repeatable
- 2. No standard way to easily create test data
    - 2-1. Use patched version of `mergo` to create data

---

### 1-1. TestMain to setup/teardown database(or schema)

- Inside `TestMain`
    - Create schema for test (we are using PostgreSQL, so using schema instead of database)
    - Create tables in that schema
    - Run tests
    - Drop cascade test schema

---

### 1-1. TestMain to setup/teardown database(or schema)

- TestingDBSetup
    * create `store_api_schema`
- TestingTableCreate
    * create database objects in `store_api_schema`
- TestingDBTeardown
    * drop cascade `store_api_schema`
- https://github.com/achiku/testable-go-rdbms/blob/master/model/main_test.go

---

```go
package model

import (
	"flag"
	"os"
	"testing"

	"log"
)

// TestMain model package setup/teardonw
func TestMain(m *testing.M) {
	flag.Parse()

	// for create/drop schema
	createSchemaCon := "postgres://pgtest@localhost:5432/pgtest?sslmode=disable"
	// for create database objects
	createTableCon := "postgres://store_api_test@localhost:5432/pgtest?sslmode=disable"

	TestingDBTeardown(createSchemaCon)
	if err := TestingDBSetup(createSchemaCon); err != nil {
		log.Fatal(err)
	}
	if err := TestingTableCreate(createTableCon); err != nil {
		log.Fatal(err)
	}
	code := m.Run()
	if err := TestingDBTeardown(createSchemaCon); err != nil {
		log.Fatal(err)
	}
	os.Exit(code)
}
```

---

- https://github.com/achiku/testable-go-rdbms/blob/master/model/testing.go

```go
package model

import (
	"database/sql"
	"io/ioutil"
	"testing"

	txdb "github.com/achiku/pgtxdb"
	_ "github.com/lib/pq" // postgres
	"github.com/pkg/errors"
)

// CREATE USER pgtest; -- superuser
// CREATE USER store_api; -- this user is for development
// CREATE SCHEMA store_api AUTHORIZATION store_api;
// CREATE USER store_api_test; -- this user is for test

// TestingDBSetup set up test schema
func TestingDBSetup(conStr string) error {
	con, err := sql.Open("postgres", conStr)
	if err != nil {
		return errors.Wrap(err, "failed to open connection")
	}
	defer con.Close()

	_, err = con.Exec("CREATE SCHEMA store_api_test AUTHORIZATION store_api_test")
	if err != nil {
		return errors.Wrap(err, "failed to create test schema")
	}
	return nil
}

// TestingTableCreate create test tables
func TestingTableCreate(conStr string) error {
	con, err := sql.Open("postgres", conStr)
	if err != nil {
		return errors.Wrap(err, "failed to open connection")
	}
	defer con.Close()

	ddl, err := ioutil.ReadFile("./ddl.sql")
	if err != nil {
		return errors.Wrap(err, "failed to read ddl.sql")
	}
	_, err = con.Exec(string(ddl))
	if err != nil {
		return errors.Wrap(err, "failed to execute ddl.sql")
	}
	return nil
}

// TestingDBTeardown drop test schema
func TestingDBTeardown(conStr string) error {
	con, err := sql.Open("postgres", conStr)
	if err != nil {
		return errors.Wrap(err, "failed to open connection")
	}
	defer con.Close()

	_, err = con.Exec("DROP SCHEMA store_api_test CASCADE")
	if err != nil {
		return errors.Wrap(err, "failed to drop schema")
	}
	return nil
}
```

---


### 1-1. TestMain to setup/teardown database(or schema)

- We are using SQLAlchemy + alembic to create database objects like tables, index, and constraints
    * As you can see above example, it gets more and more difficult to manage plane SQL to manage database objects
- This combination (SQLAlchemy + alembic) can be also used for table migration management in production
- Furthermore, we are making these functions public so that other packages depend on `model` package can also setup/teardown the test schema
    * https://speakerdeck.com/mitchellh/advanced-testing-with-go
    * p52: Testing as a Public API

---

### 1-2. Single transaction database driver to make test independent and repeatable

- Use single transaction database driver to make test independent and repeatable
    - What's that?
    - https://github.com/DATA-DOG/go-txdb (for MySQL)
    - https://github.com/achiku/pgtxdb (for PostgreSQL)

> Package txdb is a single transaction based database sql driver. When the connection is opened, it starts a transaction and all operations performed on this sql.DB will be within that transaction. If concurrent actions are performed, the lock is acquired and connection is always released the statements and rows are not holding the connection.

---

### 1-2. Single transaction database driver to make test independent and repeatable

```go
func init() {
	txdb.Register("txdb", "postgres", "postgres://store_api_test@localhost:5432/pgtest?sslmode=disable")
}
```

```go
// TestSetupTx create tx and cleanup func for test
func TestSetupTx(t *testing.T) (Txer, func()) {
	db, err := sql.Open("txdb", "dummy")
	if err != nil {
		t.Fatal(err)
	}
	tx, err := db.Begin()
	if err != nil {
		t.Fatal(err)
	}

	cleanup := func() {
		tx.Rollback()
		db.Close()
	}
	return tx, cleanup
}

// TestSetupDB create db and cleanup func for test
func TestSetupDB(t *testing.T) (DBer, func()) {
	db, err := sql.Open("txdb", "dummy")
	if err != nil {
		t.Fatal(err)
	}

	cleanup := func() {
		db.Close()
	}
	return db, cleanup
}
```

---


### 1-2. Single transaction database driver to make test independent and repeatable

- We are extensibly using PostgreSQL
    * I forked great work of datadog, `go-txdb` and add some functionalities for PostgreSQL https://github.com/achiku/pgtxdb
    * PostgreSQL can execute `savepoint`
    * Using `pgtxdb`, developers can test target code with multiple transactions including rollback
- When `conn.Bigin()` is called, this library executes `SAVEPOINT pgtxdb_xxx;` instead of actually begins transaction.
- tx.Commit() does nothing.
- `ROLLBACK TO SAVEPOINT pgtxdb_xxx;` will be executed upon `tx.Rollback()` call so that it can emulate transaction rollback.
- Above features enable us to emulate multiple transactions in one test case.


---

### 1-2. Single transaction database driver to make test independent and repeatable

- Tips 1: pass `*testing.T` for all the helper functions, and `t.Fatal(err)` if error occur
    * https://speakerdeck.com/mitchellh/advanced-testing-with-go?slide=28
    * this reduce error handling boilerplate, and makes test code cleaner
- Tips 2: use `cleanup` function which does all the cleanup-after-test kind of tasks
    * https://speakerdeck.com/mitchellh/advanced-testing-with-go?slide=29
    * using this func, you don't have to have multiple `defer` in the test
- Tips 3: define interface for database so that you can switch implementation for the test


```go
package model

import "database/sql"

// Query database/sql compatible query interface
type Query interface {
	Exec(string, ...interface{}) (sql.Result, error)
	Query(string, ...interface{}) (*sql.Rows, error)
	QueryRow(string, ...interface{}) *sql.Row
}

// Tx represents database transaction interface
type Tx interface {
	Query
	Commit() error
	Rollback() error
}

// DB database/sql interface
type DB interface {
	Query
	Begin() (*sql.Tx, error)
	Close() error
	Ping() error
}
```

---

### 1-2. Single transaction database driver to make test independent and repeatable

```go
func TestItem_ItemCreate(t *testing.T) {
	tx, cleanup := TestSetupTx(t)
	defer cleanup()

	item := Item{
		Name:        "test item1",
		Price:       decimal.NewFromFloat(1000),
		Description: sql.NullString{String: "this is test"},
	}
	if err := item.Create(tx); err != nil {
		t.Fatal(err)
	}
	t.Logf("%+v", item)

	targetItem, err := GetItemByPk(tx, item.ID)
	if err != nil {
		t.Fatal(err)
	}
	if targetItem.Name != item.Name {
		t.Errorf("want %s got %s", item.Name, targetItem.Name)
	}
}
```

---


### 2-1. Patched version of `mergo` to create data

- Use patched version of `mergo` to create data
    - https://github.com/achiku/mergo


### 2-1. Patched version of `mergo` to create data


```go
// GetDailySummary get daily summary
func GetDailySummary(tx Query, d time.Time) ([]DailySummaryStats, error) {
	from := time.Date(d.Year(), d.Month(), d.Day(), 0, 0, 0, 0, time.Local)
	rows, err := tx.Query(`
	SELECT
		date_trunc('day', s.sold_at)
		, i.id
		, i.name
		, sum(s.paid_amount)
	FROM sale s
	JOIN item i
	ON s.item_id = i.id
	WHERE s.sold_at >= $1
	AND s.sold_at < $2
	GROUP BY
		date_trunc('day', s.sold_at)
		, i.id
		, i.name
	`, from, to)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var sts []DailySummaryStats
	for rows.Next() {
		var st DailySummaryStats
		rows.Scan(
			&st.Date,
			&st.ItemID,
			&st.ItemName,
			&st.SaleAmount,
		)
		sts = append(sts, st)
	}
	return sts, nil
}
```

```go
package model

import (
	"testing"
	"time"

	"github.com/AdrianLungu/decimal"
)

func TestSale_GetDailySummary(t *testing.T) {
	tx, cleanup := TestSetupTx(t)
	defer cleanup()

	dt := time.Now()
	u := TestCreateUserAccountData(t, tx, &UserAccount{})
	i1 := TestCreateItemData(t, tx, &Item{
		Name:  "beer",
		Price: decimal.NewFromFloat(500),
	})
	TestCreateSaleData(t, tx, i1, u, &Sale{
		SoldAt: dt,
	})
	i2 := TestCreateItemData(t, tx, &Item{
		Name:  "pizza",
		Price: decimal.NewFromFloat(1200),
	})
	TestCreateSaleData(t, tx, i2, u, &Sale{
		SoldAt: dt,
	})
	TestCreateSaleData(t, tx, i2, u, &Sale{
		SoldAt: dt.AddDate(0, 0, -2),
	})

	sts, err := GetDailySummary(tx, dt)
	if err != nil {
		t.Fatal(err)
	}
	t.Logf("%+v", sts)
}
```

```go
package model

import (
	"database/sql"
	"log"
	"testing"

	"github.com/AdrianLungu/decimal"
	"github.com/achiku/mergo"
)

// TestCreateItemData create sale test data
func TestCreateItemData(t *testing.T, tx Query, item *Item) *Item {
	itemDefault := &Item{
		Price:       decimal.NewFromFloat(1000),
		Name:        "item1",
		Description: sql.NullString{String: "test desc"},
	}
	if err := mergo.MergeWithOverwrite(itemDefault, item, TestStructMergeFunc); err != nil {
		log.Fatal(err)
	}
	if err := itemDefault.Create(tx); err != nil {
		t.Fatal(err)
	}
	return itemDefault
}
```
