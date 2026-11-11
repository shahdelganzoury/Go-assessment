# Go Assessment for Junior Developers

This assessment aims to test junior developers' deep understanding of the Go language, moving beyond mere syntax to focus on mental models and how Go works under the hood. The assessment consists of 26 questions divided into four main sections.

1.  **Question:** When is memory allocated on the stack and when on the heap in Go? What are the performance implications?

   
    **Answer:** Memory is allocated on the **stack** for values whose lifetime and size can be determined by the compiler at compile time. These are typically local variables of functions. Memory is allocated on the **heap** for values whose lifetime or size cannot be determined at compile time, or which must persist after the function that created them has exited. This is determined by the compiler's escape analysis. Stack allocation is significantly faster than heap allocation because it does not require garbage collection and does not involve memory locking operations. Heap allocation increases pressure on the garbage collector, which can impact performance.

2.  **Question:** Explain how a slice is represented in memory in Go. What is the difference between a slice and an array?

   
    **Answer:** A slice in Go is a three-element data structure consisting of: a pointer to the underlying array where the data is stored, the length of the slice (the number of visible elements), and the capacity of the slice (the maximum number of elements the slice can hold without reallocating the underlying array). An array is a fixed-size sequence of elements of the same type, and its size is determined at compile time. A slice, unlike an array, is a dynamic type whose size can change, providing a flexible way to work with data sequences. Multiple slices can refer to the same underlying array or parts of it.

5.  **Question:** What is a potential problem when creating a slice from a larger slice if you are not careful? How can it be avoided?


    **Answer:** The potential problem is **unnecessary memory retention**. If you create a small slice from a larger one, the small slice will still refer to the same large underlying array. If the large slice is no longer used, the garbage collector will not be able to free the memory occupied by the entire underlying array as long as the small slice still refers to it. To avoid this, you can copy the required elements into a completely new slice using `copy` or by reallocating the new slice: `newSlice := make([]T, len(oldSlicePart)); copy(newSlice, oldSlicePart)`.

7.  **Question:** What is the impact of using pointers on Go's garbage collector?


    **Answer:** The use of pointers increases the complexity of the garbage collector's work. When a pointer refers to an object in memory, that object is considered reachable and cannot be garbage collected as long as the pointer still refers to it. If there are many pointers referring to different objects, or if there are long chains of pointers, the garbage collector needs to track these relationships to determine which objects are no longer in use and can be freed. This can increase the garbage collection (GC) cycle time and affect the application's latency.

9.  **Question:** How can a closure lead to a memory problem if not used carefully?


     **Answer:** A closure in Go is a function that can access variables from the scope in which it was defined, even after the original function has exited. The potential problem is **memory retention**. If the closure refers to large variables or complex data structures from the outer scope, and the closure itself is not released, these variables will not be garbage collected, leading to a memory leak over time. It is important to ensure that closures that are no longer needed are released or allowed to be garbage collected.

11. **Question:** Explain the importance of `go fmt` and `go vet` in Go development. What role do they play in maintaining code quality?


     **Answer:**
    *   **`go fmt`:** Is an automatic code formatting tool in Go. `go fmt` enforces a standard and unified formatting style for all Go code. This eliminates discussions about formatting style and makes the code more readable and understandable across different projects and developers. It contributes to maintaining code quality by ensuring consistency and clarity.
    *   **`go vet`:** Is a static analysis tool that searches for potential errors or suspicious constructs in Go code that the compiler might not catch but could lead to runtime errors. For example, it can detect typos in `fmt.Printf` format strings, or incorrect mutex usage. `go vet` contributes to code quality by identifying potential issues early in the development cycle, reducing bugs and improving program reliability.

### Q7
**Type:** Debug  
**Tag:** Slices — range over struct slice

The developer wants to double the `Value` field of every item. The code compiles but produces wrong output. Why and how to fix it?

