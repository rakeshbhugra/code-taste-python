# Architecture Concepts

A working glossary of the terms from "Architecture Patterns with Python" (Cosmic Python) and DDD, in plain English. Taught in dependency order — each concept builds on the previous.

## Reading order

1. [Domain](./domain.md) — what your app is fundamentally about
2. [Repository](./repository.md) — the one place that knows how to load and save domain entities
3. Service — *coming next*
4. Adapter — *coming next*
5. Unit of Work — *coming next*

## Why this order

Each concept assumes the previous one:

- **Repository** only makes sense once you know what a *Domain* is (because a Repository is "a thing that stores Domain entities").
- **Service** only makes sense once you have a Domain *and* a Repository (because a Service orchestrates between them).
- **Adapter** is a generalization of the Repository pattern — easier once you've seen one specific case.
- **Unit of Work** is the last piece — it ties Services and Repositories together at transaction boundaries.

Reading them out of order is how the jargon stays confusing.

## How each doc is structured

Every concept doc follows the same shape:

1. **One-line definition**
2. **Real examples** — anchored to projects you know (tytona, hypedar, requests)
3. **What's in / what's out** — clear boundary
4. **The trap** — the mistake most people make with this concept
5. **How to spot it in your code** — concrete check using your existing projects
6. **Self-check** — questions to test understanding before moving on
7. **Things to steal** — the takeaways worth keeping

If you can answer the self-check questions without looking, you've got that concept. Move on.
