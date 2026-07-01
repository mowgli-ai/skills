<p align="center">
  <a href="https://mowgli.ai">
    <img src="assets/mowgli-logo.png" alt="Mowgli" width="96" height="96" />
  </a>
</p>

<h1 align="center">Mowgli Skill</h1>

<p align="center">
  Agent Skills for <a href="https://mowgli.ai">Mowgli</a>: Intelligent canvas for product design with context and taste.
</p>

---

## What is Mowgli?

[Mowgli](https://mowgli.ai) is an intelligent, spec-driven canvas for product
design. It stores a frontend as a set of screens, each in a variety of states,
plus a specification with user journeys that serves as the source-of-truth.
Mowgli screens are just code, which means there is zero drift between the design
and the implementation.

Mowgli allows a user to quickly and easily experiment, tweak, extending, and
brainstorm on a design and has full version control. Coupled with its
collaboration & team features and coding agent integration, it's a powerful tool
for aligning on how the product will look and feel before diving into
implementation.

## The Mowgli skill

This repository publishes a single skill, **`mowgli`**, which lets a coding
agent push designs from a real codebase into a Mowgli project and pull them back.
This is achieved using the `mowgli` CLI, published on
[npm](https://www.npmjs.com/mowgli-cli). It keeps the screens and the
specification in sync so the project stays spec-backed on every change.

## Install

Install the skill into any skills-aware agent with a single command:

```bash
npx skills add mowgli-ai/skills
```

## No skills support? Use the MCP server instead

If your agent does not support Agent Skills, or cannot run terminal commands,
you can get the same Mowgli interoperability by connecting the Mowgli MCP server
at:

```
https://app.mowgli.ai/mcp
```

## Links

- Mowgli — https://mowgli.ai/agent
- Vercel Skills directory — https://skills.sh