```go
package main

import "fmt"

type Item struct {
	Value int
}

func main() {
	items := []Item{{1}, {2}, {3}}

	for _, item := range items {
		item.Value *= 2
	}

	fmt.Println(items[0].Value, items[1].Value, items[2].Value)
}
```

**Correct answer:**  
Range over a slice of structs gives a **copy** of each struct, not a reference. Modifying `item.Value` only modifies the local copy.

**Explanation:**  
Fix using the index:
```go
for i := range items {
	items[i].Value *= 2
}
```

---

### Q8
**Type:** Code Output  
**Tag:** Slices — append triggers reallocation in function

What does this code print?

```go
package main

import "fmt"

func addItem(s []int) []int {
	s = append(s, 100)
	s[0] = 999
	return s
}

func main() {
	original := make([]int, 2, 3)
	original[0] = 1

	original = append(original, 100)
	result := addItem(original)

	fmt.Println(original[0])
	fmt.Println(result[0])
}
```

**Correct answer:**
```
1
999
```

**Explanation:**  
`original` has len=3 and cap=3. When `addItem` calls `append(s, 100)`, the capacity is exceeded — Go allocates a new backing array for `s`. The function now modifies `s[0]` on the new array. `original` still points to the old array, unchanged.

---

### Q9
**Type:** Code Output  
**Tag:** Slices — nil vs empty in JSON

What does this code print?

```go
package main

import (
	"encoding/json"
	"fmt"
)

func main() {
	var a []int
	b := []int{}

	aj, _ := json.Marshal(a)
	bj, _ := json.Marshal(b)

	fmt.Println(string(aj))
	fmt.Println(string(bj))
}
```

**Correct answer:**
```
null
[]
```

**Explanation:**  
A nil slice marshals to `null` in JSON. An empty (but initialized) slice marshals to `[]`. This is a real production bug — APIs receiving `null` and `[]` may behave differently. Always initialize slices that will be JSON-encoded: `b := []int{}` or `b := make([]int, 0)`.

---

### Q10
**Type:** Debug  
**Tag:** Slices — pointer elements vs value elements

Why does `doubleValues` not work but `doublePointers` does?

```go
package main

import "fmt"

func doubleValues(nums []int) {
	for _, n := range nums {
		n *= 2
	}
}

func doublePointers(nums []*int) {
	for _, n := range nums {
		*n *= 2
	}
}

func main() {
	a := []int{1, 2, 3}
	doubleValues(a)
	fmt.Println(a)

	x, y, z := 1, 2, 3
	b := []*int{&x, &y, &z}
	doublePointers(b)
	fmt.Println(x, y, z)
}
```

**Correct answer:**
```
[1 2 3]
2 4 6
```

**Explanation:**  
In `doubleValues`, the slice header is passed by value. `n` in the range loop is a copy of each int. Changes to `n` don't affect the slice. In `doublePointers`, each `n` is a pointer to the original variable. `*n *= 2` dereferences and modifies the actual memory.

---

### Q11
**Type:** Code Output  
**Tag:** Slices — copy() behavior

What does this code print?

```go
package main

import "fmt"

func main() {
	src := []int{1, 2, 3, 4, 5}
	dst := make([]int, 3)

	n := copy(dst, src)

	fmt.Println(dst)
	fmt.Println(n)
	fmt.Println(src)
}
```

**Correct answer:**
```
[1 2 3]
3
[1 2 3 4 5]
```

**Explanation:**  
`copy` copies `min(len(dst), len(src))` elements — here 3. It returns the number of elements copied. The source slice is never modified. `dst` is now independent from `src`.

---

### Q12
**Type:** Conceptual  
**Tag:** Memory — escape analysis

This code works correctly and prints `42`. Doesn't `x` get destroyed when `newInt` returns?

```go
package main

import "fmt"

func newInt() *int {
	x := 42
	return &x
}

func main() {
	p := newInt()
	fmt.Println(*p)
}
```

Explain exactly what Go does here and why this is safe.

