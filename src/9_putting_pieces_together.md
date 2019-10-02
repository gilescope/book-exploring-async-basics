# Putting the pieces together

The first step to get this to run is obvious. We need a main function. Fortunately
it's a short one:

```rust
fn main() {
    let rt = Runtime::new();
    rt.run(javascript);
}
```

We instanciate our runtime, and we pass inn a pointer to our `javascript` function.

The next parts are just helper methods to let us print out interesting things
about our `Runtime` while it's running.

For those new to Rust I'll spend the time to explain them anyway:

```rust
fn current() -> String {
    thread::current().name().unwrap().to_string()
}
```
The `current` method prints out the name of the current thread it's called from.

Since we have several threads handling tasks for us, this will help us keep track
of what goes on where.


```rust
fn print(t: impl std::fmt::Display) {
    println!("Thread: {}\t {}", current(), t);
}
```

This function takes an argument that implements the `Display` trait. This trait
defines how we want a type to displayed as text. We call our `current` function to
get the name of the current thread and outputs it `stdout`.

The next function us a bit of an introduction to iterators. When we have content
to print we use this one:

```rust
fn print_content(t: impl std::fmt::Display, descr: &str) {
    println!(
        "\n===== THREAD {} START CONTENT - {} =====",
        current(),
        descr.to_uppercase()
    );

    let content = format!("{}", t); 
    let lines = content.lines().take(2);
    let main_cont: String = lines.map(|l| format!("{}\n", l)).collect();
    let opt_location = content.find("Location");

    let opt_location = opt_location.map(|loc| {
        content[loc..]
        .lines()
        .nth(0)
        .map(|l| format!("{}\n",l))
        .unwrap_or(String::new())
    });    

    println!(
        "{}{}... [Note: Abbreviated for display] ...",
        main_cont,
        opt_location.unwrap_or(String::new())
    );
    
    println!("===== END CONTENT =====\n");
}

```

Let's explain the iterators step by step:

First we have:
```rust
let content = format!("{}", t); 
let lines = content.lines().take(2);
let main_cont: String = lines.map(|l| format!("{}\n", l)).collect();
```

Content is just the content we want to print out. This is tailor made to print
out the interesting parts of our `http` reponse, namely the first few lines and
the `location` header so we know which call it was that got returned.

We only want to print out the two first lines so we get an iterator over those
by calling `content.lines().take(2)`.

The next step is to create a `String` out of these two lines.

`lines.map(|l| format!("{}\n", l)).collect();` Takes every line and `maps` them as
a new string which we append `\n` to to preserve the line breaks (if we don't do
that the lines will come out as a single line). Then we `collect` that into a `String`.

We know they collect to a `String` since we annotated the type of `main_cont` in `let main_cont: String`.

```rust
let opt_location = content.find("Location");
let opt_location = opt_location.map(|loc| {
        content[loc..]
        .lines()
        .nth(0)
        .map(|l| format!("{}\n",l))
        .unwrap_or(String::new())
    }); 
```

The next part finds the location of the "Location" header. `find` returns an `Option`,
just keep that in mind.

The next step, we use `map` again, but this time we use `map` on an `Option` type.
In this context, map means that **if** there is `Some` value, we want to work with that,
which means that `map(|loc| {...` loc is an index of the where the `Location` header was found.

Now what we do is that we take a range of our `content` string starting from the index
of where we found the `Location` header, and all the way to the end of the string.

From this range we access the iterator over its lines by calling `lines()`, we take the first
line from this iterator `nth(0)`. Now again, this returns an `Option`, so we use `map`
again to define what we'll do in the case if it's `Some`. 

This means that **if** the first line of `content[loc..]` is something we pass that
into our closure `map(|l| format!("{}\n",l))`, which results in `l` as this line.

We simply map this line into a `String` which we have appended a line break to.

Lastly we `unwrap` the result of these operations. If you kept your thoungue right
all the way through this should either be Some(the_first_line_starting_with_Location),
or None. If it's None we just pass inn an empty `String`.

Lastly we output everything.