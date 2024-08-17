# Gormless

This is such an obvious idea that I'm sure it's been done, but I can't find it.
Gorm Gen will help get you some of the safety that this gives, but this gives it 
to you with Gorm itself (all the same capabilities, but with a couple minor 
changes).

# The Problem

If you read the Gorm documentation carefully (and you absolutely should because
of this problem), you will find the [Security](https://gorm.io/docs/security.html)
page. On it, it is revealed that SQL injection is considered a feature of Gorm.

That's not, of course, how the devs see it, but I believe they are simply wrong.
SQL injection is an obvious issue with a method like `db.Where` or `db.Raw`
where you know you're writing a raw SQL query. But what you don't expect is
something like this to be vulnerable:

```go
// BAD CODE DO NOT USE!!!

package main

import (
    "fmt"
	"os"
	
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

type User struct {
    ID   int
    Name string
}

func main() {
	if len(os.Args) != 2 {
        fmt.Println("usage: ", os.Args[0], " <id>")
        os.Exit(1)
	}
	
    db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})
    if err != nil {
        panic("failed to connect database: " + err.Error())
    }
	
	// VERY BAD CODE HERE!!! DO NOT USE!!!
	var u User
    err = db.First(&u, os.Args[1]).Error // <=== VULNERABILITY HERE
    if err != nil {
        panic("failed to read user: " + err.Error())
    }	
}
```

Run this like so and you'll see the problem:

```sh
go run main.go "1; drop table users"
```

The most strident commentary when this issue is raised
was, ["Absolutely, this is bad API design, but it's not a vulnerability."](https://github.com/go-gorm/gorm/issues/2517#issuecomment-638459166). The Gorm devs' solution
was to provide the security page linked above. That's not a solution. That's
negligence. I've given talks on the OWASP Top 10 for more than a decade. This
exact problem remains one of the top issues in applications today for all this
time because devs don't take it seriously.

Regarding this being bad API design and not a vulnerability, I quote Dwight 
Schrute, "False." It ceases to be bad API design when you realize 
that the various `First`, `Find`, etc. methods actually look at the input and 
try to magically decide how to treat the input. For example,

 * If the input is an `int`, it turns into something like `id = <input>`. 
 * If it's a `string` containing a "?", it treats it as a prepared statement with bindings. 
 * If it's some other string, it might treat it as the `id = <input>` case or it might
treat it as a raw SQL query. 

That is the absolute definition of a SQL injection vulnerability. If the input 
is safe some of the time, but not all of the, that's SQL injection. They have
created the illusion of safety. Period. There's not even an opinion, but clear 
fact, as far as I'm concerned.

To me, part of the purpose of using an ORM rather than raw SQL and a language 
like Go instead of C is to avoid footguns as big as this one. So...

# The Solution

Since the authors of Gorm seem unwilling to fix this or to even admit that need
a v3 that obliterate this "bad API design." I have created this library named
Gormless. I'll you interpret the name as you will.

This is a straight up wrapper of `gorm.DB` that adds in the powers
of `google.com/go-safeweb/safesql`
and neuters the unsafe syntax. If you want to get back to the unsafe syntax,
you can call `Unsafe()` on the `Gormless` object.

This package is built using code generation using reflection, so it should be
able to keep up with Gorm's changes. If it doesn't, please file an issue.

# Installation

```sh
go get github.com/zostay/gormless
```

# Usage

```go
package main

import (
    "fmt"
    "os"
    
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
    "github.com/zostay/gormless"
)

type User struct {
    ID   int
    Name string
}

func main() {
    if len(os.Args) != 2 {
        fmt.Println("usage: ", os.Args[0], " <id>")
        os.Exit(1)
    }
    
    unsafeDB, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})
    if err != nil {
        panic("failed to connect database: " + err.Error())
    }
    
    db := gormless.New(unsafeDB)
    
    var u User
    err = FirstID(db, &u, os.Args[1]).Error
    if err != nil {
        panic("failed to read user: " + err.Error())
    }   
	
    // OR the above could be written using google.com/go-safeweb/safesql
	// err = db.First(&u, safesql.New("id = ?"), os.Args[1]).Error
}
```

# License

Copyright © 2024 Qubling LLC

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the “Software”), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
the Software, and to permit persons to whom the Software is furnished to do so,
subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
