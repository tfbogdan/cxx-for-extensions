# Proposed extensions to C++ `for` loops

## Background

The introduction of _range based for loops_ to C++ has made for loops both easier to work with and much safer. Without further ado here's a short recap of what they are and what they do.

```C++
std::string my_string = "Hello World!!!";
for (std::string::const_iterator c = my_string.begin(); c != my_string.end(); ++c) {
    // Work with c
}

// Is now as simple as:
for(const auto c: my_string) {
    // Work with c
}
```

The second is preferable for at least the following reasons:

- More terse, easier on the eyes
- Less potential for mistakes, such as using begin and end iterators from different containers

To many programmers, making something both easier and safer to use at once is all it takes to embrace a new technology so fast forward a few years, range based for loop is the idiomatic way of iterating through containers now.

## Limitations

The same features that make the range based for loop safe to use also make it hard to control the iteration at a finer level. The only tools you have as a programmer are `continue` and `break` statements and while very useful on their own, they are low level primitives and need to be surrounded by a fair amount of code.

### Sub-range iteration

A tricky thing to do with range based for loops but also with it's precursors is iterating through a part of the container in a way that differs from the general loop. To illustrate this limitation, the following code snippet does a naive parse of a string, filtering out the parts that are parenthesized.

```C++
std::string_view the_string = "This is a text (not very long) that is to serve as an example(perhaps not the best one)";
std::ostringstream output;
// Since a range based for loop does not expose an iterator it cannot be used for this task
for (auto c = the_string.begin(); c != the_string.end(); ++c) {
    if (*c != '(') {
        output << *c;
    } else {
        while (c != the_string.end() && *c != ')') {
            ++c;
        }
        if (c == the_string.end()) {
            throw std::runtime_error("Expected ')' found end of string.");
        }
    }
}

assert(output.str() == "This is a text  that is to serve as an example");
```

## Proposed solution

Given the following informal definition of the `for` loop:

```C++
for ( init /*optional*/; condition /*optional*/; iterate /*optional*/) statement
```

It's flow can be broken down to:

1. run `init` expression if found
2. run `condition` if found. If not found or if it evaluates to `true`, continue to step 3. Otherwise, jump straight to step 6.
3. run `statement`.
4. run `iterate`
5. go back to step 2.
6. out of the loop

This is it's equivalent `while` statement:

```C++
init;
while(condition) {
    statement;
    iterate;
}
```

And the acknowledgement that the range based for loop is simply syntactic sugar for the plain for with the compiler filling in all three expressions, we can do the following:

### `continue while`

The `continue while` statement is defined as:

```C++
for ( init /*optional*/ ; condition /*!!not optional!!*/; iterate /*!!not optional!!*/ ) {
    preceding_stmt
    continue while (condition2) inner_statement catch /*optional*/ catch_stmt else /*optional*/ else_stmt
    succeeding_stmt
}
```

The `continue while` statement may only be used inside a for loop as long as the following constraints are met:

- The outer `for` **has a condition expression**
- The outer `for` **has an iteration expression**

This makes it implicitly useable within a range based for loop.

In it's presence, the `for` loop flow changes to:

1. run `init` if found.
2. run `condition`. If it evaluates to `true`, continue to step **3** otherwise jump to **12**.
3. run `preceding_stmt`.
4. run `condition2`. If `true`, continue to step **5** otherwise jump to **8**.
5. run `inner_statement`.
6. run `iterate`.
7. run `condition`. If true, proceed to step **4** otherwise jump to **10**
8. run `succeeding_stmt`.
9. run `iterate`. Go to step **2**.
10. run `else_stmt` if found.
11. run `catch_stmt` if found.
12. out of the loop.

It can also be translated to it's equivalent while loop:

```C++
init;
while(condition) {
    preceding_stmt
    while(condition && condition2) {
        inner_statement;
        iterate;
    }
    if (!condition || !condition2) {
        // The optional else block
        else_stmt;
        goto out_of_the_loop;
    }
    if (!condition2) {
        // The optional catch block
        catch_stmt;
        goto out_of_the_loop;
    }
    succeeding_stmt;
    iterate;
}
out_of_the_loop:
    // continue here
```

Notice how step **7** bypasses step **3** and jumps straight to **4**. This is intended as it allows all variables declared during `preceding_stmt` to stay in scope for the whole duration of the `continue while` loop.

The `else` statement of the `continue while` is executed when `condition` evaluates to `false` **or** when `condition2` evaluates to `false` allowing the inner loop to wrap up it's business in either case. When the outer loop is a _range based for loop_ this block may **not** access the loop variable as it will be out of range if the outer loop finished before the inner one.

The `catch` statement is executed when `condition` evaluates to `false`, that is, when the outer loop is done. It allows the inner loop to handle errors if `condition2` is expected to become `false`, that is, the inner loop is expected to finish iterating before the outer one. Just like the `else` block, when the outer loop is a _range based for loop_ this block may **not** access the loop variable.

This is compelling because:

- It can be used when the outer loop is a _range based for loop_ giving back some control over the iteration to the programmer.
- It works off the contract established by the outer loop. There is no need to duplicate the outer condition or the iteration expression in the inner loop. That makes it safer than hand written alternatives.
- It is more readable than hand written alternatives.
- Backwards compatible both syntactically and semantically.
- Built entirely on existing keywords.

Some drawbacks:

- Adds to the list of _magic_ things that the compiler does behind the curtain.
- Possibly requires hidden variables that the compiler would have to generate.
- `else` and `catch` are not the best keywords given their role in this proposal but they make it possible to do this without introducing new ones.

With this construct, we can rewrite the parser to look like this:

```C++
std::string_view the_string = "This is a text (not very long) that is to serve as an example(perhaps not the best one)";
std::ostringstream output;
// Range based for loop is suddenly back in the game
for (const auto c: the_string) {
    if (c != '(') {
        output << c;
    } else {
        continue while (c != ')') {
            // No need to duplicate the range check and the iteration expression.
        }  catch {
           // And we'll always be guaranteed to know when the outer loop finished before the inner one could do so
           throw std::runtime_error("Expected ')' found end of string.");
        }
    }
}

assert(output.str() == "This is a text  that is to serve as an example");

```

With this construct and `std::for_index`, we can rewrite the parser once more to become a bit more efficient:

```C++
std::string_view the_string = "This is a text (not very long) that is to serve as an example(perhaps not the best one)";
std::ostringstream output;
for (const auto c: the_string) {
    if (c != '(') {
        auto starting_index = std::for_index;
        continue while (c != '(') ;
        else output << the_string.substr(starting_index, std::for_index);
    } else {
        continue while (c != ')');
        catch throw std::runtime_error("Expected ')' found end of string.");
    }
}

assert(output.str() == "This is a text  that is to serve as an example");

```

### `continue for`

_To be defined_ but very similar to `continue while` with the added benefit of being able to nest an arbitrary level of `continue for` loops.

Copyright (c) Bogdan T. tfbogdan@gmail.com
