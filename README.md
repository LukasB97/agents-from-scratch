# Agents From Scratch

We’re building Emerge, a fully working, capable agent harness, from the ground up. This series explores how to get there from first principles, one step at a time.

We begin with a simple question: how does a model that generates text become an agent that can do useful work? From there, we develop the loop, give it tools, and explore what it takes to keep working through longer tasks—with a growing history, limited context, and other agents helping along the way.

The goal is to understand the whole system, from its smallest parts to how we work with it. Code examples make the ideas concrete. Runnable reference implementations will follow as Emerge takes shape.

## The series

| Chapter | What we explore | Status |
| --- | --- | --- |
| [01 · The Essence of AI Agents](blog/01_introduction.md) | How text completion becomes conversation, tool use, and an agent loop | Ready |
| [02 · State and the Agent Loop](blog/02_agent-loop.md) | Representing a conversation and expressing the loop in TypeScript | Ready |
| [03 · The Shell](blog/03_shell.md) | Running programs and keeping processes available between calls | Ready |
| 04 · Reading and Changing Files | Giving the agent a reliable way to inspect and edit files | Planned |
| 05 · Streaming the Agent’s Work | Seeing responses and tool activity as they happen | Planned |
| 06 · Persistence | Recording progress and continuing after a restart | Planned |
| 07 · Context Management | Keeping useful information as the conversation grows | Planned |
| 08 · Subagents | Sharing work between agents with their own context | Planned |
| 09 · Interrupts and Steering | Guiding, pausing, and stopping ongoing work | Planned |
| 10 · The User Interface | Working with the agent through a terminal | Planned |
| 11 · The App Server | Keeping work running independently of the interface | Planned |

The planned titles and order may evolve.

## Further explorations

Once the foundation is in place, we’ll explore what it makes possible.

**Self-evolving agents** can reshape their own context, organizing what they’ve learned for the work ahead.

**Programmatic tool calling** lets agents combine tools through code, using loops, conditions, and intermediate results within a single call.

**Computer and browser use** extends their reach into applications and websites.

## Reference implementations

The reference implementations will bring the designs together into a runnable harness, beginning with TypeScript. You’ll be able to run Emerge, experiment with its parts, and build on it.

Chapters and code will be published here as they become ready.
