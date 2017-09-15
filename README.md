# go-queryset [![GoDoc](https://godoc.org/github.com/jirfag/go-queryset?status.png)](http://godoc.org/github.com/jirfag/go-queryset) [![Go Report Card](https://goreportcard.com/badge/github.com/jirfag/go-queryset)](https://goreportcard.com/report/github.com/jirfag/go-queryset) [![Codacy Badge](https://api.codacy.com/project/badge/Grade/04f432950cbc4e8cb0ec4ef36f4e90bb)](https://www.codacy.com/app/jirfag/go-queryset?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=jirfag/go-queryset&amp;utm_campaign=Badge_Grade) [![Build Status](https://travis-ci.org/jirfag/go-queryset.svg?branch=master)](https://travis-ci.org/jirfag/go-queryset) [![Coverage Status](https://coveralls.io/repos/github/jirfag/go-queryset/badge.svg?branch=master)](https://coveralls.io/github/jirfag/go-queryset?branch=master)

100% type-safe ORM for Go (Golang) with code generation and MySQL, PostgreSQL, Sqlite3, SQL Server support. GORM under the hood.


## Contents

* [Installation](#installation)
* [Usage](#usage)
  * [Define models](#define-models)
  * [Relation with GORM](#relation-with-gorm)
  * [Create models](#create)
  * [Select models](#select)
  * [Update models](#update)
  * [Delete models](#delete)
  * [Full list of generated methods](#full-list-of-generated-methods)
* [Golang version](#golang-version)
* [Why](#why)
  * [Why not just use GORM?](#why-not-just-use-gorm)
  * [Why not another ORM?](#why-not-another-orm)
  * [Why not any ORM?](#why-not-any-orm)
  * [Why not raw SQL queries?](#why-not-raw-sql-queries)
  * [Why not go-kallax?](#why-not-go-kallax)
* [How it relates to another languages ORMs](#how-it-relates-to-another-languages-orms)
* [Features](#features)
* [Limitations](#limitations)
* [Performance](#performance)


# Installation
```bash
go get -u github.com/jirfag/go-queryset/cmd/goqueryset
```

# Usage
## Define models
Imagine you have model `User` in your [`models.go`](https://github.com/jirfag/go-queryset/blob/master/examples/comparison/gorm1/gorm1.go#L16) file:

```go
type User struct {
	gorm.Model
	Rating      int
	RatingMarks int
}
```

Now transform it by [adding comments for query set generation](https://github.com/jirfag/go-queryset/blob/master/examples/comparison/gorm4/gorm4.go#L15):
```go
//go:generate goqueryset -in models.go

// User struct represent user model. Next line (gen:qs) is needed to autogenerate UserQuerySet.
// gen:qs
type User struct {
	gorm.Model
	Rating      int
	RatingMarks int
}
```

Take a loot at line `// gen:qs`. It's a necessary line to enable querysets for this struct. You can put it at any line in struct's doc-comment.

Then execute next shell command:
```bash
go generate ./...
```

And you will get file [`autogenerated_models.go`](https://github.com/jirfag/go-queryset/blob/master/examples/comparison/gorm4/autogenerated_gorm4.go) in the same directory (and package) as `models.go`.

In this autogenerated file you will find a lot of autogenerated typesafe methods like these:
```go
func (qs UserQuerySet) CreatedAtGte(createdAt time.Time) UserQuerySet {
	return qs.w(qs.db.Where("created_at >= ?", createdAt))
}

func (qs UserQuerySet) RatingGt(rating int) UserQuerySet {
	return qs.w(qs.db.Where("rating > ?", rating))
}

func (qs UserQuerySet) IDEq(ID uint) UserQuerySet {
	return qs.w(qs.db.Where("id = ?", ID))
}

func (qs UserQuerySet) DeletedAtIsNull() UserQuerySet {
	return qs.w(qs.db.Where("deleted_at IS NULL"))
}

func (o *User) Delete(db *gorm.DB) error {
	return db.Delete(o).Error
}

func (qs UserQuerySet) OrderAscByCreatedAt() UserQuerySet {
	return qs.w(qs.db.Order("created_at ASC"))
}
```

See full autogenerated file [here](https://github.com/jirfag/go-queryset/blob/master/examples/comparison/gorm4/autogenerated_gorm4.go).

Now you can use this queryset for creating/reading/updating/deleting. Let's take a lot at these operations.

## Relation with GORM
You can embed and not embed `gorm.Model` into your model (e.g. if you don't need `DeletedAt` field), but you must use `*gorm.DB`
to properly work. Don't worry if you don't use GORM yet, it's [easy to create `*gorm.DB`](http://jinzhu.me/gorm/database.html#connecting-to-a-database):
```go
import (
    "github.com/jinzhu/gorm"
    _ "github.com/jinzhu/gorm/dialects/mysql"
)

func getGormDB() *gorm.DB {
	db, err := gorm.Open("mysql", "user:password@/dbname?charset=utf8&parseTime=True&loc=Local")
	// ...
}
```

If you already use another ORM or raw `sql.DB`, you can reuse your `sql.DB` object (to reuse connections pool):
```go
	var sqlDB *sql.DB = getSQLDBFromAnotherORM()
	var gormDB *gorm.DB
	gormDB, err = gorm.Open("mysql", sqlDB)
```

## Create
```go
u := User{
	Rating: 5,
	RatingMarks: 0,
}
err := u.Create(getGormDB())
```
Under the hood `Create` method [just calls `db.Create(&u)`](https://github.com/jirfag/go-queryset/blob/master/examples/comparison/gorm4/autogenerated_gorm4.go#L38).

## Select
It's the most powerful feature of query set. Let's execute some queries:
### Select all users
```go
var users []User
err := NewUserQuerySet(getGormDB()).All(&users)
if err == gorm.ErrRecordNotFound {
	// no records were found
}
```

It generates this SQL request for MySQL:
```sql
SELECT * FROM `users` WHERE `users`.deleted_at IS NULL
```

`deleted_at` filtering is added by GORM (soft-delete), to disable it use [`Unscoped`](http://jinzhu.me/gorm/crud.html#delete).

### Select one user
```go
var user User
err := NewUserQuerySet(getGormDB()).One(&user)
```

### Select N users with highest rating
```go
var users []User
err := NewUserQuerySet(getGormDB()).
	RatingMarksGte(minMarks).
	OrderDescByRating().
	Limit(N)
	All(&users)
```

### Select users registered today
In this example we will define custom method on generated `UserQuerySet` for later reuse in multiple functions:
```go
func (qs UserQuerySet) RegisteredToday() UserQuerySet {
	// autogenerated typesafe method CreatedAtGte(time.Time)
	return qs.CreatedAtGte(getTodayBegin())
}

...
var users []User
err := NewUserQuerySet(getGormDB()).
	RegisteredToday().
	OrderDescByCreatedAt().
	Limit(N)
	All(&users)
```

## Update

### Update one record by primary key
```go
u := User{
	Model: gorm.Model{
		ID: uint(7),
	},
	Rating: 1,
}
err := u.Update(getGormDB(), UserDBSchema.Rating)
```

Goqueryset generates DB names for struct fields into [`UserDBSchema`](https://github.com/jirfag/go-queryset/blob/master/examples/comparison/gorm4/autogenerated_gorm4.go#L414) variable.
In this example we used [`UserDBSchema.Rating`](https://github.com/jirfag/go-queryset/blob/master/examples/comparison/gorm4/autogenerated_gorm4.go#L419).

And this code generates next SQL:
```sql
UPDATE `users` SET `rating` = ? WHERE `users`.deleted_at IS NULL AND `users`.`id` = ?
```

### Update multiple record or without model object
Sometimes we don't have model object or we are updating multiple rows in DB.
For these cases there is another typesafe interface:

```go
err := NewUserQuerySet(getGormDB()).
	RatingLt(1).
	GetUpdater().
	SetRatingMarks(0).
	Update()
```
```sql
UPDATE `users` SET `rating_marks` = ? WHERE `users`.deleted_at IS NULL AND ((rating < ?))
```

## Delete
### Delete one record by primary key
```go
u := User{
	Model: gorm.Model{
		ID: uint(7),
	},
}
err := u.Delete(getGormDB())
```

### Delete multiple records
```go
err := NewUserQuerySet(getGormDB()).
	RatingMarksEq(0).
	Delete()
```

## Full list of generated methods
### QuerySet methods - `func (qs {StructName}QuerySet)`
* create new queryset: `New{StructName}QuerySet(db *gorm.DB)`
```go
func NewUserQuerySet(db *gorm.DB) UserQuerySet
```
* filter by field (`where`)
	* all field types
		* Equals: `{FieldName}(Eq|Ne)(arg {FieldType})`
		```go
		func (qs UserQuerySet) RatingEq(rating int) UserQuerySet
		```
		* In: `{FieldName}(Not)In(arg {FieldType}, argsRest ...{FieldType})`
		```go
		func (qs UserQuerySet) NameIn(name string, nameRest ...string) UserQuerySet {}
		func (qs UserQuerySet) NameNotIn(name string, nameRest ...string) UserQuerySet {}
		```
	* numeric types (`int`, `int64`, `uint` etc + `time.Time`):
 		* `{FieldName}(Lt|Lte|Gt|Gte)(arg {FieldType)`
		```go
		func (qs UserQuerySet) RatingGt(rating int) UserQuerySet
		```
		* `Order(Asc|Desc)By{FieldName}()`
		```go
		func (qs UserQuerySet) OrderDescByRating() UserQuerySet
		```
	* pointer fields: `{FieldName}IsNull()`, `{FieldName}IsNotNull()`
	```go
	func (qs UserQuerySet) ProfileIsNull() UserQuerySet {}
	func (qs UserQuerySet) ProfileIsNotNull() UserQuerySet {}
	```
* preload related object (for structs fields or pointers to structs fields): `Preload{FieldName}()`
	For struct
	```go
		type User struct {
			profile *Profile
		}
	```
	will be generated:
	```go
	func (qs UserQuerySet) PreloadProfile() UserQuerySet
	```
	`Preload` functions call `gorm.Preload` to preload related object.

* selectors
	* Select all objects, return `gorm.ErrRecordNotFound` if no records
	```go
	func (qs UserQuerySet) All(users *[]User) error
	```
	* Select one object, return `gorm.ErrRecordNotFound` if no records
	```go
	func (qs UserQuerySet) One(user *User) error
	```
* Limit
```go
func (qs UserQuerySet) Limit(limit int) UserQuerySet
```
* [get updater](#update-multiple-record-or-without-model-object) (for update + where, based on current queryset):
```go
func (qs UserQuerySet) GetUpdater() UserUpdater
```
* delete with conditions from current queryset: `Delete()`
```go
func (qs UserQuerySet) Delete() error
```
* Aggregations
	* Count
	```go
	func (qs UserQuerySet) Count() (int, error)
	```

### Object methods - `func (u *User)`
* create object
```go
func (o *User) Create(db *gorm.DB) error
```
* delete object by PK
```go
func (o *User) Delete(db *gorm.DB) error
```
* update object by PK
```go
func (o *User) Update(db *gorm.DB, fields ...userDBSchemaField) error
```
Pay attention that field names are automatically generated into variable
```go
type userDBSchemaField string

// UserDBSchema stores db field names of User
var UserDBSchema = struct {
	ID          userDBSchemaField
	CreatedAt   userDBSchemaField
	UpdatedAt   userDBSchemaField
	DeletedAt   userDBSchemaField
	Rating      userDBSchemaField
	RatingMarks userDBSchemaField
}{

	ID:          userDBSchemaField("id"),
	CreatedAt:   userDBSchemaField("created_at"),
	UpdatedAt:   userDBSchemaField("updated_at"),
	DeletedAt:   userDBSchemaField("deleted_at"),
	Rating:      userDBSchemaField("rating"),
	RatingMarks: userDBSchemaField("rating_marks"),
}
```

And they are typed, so you won't have string-misprint error.


### Updater methods - `func (u UserUpdater)`
* set field: `Set{FieldName}`
```go
func (u UserUpdater) SetCreatedAt(createdAt time.Time) UserUpdater
```
* execute update: `Update()`
```go
func (u UserUpdater) Update() error
```

# Golang version
Golang >= 1.7 is required. Tested on go 1.7, 1.8, 1.9 versions by [Travis CI](https://travis-ci.org/jirfag/go-queryset)

# Why?
## Why not just use GORM?
I like GORM: it's the best ORM for golang, it has fantastic documentation, but as a Golang developers team lead I can point out some troubles with it:
1. GORM isn't typesafe: it's so easy to spend 1 hour trying to execute simple Update. GORM gets all arguments as `interface{}`
and in the case of invalid GORM usage you won't get error: you will get invalid SQL, no SQL (!) and `error == nil` etc.
It's easy to get `SELECT * FROM t WHERE string_field == 1` SQL in production without type safety.
2. GORM is difficult for beginners because of unclear `interface{}` interfaces: one can't easily find which arguments to pass to GORM methods.

## Why not another ORM?
Type-safety, like with GORM.

## Why not any ORM?
I didn't see any ORM that properly handles code duplication. GORM is the best with `Scopes` support, but even it's far from ideal. E.g. we have GORM and [next typical code](https://github.com/jirfag/go-queryset/blob/master/examples/comparison/gorm1/gorm1.go#L16):
```go
type User struct {
	gorm.Model
	Rating      int
	RatingMarks int
}

func GetUsersWithMaxRating(limit int) ([]User, error) {
	var users []User
	if err := getGormDB().Order("rating DESC").Limit(limit).Find(&users).Error; err != nil {
		return nil, err
	}
	return users, nil
}

func GetUsersRegisteredToday(limit int) ([]User, error) {
	var users []User
	today := getTodayBegin()
	err := getGormDB().Where("created_at >= ?", today).Limit(limit).Find(&users).Error
	if err != nil {
		return nil, err
	}
	return users, nil
}
```

At one moment PM asks us to implement new function, returning list of users registered today AND sorted by rating. Copy-paste way is to add `Order("rating DESC")` to `GetUsersRegisteredToday`. But it leads to typical copy-paste troubles: when we change rating calculation logics (e.g. to `.Where("rating_marks >= ?", 10).Order("rating DESC")`) we must change it in two places.

How to solve it? First idea is to [make reusable functions](https://github.com/jirfag/go-queryset/blob/master/examples/comparison/gorm2/gorm2.go#L27):
```go
func queryUsersWithMaxRating(db *gorm.DB, limit int) *gorm.DB {
	return db.Order("rating DESC").Limit(limit)
}

func queryUsersRegisteredToday(db *gorm.DB, limit int) *gorm.DB {
	today := getTodayBegin()
	return db.Where("created_at >= ?", today).Limit(limit)
}

func GetUsersWithMaxRating(limit int) ([]User, error) {
	var users []User
	if err := queryUsersWithMaxRating(getGormDB(), limit).Find(&users).Error; err != nil {
		return nil, err
	}
	return users, nil
}

func GetUsersRegisteredToday(limit int) ([]User, error) {
	var users []User
	if err := queryUsersRegisteredToday(getGormDB(), limit).Find(&users).Error; err != nil {
		return nil, err
	}
	return users, nil
}

func GetUsersRegisteredTodayWithMaxRating(limit int) ([]User, error) {
	var users []User
	err := queryUsersWithMaxRating(queryUsersRegisteredToday(getGormDB(), limit), limit).
		Find(&users).Error
	if err != nil {
		return nil, err
	}
	return users, nil
}
```

We can use GORM [Scopes](http://jinzhu.me/gorm/crud.html#scopes) to improve [how it looks](https://github.com/jirfag/go-queryset/blob/master/examples/comparison/gorm3/gorm3.go#L64):
```go
func queryUsersWithMaxRating(db *gorm.DB) *gorm.DB {
	return db.Order("rating DESC")
}

func queryUsersRegisteredToday(db *gorm.DB) *gorm.DB {
	return db.Where("created_at >= ?", getTodayBegin())
}

func GetUsersRegisteredTodayWithMaxRating(limit int) ([]User, error) {
	var users []User
	err := getGormDB().
		Scopes(queryUsersWithMaxRating, queryUsersRegisteredToday).
		Limit(limit).
		Find(&users).Error
	if err != nil {
		return nil, err
	}
	return users, nil
}
```

Looks nice, but we loosed ability to parametrize our reusable GORM queries (scopes): they must have only one argument of type `*gorm.DB`. It means that we must move out `Limit` from them (let's say we get it from user). If we need to implement query `QueryUsersRegisteredAfter(db *gorm.DB, t time.Time)` - we can't do it.

Now compare it with [go-queryset solution](https://github.com/jirfag/go-queryset/blob/master/examples/comparison/gorm4/gorm4.go#L30):
```go
// UserQuerySet is an autogenerated struct with a lot of typesafe methods.
// We can define any methods on it because it's in the same package
func (qs UserQuerySet) WithMaxRating(minMarks int) UserQuerySet {
	return qs.RatingMarksGte(minMarks).OrderDescByRating()
}

func (qs UserQuerySet) RegisteredToday() UserQuerySet {
	// autogenerated typesafe method CreatedAtGte(time.Time)
	return qs.CreatedAtGte(getTodayBegin())
}

// now we can parametrize it
const minRatingMarks = 10

func GetUsersWithMaxRating(limit int) ([]User, error) {
	var users []User
	err := NewUserQuerySet(getGormDB()).
		WithMaxRating(minRatingMarks). // reuse our method
		Limit(limit).                  // autogenerated typesafe method Limit(int)
		All(&users)                    // autogenerated typesafe method All(*[]User)
	if err != nil {
		return nil, err
	}
	return users, nil
}

func GetUsersRegisteredToday(limit int) ([]User, error) {
	var users []User
	err := NewUserQuerySet(getGormDB()).
		RegisteredToday(). // reuse our method
		Limit(limit).      // autogenerated typesafe method Limit(int)
		All(&users)        // autogenerated typesafe method All(*[]User)
	if err != nil {
		return nil, err
	}
	return users, nil
}

func GetUsersRegisteredTodayWithMaxRating(limit int) ([]User, error) {
	var users []User
	err := NewUserQuerySet(getGormDB()).
		RegisteredToday().             // reuse our method
		WithMaxRating(minRatingMarks). // reuse our method
		Limit(limit).
		All(&users) // autogenerated typesafe method All(*[]User)
	if err != nil {
		return nil, err
	}
	return users, nil
}
```
## Why not raw SQL queries?
No type-safety, a lot of boilerplate code.

## Why not [go-kallax](https://github.com/src-d/go-kallax)?
1. It works only with PostgreSQL. Go-queryset supports mysql, postgresql, sqlite, mssql etc (all that gorm supports).
2. Lacks simplier model updating interface

## How it relates to another languages ORMs
QuerySet pattern is similar to:
* [Django QuerySet](https://docs.djangoproject.com/en/dev/ref/models/querysets/), but better than it because of type-safety (Python)
* [Rails Active Record and it's scopes](http://guides.rubyonrails.org/active_record_querying.html#scopes), but better than it because of type-safety (Ruby)

# Features
* 100% typesafe: there is no one method with `interface{}` arguments.
* QuerySet pattern allows to reuse queries by defining [custom methods](https://github.com/jirfag/go-queryset/blob/master/examples/comparison/gorm4/gorm4.go#L30) on it.
* Supports all DBMS that GORM supports: MySQL, PostgreSQL, Sqlite3, SQL Server.
* Supports creating, selecting, updating, deleting of objects.

# Limitations
* Joins aren't supported
* Struct tags aren't supported

# Performance
## Runtime
Performance is similar to GORM performance. GORM uses reflection and it may be slow, so why don't we generate raw SQL code?
1. Despite the fact GORM uses reflection, it's the most popular ORM for golang. There are really few tasks where you are CPU-bound while working with DB, usually you are CPU-bound in machine with DB and network/disk bound on machine with golang server.
2. Premature optimization is the root of all evil.
3. Go-queryset is fully compatible with GORM.
4. Code generation is used here not to speedup things, but to create nice interfaces.
5. The main purpose of go-queryset isn't speed, but usage convenience.

## Code generation
Code generation is fast:
1. We parse AST of needed file and find needed structs.
2. We load package and parse it by `go/types`
3. We don't use `reflect` module for parsing, because it's slow