**Correct answer:**  
The Go compiler performs **escape analysis**. It detects that `x`'s address is returned from the function — meaning `x` outlives the stack frame of `newInt`. So the compiler allocates `x` on the **heap** instead of the stack. Heap memory persists until the garbage collector determines no pointers reference it. The returned pointer is valid and safe.

---

### Q13
**Type:** Conceptual  
**Tag:** Memory — new vs make with maps

What is wrong with this code? How does it relate to the difference between `new` and `make`?

```go
m := new(map[string]int)
(*m)["key"] = 1
```

**Correct answer:**  
`new(map[string]int)` allocates a `*map[string]int` pointing to a **nil map** (the zero value of a map). Writing to a nil map panics at runtime. `new` only zeroes memory — it does not initialize reference types.  
The correct approach:
```go
m := make(map[string]int)
m["key"] = 1
```
`make` allocates and initializes the internal hash table, making the map ready to use.

---


### Q14
**Type:** Code Output  
**Tag:** Pointers — independent heap allocations

What does this code print?

```go
package main

import "fmt"

func makeCounter() *int {
	count := 0
	return &count
}

func main() {
	c1 := makeCounter()
	c2 := makeCounter()

	*c1++
	*c1++
	*c2++

	fmt.Println(*c1, *c2)
}
```

**Correct answer:** `2 1`

**Explanation:**  
Each call to `makeCounter` creates a **separate heap-allocated int** (escape analysis promotes `count` to the heap). `c1` and `c2` are completely independent pointers. Incrementing `c1` twice and `c2` once gives `2` and `1`.

---

### Q15
**Type:** MCQ  
**Tag:** Maps — iteration order

What can you say with certainty about the output of this code?

```go
package main

import "fmt"

func main() {
	m := map[string]int{"a": 1, "b": 2, "c": 3}
	for k, v := range m {
		fmt.Println(k, v)
	}
}
```

**Choices:**

A. It always prints in alphabetical key order  
B. It prints in insertion order  
C. The iteration order is randomized on purpose in Go  
D. It will only print two of the three entries

**Correct answer:** C

**Explanation:**  
Go deliberately randomizes map iteration order to prevent developers from accidentally depending on it. Each run may produce a different order. If you need sorted output, collect keys into a slice, sort them with `sort.Strings`, and then iterate.

---

### Q16
**Type:** Debug  
**Tag:** Maps — struct values are not addressable

This code does not compile. Why, and what are the one ways to fix it?

```go
package main

import "fmt"

type Point struct{ X, Y int }

func main() {
	points := map[string]Point{"origin": {0, 0}}
	points["origin"].X = 10  // compile error
	fmt.Println(points["origin"])
}
```

**Correct answer:**  
Map values are **not addressable**. You cannot take `&points["origin"]` and therefore cannot assign directly to a field. Two fixes:

```go
// Fix 1 — get copy, modify, write back
p := points["origin"]
p.X = 10
points["origin"] = p

// Fix 2 — store pointers in the map
points := map[string]*Point{"origin": {0, 0}}
points["origin"].X = 10  // this works: pointer is addressable
```

---

### Q17
**Type:** Debug  
**Tag:** Maps — concurrent access

What is wrong with this code? What will happen at runtime?

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	m := make(map[int]int)
	var wg sync.WaitGroup

	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			m[n] = n * 2
		}(i)
	}

	wg.Wait()
	fmt.Println("done", len(m))
}
```

**Correct answer:**  
Multiple goroutines write to the map concurrently without synchronization — this is a **data race**. Go's runtime detects concurrent map writes and panics with `"concurrent map writes"`. Fix with a mutex:

```go
var mu sync.Mutex
// inside goroutine:
mu.Lock()
m[n] = n * 2
mu.Unlock()
```

Or use `sync.Map` which is designed for concurrent use.

---

### Q18
**Type:** Code Output  
**Tag:** Error handling — defer + named return values

What does this code print?

```go
package main

import (
	"errors"
	"fmt"
)

func divide(a, b float64) (result float64, err error) {
	defer func() {
		if err != nil {
			err = fmt.Errorf("divide failed: %w", err)
		}
	}()

	if b == 0 {
		err = errors.New("division by zero")
		return
	}
	result = a / b
	return
}

