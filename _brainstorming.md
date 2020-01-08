Quick and unfinished ideas, most of them just me brainstorming :-D
*	Have a trait `DropEarly`
	*	(Ideally / at least conceptually) supertrait of `Copy`,
		ideally an auto trait although that would make it quite the
		breaking change, if it’s not auto but supertrait of `Copy` that would be breaking too.
	*	Makes the compiler drop a value of this type not at the end of it’s static scope but as
		early as possible. (Or perhaps just allows dropping at any point from earliest possible
		to end of scope?)
	*	Hence some (or most) drop flags would become unnecessary; furthermore some moves can be eliminated,
		(suppose, `String` implements `DropEarly`) for example:
		```rust
		fn main() {
			let a = String::from("foo");
			if EXPR {
				// do something that moves a like
				drop(a)
				// do something else
			} else {
				// don’t use a
			}
			// some other code
		}
		```
		currently becomes something like
		```rust
		fn main() {
			let a = String::from("foo");
			let a_dropped = false;
			if EXPR {
				// do something that moves a like
				drop(a)
				a_dropped = true;
				// do something else
			} else {
				// don’t use a
			}
			// some other code
			if !a_dropped { drop(a) } // of course not legal rust, since a isn’t usable here 
		}
		```
		But it could instead become:
		```rust
		fn main() {
			let a = String::from("foo");
			if EXPR {
				// do something that moves a like
				drop(a)
				// do something else
			} else {
				drop(a);
				// don’t use a
			}
		}
		```
		
		About avoiding moves:
		TODO: find that example where `Drop` enforces a copy in one of the books again.
		
		Definitely helps at least save some stack space in some cases. (Should be obvious.)
		
		Solves problems with memory leaks in a naive REPL (i.e. REPL interface with the approach:
		what you write constitures the body of a very long `main()` function. There rerunning commands
		shadowing earlier bindings will allow drops of `DropEarly` values (if no reference, etc. is retained either).
		
	*	The exact (or perhaps earliest possible) point of drop is the end of the smalles possible lifetime of the variable.
	*	There should be `impl`s such as for example:
		```rust
		impl<T: ?Sized + DropEarly> DropEarly for Box<T> {}
		impl<T: ?Sized + DropEarly> DropEarly for Vec<T> {}
		```
		And every type that doesn’t (directly or inderectly) need any custom `drop` call should behave
		as if it’s `DropEarly` anyways.
