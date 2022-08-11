# Turn a module IBC-enabled

In this section, you'll build a conceptual SDK blockchain with one module. The first time, a regular module and the second time an IBC module. This will introduce us to what makes a module IBC-enabled.

## Scaffold a Leaderboard chain

By now, you are familiar with scaffolding a chain with Ignite CLI (if not, check out the [Create Your Own Chain](insert_link) section).

Let's scaffold a `leaderboard` chain:

```bash
$ ignite scaffold chain github.com/cosmonaut/leaderboard
```

This creates a chain with the `x/leaderboard` a regular SDK module.

Next, scaffold another chain (for example in another git branch) but decide to add the `--no-module` flag:

```bash
$ ignite scaffold chain github.com/cosmonaut/leaderboard --no-module
```

Now we add the `x/leaderboard` module as an IBC module with the `--ibc` flag:

```bash
$ ignite scaffold module leaderboard --ibc
```

The output you see on the terminal when the module has finished scaffolding already gives a sense of what has to be implemented to create an IBC module:

```bash
modify app/app.go
modify proto/leaderboard/genesis.proto
create proto/leaderboard/packet.proto
modify testutil/keeper/leaderboard.go
modify x/leaderboard/genesis.go
create x/leaderboard/module_ibc.go
create x/leaderboard/types/events_ibc.go
modify x/leaderboard/types/genesis.go
modify x/leaderboard/types/keys.go
```

To have a more detailed view, we can now compare both versions with a [git diff](https://diffy.org/diff/33f304d315465).

## IBC application module requirements

What does Ignite CLI do behind the scenes when creating an IBC module for us? What do we need to implement if we want to upgrade a regular custom application module to an IBC one?