func main() {
	_, err := divide(10, 0)
	fmt.Println(err)
}
```

**Correct answer:** `divide failed: division by zero`

**Explanation:**  
Named return values (`result`, `err`) are accessible inside deferred functions. When `b == 0`, `err` is set and the function returns early. The deferred function runs before the return completes, sees `err != nil`, and wraps it with context. The caller receives the wrapped error.

---

### Q19
**Type:** Conceptual  
**Tag:** init — order across packages

You have three packages: `main` imports `A`, `A` imports `B`. Each has one `init()` function and several package-level variable declarations. Describe the complete initialization order.

**Correct answer:**  
1. All package-level variable declarations in `B` are evaluated (in dependency order)
2. `init()` in `B` runs
3. All package-level variable declarations in `A` are evaluated
4. `init()` in `A` runs
5. All package-level variable declarations in `main` are evaluated
6. `init()` in `main` runs
7. `main()` runs

Key rules: imported packages always initialize first. Within a package, multiple `init()` functions run in the order they appear in source files. `init()` cannot be called manually — it is invoked only by the Go runtime.

---

### Q20
**Type:** Conceptual  
**Tag:** Standard library — string building efficiency

Why is this code inefficient? What should be used instead and why is it faster?

```go
result := ""
for i := 0; i < 10000; i++ {
	result += fmt.Sprintf("item %d\n", i)
}
```

**Correct answer:**  
Strings in Go are **immutable**. Every `+=` creates a brand new string by allocating new memory and copying the old content — O(n²) total allocations and copies for a loop of n iterations.

Use `strings.Builder`:
```go
var sb strings.Builder
for i := 0; i < 10000; i++ {
	fmt.Fprintf(&sb, "item %d\n", i)
}
result := sb.String()
```

`strings.Builder` maintains a growable `[]byte` buffer internally. It only converts to a string once at the end. This is O(n) total work with minimal allocations.

---

### Q21
**Type:** Code Output  
**Tag:** Idiomatic Go — type switch

What does this code print?

```go
package main

import "fmt"

func describe(i interface{}) string {
	switch v := i.(type) {
	case int:
		return fmt.Sprintf("int: %d", v)
	case string:
		return fmt.Sprintf("string: %q", v)
	case []int:
		return fmt.Sprintf("slice of %d ints", len(v))
	default:
		return fmt.Sprintf("unknown: %T", v)
	}
}

func main() {
	fmt.Println(describe(42))
	fmt.Println(describe("hello"))
	fmt.Println(describe([]int{1, 2, 3}))
	fmt.Println(describe(3.14))
}
```

**Correct answer:**
```
int: 42
string: "hello"
slice of 3 ints
unknown: float64
```

**Explanation:**  
`v := i.(type)` is a type switch. In each `case`, `v` takes the concrete type of that branch — `int`, `string`, `[]int`. In `default`, `v` remains the interface type, and `%T` prints the actual runtime type name.

---

### Q22
**Type:** Conceptual  
**Tag:** Idiomatic Go — context cancellation

A web handler starts a database query in a goroutine. The client disconnects halfway through. Write the correct pattern to cancel the database query when the client disconnects, and explain what `ctx.Done()` returns and when it is closed.

**Correct answer:**

```go
func handler(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	resultCh := make(chan string, 1)

	go func() {
		result, err := db.QueryContext(ctx, "SELECT ...")
		if err != nil {
			return
		}
		resultCh <- result
	}()

	select {
	case result := <-resultCh:
		fmt.Fprintln(w, result)
	case <-ctx.Done():
		http.Error(w, "request canceled", http.StatusRequestTimeout)
	}
}
```

`ctx.Done()` returns a **receive-only channel** that is closed when the context is canceled or its deadline is exceeded. A closed channel is always readable — all goroutines waiting on it unblock immediately. `r.Context()` in an HTTP handler is automatically canceled when the client disconnects.

---

### Q23

**Type:** Debug

**Tag:** Methods — pointer vs value receivers

The developer wrote a method to increment a user's login count, but the count remains zero. Why does this happen and how do you fix it?

```go
package main

