# Gormless

This is such an obvious idea that I'm sure it's been done, but I can't find it.
Gorm Gen will help get you some of the safety that this gives, but this gives it 
to you with Gorm itself (all the same capabilities, but with a couple minor 
changes).

# The Problem

If you read the Gorm documentation carefully (and you absolutely should because
of this problem), you will find the [Security](https://gorm.io/docs/security.html)
page. On it, it is revealed that SQL injection is considered a feature of Gorm.

For a reminder of what SQL injection is, see XKCD's [Exploits of a Mom](https://xkcd.com/327/).

![A comic about a mom who destroys the school database because she named her boy "Robert'); DROP TABLE Students;--"](https://imgs.xkcd.com/comics/exploits_of_a_mom.png)

The gorm devs do not consider this to be a vulnerability, but they are simply 
wrong. SQL injection is an obvious issue with a method like `db.Where` or 
`db.Raw` where you know you're writing a raw SQL query. If these were the
methods we were talking about, I have concerns to make sure devs use them
carefully, but I wouldn't say they are inherently vulnerable.

However, if you have a method that is safe under some circumstances, but not
under others, that is a vulnerability. Consider the following program:

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

    fmt.Println("Name:", u.Name)
}
```

Run this like so and you'll see the problem:

```sh
go run main.go "1; drop table users"
```

The most strident commentary when this issue is raised
was, ["Absolutely, this is bad API design, but it's not a vulnerability."](https://github.com/go-gorm/gorm/issues/2517#issuecomment-638459166). The Gorm devs' solution
was to provide the security page linked above. That's not a solution. That's
negligence. I've given talks on
the [OWASP Top 10](https://owasp.org/www-project-top-ten/) for more than a
decade. This
exact problem, code injection generally and SQL injection particularly, remains
one of the top issues in applications today for all this time because devs like
those working on Gorm have continually avoided taking it seriously.

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
created the illusion of safety. Period. That's not even an opinion, but clear 
fact, as far as I'm concerned.

To me, part of the purpose of using an ORM rather than raw SQL and a language 
like Go instead of C is to avoid footguns as big as this one. So...

# The Solution

Since the authors of Gorm seem unwilling to fix this or to even admit that need
a v3 that obliterate this "bad API design." I have created this library named
Gormless. I'll let you interpret the name as you will.

This is a straight up wrapper of `gorm.DB` that adds in the powers
of `google.com/go-safeweb/safesql`
and neuters the unsafe syntax. If you want to get back to the unsafe syntax,
you can call `Unsafe()` on the `gormless.DB` object.

This package is built using code generation using reflection, so it should be
able to keep up with Gorm's changes. If it doesn't, please file an issue.

This solution is absolutely overkill. It's a sledgehammer to a nail. All that
really needs to be fixed to resolve the major issue is the `First`, `Find`, and
other methods that might be safe sometimes, but not all the time. On the other
hande, `safesql` is a brilliantly simple solution to add in, so why not?

# Patches Welcome

If there's something you cannot do with this library in place without
calling `Unsafe()` that you believe is safe. Please submit a PR and I will 
happily add it in, so long as it doesn't reintroduce the vulnerability.

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
    "github.com/google/go-safeweb/safesql"
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
    err = db.First(&u, safesql.New("id = ?"), os.Args[1]).Error()
    if err != nil {
        panic("failed to read user: " + err.Error())
    }   

    fmt.Println("Name:", u.Name)
}
```

# Callback Support

This library is unable to support the `gorm.DB.Callback` method. This is due to
the fact that that method returns a private type, so it can't be wrapped in a
straight-forward fashion. Anyway, I can think of a few workarounds, but I just
want to highlight here that lack of callback support is deliberate at this time.

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
