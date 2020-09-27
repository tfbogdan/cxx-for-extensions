# Proposed extensions to C+ `for` loops

## Background

The introduction of _range based for loops_ (_**RBFL**_ _for the rest of this document_) to C++ has made for loops both easier to work with and much safer. Without further ado here's a short recap of what they are and what they do.

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

To many programmers, making something both easier and safer to use at once is all it takes to embrace a new technology so fast forward a few years, _**RBFL**_ is the idiomatic way of iterating through containers now.

## Limitations

There still are plenty of situations though where using the _**RBFL**_ is undesirable or insufficient on it's own. A few samples illustrating a few such situations will be listed below.

### Assessing how far in the loop we are

On a whim, one might be tempted to assume that `std::distance` could be used to assess how far in the container we are at a given cycle through the loop. However, since the underlying iterator itself cannot be accessed, one would quickly realize this wouldn't work. So even if the code below is invalid and wouldn't compile we'll write it down:

```C++
std::string my_string = "Hello again!!!";
for(const auto c: my_string) {
    auto index = std::distance(my_string.begin(), c); // Whoops, c is not an iterator
}
```

This is not a criticism of the decision to not expose the underlying iterator though. On the contrary, that decision was a very sound one and it contributed towards making _**RBFL**_ a very robust construct.

To solve this problem, usually one needs to introduce another variable in the outer scope to keep track of an index and increment that in the body of the loop:

```C++
std::string my_string = "Uhm.. Hello!!!";
auto index = 0;
for(const auto c: my_string) {
    ++index;
}
```

This is a low cost to pay to keep track of the index in this context but not all loops are as short as the one presented above. Some of them are complex and are sprinkled with `continue` statements that make maintaining an index a bug-prone task. Whether loops in general should be complex or not is out of scope for this discussion. One could argue that they shouldn't be and many of them could probably be simplified with some judicious restructuring but the status quo is that sometimes they just are.

Another way to check for an index that will only work for contiguous containers is:

```C++
std::string_view my_string = "Bit tired of saying hello and getting nothing back but hello!!!";
for (const auto& c: my_string) {
    auto index = &c - &my_string.front();
}
```

Since it's use is limited to contiguous containers and makes use of knowledge of the internals of the container it is less than ideal.

Eventually, just like it can be argued that complex loops should be simplified it could also be argued that relying on an index can be avoided but while that that might be true, it's also true that having it can make some things easier.

### Sub-range iteration

Another tricky thing to do with range based for loops but also with it's precursors is iterating through a part of the container in a way that differs from the general loop. To illustrate this limitation, the following code snippet does a naive parse of a string, filtering out the parts that are parenthesized.

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

### `std::for_index` and `std::for_sindex`

The first limitation can easily be overcome by having a special variable whose name could _in lack of a better name right now_ be `std::for_index` that the compiler would **_always_** increment at the iteration expression stage, when the compiler normally increments the iterator. This would be of `std::size_t` type. For completeness sake and other reasons, an `std::ssize_t` alternative is provided too.

Their formal definition is:

```C++
const std::size_t &std::for_index;
const std::ssize_t &std::for_sindex;
```

And they could simply be used inside any for loop:

```C++
auto vec = std::vector {2, 3, 4};
for (auto v: vec) {
    std::cout << std::for_index << '\n';
}
```

This variable will be const in the body of the loop and it would always be the index inside the parent loops closest to where the variable is used. to be consistent with C++'s goal of only forcing you to pay for what you use, it would only be introduced by the compiler when used.

It's declared in the `std` namespace to avoid clashes thus making it's fairly backwards compatible.

### For loop extensions

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

#### `continue while`

The `continue while` statement has the following definition:

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

#### `continue for`

_To be defined_ but very similar to `continue while` with the added benefit of being able to nest an arbitrary level of `continue for` loops.

Copyright (c) Bogdan T. tfbogdan@gmail.com
