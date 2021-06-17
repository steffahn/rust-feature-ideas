WIP, unchecked for typos, ideas on possible if-let, let-chain, and let-else syntax, using commas and a contextual `and` keyword.

```rust
fn f() {
    let Some(x) = foo(),
    and let Ok(y) = bar(x),
    and y <= z,
    else {
    
    }
    
    let Some(x) = foo.next(), else {
        return;
    }
    
    if y <= x, {
        
    }
    if y <= X {}, {
        
    }
    if a < b, and c < d {
        
    }
    
    let x = foo(), else {
        
    }
    
    if let Some(x) = bar() {
        
    }

    if !(foo() && bar()) {

    }

    if !foo() || !bar() {

    }

    foo() && bar(), else {

    }

    foo(), and bar(), else {

    }

    foo(),
    and bar(),
    else {

    }

    let Some(x) = foo(v),
    and bar(x) < z,
    else if v.abc() {
        // diverge
    } else {
        // diverge
    }

    if let Some(x) = a + b, and x > 42 {

    }

    if let Some(x) = a + b,
    and x > 42,
    and let Ok(foo) = x.try_add(100.pow(42)),
    {

    }

    if FooB {
        x: 42,
        y: 24,
    }.try_build().is_some(),
    {
	// …
    }

    // subtle around commas though
    let x: () = if foo {} else { /* diverge */ };

    let x: () = if foo {},
    else {
	/* diverge */
    }
    ;
}
```
