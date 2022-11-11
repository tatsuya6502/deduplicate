# deduplicate
asynchronous deduplicator with optional LRU caching

If you have "slow", "expensive" or "flaky" tasks for which you'd like to provide de-duplication and, optionally, result caching, then this may be the crate you are looking for.

The `Deduplicate` struct controls concurrent access to data via a delegated function provided when you create your `Deduplicate` instance.

```
use std::sync::Arc;
use std::time::Instant;

use deduplicate::Deduplicate;
use deduplicate::DeduplicateFuture;

use rand::Rng;

/// If our delegated getter panics, all our concurrent gets will
/// fail. Let's cause that to happen sometimes by panicking on even
/// numbers.
fn get(_key: usize) -> DeduplicateFuture<String> {
    let fut = async {
        let num = rand::thread_rng().gen_range(1000..2000);
        tokio::time::sleep(tokio::time::Duration::from_millis(num)).await;

        if num % 2 == 0 {
            panic!("BAD NUMBER");
        }
        Some("test".to_string())
    };
    Box::pin(fut)
}

/// Create our deduplicate and then loop around 5 times creating 100
/// jobs which all call our delegated get function.
/// We print out data about each iteration where we see how many
/// succeed, the range of times between each invocation, the set
/// of results and how long the iteration took.
/// The results of running this will vary depending on whether or not
/// our random number generator provides us with an even number.
/// As long as we get even numbers, all of our gets will fail and
/// the delegated get will continue to be invoked. As soon as we
/// get a delegated call that succeeds, all of our remaing loops
/// will succeed since they'll get the value from the cache.
#[tokio::main]
async fn main() {
    let deduplicate = Arc::new(Deduplicate::new(get));

    for _i in 0..5 {
        let mut hdls = vec![];
        let start = Instant::now();
        for _i in 0..100 {
            let my_deduplicate = deduplicate.clone();
            hdls.push(async move {
                let is_ok = my_deduplicate.get(5).await.is_ok();
                (Instant::now(), is_ok)
            });
        }
        let mut result: Vec<(Instant, bool)> =
            futures::future::join_all(hdls).await.into_iter().collect();
        result.sort();
        println!(
            "range: {:?}",
            result.last().unwrap().0 - result.first().unwrap().0
        );
        println!(
            "passed: {:?}",
            result
                .iter()
                .fold(0, |acc, x| if x.1 { acc + 1 } else { acc })
        );
        println!("result: {:?}", result);
        println!("elapsed: {:?}\n", Instant::now() - start);
    }
}
```

[![Crates.io](https://img.shields.io/crates/v/deduplicate.svg)](https://crates.io/crates/deduplicate)

[API Docs](https://docs.rs/deduplicate/latest/deduplicate)

## Installation

```toml
[dependencies]
deduplicate = "0.3"
```

## Acknowledgements

This crate build upon the hard work and inspiration of several folks, some of whom I have worked with directly and some from whom I have taken indirect inspiration:
 - https://github.com/Geal
 - https://github.com/cecton
 - https://fasterthanli.me/articles/request-coalescing-in-async-rust
 - various apollographql router developers

Thanks for the input and good advice. All mistakes/errors are of course mine.

## License

Apache 2.0 licensed. See LICENSE for details.
