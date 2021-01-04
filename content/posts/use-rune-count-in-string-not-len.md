+++
title = "If you use 'len(s)' to check a string's length, YOU'RE DOING IT WRONG!!"
author = ["Pavit Kiatkraipob"]
date = 2021-01-03T00:00:00+07:00
draft = false
+++

Maybe `len(s)` doesn't work as you expect. If you use it to determine the length of the string, you must read this post!


## One-line Summary {#one-line-summary}

Use `utf8.RuneCountInString(s)` instead when you want to count the number of characters in `s`.


## What happens if I use `len(s)` {#what-happens-if-i-use-len--s}

Let's start with using `len(s)` to determine string's length of `A`:

```go
package main

import "fmt"

func main() {
      fmt.Println(len("A"))
}
```

Output:

```go
1
```

The result is 1! which is what we expects. But what if you use other UTF-8 character, is the behavior of `len(s)` still be the same?

```go
package main

import "fmt"

func main() {
      fmt.Println(len("ก"))
}
```

Output:

```go
3
```

`ก` is a single UTF-8 thai characters. But somehow, `len("ก")` gives us `3`! Is `len(s)` is broken? Let's see...


## What is happening? {#what-is-happening}

According to [Go `builtin` document](https://golang.org/pkg/builtin/#len), `func len(v Type) int` returns the number of **BYTES** in `v` if `v` is `String`. And because Go's [source code is UTF-8](https://blog.golang.org/strings), `len(s)` will returns the number of bytes when encoding `s` using UTF-8. So, let's see the bytes representation of `A` and `ก`, is it really UTF-8?:

```go
package main

import "fmt"

func main() {
      fmt.Printf("UTF-8 bytes representation of %s: % x\n", "A", "A")
      fmt.Printf("UTF-8 bytes representation of %s: % x", "ก", "ก")
}
```

Output:

```go
UTF-8 bytes representation of A: 41
UTF-8 bytes representation of ก: e0 b8 81
```

From the result above, `A` is composed of one byte, `41`, and `ก` is composed of three bytes, `e0`, `b8` and `81` (these numbers are hex-encoded). If you look through the complete list of UTF-8 charset [here](https://www.fileformat.info/info/charset/UTF-8/list.htm), you will see that the bytes representation above match the characters `A` and `ก`.
![](/ox-hugo/image_1.png)
![](/ox-hugo/image_2.png)
So, if the string in Go's source code is acutally UTF-8 and `len(s)` actually return the number of bytes not the number of the character in string, what should we use then?


## Here's come utf8 package {#here-s-come-utf8-package}

Luckily, Go standard packages contains `utf8` which used to deal with UTF-8 charset. If you take a look at its document page [here](https://golang.org/pkg/unicode/utf8/#RuneCountInString), you see that there is a function called `RuneCountInString(s)`. This function iterates the bytes of the given string and count the UTF-8 character. Let's use `RuneCountInString(s)` on `A` and `ก` and see whether it give us the result we expects:

```go
package main

import (
      "fmt"
      "unicode/utf8"
)

func main() {
      fmt.Printf("Output from utf8.RuneCountInString(`A`): %d\n", utf8.RuneCountInString("A"))
      fmt.Printf("Output from utf8.RuneCountInString(`ก`): %d", utf8.RuneCountInString("ก"))
}
```

Output:

```go
Output from utf8.RuneCountInString(`A`): 1
Output from utf8.RuneCountInString(`ก`): 1
```

The output from `utf8.RuneCountInString(s)` is what we expect!


## Summary {#summary}

So, if you're reading til here, please use `utf8.RuneCountInString(s)` instead `len(s)` when you want to count the number of characters in `s` because

1.  `len(s)` counts the number of bytes in `s`
2.  Go's treats string as UTF-8 character
3.  Some UTF-8 character can compose of more than one byte!
