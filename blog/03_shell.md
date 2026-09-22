# The Shell

In Chapter 1, we described a coding agent that reads files, changes code, and runs tests. In Chapter 2, we built the loop that lets the model request these actions and receive their results. Now we need tools through which the agent can perform that work on the user’s computer.

We could give it a separate tool for each operation: reading a file, editing code, inspecting Git changes, running tests. But many of these operations are already available through command-line programs. Giving the agent a shell lets it use those programs directly.

The model supplies commands as text, and the installed programs perform the work. With Git installed, the agent can inspect changes. With Python installed, it can write and execute a program. The shell gives the agent broad capabilities without requiring a separate harness function for every action.

This raises a design question: **if the agent has a shell, which operations still benefit from a specialized tool?**

Consider a simple example. Suppose the agent needs to add two numbers. With Python available, it can run:

```sh
python -c "print(137 + 284)"
```

We could provide a dedicated `add({ left: 137, right: 284 })` tool. But for this task, the tool saves little work. The calculation is already easy to express and execute.

Now consider retrieving the contents of a web page. The agent could use Python’s `requests` library to fetch the HTML. But getting useful page content may require more: handling failed requests, rendering JavaScript, or extracting the article from navigation bars and other surrounding text.

The agent can write code for those tasks too. But if it frequently needs web pages, repeatedly constructing that code costs time and context space. A maintained `scrape({ url })` tool can handle that repeated work.

Its value comes from the behavior behind the interface: what it handles, how reliably it works, and how useful its output is. A thin wrapper around the same HTTP request would offer much less.

This gives us two questions for evaluating specialized tools: how often does the agent need the operation, and how much does the tool improve on the solution it would otherwise construct? Frequent use makes even a modest improvement valuable. Little use and little improvement give us little reason to add another tool.

The shell is our starting point for that comparison. We now need to decide how commands enter it and how their results return to the model.

## Running a command

Imagine giving the agent a function that accepts a command. The harness starts a shell, runs the command, waits for it to finish, and returns the output. The shell exits afterward.

The `type` argument selects which shell to start. The harness exposes the shell types available in its environment:

```ts
shell({
  type: "bash",
  input: "git status",
});
```

To see where this becomes limiting, recall how Chapter 2 executes tools. A model response can contain several calls. The harness starts them concurrently and uses `Promise.all` to collect their results before calling the model again.

