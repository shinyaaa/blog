---
title: "Can we use Rust to Develop Extensions for PostgreSQL?"
published_at: 2025-11-24
tags: postgres, rust, pgrx
---

Although it has been a while since the event, I am sharing the full transcript of my talk from [POSETTE 2025](https://posetteconf.com/2025/talks/can-we-use-rust-to-develop-extensions-for-postgresql/). Please note that some information may be outdated as time has passed.

Additionally, the Call for Proposals (CFP) for [POSETTE 2026](https://posetteconf.com/2026/cfp/) is now open. Let's make the next POSETTE a great success together!

Slides: {% slideshare lJRWJL65nyDs2A %}
YouTube: {% youtube 0NKFVQq0YuA %}

---

### Can We Use Rust to Develop Extensions for PostgreSQL? #1
![Can We Use Rust to Develop Extensions for PostgreSQL? #1](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/j1e0vlg5wiyv34w1icol.jpg)

Hello everyone.
Thank you for joining my session today.
My name is Shinya Kato and I'm a database engineer at NTT DATA Japan Corporation.

Today, I will talk about Postgres extensions.
The title of my talk is "Can We Use Rust to Develop Extensions for PostgreSQL".
If you're someone who's passionate about Postgres extension development or curious about how Rust fits into the picture, this presentation is for you.
Let's dive into it.

---

### About me #2
![About me #2](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6xrfo1mu3st353rg9nt1.jpg)

Here we go.

Let me introduce myself a little bit.
As I said, I'm a database engineer at NTT Data Japan Corporation which is the largest IT company in Japan.
I've been here with our company for about 5 years.

I specialize in Postgres, offering technical support, conducting R&D, contributing to the Postgres project, and maintaining two key extensions: pg_bulkload and pg_rman.

---

### Can we use Rust to develop extensions for PostgreSQL? #3
![Can we use Rust to develop extensions for PostgreSQL? #3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hcig3hutojxrrupop8hh.jpg)

To begin, can we use Rust to develop extensions for PostgreSQL?

---

### Yes, we can ✅ #4
![Yes, we can ✅ #4](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/sfvr5vvzyknoenjy6y7t.jpg)

Yes, we can.
Instead of legacy C language, it's possible to develop Postgres extensions using modern Rust and frameworks.

---

### But limited resources make it challenging ❎ #5
![But limited resources make it challenging ❎ #5](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2qojo3iukci7265v9ciz.jpg)

However, getting started with Postgres extension development in Rust can be a little bit challenging due to limited information.
This talk aims to provide assistance for developing Postgres extensions using Rust.
Furthermore, drawing from my own experience implementing Postgres extensions in Rust, I will share practical insights, covering both the pros and cons.

---

### Why Rust? #6
![Why Rust? #6](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/teum8sj0f5immb1pmu5p.jpg)

Why Rust?
Currently, system security is compromised by numerous vulnerabilities caused by memory corruption issues.
Google's security team, Project Zero, reports that about 68% of zero-day vulnerabilities are caused by memory-related problems.
These critical memory safety risks are commonly found in C and C++ applications.

On the other hand, Rust's strong memory safety features make these issues much less likely to occur.
Furthermore, Rust's ownership system guarantees memory safety without sacrificing performance.
This is why a partial migration from C and C++ to Rust is underway.

[1] https://googleprojectzero.blogspot.com/p/0day.html

---

### Why Rust? #7
![Why Rust? #7](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9xlttpaemzxubbnbpcsy.jpg)

For example, we can point to projects like 'Rust for Linux', which enables implementing Linux kernel modules and device drivers in Rust.
Another significant development is Chrome's ongoing migration of its font handling library from C's FreeType to Rust's Skrifa to ensure safer font processing.
Additionally, in the Postgres ecosystem, Neon is notable for implementing Postgres' storage engine in Rust.

[2] https://rust-for-linux.com
[3] https://developer.chrome.com/blog/memory-safety-fonts
[4] https://neon.tech/blog/how-we-scale-an-open-source-multi-tenant-storage-engine-for-postgres-written-rust

---

### What is pgrx? #8
![What is pgrx? #8](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/he30sfo1yw01979la1xb.jpg)

So, how can we implement Postgres extensions in Rust?
You can easily develop them using pgrx, which is a framework specifically designed for developing Postgres extensions in Rust.
Let me explain the key features of pgrx.
First, pgrx provides an integrated development environment.
Using this environment, you can easily implement Postgres extension features.
Second, it safely exposes the Postgres C API to Rust.
This allows you to securely call Postgres functions from your Rust implementation.
Third, it automatically handles the mapping between Rust types and Postgres types.
For example, if a Rust function returns an `i32` type, pgrx converts it to the `integer` type;
if it returns an `i64` type, it converts it to `bigint`;
and if it returns a `string` type, it converts it to `TEXT`.

---

### Popular PostgreSQL extensions written in Rust #9
![Popular PostgreSQL extensions written in Rust #9](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/92s9jqkpkd3cw3so7rfy.jpg)

There are many popular Postgres extensions written in Rust using pgrx.
ParadeDB is a modern Elasticsearch alternative built on Postgres.
ZomboDB integrates Postgres and Elasticsearch.
PostgresML is a powerful Postgres extension that seamlessly combines data storage and machine learning inference within a database.
PL/Rust is a Rust procedural language handler for Postgres.

Various other extensions are also being developed, so please refer to GitHub for more details.

Okey, let's move to next part demo.

---

### DEMO #10
![DEMO #10](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bnoup7v006to0qtjepwq.jpg)

---

### pgrx: Pros & Cons #11
![pgrx: Pros & Cons #11](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xnuafldgkwsnlddperd1.jpg)

Alright, that's it for the demo.
Based on the explanation so far, I hope you now have a general idea of how to use pgrx.
From this point, I'd like to discuss the pros and cons of pgrx, based on my own experience developing extensions with it.

---

### Setting up development environment ✅ #12
![Setting up development environment ✅ #12](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/795hb8c8q1zpbwaxf8an.jpg)

The first point I'd like to introduce is about setting up the development environment.
As you saw in the demo earlier, you can set up the development environment very easily.
With the command `cargo pgrx init`, you can set up development environments for all Postgres versions supported by pgrx, from 13 to 17.
You can also create an environment for only a specific version, for example, using a flag like `--pg17`.
I believe this feature is very user-friendly, especially for beginners in Postgres extension development.

---

### Setting up development environment ❎ #13
![Setting up development environment ❎ #13](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/aji61mc5d0bhtmp175tg.jpg)

However, you can only specify major Postgres versions; minor versions cannot be specified.
While you do not often need to worry about minor versions, the ability to specify them would be convenient, for instance, when you want to check the impact of a bug fixed in a specific minor release.
Furthermore, you cannot specify the Postgres master branch, which is currently under development.
Developing a Postgres extension isn't a one-time task; it needs to keep pace with Postgres version upgrades.
When a new major version of Postgres is released, pgrx will likely add support for it eventually.
However, waiting for pgrx to support new major versions after their official release slows down the delivery of updated extensions.
If it were possible to develop extensions against the master branch, I believe pgrx would become an even more practical framework for extension development.

---

### Requirements for C-Based Extensions #14
![Requirements for C-Based Extensions #14](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/263idcsko4jllcg11tf8.jpg)

The second point I'd like to introduce is automatic SQL generation.
With pgrx, SQL statements like `CREATE FUNCTION` can be generated automatically.
But first, let's review how user-defined functions and other components are typically implemented using the traditional C language approach.

When implementing a user-defined function called `func` and providing it as part of an extension using C, you need both a C source file and an SQL file.
On the left, I'm showing the C source file, and on the right is the SQL file.
After implementing the function logic in C, you then define the SQL function using a `CREATE FUNCTION` statement that calls the C function.
Furthermore, if your extension provides more complex features like custom types or index support, you also need corresponding SQL definitions such as `CREATE OPERATOR`, `CREATE OPERATOR CLASS`, or `CREATE OPERATOR FAMILY`.
As you can see, this means you need to implement and maintain both the C source file and all these SQL definition files.

---

### Automatic SQL generation ✅ #15
![Automatic SQL generation ✅ #15](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xk5fbil058nb7lv5ecnj.jpg)

Using pgrx, the SQL file is automatically generated based on the Rust function's attributes, argument types, and return value types.

---

### Automatic SQL generation ✅ #16
![Automatic SQL generation ✅ #16](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/cuql6dt3pmg6c4b4chu3.jpg)

In the `pg_extern` attribute, you specify parameters equivalent to those used in `CREATE FUNCTION`, such as `IMMUTABLE`, `STRICT`, `STABLE`, or `VOLATILE`.

---

### Automatic SQL generation ✅ #17
![Automatic SQL generation ✅ #17](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bemuszdye84ct3hou5cg.jpg)

Additionally, the Rust function's argument and return value types are mapped directly to the arguments and return type in the generated `CREATE FUNCTION` statement.
In this way, automatically generating the SQL reduces the cost of implementing and maintaining SQL files.

---

### Automatic SQL generation ✅ #18
![Automatic SQL generation ✅ #18](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/dk1e7mv4bufdsr6tjsxa.jpg)

Similar to functions, pgrx can also generate the necessary `CREATE FUNCTION` and `CREATE OPERATOR` statements for operators.
Just like before, in the `pg_operator` attribute, you specify parameters for `CREATE FUNCTION`, such as `STABLE` or `STRICT`.

---

### Automatic SQL generation ✅ #19
![Automatic SQL generation ✅ #19](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7ujvgxc0oms2wjnkdegt.jpg)

In the `opname` attribute, you specify the desired operator name.
In the `restrict` and `join` attributes, you specify operator-specific parameters.

---

### Automatic SQL generation ❎ #20
![Automatic SQL generation ❎ #20](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zqr5b5n9k4ckkiptcsea.jpg)

Automatic SQL generation is also available for operator classes, similar to functions and operators.
However, this capability is currently restricted to B-tree and Hash indexes only.
Support for other index types like GIN, GiST, and SP-GiST is currently lacking, which is an important limitation to consider.
For instance, an extension I developed required GIN indexes, so this auto-generation feature wasn't applicable, and I needed to write the necessary SQL definitions directly.

---

### Memory management in C/PostgreSQL #21
![Memory management in C/PostgreSQL #21](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ls60rmwfik98lwio60xh.jpg)

The third point I'd like to introduce is memory management.

Memory management in the C language requires explicit memory allocation and deallocation using malloc and free.
Postgres' implementation, uses a region-based system for memory management, utilizing `MemoryContexts`.
This involves creating these `MemoryContexts` and allocating memory associated with a specific context.
To free memory, you simply delete the `MemoryContext`, which releases all memory associated with it at once.
In this way, Postgres provides an implementation for efficient and relatively easy memory management.
However, even with this system, developers still need to be careful about issues like memory leaks, double frees, and dangling pointers.

---

### Memory management in Rust #22
![Memory management in Rust #22](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uta6rrwu082wka19zpmq.jpg)

Next, I will very briefly explain Rust's memory management.
Rust manages memory using a concept called the ownership system, which has three main rules:
Each value in Rust has an owner.
There can only be one owner at a time.
When the owner goes out of scope, the value will be dropped.

---

### Memory management in Rust #23
![Memory management in Rust #23](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/f52xl2bit6mk6b97fgm9.jpg)

As shown in this diagram, when `s1` is declared, `s1` becomes the owner of the `hello`.

---

### Memory management in Rust #24
![Memory management in Rust #24](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uoolj407jw0x5ncr85be.jpg)

Similarly, `s2` becomes the owner of `world`.

---

### Memory management in Rust #25
![Memory management in Rust #25](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8eslwnvtmw9lqu43rfbj.jpg)

Next, `s2` goes out of scope and then is dropped, freeing its memory.

---

### Memory management in Rust #26
![Memory management in Rust #26](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wfjggazqjelkxofs4dcp.jpg)

Likewise, `s1` goes out of scope and is dropped, freeing its memory.

---

### Memory management in Rust ✅ #27
![Memory management in Rust ✅ #27](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gs2vf18quk06qor7kahw.jpg)

pgrx's memory management system is designed to bridge the gap between Postgres' MemoryContexts and Rust's ownership.
Specifically, deallocation of pointers received from the C API is handled by Postgres.
Conversely, objects created in Rust are deallocated by Rust itself, based on its ownership rules.
To help safely manage pointers allocated by C from within Rust, pgrx provides the `PgBox` type.

---

### Memory management in Rust ❎ #28
![Memory management in Rust ❎ #28](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pf84g2unytfg1e1cfuc1.jpg)

It's already difficult to understand C and Postgres memory management and Rust's memory management individually.
Then, you also need to learn pgrx memory management.
This makes it much harder.
Let me give some difficult examples.
One is handling memory created in C from your Rust code. Another is passing Rust memory to C and telling Rust not to free it, which goes against Rust's ownership system.

---

### Toolchain ✅ #29
![Toolchain ✅ #29](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qj1oa4criwp0tt2y0bdi.jpg)

The fourth point I'd like to introduce is toolchain.
When developing with pgrx, you directly benefit from the rich Rust ecosystem, particularly its excellent toolchain.
This familiarity offers great convenience.
For instance, you have `cargo clippy` for Linting to help write more robust and idiomatic Rust.
Code style consistency is easily maintained using `cargo fmt` for automatic Formatting.
And crucially for extensions, `cargo pgrx test` provides a streamlined way to handle integration testing directly against Postgres.
This ability to use standard Rust tooling really makes the development workflow much smoother.

---

### Toolchain ❎ #30
![Toolchain ❎ #30](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/493zbl7i7n50j2q0kqb9.jpg)

Now, let's look at a point marked here as a challenge, which relates to toolchain integration.
While the Rust side is well-supported, there's currently weaker integration with some traditional Postgres tools.
Specifically, as noted here, there isn't direct integration with `pg_regress`, the standard framework for running SQL-level regression tests in the Postgres ecosystem.
While pgrx provides its own testing mechanism, it doesn't replace the `pg_regress` workflow.
This is a recognized limitation, as shown by the GitHub issues listed on the slide.
For now, this means that setting up `pg_regress`-style tests might require separate configuration.

Note: ![ZomboDB](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4ytqizsvctd4yibcaps2.png)

[5] https://github.com/pgcentralfoundation/pgrx/issues/284
[6] https://github.com/pgcentralfoundation/pgrx/issues/1511

---

### Documentation ✅ #31
![Documentation ✅ #31](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o22eji2xwu6otx0m6kr3.jpg)

The fifth point I'd like to introduce is documentation.

While learning pgrx involves navigating different memory models as we discussed, there are several excellent resources available to help you.
Let me highlight the key ones shown on this slide.
First, for the specific details of the pgrx crate's functions, types, and macros, your primary reference is `Docs.rs`, which is the API documentation.
Next, `The PGRX Book` serves as the official guide and documentation.
This is the best place to understand core concepts, learn how to set up your environment, and follow tutorials.
Finally, if you want to see practical examples and common patterns, the `pgrx-examples` is invaluable.
It contains various code examples that you can learn from and adapt.
I highly recommend utilizing these resources as you explore pgrx development.

[7] https://docs.rs/pgrx/latest/pgrx/
[8] https://github.com/pgcentralfoundation/pgrx/blob/develop/docs/src/SUMMARY.md
[9] https://github.com/pgcentralfoundation/pgrx/tree/develop/pgrx-examples

---

### Documentation ❎ #32
![Documentation ❎ #32](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pbhio3ab8rwmuiefx56f.jpg)

While resources are growing, it's true that there is currently less information available for pgrx development compared to the vast resources accumulated over many years for traditional C extensions.
So, because there isn't enough documentation or examples yet, development can still be quite hard sometimes.
This is especially true for complex problems or unusual situations.

I hope this presentation was helpful for you.

---

### PostgreSQL & Rust: Core vs. Extensions #33
![PostgreSQL & Rust: Core vs. Extensions #33](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vuskqq31v4azw6z9jrfd.jpg)

Finally, let's discuss the future of Postgres and Rust.
There have been multiple proposals in the past suggesting the use of Rust for implementing Postgres' core.
However, due to various reasons, the adoption of Rust for the core has been deferred, so the chance of this happening is quite low.
For example, there's the huge existing C codebase, Postgres' unique memory management system, and its specific error handling mechanisms (like `PG_TRY()`).
Additionally, Postgres needs to support a wide range of platforms, which presents portability challenges.
But, as mentioned in this talk, Rust-based Postgres extensions are expected to grow substantially in the future.

[10] https://www.postgresql.org/message-id/flat/CAASwCXdQUiuUnhycdRvrUmHuzk5PsaGxr54U4t34teQjcjb%3DAQ%40mail.gmail.com
[11] https://www.postgresql.org/message-id/flat/CANvSX4wsVhDrqy4Ku81wHs--aViYthJ_G5G-2VeZ%3DJboEE9TNQ%40mail.gmail.com

---

### Summary #34
![Summary #34](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qe3u5bzclemsu7gkdegu.jpg)

Okay, let's quickly summarize the main points of this presentation.
First, I explained how you can implement Postgres extensions using Rust with the help of pgrx.
We also took some time to discuss the pros and cons of using the pgrx framework.
pgrx offers a powerful way to build extensions.
So, let's try it out!
Thank you.

{% github https://github.com/pgcentralfoundation/pgrx %}
