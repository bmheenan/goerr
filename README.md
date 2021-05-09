# Go error handling proposal

## Concise, with more obvious control flow

This proposal suggests a modfication to the [try built-in](https://go.googlesource.com/proposal/+/master/design/32437-try-builtin.md). If you haven't already, read that proposal first; this proposal builds off it. Here, we'll aim to:

* make code slightly more concise
* make it more obvious at a glance where code execution may end in a function
* keep go's principles around error handling intact
* change as little as possible, without breaking existing code

Let's just start with what it looks like:

```
// Adds two integers given as strings
func strAdd(a, b string) (sum string, err error) {
    defer fmt.Wrapf(&err, "Sum strings '%v' and '%v'", a, b)
    try sum = fmt.Sprintf("%d", strconv.Atoi(a)? + strconv.Atoi(b)?)
    return
}
```

What's happening in that code? Let's look at the `try` line first, then the `defer` line above it.

## `try` at the front, `?` to be explicit

Inside a function that returns an error as its last return value, a statement may begin with the keyword `try`. When it does, the statement takes the form: `try <abandonable statement>`. An abandonable statement is a lot like any other statement, except it gets an additional superpower: When you make a function call to a function that returns an error as its last return value, you can add a `?` to the end of the function call. Doing so has the same effect as the `try()` built-in from the original proposal.

Why is this better?

* Bringing `try` to the front of the line makes it easier to read at a glance, and see where code execution may cease in a function. Just as `return` and `panic` stick out, so should `try`. Compare:
    ```
    f := try(os.Open(filename))
    ```
    ```
    try f := os.Open(filename)?
    ```
* In cases where you need multiple tries in a single statement, it requires less typing and reads better. Compare:
    ```
    sum = fmt.Sprintf("%d", try(strconv.Atoi(a)) + try(strconv.Atoi(b)))
    ```
    ```
    try sum = fmt.Sprintf("%d", strconv.Atoi(a)? + strconv.Atoi(b)?)
    ```

Why even have the `?` then? We could simply allow abandonable statements to include expressions that drop the final (error) return value silently. However, one of go's strengths is forcing thought around error handling. The programmer must still think about each case and decide whether they want to handle an error explictly or simply return (and possibly wrap) the error. If a programmer forgets that a function may return an error, we want the complier to remind them.

## Handling errors through `defer`

In the spirit of changing as little as necessary, not much has changed here over the original proposal.

The standard helper function should allow the programmer to choose to wrap the error (like `fmt.Errorf("%w", err)`) or not (like `fmt.Errorf("%v", err)`). So, this proposal suggests having two functions within `fmt`:

* `fmt.Handlef` has the same behavior as `fmt.HandleErrorf` from the original proposal
* `fmt.Wrapf` is almost the same, but wraps the existing error, as if using the "%w" verb instead of "%v"
