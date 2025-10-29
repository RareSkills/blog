# Solana Smart Contract Programming Language


![Rust code from the Metaplex library](https://static.wixstatic.com/media/935a00_3f5e8b99b9474c1d9ba116a2ad27ef12~mv2.webp/v1/fill/w_1480,h_878,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_3f5e8b99b9474c1d9ba116a2ad27ef12~mv2.webp)

## Solana programming language

The primary [Solana programming language](https://www.rareskills.io/solana-tutorial) is [Rust](https://www.rareskills.io/rust-bootcamp), but [C, C++](https://docs.solana.com/developing/on-chain-programs/developing-c), and even [Python](https://seahorse-lang.org/) are supported.

Solana coding language uses Rust, C, and C++ in a similar way. We’ll discuss Python in a later section.

Rust is a compiled language. If you compile Rust on your computer, it will ultimately turn into LLVM-IR (low level virtual machine intermediate representation), and LLVM turns it into the bytecode that can run on your machine (x86, arm64, etc.).

In Solana, the sequence looks like this: 1) Compile Rust to LLVM-IR, then to BPF (Berkeley Packet Filter) and store the bytecode on the [blockchain](https://www.rareskills.io/web3-blockchain-bootcamps). 2) The validators JIT compile ([just in time compile](https://en.wikipedia.org/wiki/Just-in-time_compilation)) the BPF to the instruction set compatible with their hardware, usually x86, but arm64 might be another common target.

*Packet Filter*, you ask? Why would the Solana programming language have to do with internet packets?

This is actually a clever design choice on Solana’s part.

## User space vs kernel space

Linux has a notion of kernel space and a user space. If you want to do things like open a file or start another process, your executable needs to ask the operating system to do that for it. Let’s say you write a python script to open a file and print out every even line. The actual loading of the file’s bytecode happens in kernel space, but once the bytecode is given to the script, the interpretation to ASCII and determining if a line number is even or odd happens in user space.

This abstraction exists for several reasons, but one obvious one is security. Not every user or executable should be able to open or execute arbitrary files. The operating system determines which “APIs” are permitted. (By the way, the “API” to open a file is technically called a “system call” in operating system speak).

Similarly, programs and executables should not be allowed to arbitrarily access incoming internet packets. They must, by default, make system calls to ask the operating system permission to view packets, which can only be accessed from kernel space.

One important concept must be emphasized here: transitioning back and forth between user space and kernel space is generally slow.

If you are filtering incoming internet packets, then that is a ***lot*** of jumps back and forth from user space and kernel space. Imagine copying every incoming packet from kernel space to user space. That would create a lot of overhead.

This is why BPF was invented. You can run executables inside the kernel space to avoid this jumping.

But if you know anything about having kernel privileges, you know this is extremely dangerous! Having control over the kernel (operating system) could cause the computer to crash if there is a bug. Worse, if malicious code gets executed, the damage is limitless.

Of course, the BPF designers thought of this. Before BPF code is executed, it gets validated to ensure that it runs for a fixed amount of time, i.e. must terminate, can only access a designated memory area, and follows other suitable restrictions.

As an aside, since its invention, BPF has expanded its use beyond just filtering packets, but its name has stuck.

## Why Solana language uses BPF

By leveraging the existing research that went into making BPF programs safe, Solana can run smart contracts where they run the fastest — inside the kernel! This is pretty remarkable if you think about it. You can run untrusted smart contracts which could have been written by anyone in the most sensitive (but efficient) part of the operating system — the operating system kernel. Solana gets to leverage decades of research and investment in this area to get a nice performance boost.

BPF is not machine instructions; it’s still its own set of bytecode. However, it can be JIT compiled to a variety of CPU architectures.

## Back to Rust, C, and C++

These three programming languages have long been supported by LLVM. This is another area where Solana can take advantage of decades of investment (yes, it’s odd to speak of tech in terms of decades, but [LLVM came out in 2003](https://en.wikipedia.org/wiki/LLVM)).

You can use C or C++ as a Solana programming language, but you will get far less support with tooling. It’s generally recommended that you use Rust even if you are a C or C++ expert. Rust will be easy to learn if you are already fluent in C++.

## How much Rust do you need to know to program Solana?

Not that much, but it still requires some study. Rust is not a language that you can “google your way through.” For example, suppose you are programming in Ruby from a Java or Scala background. In that case, you can pretty easily ask google for the equivalent programming patterns (your code might not look idiomatic, but it will be readable and functional). If (when) you copy and paste code from Stack Overflow, you’ll still have a good intuition about what the code is doing.

However, if you do this with Rust, you will run into some frustrating roadblocks. Rust has syntax that is hard to look up (try looking up a “#” in the search engine), and it has concepts that aren’t found in other programming languages.

Rust is a vast language. However, you only need to know a subset of it. Our [Solana development course](https://rareskills.io/solana-tutorial) teaches Rust alongside Solana so that you only focus on the parts of Rust that you need.

## How Solana can use Python

Compiling Rust, C, or C++ to BPF is straightforward. With Python, it’s quite different; Python, obviously, is an interpreted language, not a language that can be compiled with LLVM.

To oversimplify, Python is [transpiled](https://en.wikipedia.org/wiki/Source-to-source_compiler) to Rust, and then everything behaves as above.

If you want the exact workflow, it’s documented visually on the [seahorse-lang github](https://github.com/ameliatastic/seahorse-lang)

Solana programs usually aren’t written in raw Rust; most developers use the Anchor Framework. Therefore, although Seahorse does fairly typical transpilation, it’s also taking advantage of planned framework similarities.

The Seahorse Framework closely models the Anchor Framework so that the Python code can be translated into Rust code that closely models the way it would be written in an Anchor framework.

Note that this project is in beta right now.

## Does Solana use Solidity?

Yes, it is possible to write Solana applications in Solidity but somewhat experimental. The [solang solidity compiler](https://solang.readthedocs.io/en/latest/) was built to support compiling Solidity to BPF.

## The Solana client programming language

The [Solana client](https://github.com/solana-labs/solana) itself, the program that runs on nodes that powers the blockchain, as opposed to programs (smart contracts), is written in Rust. There is currently an in-progress re-implementation of the Solana client, [firedancer](https://jumpcrypto.com/firedancer/) by Jump Crypto, which is written entirely in C.

## Addendum: Rust is migrating BPF to SBF.

As of October 2022, Solana began migrating from BPF to SBF (Solana binary format). As of the time of writing, the documentation on this change is quite sparse, but this won’t affect most developers. If your build tool is configured to compile to BPF, you will get a deprecation warning, but everything will still run. Just change your build flags.

## Learn More from RareSkills

Want to master Ethereum development? See our [Solidity Bootcamp](https://www.rareskills.io/solidity-bootcamp)

### Resources

[https://www.kernel.org/doc/html/latest/bpf/instruction-set.html](https://www.kernel.org/doc/html/latest/bpf/instruction-set.html)
[https://docs.rs/solana_rbpf/latest/solana_rbpf/ebpf/index.html](https://docs.rs/solana_rbpf/latest/solana_rbpf/ebpf/index.html)
[https://www.youtube.com/watch?v=5jQvuPWpzcE](https://www.youtube.com/watch?v=5jQvuPWpzcE)
[https://ebpf.io/what-is-ebpf/](https://ebpf.io/what-is-ebpf/)
[https://www.youtube.com/watch?v=Q8eY67hDvkc](https://www.youtube.com/watch?v=Q8eY67hDvkc)

*Originally Published Dec 1, 2022*
