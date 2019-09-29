# Putting the pieces together

```rust
fn main() {
    let rt = Runtime::new();
    rt.run(javascript);
}
```

```rust
fn print(t: impl std::fmt::Display) {
    println!("Thread: {}\t {}", current(), t);
}

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

fn current() -> String {
    thread::current().name().unwrap().to_string()
}
```