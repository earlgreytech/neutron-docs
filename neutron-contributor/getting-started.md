---
description: How to build Neutron and run a smart contract
---

# Getting Started

Currently Neutron is spread out through many different projects and due to its early development state, is not packaged up in anyway. The following is a recommendation for how to build Neutron currently in this state.

First, you'll need to install Rust and its accompanying utilities such as Cargo and Rustup. You can do this by following the instructions here or by using your favorite OS package manager: [https://www.rust-lang.org/tools/install](https://www.rust-lang.org/tools/install)

Next, you'll need to install ARMv6 targeting for the Rust compiler. This can be done by executing the following command:

```text
rustup target add thumbv6m-none-eabi
```

Next, create a new directory for Neutron development and clone all of the relevant repositories:

```text
mkdir ~/neutron
cd ~/neutron
# the "core" of Neutron
git clone https://github.com/earlgreytech/neutron-host
# various shared constants and structures between the "client" and "host" side of Neutron
git clone https://github.com/earlgreytech/neutron-common
# the Neutron ARM VM
git clone https://github.com/earlgreytech/narm
# the basic Neutron "client" runtime with system call stubs etc
git clone https://github.com/earlgreytech/neutron-star-rt
# the abstraction layer for writing Neutron smart contracts
git clone https://github.com/earlgreytech/neutron-star
```

Afterwards you should be able to go into `neutron-host/neutron-testbench` and build it

```text
cd ~/neutron/neutron-host/neutron-testbench
cargo build
```

Assuming everything is successful, you've now built the Neutron testbench program which can be used to execute smart contracts! Next, we need a smart contract to actually run. You now need to create a new cargo project and link it up to the required dependencies and instruct Rust to build an ARMv6 executable file. 

```text
cd ~/neutron
cargo new neutron-test-contract
cd neutron-test-contract
mkdir .cargo && touch .cargo/config
```

Now edit the files in that directory to have the contents shown.

```text
File: .cargo/config

rustflags = [
  # This is needed if your flash or ram addresses are not aligned to 0x10000 in memory.x
  # See https://github.com/rust-embedded/cortex-m-quickstart/pull/95
  # "-C", "link-arg=--nmagic",

  # LLD (shipped with the Rust toolchain) is used as the default linker
  "-C", "link-arg=-Tlink.x",
  "-C", "relocation-model=static",
  "-C", "target-feature=+crt-static"

  # if you run into problems with LLD switch to the GNU linker by commenting out
  # this line
  # "-C", "linker=arm-none-eabi-ld",

  # if you need to link to pre-compiled C libraries provided by a C toolchain
  # use GCC as the linker by commenting out both lines above and then
  # uncommenting the three lines below
  # "-C", "linker=arm-none-eabi-gcc",
  # "-C", "link-arg=-Wl,-Tlink.x",
  # "-C", "link-arg=-nostartfiles",
]

[build]
target = "thumbv6m-none-eabi"        # Cortex-M0 and Cortex-M0+
```

```text
File: Cargo.toml

[package]
name = "neutron-test-contract"
version = "0.1.0"
authors = ["earlz <earlz@earlz.net>"]
edition = "2018"

[[bin]]
name = "neutron-test-contract"
test = false
bench = false

[dependencies]
neutron-star-rt = { path = "../neutron-star-rt" }
neutron-star = { path = "../neutron-star" }
panic-halt = "0.2.0"
```

```text
File: src/main.rs

#![no_main]
#![no_std]

use neutron_star_rt::*;
use neutron_star::*;
extern crate panic_halt;


#[no_mangle]
pub unsafe extern "C" fn main() -> ! {
    println!("Hello world! {}, {}", 0, 2);
    __exit(5);
}
```

Now use `cargo build` to actually build the smart contract. 

Finally, you can now run the smart contract using the testbench! 

```text
cd ~/neutron/neutron-host/neutron-testbench
cargo run ~/neutron/neutron-test-contract/target/thumbv6m-none-eabi/debug/neutron-test-contract
```

You should see a result like so:

```text
Beginning contract execution
NEUTRON INFO: Hello world! 0, 2

Contract executed successfully!
Gas used: 1369
Status code: 5
```

Congrats! You've now ran your first smart contract on Neutron and are ready to start development. 

One additional dependency is needed if you are adding system calls or otherwise messing with the assembly code in `neutron-star-rt`. Specifically you need an ARM assembler, specifically of the form `arm-none-eabi-as` . This can be installed using your OS package manager or by installing the ARM Embedded GNU Toolchain: [https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm/downloads](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm/downloads). Make sure to add the executables into your `PATH` if needed. When editing the file `asm.s` within `neutron-star-rt`, you must manually recompile it by executing `./assemble.sh`. In pull requests etc to this project, you should include the assembled output files of this operation. This is done so that a specific ARM assembler is not required to use the `neutron-star-rt` project as a dependency, even though it is a little messy to have compiled binaries in a git repo. Unless you are modifying the assembly code of this project though, there is no need for the GNU ARM assembler. 

