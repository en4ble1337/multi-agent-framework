# Example Deployments

This repository is intentionally generic. The goal is to preserve the operating model, not to force one stack.

The core invariants are always the same:

- one private workspace per agent
- one shared surface for public coordination
- one visible chat path for humans and agents
- one locking rule for multi-writer files

## Example 1: Virtual Machines Plus Shared Filesystem

Shape:

- one isolated virtual machine per agent
- one mounted shared filesystem for `/shared`
- one chat platform for operator oversight

Why it works:

- machine-level isolation is easy to reason about
- browser sessions and local credentials stay separated
- cloning a prepared image is straightforward

Tradeoffs:

- heavier provisioning than containers
- more operating system management
- shared storage becomes an important dependency

## Example 2: Containers Plus Shared Volume

Shape:

- one container per agent
- one mounted shared volume
- one chat platform for operator oversight

Why it works:

- faster provisioning
- easier density on one host or cluster
- compatible with existing container infrastructure

Tradeoffs:

- weaker isolation than full virtual machines
- browser automation and host integration may need extra care
- storage semantics must still support the lock protocol

## Example 3: Single Host Plus Per-Agent Users

Shape:

- one machine
- one OS user or directory per agent
- one shared local folder

Why it works:

- lowest setup overhead
- good for prototyping or small experiments
- easy local debugging

Tradeoffs:

- weakest failure isolation
- easier for one agent process to impact others
- not ideal when agents need separate browser state or credentials

## Example 4: Cloud Instances Plus Managed Storage

Shape:

- one cloud instance per agent
- one managed shared filesystem or synchronized storage layer
- one chat platform for operator oversight

Why it works:

- flexible scaling
- works without local hardware
- easy geographic placement when needed

Tradeoffs:

- higher cost
- more network dependency
- managed storage semantics must be tested for locking behavior

## Translation Guide

Map the pattern by responsibilities, not product names:

| Need | Valid implementation choices | Invariant to keep |
|---|---|---|
| Isolation boundary | VM, container, user account, sandbox | Each agent needs a private workspace and bounded blast radius |
| Shared surface | Network share, synced folder, mounted volume, local directory | Shared artifacts must be visible to every intended writer and reader |
| Human coordination | Chat rooms, direct messages, ticketing plus chat | Humans must be able to direct work and observe outcomes |
| Agent runtime | Any process that can read files, write files, and send messages | Agents must be able to act on both local and shared context |
| Lock primitive | Directory lock, file lock, service lock, transactional write | Multi-writer files must not rely on blind concurrent writes |

## Choosing Between Shapes

Pick the lightest option that still preserves the boundaries you need:

- If secrets, browsers, and failure isolation matter, prefer stronger isolation.
- If setup speed matters most, prefer lighter isolation.
- If shared files change often, validate storage visibility and lock semantics first.
- If human oversight matters, keep chat rooms simple and visible.

## Minimal Adoption Checklist

Before using this pattern anywhere, confirm that you have:

1. A private workspace for each agent.
2. A shared directory or equivalent for public coordination.
3. A clear list of private, shared, single-owner, and multi-writer files.
4. A chat surface for operator oversight.
5. A documented write-locking rule for shared files.
