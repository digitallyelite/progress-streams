# progress-streams

Rust crate to provide progress callbacks for types which implement `io::Read` or `io::Write`.

## Examples

### Reader

```rust
extern crate progress_streams;

use progress_streams::ProgressReader;
use std::fs::File;
use std::io::Read;
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::thread;
use std::time::Duration;

fn main() {
    let total = Arc::new(AtomicUsize::new(0));
    let mut file = File::open("/dev/urandom").unwrap();
    let mut reader = ProgressReader::new(&mut file, |progress: usize| {
        total.fetch_add(progress, Ordering::SeqCst);
    });

    {
        let total = total.clone();
        thread::spawn(move || {
            loop {
                println!("Read {} KiB", total.load(Ordering::SeqCst) / 1024);
                thread::sleep(Duration::from_millis(16));
            }
        });
    }

    let mut buffer = [0u8; 8192];
    while total.load(Ordering::SeqCst) < 100 * 1024 * 1024 {
        reader.read(&mut buffer).unwrap();
    }
}
```

### Writer

```rust
extern crate progress_streams;

use progress_streams::ProgressWriter;
use std::io::{Cursor, Write};
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::thread;
use std::time::Duration;

fn main() {
    let total = Arc::new(AtomicUsize::new(0));
    let mut file = Cursor::new(Vec::new());
    let mut writer = ProgressWriter::new(&mut file, |progress: usize| {
        total.fetch_add(progress, Ordering::SeqCst);
    });

    {
        let total = total.clone();
        thread::spawn(move || {
            loop {
                println!("Written {} Kib", total.load(Ordering::SeqCst) / 1024);
                thread::sleep(Duration::from_millis(16));
            }
        });
    }

    let buffer = [0u8; 8192];
    while total.load(Ordering::SeqCst) < 1000 * 1024 * 1024 {
        writer.write(&buffer).unwrap();
    }
}
```