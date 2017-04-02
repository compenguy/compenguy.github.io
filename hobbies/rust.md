## Rust

Lately, I've been learning [Rust](https://www.rust-lang.org/).

### Why Rust?

Because it's [like C](https://locka99.gitbooks.io/a-guide-to-porting-c-to-rust/content/), but adds memory and concurrency safety, and adds the structure of [OOP-like](https://www.reddit.com/r/rust/comments/27sbgr/oop_in_rust/) programming.

Because it's [like C++](https://locka99.gitbooks.io/a-guide-to-porting-c-to-rust/content/), but adds memory and concurrency safety, and boils down OOP to its bare essence making it a lot simpler and harder to screw up.

Because it can be used [runtimeless](https://www.rust-lang.org/en-US/faq.html#does-rust-have-a-runtime), making it suitable for [kernel-mode](https://github.com/tsgates/rust.ko/blob/master/src/lib.rs) use and in deeply-embedded systems.  There's even a crate for [intrusive datatypes](https://github.com/Amanieu/intrusive-rs).

Because [semantic macros](https://danielkeep.github.io/tlborm/book/mbe-syn-source-analysis.html) require less [mental gymnastics](http://www.ioccc.org/) and fewer [WTF moments](http://thedailywtf.com/).

### Really?  Sounds more like hype.

Yeah, a little bit. Rust is still pretty young, and it's trying out a number of fairly new ideas.  That doesn't mean isn't useful for real work, though.

Rust was basically [invented](https://en.wikipedia.org/wiki/Rust_(programming_language)#History) to be the foundation for Mozilla's research browser engine Servo.  Servo recently hit two milestones: converting portions of the shipping Mozilla browser to [use their rusty Servo counterparts instead](https://wiki.mozilla.org/Oxidation), and benchmarks of very complex javascript demos between browsers showing [servo hardly breaking a sweat](https://www.youtube.com/watch?v=u0hYIRQRiws).

Dropbox has been porting performance-critical (especially memory-critical) portions of their infrastructure from [Go to Rust](https://news.ycombinator.com/item?id=11282948).

Tor extensively evaluated a move to either Go or Rust, and [settled on Rust](https://lists.torproject.org/pipermail/tor-dev/2017-March/012088.html).

npm, coursera, Samsung SmartThings - [the list of major projects standing behind rust is growing longer](https://www.rust-lang.org/en-US/friends.html). There's even [a game on Steam written in rust](https://www.reddit.com/r/rust_gamedev/comments/5vqlln/shar_one_year_with_rust/).


And Rust is not perfect.  Writing [unsafe code](https://doc.rust-lang.org/beta/nomicon/meet-safe-and-unsafe.html) can't always be avoided.  There are some [degenerate cases](https://doc.rust-lang.org/beta/nomicon/leaking.html) where leaks are possible.  Very few batteries included - instead of a rich standard library system, it relies on the crate ecosystem, a darwinistic environment of rapidly-evolving choices, although people have been making efforts to [collect](https://github.com/brson/stdx) and [advertise](https://github.com/llogiq/stdx-dev) important crates that are high quality and stable.

### How to Rust

* [The Rust book/guide](http://doc.rust-lang.org/book/)
* [Rust API documentation](http://doc.rust-lang.org/std/index.html)
* [The Rust reference manual](http://doc.rust-lang.org/reference.html)
* [Rust for C/C++ developers](https://github.com/nrc/r4cppp)

### My Rust Projects
- [x] [ricracroe](https://github.com/compenguy/ricracroe) - a CLI tic-tac-toe game
- [ ] nesugo - a rust-based NES emulator
- [ ] clickrs - an X11 automated mouse and keyboard clicker