import "fmt"

type User struct {
    Name       string
    LoginCount int
}

func (u User) RecordLogin() {
    u.LoginCount++
}

func main() {
    user := User{Name: "Alice", LoginCount: 0}
    user.RecordLogin()
    fmt.Println(user.LoginCount)
}

```

**Correct answer:** The code prints `0`. The method `RecordLogin` has a **value receiver** (`u User`).

**Explanation:** When a method has a value receiver, Go passes a complete copy of the struct to the method. `u.LoginCount++` modifies the local copy inside the method, not the original `user` struct in `main`. To mutate the original struct, the method must use a **pointer receiver**: `func (u *User) RecordLogin()`.

---

### Q24

**Type:** Code Output

**Tag:** Interfaces — the nil interface trap

What does this code print? Is the error `nil`?

```go
package main

import "fmt"

type CustomError struct {
    Msg string
}

func (c *CustomError) Error() string {
    return c.Msg
}

func doWork() error {
    var err *CustomError = nil
    // ... some work that succeeds ...
    return err
}

func main() {
    err := doWork()
    if err != nil {
        fmt.Println("An error occurred!")
    } else {
        fmt.Println("Success!")
    }
}

```

**Correct answer:** 
An error occurred!



**Explanation:** This is the classic Go "nil interface trap". An interface in Go is represented under the hood as a tuple containing two pointers: `(type, value)`. An interface is only truly `nil` if **both** type and value are nil. 
When `doWork` returns, it packages the typed `nil` pointer into the `error` interface. The interface becomes `(type=*CustomError, value=nil)`. Because the type is not nil, `err != nil` evaluates to `true`. To fix this, explicitly return `nil` when there is no error: `return nil`.

---

### Q25
**Type:** Debug  
**Tag:** Concurrency — goroutine leaks and channels

A web handler relies on this function to fetch data with a timeout. If the database is consistently slow and hits the timeout, what happens to the application's memory over time?

```go
func fetchData() (string, error) {
    ch := make(chan string)

    go func() {
        // Simulating a slow database query
        time.Sleep(2 * time.Second)
        ch <- "data retrieved"
    }()

    select {
    case res := <-ch:
        return res, nil
    case <-time.After(1 * time.Second):
        return "", errors.New("timeout")
    }
}

```

**Correct answer:** The application will suffer from a **goroutine leak**, eventually exhausting memory.

**Explanation:** The channel `ch` is **unbuffered**. If the timeout is reached first, `fetchData` returns and the `select` statement is abandoned. However, the background goroutine is still running. When it finally finishes sleeping, it tries to send `"data retrieved"` to `ch`. Because the channel is unbuffered and there are no longer any receivers, the send operation **blocks forever**. The goroutine can never exit and cannot be garbage collected.
Fix: Use a buffered channel `ch := make(chan string, 1)` so the goroutine can complete its send and exit, even if nobody is reading from it.

---

### Q26
**Type:** Code Output  
**Tag:** Defer — argument evaluation and execution order

What does this code print?

```go
package main

import "fmt"

func main() {
    i := 1

    defer fmt.Println("A:", i)
    
    defer func() {
        fmt.Println("B:", i)
    }()

    i = 2
}

```

**Correct answer:** 
B: 2
A: 1


**Explanation:** There are two rules at play here:
1. **LIFO Execution:** Deferred functions are executed in Last-In, First-Out order. The second defer (B) runs before the first (A).
2. **Argument Evaluation:** Arguments passed directly to a deferred function are evaluated *immediately* when the defer statement is encountered. For "A", `i` is evaluated as `1` immediately. For "B", `i` is not passed as an argument; it is part of a closure. The closure evaluates `i` when it *executes* at the end of the function, by which point `i` has been reassigned to `2`.

---
