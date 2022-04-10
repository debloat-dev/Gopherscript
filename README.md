# Gopherscript

Gopherscript is a secure scripting/configuration language written in Go. 
It features a fined-grain permission system and enforces a strong isolation of dependencies.
Gopherscript is not production ready yet : if you find a bug or want to suggest a feature create an issue please !
See [here](#installation) for installation & usage.

## Security & Minimalism

- The codebase is small on purpose (a single Go file with less than 6K lines and only std lib dependencies)
- The default global scope has ZERO variables/functions. (only add what you need)
- A strict permission system allows you to precisely control what is allowed (almost no permissions by default). 
  For more details go to the [permission section](#permissions).
- Paths, path patterns, URLs are literals and dynamic paths are only possible as path expressions. You cannot create them from strings at runtime ! That facilitates the listing of permissions and helps static analysis.
- Properties cannot be accessed with a dynamic name ( ``$obj[$name]`` ), only Go functions that are passed objects in can (you have to trust them anyway).

## Features

### Basic

![image](https://user-images.githubusercontent.com/84961291/162411233-e183312b-3cb3-424c-baf0-ad32d3ab87b3.png)

<!--

$integer = 1 
$float = 1.0

if true {
    log 1 2
} else {
    log(3,4)
}

$slice1 = ["a", "b", 3]
$slice2 = [
    "a" 
    "b"
    3
]

$object = {name: "Foo"}
-->

### Permissions

Required permissions are specified at the top of each module (file).

![image](https://user-images.githubusercontent.com/84961291/162411317-58513db5-87c3-4f9d-a5f2-7204a0951303.png)

<!--
require {
    read: {
        # access to all HTTPS hosts (GET, HEAD, QUERY)
        : https://*

        # access to all paths prefixed with /home/project
        : /home/project/...
    }
    use: {
        globals: "*"
    }
    create: {
        globals: ["myvar"]
        : https://* # POST to all HTTPS hosts
    }
}
-->


There are several permission kinds: Create, Update, Read, Delete, Use, Consume, Provide.
Some permission types are already provided: FilesystemPermission, HttpPermission, StackPermission, GlobalVarPermission.
You can specify your own permissions by implementing the Permission interface

```go
type Permission interface {
	Kind() PermissionKind
	Includes(Permission) bool
}
```

### Special literals & expressions

Path literals

![image](https://user-images.githubusercontent.com/84961291/162411824-f87ffda1-7de4-4eca-8e9a-1e9329964d88.png)

<!--
/home/user/
./file.json
-->

Path patterns support basic globbing (*, \[set], ?) and prefixing (not both at the same time).

![image](https://user-images.githubusercontent.com/84961291/162411903-41f33536-db98-420f-a0de-ab433380c5b8.png)

<!--
./data/*.json
/app/logs/...

/app/*/...      # invalid (this might be allowed in the future though)
-->

HTTP host literals:

![image](https://user-images.githubusercontent.com/84961291/162412319-a18945af-987d-4d04-982a-7d1a51313021.png)

<!--
https://example.com
https://example.com:443
-->

HTTP host pattern literals:

![image](https://user-images.githubusercontent.com/84961291/162412383-dcacd28e-bd12-4fbe-94be-036cdf14c7b5.png)

<!--
https://*               # any HTTPS host
https://*.com           # any domain with .com as TLD, will not match against subdomains
https://*.example.com   # any subdomain of example.com
-->


URL literals:

![image](https://user-images.githubusercontent.com/84961291/162412439-4049ba90-644b-4dda-bc87-cee49873047b.png)

<!--
https://example.com/
https://example.com/index.html
https://localhost/
-->

URL pattern literals (only prefixs supported):

![image](https://user-images.githubusercontent.com/84961291/162470615-be697feb-8c21-4e17-89fd-30e503214b0e.png)

<!--
https://example.com/users/...
-->

### Gopherscript types

Basic types 

```go
string, bool, int, float64
```

Special string types
```go
type Path string
type PathPattern string
type URL string
type HTTPHost string
type HTTPHostPattern string
type URLPattern string
```

Complex types
```go
type Object map[string]interface{}
type List []interface{}
type KeyList []string
type Func Node
type ExternalValue struct {
	state *State
	value interface{}
}
```

Any other type will be considered as a "Go value".

### Functions

Functions can be declared with the following syntax.
```
fn f($x){
    return ($x + 1)
}
$y = f(1)
```

you can also use function expressions:
```
$f = fn(){
    log "hello"
}
$f()
```

Declared functions can be called without parenthesis:

```
myfunc 1 { }
```

```
myfunc 1 { 

}
```

You can make several calls on a single line by using a semicolon.

```
f 1 { }; f 2 { }
```

### Calling Go functions & Go values

When you create the initial global state you can expose Go functions or values.
Go values & functions are "wrapped" in a reflect.Value. Reflection is used when calling Go functions or accessing fields.

```go

type User struct {
    Name string
}

...

gos.NewState(ctx, map[string]interface{}{
    "user": User{Name: "Foo"},
    "makeUser": func(ctx *gos.Context) User {
        return User{Name: "Bar"}
    },
    /* Golang values are unwrapped before being passed to a Go function.
       Functions results that are not valid Gopherscript values are wrapped.
       If a Go function returns multiple results there will be returned in a List{}.
    */
    "hasName": func(ctx *gos.Context, user User, name string) bool {
        return user.Name == name
    },
})

```

Gopherscript:

```
$user = makeUser()
$user.Name              # "Bar"
hasName($user, "Bar")   # true

$$user                  # User{Name: "Foo"}
```

### Imports

Syntax:
```
import <varname> <url> <file-content-sha256>  { <global variables> } allow { <permission> }
```

Importing a module is like executing a script with the passed globals and granted permissions.

```
# content of https://example.com/return_1.gos
return 1
```

Script:

![image](https://user-images.githubusercontent.com/84961291/162414629-e6426d1c-e135-4bbf-aee6-1b778817cbf6.png)

<!--
import modresult https://example.com/return_1.gos "SG2a/7YNuwBjsD2OI6bM9jZM4gPcOp9W8g51DrQeyt4=" {MY_GLOBVAR: "a"} allow {}
-->

### Routines

Routines are mainly used for concurrent work and isolation. Each routine has its own goroutine and state.

Syntax for spawning routines:
```
$routine = sr [group] <globals> <module | call | variable>
``` 

Call (all permissions are inherited).
```
$routine = sr nil f()
```

Embedded module:

![image](https://user-images.githubusercontent.com/84961291/162414775-75d0562c-0e99-402f-8a66-b85fdb730a09.png)

<!--
$routine = sr {http: $$http} {
    return http.get(https://example.com/)!
} allow { 
    use: {globals: ["http"]} 
}
-->

You can wait for the routine's result by calling the WaitResult method:
```
$result = $routine.WaitResult()!
```

Routines can optionally be part of a "routine group" that allows easier control of multiple routines. The group variable is defined (and updated) when the spawn expression is evaluated.

```
for (1 .. 10) {
    sr req_group nil read(https://jsonplaceholder.typicode.com/posts/)!
}

$results = $req_group.WaitAllResults()!
```

### Objects

Objects are just map\[string]interface{}.

![image](https://user-images.githubusercontent.com/84961291/162415297-c0f9fe11-6976-4d1a-87e9-5a0754004978.png)

<!--
$object = {   
    # properties can be separated with commas
    count: 0, duration: 10ms

    #or this way
    a: 1 
    b: 2

    # or just with space (this might be removed in the future though)
    c: 3 d: 4

    # implicit-key properties, implicit keys starts at 0
    : "a"
    : "b"
}
-->

Objects with at least one implicit-key property are also given an additional property
to represent the "length" of the object : "__len" which has a value of type ``int``.

### Loops & Iterables

Iteration of a ``List``:
```
for $i, $e in ["a", "b", "c"] {
    # 0 "a"
    # 1 "b"
    # 2 "c"
}
```

Iteration of an ``Object``:
```
for $k, $v in {:"a", :"b", name: "Foo", "count": 10} {
    # note: iteration order is random
    # "0" "a"
    # "1" "b"
    # "name" "Foo"
    # "count" 10
    # "__len" 2
}
```

The ``for .. in`` loop can also iterate through Go values that implement the following Iterable interface.

```go
type Iterable interface {
	Iterator() Iterator
}

type Iterator interface {
	HasNext() bool
	GetNext() interface{}
}
```


The IntRange type implements this interface so it is iterable:

```
$iterable = (0 .. 2) # use ..< for exclusive end

for $i, $e in $iterable {
    # 0 0
    # 1 1
    # 2 2
}
```

### Binary expressions

Examples:
```
(1 + 2)
(1.0 +. 3.5)
(1 == 3)
("name" keyof {"name": "Foo"})
(1 in [1, 2, 3])
(0 .. 2)
(0 ..< 2)
```

Noote: Binary expressions are always expressed inside parenthesis.

### Quantity literals

```
10s
10ms
10%

sleep 100ms
```

## Installation

As said in the security section, Gopherscript is very minimal. You can use it as a library and only add whay you need to the global scope (see following example).\
You can also use the ``gos`` executable, it allows you to execute scripts and provides a REPL. See the documentation [here](./gos.md).

```go
package main

import (
	gos "gopherscript"
	"log"
)

type User struct {
	Name string
}

func main() {
	grantedPerms := []gos.Permission{
		gos.GlobalVarPermission{gos.UsePerm, "*"},
	}
	ctx := gos.NewContext(grantedPerms)
	state := gos.NewState(ctx, map[string]interface{}{
		//initial globals
		"makeUser": func(ctx *gos.Context) User {
			return User{Name: "Bar"}
		},
	})

	mod, err := gos.ParseAndCheckModule(`
            # permissions must be requested at the top of the file AND granted
            require {
                use: {globals: "*"} 
            }
            $a = 1
            $user = makeUser()
            return [
                ($a + 2),
                $user.Name
            ]
    `, "")
	if err != nil {
		log.Panicln(err)
	}

	res, err := gos.Eval(mod, state)
	if err != nil {
		log.Panicln(err)
	}

	log.Printf("%#v", res)
}


```

## Implementation

- Why use a tree walk interpreter instead of a bytecode interpreter ?\
  -> Tree walk interpreters are slower but simpler : that means less bugs and vulnerabilities. A Gopherscript implementation that uses a bytecode interpreter might be developed in the future though.

## Executables

When something tries to do too many things it becomes impossible to understand for the average developper. That makes audits and customization harder.
Goperscript aims to have a different model. Depending on the features you need, you install one one more executables that try to do one thing well without uncessary bloat (each one providing specific globals to Gophercript).