Collecting those results also satisfies a protocol requirement. Anthropic, for example, requires a matching result for every tool call in the batch. Five calls require five results, including results for calls that fail. [Its documentation specifies this requirement.](https://platform.claude.com/docs/en/agents-and-tools/tool-use/parallel-tool-use)

For a fast operation like reading a file, waiting makes sense. The operation is short, and its result is what the model needs next.

But a shell command might run for several minutes. A dataset download could continue while the agent does other useful work. With our current implementation, that download holds back its tool result, so the harness cannot make the next model call. Starting the other calls concurrently does not remove this dependency.

We could add a timeout that kills the command after thirty seconds:

```ts
shell({
  type: "bash",
  input: "curl -L -o dataset.zip https://example.com/dataset.zip",
  timeout: 30,
});
```

This bounds the delay, but it also stops useful work. A download that needed one more second would be terminated simply because we wanted the model to continue.

We need to separate two decisions: **how long a command may run**, and **how long we are willing to wait for it right now**.

To let the agent loop continue while a command is running, the shell tool must be able to return before the command finishes. That result can confirm that the work started without claiming that it succeeded.

Once the model continues, it needs a way to refer to that work. We give the managed execution a reference, which we call a **handle**. The harness tracks the process and its eventual result under this handle.

Output may arrive after the tool call has returned, so the harness also needs to retain it independently of that call. **Every shell execution gets its own append-only logfile from the moment it starts, regardless of output size.** The log records submitted input and received output, clearly distinguishing the two.

A successful shell creation returns both the handle and the logfile path:

```json
{
  "handle": "execution_1",
  "logPath": "/logs/execution_1.log"
}
```

The tool call has supplied its required response, while the execution can continue independently. The model knows where to inspect its log and has a handle through which it can request later observations.

We call an execution **awaitable** when its progress and results can be observed through a handle independently of the call that started it.

The agent can now make decisions while a command runs. But how does it wait when its next action depends on the command’s result?

## Waiting for results

One option is to add a `waitFor` argument to each shell call. The call starts its command, waits for up to a specified duration, and returns whatever is available:

```ts
shell({
  type: "bash",
  input: "npm test",
  waitFor: 5,
});
```

For a single command, this works. Several calls in the same response make the timing less straightforward.

Suppose one call waits for one second and another waits for ten. The first returns a snapshot after one second, but the harness still needs the second call’s result before invoking the model. By the time the model receives both, the first snapshot may be nine seconds old.

We could refresh the earlier snapshot before returning the batch. Or we could return when the first call finishes, end the other waits, and collect their current states. Either approach requires coordinating the separate waits.

Since the model receives one batch of results, we make waiting a decision about the group of executions it wants to observe. We expose that decision through a separate tool: `waitFor`.

The `shell` tool starts work and returns without waiting for it to finish. `waitFor` observes selected awaitable executions. There is nothing shell-specific about this: any tool that exposes a handle for continuing work can use the same waiting mechanism.

It takes three optional arguments:

```ts
type WaitForInput = {
  handles?: string[];
  when?: "any" | "all";
  timeout?: number;
};
```

- `handles` selects which executions to observe. If omitted, it selects the handles exposed by the preceding calls in the same batch.
- `when` selects the completion condition. `any` waits until at least one selected execution has finished its submitted work; `all` waits until every selected execution has. It defaults to `any`.
- `timeout` limits the wait in seconds. If omitted, the harness uses a finite default.

For the commands we have introduced so far, a process exit supplies the completion event, whether the command succeeds or fails. Progress updates and output arriving on their own do not satisfy the wait. A command that has already exited satisfies the completion condition immediately, even if an earlier wait reported its result.

Often, the model already knows it wants to wait when it requests the work. Requiring another generation just to say “now wait for those commands” would add a round trip without adding a decision.

We therefore allow `waitFor` as the final call in the same response. These three calls belong to one model response:

```ts
shell({ type: "bash", input: "npm run test:unit" });
shell({ type: "bash", input: "npm run test:integration" });
waitFor({ when: "all" });
```

The harness starts both test runs concurrently. With no explicit handles, the final call selects those two executions and waits for both.

Before evaluating `waitFor`, the harness must know which execution handles belong to the preceding calls in the same response. It establishes these handles when starting those calls. The executions themselves continue concurrently, and `waitFor` observes them together.

`Promise.all` still collects one result per tool call. The shell results acknowledge the started work; the `waitFor` result supplies the later observations.

Calling `waitFor()` with no arguments uses the same batch selection, waits for any selected execution to finish its submitted work, and applies the default timeout.

To wait on work from an earlier response, the model supplies its handles explicitly:

```ts
waitFor({
  handles: ["execution_1", "execution_2"],
  when: "any",
  timeout: 5,
});
```

This waits for at most five seconds. If one execution has already finished its submitted work, the wait returns immediately; otherwise, it returns when one finishes or the timeout expires. Reaching the wait timeout is a normal outcome: the tool returns the current states and available output. **Unfinished work continues.**

Execution output and completion results belong in the `waitFor` result, grouped by handle. It reports the current state, newly available execution results, and new output from every selected execution.

For the two executions selected above, that could look like:

```json
{
  "execution_1": {
    "status": "running",
    "results": [],
    "newOutput": "PASS: src/agent.test.ts\n",
    "logPath": "/logs/execution_1.log"
  },
  "execution_2": {
    "status": "exited",
    "results": [{ "type": "exit", "exitCode": 0 }],
    "newOutput": "Build successful.\n",
    "logPath": "/logs/execution_2.log"
  }
}
```

Suppose the agent starts three commands and waits with `when: "any"`. The second command finishes while the first and third are still running. The wait returns the second command’s completed result together with the current states and partial output of the other two. The wait call has completed, even though some of the work it observes continues.

Keeping these observations in `waitFor` gives us the same behavior whether it appears in the original batch or a later response. Each wait produces a new observation. The earlier shell results remain acknowledgments of the work they started.

Completion and reporting are separate. Suppose a wait with `when: "all"` for A and B times out after A finishes, and reports A's result. A later wait with `when: "all"` for the same handles still counts A as complete and waits only for B. A remains in the returned observations, with an empty `results` array and empty `newOutput` if nothing new is available.

“New output” means output since the last observation delivered to the model. It includes output produced while the model was thinking, before the current wait began. The harness therefore tracks the log position covered by each observation.

A process can produce far more output than fits in the model’s context. Returning everything could make the next model request too large. Simply discarding the excess would prevent the agent from investigating information it had never seen.

We therefore limit the output included in a response while preserving the complete record in the execution’s logfile. Small amounts of new output return directly. When the new output exceeds the limit, `waitFor` returns its beginning and end, separated by a marker stating the exact number of omitted characters, alongside the logfile reference:

```text
[first part of new output]

… [213,213 characters omitted] …

[last part of new output]
```

The excerpt and marker together fit within the output limit. This applies to the new output since the previous observation, so each response does not repeat the beginning of the entire logfile.

The agent can then use the shell to search or inspect the saved log. A large test report remains available even when only its beginning and end fit in the immediate response.

## Using the same shell again

So far, each shell execution runs one command and then exits. Its result and logfile remain available afterward. This is enough for commands like `git status`, test runs, and downloads. A command can take a long time without needing a reusable shell.

Other workflows need state to survive across inputs. The agent may change a working directory or set environment variables and want later commands to use them. Or it may open an interactive Python interpreter and hold data in memory while exploring it through several expressions.

For these workflows, we keep the shell or interpreter alive between calls. We call this a **persistent session**.

The harness starts persistent shells and interpreters in interactive mode with a terminal.

We add a `persistent` argument to choose this behavior. It defaults to `false`, meaning the shell exits after its command finishes. Setting it to `true` keeps the shell open for further input. The `type` argument still chooses which shell to launch.

As the agent starts more executions, meaningful names can make them easier to track. We therefore also let it supply an unused handle. If it omits the handle, the harness generates one.

```ts
shell({
  type: "bash",
  persistent: true,
  handle: "work",
  input: "cd src\n",
});
```

A persistent shell receives text through its input stream. The trailing `\n` submits the command.

The handle `work` identifies this persistent shell execution. Its creation returns the handle and logfile path, just as it does for a nonpersistent execution.

A later call sends more input to the same shell through its `handle`:

```ts
shell({
  handle: "work",
  input: "pwd\n",
});
```

The existing session already has a shell type and a lifetime, so this call omits `type` and `persistent`. Supplying `type` requests a new execution; supplying `handle` and `input` without the creation arguments addresses an existing one. An unknown handle is an error, never an implicit request to create a shell.

Because both inputs reach the same process, the working directory survives. Writing to that process does not start another managed execution. The call acknowledges that it supplied the input and returns the existing `work` handle.

This gives `shell` three uses:

- With `type` supplied and `persistent` omitted or set to `false`, it starts the chosen shell, runs a command, and exits afterward.
- With `type` and `persistent: true`, it creates a persistent shell.
- With `handle` and `input`, and no creation arguments, it writes into an existing shell.

All three return without waiting for the submitted work to finish. Waiting remains the responsibility of `waitFor`.

For a nonpersistent shell, the handle identifies one command’s execution. For a persistent shell, it identifies the session across multiple inputs. Each new nonpersistent execution gets a new handle; further input to a persistent shell reuses its existing one.

The session’s logfile follows the same lifetime. Each submitted input and all received output are appended to that one file, including output that arrives between observations. The agent can inspect the interaction as a whole whenever it needs to.

## Recognizing a result

We now need to be precise about what it means for an execution to produce a result.

For a command whose shell exits afterward, the process exit supplies the result. A persistent shell, however, stays alive after a command finishes. Waiting for that shell to exit would miss the point when the agent can continue using it.

The harness needs to recognize when the managed shell has finished evaluating the submitted input. A shell prompt looks like a natural place to find that signal. Bash, for example, provides a hook before displaying its next primary prompt. An integration can use it to emit a marker the harness recognizes. [The Bash manual describes this hook.](https://www.gnu.org/software/bash/manual/html_node/Interactive-Shell-Behavior.html)

A prompt alone does not necessarily mean that the entire submission has finished. Consider:

```sh
echo ready
sleep 60
```

The shell may return to its prompt after `echo ready`, while `sleep 60` still remains to be executed. Reporting completion at that point would be too early.

Each supported shell type therefore needs an integration that recognizes completion of the entire submission, whether evaluation succeeds or ends with an error. The harness knows which shell it started and uses the corresponding integration.

With that signal, `waitFor` can observe the submission's completion while leaving the session available for another input. A pause in output cannot serve the same purpose: a quiet command may still be running. Completion here means the interpreter has finished evaluating the submission; work it explicitly launches in the background may continue.

This makes the relationship between handles and results explicit: **a handle identifies an execution; a result is a completion event produced by that execution.**

A nonpersistent execution produces its final result when it exits. A persistent shell can report several command completions over its lifetime, followed eventually by its own exit. The handle stays the same throughout.

For a persistent shell, the `results` array can contain a command completion such as `{"type": "command_completed", "exitCode": 0}` while the shell’s process status remains `running`.

The harness keeps track of which completion events it has already reported for each handle. An event that arrived before the current wait began is still included if no earlier observation reported it. If several events accumulated between observations, the next wait reports them together. Reporting an event changes what the next observation includes, not whether the work is complete.

For a persistent shell, completion of its current submission satisfies the wait until further input is supplied; the shell's exit also satisfies it. Further input establishes the completion condition for the new submission. With persistent shells, `when: "all"` therefore waits until every selected shell meets its completion condition or exits. The shells can remain alive after the wait returns. As with any wait, the timeout still bounds how long the call waits.

Each `waitFor` call still returns one tool result. That tool result contains observations from the selected executions. Completing a wait does not complete the executions it observes.

Nested interactive programs reveal the limit of the managed shell's integration.

Suppose the agent starts Python inside the Bash session `work`. Python now receives the input. It can evaluate an expression and become ready for another while Bash is still waiting for Python to exit. Python has its own prompts and input rules. [Its interpreter documentation describes these interactions.](https://docs.python.org/3/tutorial/interpreter.html)

A completion marker from Bash can tell us when control returns to Bash. It cannot tell us when Python has finished evaluating an individual expression.

If the harness manages Python directly, it can use an integration that understands Python’s input boundaries. Starting Python inside another shell does not automatically provide that integration.

For interactions outside that integration, the agent can supply an optional `doneWhen` output marker. This replaces automatic completion detection for that submission. The input is passed through without adding completion commands for the outer shell.

For example, with Python running inside `work`, the agent can request:

```ts
shell({
  handle: "work",
  input: 'import time; time.sleep(60); print("__done_7f3a__")\n',
  doneWhen: "__done_7f3a__",
});
waitFor({ timeout: 90 });
```

Python prints the marker after the sleep finishes. That exact output line satisfies the completion condition, even though Python remains open and control has not returned to Bash. The marker must be distinct from ordinary output, and its appearance in logged or terminal-echoed input does not count. Matching begins with the new output for this submission. Reaching a marker confirms that the program reached that point; it does not by itself provide an exit code or prove success.

The managed process exiting still satisfies the wait. If no completion signal arrives before the timeout, `waitFor` returns the current observation. The agent can inspect the session log and send further input based on what the program reports.

Sending input also does not mean that a new command has begun executing. While Python is active, the text goes to Python. While a program is still running, further input may be consumed by that program or remain buffered. The shell tool confirms that it supplied the input; the observed output shows what the receiving program did with it.

## Finding and stopping shells

Over the course of a task, several shells may remain open. Earlier tool results may no longer reflect their current state, so the agent needs a way to inspect what exists now.

`shell.list` returns their handles, current states, and logfile paths:

```ts
shell.list();
```

For example:

```json
[
  {
    "handle": "work",
    "status": "running",
    "logPath": "/logs/work.log"
  }
]
```

Once a shell is no longer useful, the agent needs a way to stop it. We expose explicit termination through `shell.close`:

```ts
shell.close({ handle: "work" });
```

This stops the selected shell and the processes the harness manages through it. The same tool can stop a long-running command that was started without a persistent session.

The tool confirms closure only after those processes have stopped. The agent can then proceed knowing that termination has completed.
