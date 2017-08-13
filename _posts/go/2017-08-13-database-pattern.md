---
layout: post
title:  "Go Database Code Implementation Pattern"
date:   2017-08-13 19:25:00 +0100
categories: go
tags: database
published: true
---
Whilst learning how to use Go to interact with databases for a ReST application I've been thinking about the best way to structure my code.
This post outlines the approach I am using and I'd be interested in any feedback from experienced Go developers.

## Summary
The points below are a quick summary of the approach:
* All SQL statements are prepared at start up
* Functions that need to interact with the database are passed a map of prepared statements from which they can extract and execute the statement needed.
* All SQL query strings are defined in constants within a central database package. Source files in this package align to the resource types in the ReST application.

## Benefits
* A consistent stucture makes the code easier to maintain
* It is simple to add new ReST resource types to the application and follow the same pattern
* If the database engine has to be changed then this will be simpler as the query strings are all defined in one package

## Details
The diagram below will be useful reference for this post.
![Package structure]({{ site.url }}/assets/2017-08-13-database-pattern-pkg.png)

### Application Initialisation
Within my application's main.go I have a struct type defined called "app".
This struct holds the application's configuration and pointers to resources such as the instance of sql.DB for the connection to the database.
Below is an example of the struct:
```go
type app struct {
	Router        *mux.Router
	Config        *config.Config
	DB            *sql.DB
	PreparedStmts *database.StmtMap
}
```
I create this struct within an "initialize" function in main.go. 
The initialize function opens the database connection and sets the pointer in the app struct for the *sql.DB

It also creates a map of prepared statements for interaction with the database.
The approach for creating and using this map is outlined below.

#### Prepared Statements Map
All the database statements the application uses are prepared at start up and populated into this map.
The initialize function in the main package does this simply with the following call:
```go
a.PreparedStmts, err = database.NewStmtMap(a.DB)
```
Within the application's database package the statement map type is defined:
```go
type StmtMap map[int]*sql.Stmt
```

Populating this map involves:
* A simple struct (called Statement) which pairs a SQL query string with an integer ID
* An interface for statement providers with a method that returns a slice of Statements
* Implementations of the statement provider interface for each resource type of the ReST interface
* Some functions that bring all this together to prepare the statements and add to the map

**Statement Struct**
```go
type Statement struct {
	ID    int
	Query string
}
```
**Statement Provider Interface**
```go
type stmtProvider interface{
	stmts() []Statement
}
```
**Statement Provider Implementation**

Within the "database" directory of the application I create a Go source file for each of the resource types of the ReST API.
This just helps keep things tidy. In each of these I define a struct and implement the stmts() method on it.
The ID integers and the Query strings are defined as constants within the same file. For example:
```go
package database

const (
    StmtKeyListResourceA        = 1
    queryListResourceA          = "SELECT id, name FROM resourcea"
    StmtKeyInsertResourceA      = 2
    queryInsertResourceA        = "INSERT INTO resourcea (id, name) VALUES (?, ?)"
)

type resA struct{}

func (a resA) stmts() []Statement {
	return []Statement{
		{
			ID:    StmtKeyListResourceA,
			Query: queryListResourceA,
		},
		{
			ID:    StmtKeyInsertResourceA,
			Query: queryInsertResourceA,
		},
	}
}
```
**Preparing Statements and Adding to the Map**

Actually preparing the statements and adding to the map is done by functions in the database package's database.go file:
```go
func NewStmtMap(db *sql.DB) (*StmtMap, error) {
	stmtMap := make(StmtMap)
	err := addStmt(&stmtMap, Statements(), db)
	return &stmtMap, err
}

func Statements() []Statement {
	ps := []stmtProvider{
		new(assumeRole),
		new(federationUser),
	}
	var s []Statement
	for _, p := range ps {
		s = append(s, p.stmts()...)
	}
	return s
}

func addStmt(stmtMap *StmtMap, stmts []Statement, db *sql.DB) error {
	for _, stmt := range stmts {
		s, err := db.Prepare(stmt.Query)
		if err != nil {
			return fmt.Errorf("Error preparing statement ID %d: %v", stmt.ID, err)
		}
		(*stmtMap)[stmt.ID] = s
	}
	return nil
}
```

The Statements() method is separate and made public as it is useful to have available for tests in the resource aligned packages within the application.
For more on this see my post on database unit testing (coming soon).

#### Using the Statements Map
Now that the map is all prepared other functions within the application can be passed it for use.
I tend to pass the map by value as it contains pointers to the perpared statements.
For my ReST application I have a package for each resource type. An example of a function within one of these packages is below:
```go
func (r *ResourceA) Delete(stmtMap database.StmtMap) error {
	if stmt, ok := stmtMap[database.StmtKeyDeleteResourceA]; !ok {
		return errors.New("Prepared statement for DB not found. Action: ResourceA Delete ; Statment ID: %d", database.StmtKeyDeleteResourceA)
	} else {
		r, err := stmt.Exec(r.id)
		if err != nil {
			return err
		}
		if i, _ := r.RowsAffected(); i != 1 {
			return fmt.Errorf("Expected 1 and only 1 row to be affected. Number affected was: %v", i)
		}
	}
	return nil
}
```

## Areas for Improvement and Further Thought
The following are some points for potential improvement I'm going to put some thought into:
* Would it be better to encapsulate the implementation of the statementProvider interface into the resource packages rather than the database package? This may need to careful thought to avoid circular imports.
* Should I just store the value of the stmtMap in the app struct rather than the pointer as the map contains pointers to the prepared statements?