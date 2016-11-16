---
layout: post
title:  "Rustup Tutorial: Downgrading Nightlies"
date:   2015-11-16 14:44:03
categories: jekyll update
---

I hit an ICE (internal compiler exception -- a BUG in the compiler!) on the most recent nightly. Good thing I'm a rustup user and managing toolchains is well within reach. A downgrade is in order!

So the first thing you do is check out the help of your tool:

> LP-XLANGE-OSX:~ xlange$ rustup -h
> rustup 0.6.5 (88ef618 2016-11-04)
> The Rust toolchain installer
> 
> USAGE:
>     rustup [FLAGS] [SUBCOMMAND]
> 
> FLAGS:
>     -v, --verbose    Enable verbose output
>     -h, --help       Prints help information
>     -V, --version    Prints version information
> 
> SUBCOMMANDS:
>     show           Show the active and installed toolchains
>     update         Update Rust toolchains
>     default        Set the default toolchain
>     toolchain      Modify or query the installed toolchains
>     target         Modify a toolchain's supported targets
>     component      Modify a toolchain's installed components
>     override       Modify directory toolchain overrides
>     run            Run a command with an environment configured for a given toolchain
>     which          Display which binary will be run for a given command
>     doc            Open the documentation for the current toolchain
>     man            View the man page for a given command
>     self           Modify the rustup installation
>     set            Alter rustup settings
>     completions    Generate completion scripts for your shell
>     help           Prints this message or the help of the given subcommand(s)

Now usually I just do a quick `rustup update` and watch as the latest nightly and stdlib are downloaded and
linked in to my environment. But today I have an unusuable nightly which crashes on due to some unstable
features. Unfortunately no top-level command screams `install` or even `download`. But digging a little further:


    LP-XLANGE-OSX:~ xlange$ rustup toolchain --help
    rustup-toolchain
    Modify or query the installed toolchains

    USAGE:
        rustup toolchain [SUBCOMMAND]

    FLAGS:
        -h, --help    Prints help information

    SUBCOMMANDS:
        list         List installed toolchains
        install      Install or update a given toolchain
        uninstall    Uninstall a toolchain
        link         Create a custom toolchain by symlinking to a directory
        help         Prints this message or the help of the given subcommand(s)
    // SNIP


Aha! The elusive `install` keyword. The help continues on, describing how you specify a channel and version:

    Many `rustup` commands deal with *toolchains*, a single installation
    of the Rust compiler. `rustup` supports multiple types of
    toolchains. The most basic track the official release channels:
    'stable', 'beta' and 'nightly'; but `rustup` can also install
    toolchains from the official archives, for alternate host platforms,
    and from local builds.

    Standard release channel toolchain names have the following form:

        <channel>[-<date>][-<host>]

        <channel>       = stable|beta|nightly|<version>
        <date>          = YYYY-MM-DD
        <host>          = <target-triple>

    'channel' is either a named release channel or an explicit version
    number, such as '1.8.0'. Channel names can be optionally appended with
    an archive date, as in 'nightly-2014-12-18', in which case the
    toolchain is downloaded from the archive for that date.

Cool. So today is `2016-11-16` and the nightly isn't very good. How about I go back a few days to be safe:

    $ rustup toolchain install nightly-2016-11-13
    info: syncing channel updates for 'nightly-2016-11-13-x86_64-apple-darwin'
    error: no release found for 'nightly-2016-11-13'

Cue the music, we have a mystery. Turns out not all days contain nightlies! If **any** of the platform builds
fails the whole nightly is called off and there is nothing to download. So how do you find a valid pattern?

Enter [rusty dash's nightly build result tracker](http://rusty-dash.com/nightlies). Go here, look for a green
build, and use that to specify your version!

    LP-XLANGE-OSX:~ xlange$ rustup toolchain install nightly-2016-11-06
    info: syncing channel updates for 'nightly-2016-11-06-x86_64-apple-darwin'
    info: downloading component 'rustc'
     33.7 MiB /  33.7 MiB (100 %) 320.0 KiB/s ETA:   0 s
    info: downloading component 'rust-std'
     43.6 MiB /  43.6 MiB (100 %) 403.2 KiB/s ETA:   0 s
    info: downloading component 'rust-docs'
      7.7 MiB /   7.7 MiB (100 %) 630.4 KiB/s ETA:   0 s
    info: downloading component 'cargo'
      2.5 MiB /   2.5 MiB (100 %) 671.7 KiB/s ETA:   0 s
    info: installing component 'rustc'
    info: installing component 'rust-std'
    info: installing component 'rust-docs'
    info: installing component 'cargo'

      nightly-2016-11-06-x86_64-apple-darwin installed - rustc 1.14.0-nightly (cae6ab1c4 2016-11-05)

You can set this nightly as your default:

    LP-XLANGE-OSX:~ xlange$ rustup default nightly-2016-11-06
    info: using existing install for 'nightly-2016-11-06-x86_64-apple-darwin'
    info: default toolchain set to 'nightly-2016-11-06-x86_64-apple-darwin'

      nightly-2016-11-06-x86_64-apple-darwin unchanged - rustc 1.14.0-nightly (cae6ab1c4 2016-11-05)

    LP-XLANGE-OSX:~ xlange$ rustc --version
    rustc 1.14.0-nightly (cae6ab1c4 2016-11-05)

And you can get your travis build working again, use the same version specifer in your travis.yml:

    sudo: false # run in a docker container
    language: rust
    rust:
    #  - stable
    #  - beta
      - nightly-2016-11-06

There ya go, next time you hit issues with a nightly you can figure out how to downgrade and get on with it!
