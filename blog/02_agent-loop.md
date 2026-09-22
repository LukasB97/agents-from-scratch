# State and the Agent Loop

In the [Introduction](01_introduction.md), we saw the basic cycle of an agent harness: call the model, execute the actions it requests, add their results to the context, and call the model again.

To build that cycle in TypeScript, we begin with what the next model call needs to know.

Consider the search from the Introduction:

```text
User: Who is the current CEO of Example Corporation?
Model: I'll look that up.
       webSearch({ query: "Example Corporation leadership" })
Tool: Search results for that query.
```

The next call needs the question, the search request, and the result in that order. It also needs to know where each came from. We distinguish three **roles**: the user supplies the request, the model produces a response, and a tool supplies the result of an action. A **message** groups the content supplied by one role at one point in the exchange.

## Messages contain ordered parts

While we could represent a user message as a single string, today’s AI models also accept images, audio, and documents alongside text. To represent these different kinds of content within one message, we divide it into parts. Each part has a `type` that tells our program how to interpret its data.

Text needs a string. Images, audio, and documents can all be represented as files, which need their encoded bytes and a MIME type. This gives us two initial part types:
```
type TextPart = {
  type: "text";
  text: string;
};

type FilePart = {
  type: "file";
  data: string;
  mimeType: string;
};

type UserPart = TextPart | FilePart;
```

`FilePart.data` contains the file's bytes encoded as base64. Its MIME type identifies the file's format. An image can use `image/png`; a PDF can use `application/pdf`. A particular model may support only some MIME types.

A message that contains only text uses the same representation. Our original question remains unchanged:

```ts
const question: UserPart[] = [
  {
    type: "text",
    text: "Who is the current CEO of Example Corporation?",
  },
];
```

In the introductory example, the model produces text and asks the harness to perform an operation. We already have `TextPart` to represent text. A request for an operation needs the function's name and its arguments.

Since the model can call the same tool multiple times, the conversation may contain several requests and results with the same name. To show which result belongs to which request, we assign each call an ID and include it in the result:

```ts
type ToolCallPart = {
  type: "toolCall";
  id: string;
  tool: string;
  args: unknown;
};

type ModelPart = TextPart | ToolCallPart;
```

A model response can contain text, tool calls, or both. Their order is part of the response, so we represent its contents as an array of `ModelPart` values. The model message from our example becomes:

```ts
const searchRequest: ModelPart[] = [
  { type: "text", text: "I'll look that up." },
  {
    type: "toolCall",
    id: "search_1",
    tool: "webSearch",
    args: { query: "Example Corporation leadership" },
  },
];
```

## Connecting a tool result to its call

The harness executes `webSearch` and receives its output. The next model call needs both this output and the ID of the request it answers.

The search returns structured data. Text and files already have part types, so a tool output can combine one structured value with an ordered array of those parts:

```ts
type ToolOutput = {
  structured?: Json;
  content: (TextPart | FilePart)[];
};

type ToolResultPart = {
  type: "toolResult";
  callId: string;
  result: ToolOutput;
};

type ToolPart = ToolResultPart;
```

The same `FilePart` can therefore represent an image or PDF supplied by either the user or a tool. The result for our search becomes:

```ts
const searchResult: ToolResultPart = {
  type: "toolResult",
  callId: "search_1",
  result: {
    structured: {
      results: [
        {
          title: "Example Corporation Leadership",
          url: "https://example.com/leadership",
          snippet: "The company's leadership page.",
        },
      ],
    },
    content: [],
  },
};
```

## Messages form the state

The role determines which parts a message can contain:

```ts
type Message =
  | { role: "user"; parts: UserPart[] }
  | { role: "model"; parts: ModelPart[] }
  | { role: "tool"; parts: ToolPart[] };
```

The ordered message history is the harness's **state**:

```ts
type State = Message[];
```

Our exchange becomes:

```ts
const state: State = [
  { role: "user", parts: question },
  { role: "model", parts: searchRequest },
  { role: "tool", parts: [searchResult] },
];
```

A user message begins a **turn**. Model and tool messages continue that turn until the model produces a response without another tool call. The `runTurn` function below receives a state that already contains the starting user message and returns the model and tool messages produced after it.

## Describing the available tools

Suppose a developer must use a function without seeing its implementation. Its name identifies it, but does not explain how to use it. A function named `createInvoice` might create a draft, send the invoice, or charge the customer immediately.

The developer also needs to know which arguments the function accepts and what it returns. The structure of both can be described formally, while their meaning, the function's behavior, and its effects require documentation.

The model is in the same position because it cannot inspect the functions held by the harness. We therefore collect the same information in a tool definition. A function exposed to the model through such a definition is a **tool**.

```ts
type Schema = Record<string, unknown>;

type ToolDefinition = {
  name: string;
  description: string;
  inputSchema: Schema;
  outputSchema?: Schema;
};

const searchDefinition: ToolDefinition = {
  name: "webSearch",
  description: "Search the web and return matching pages with their titles, URLs, and snippets.",
  inputSchema: {
    type: "object",
    properties: {
      query: { type: "string" },
    },
    required: ["query"],
    additionalProperties: false,
  },
  outputSchema: {
    type: "object",
    properties: {
      results: {
        type: "array",
        items: {
          type: "object",
          properties: {
            title: { type: "string" },
            url: { type: "string" },
            snippet: { type: "string" },
          },
          required: ["title", "url", "snippet"],
          additionalProperties: false,
        },
      },
    },
    required: ["results"],
    additionalProperties: false,
  },
};
```

`outputSchema` is optional. When present, it describes `ToolOutput.structured`. The model can use it to see what data the tool provides and how to access it, and the harness can validate the value.

The state records what has happened in the conversation. Tool definitions describe which functions the model can request during the current work. The system instruction from the Introduction guides the model's behavior. We therefore supply all three as the input to a model call:

```ts
type ModelInput = {
  state: State;
  tools: ToolDefinition[];
  systemInstruction?: string;
};
```

For each following model call, the loop extends the existing state with the messages it has produced. Its caller supplies the tools and system instruction. `ModelInput` now contains the provider-independent information needed for one model call.

## Obtaining one model response

We now have everything the model needs. But `ModelInput` is our own format; we cannot send it directly to OpenAI, Anthropic, or another model provider. Each provider expects a different request and returns a different response.

We therefore need a component between the loop and the provider. It converts `ModelInput` into the provider's request format, sends the request, and converts the response into our `ModelPart` values. We call this component a **model adapter**.

The loop only needs to know the operation every adapter provides:

```ts
type ModelAdapter = {
  generate(input: ModelInput): Promise<ModelPart[]>;
};
```

Calling the adapter with our model input produces the next model response:

```ts
const output = await adapter.generate({
  state,
  tools,
  systemInstruction,
});
```

## Examining the model response

`output` is the model's next response. The loop preserves its parts as a model message before deciding what to do next:

```ts
const modelMessage: Message = {
  role: "model",
  parts: output,
};
```

A `TextPart` is already content produced by the model. The harness keeps it in the model message, which makes it part of the state supplied to later model calls.

A `ToolCallPart` instead asks the harness to perform an operation. The call remains in the model message, but the next model call also needs the result of that operation. If the response contains no tool calls, the turn can end. If it contains tool calls, the harness must execute them first.

<a id="der-harness-führt-den-call-aus"></a>

## Executing a tool call

When the agent calls a tool, the harness needs to execute it. A tool available to the harness must therefore include both its definition and its implementation:

```ts
type Tool = ToolDefinition & {
  execute(args: unknown): Promise<ToolOutput>;
};
```

The definition fields are sent to the model. The `execute` function remains in the harness.

The function validates its arguments against the tool's `inputSchema`, performs the operation, and produces a `ToolOutput`.

After execution, the harness checks the structured value when the definition has an output schema. In the code below, `validateOutput` returns normally when no schema is present and throws when the structured value is missing or does not match it.

When the model produces a call, the harness uses its `tool` field to find the corresponding function. A tool name must therefore identify exactly one available tool:

```ts
async function executeToolCall(
  call: ToolCallPart,
  tools: Tool[],
): Promise<ToolResultPart> {
  const tool = tools.find((candidate) => candidate.name === call.tool);
  if (!tool) throw new Error(`Unknown tool: ${call.tool}`);

  const output = await tool.execute(call.args);
  validateOutput(tool.outputSchema, output.structured);

  return {
    type: "toolResult",
    callId: call.id,
    result: output,
  };
}
```

The output becomes the `result` of a `ToolResultPart`.

## Several calls share one generation

Suppose the model already knows the contents of five files it wants to write. Requiring a new model call after every write would produce:

```text
Generation 1 -> write file a -> result
Generation 2 -> write file b -> result
Generation 3 -> write file c -> result
Generation 4 -> write file d -> result
Generation 5 -> write file e -> result
Generation 6 -> answer
```

The model can instead request all five writes in one response. The harness executes them together and starts a second generation with all five results:

```text
Generation 1 -> write a, write b, write c, write d, write e
Execution    -> result a, result b, result c, result d, result e
Generation 2 -> answer
```

Calls from the same model response start concurrently. `Promise.all` retains the order of its input promises in the returned array, even when the operations finish in another order:

```ts
const results = await Promise.all(calls.map((call) => executeToolCall(call, tools)));
```

The model cannot use one of these results while producing another call in the same response. If the next action depends on a result, it belongs in the next iteration. Actions that require ordered execution without another model decision belong in one explicitly sequential tool.

## Constructing one iteration

One iteration obtains a completed model response and executes all calls in it. It returns a model message followed by one tool message whose results match the call order:

```ts
async function runStep(
  adapter: ModelAdapter,
  state: State,
  tools: Tool[],
  systemInstruction?: string,
): Promise<Message[]> {
  const definitions = tools.map(
    ({ name, description, inputSchema, outputSchema }) => ({
      name,
      description,
      inputSchema,
      outputSchema,
    }),
  );

  const output = await adapter.generate({
    state,
    tools: definitions,
    systemInstruction,
  });

  const modelMessage: Message = { role: "model", parts: output };
  const calls = output.filter((part) => part.type === "toolCall");
  if (calls.length === 0) return [modelMessage];

  const results = await Promise.all(calls.map((call) => executeToolCall(call, tools)));

  return [modelMessage, { role: "tool", parts: results }];
}
```

## Repeating iterations for one turn

The turn continues whenever a step returns tool results. Each new iteration receives the original state followed by the messages already produced during this turn:

```ts
async function runTurn(
  adapter: ModelAdapter,
  state: State,
  tools: Tool[],
  systemInstruction?: string,
): Promise<Message[]> {
  let newMessages: Message[] = [];

  while (true) {
    const messages = await runStep(adapter, [...state, ...newMessages], tools, systemInstruction);
    newMessages = [...newMessages, ...messages];

    if (!messages.some((message) => message.role === "tool")) {
      return newMessages;
    }
  }
}
```

`runTurn` returns only the new model and tool messages. Its caller decides whether to add them to the conversation.

## When a tool fails

A tool can be unavailable, receive invalid arguments, or fail during its operation. `ToolResultPart` currently assumes that the execution produced an output. A failed execution has no such output, but the model still needs to know what went wrong.

We therefore distinguish the two possible outcomes inside its `result` field:

```ts
type ToolResult =
  | {
      status: "success";
      structured?: Json;
      content: (TextPart | FilePart)[];
    }
  | {
      status: "error";
      error: string;
    };

type ToolResultPart = {
  type: "toolResult";
  callId: string;
  result: ToolResult;
};
```

`status: "success"` means the harness executed the tool and obtained valid output. The output can report any domain outcome, including that the requested operation was unsuccessful. `status: "error"` means execution failed before the harness obtained valid output. This includes an unknown tool, invalid arguments, an exception during execution, or structured output that does not match a declared output schema. The error branch carries the message the model needs to respond.

We replace `executeToolCall` with a version that produces either outcome:

```ts
async function executeToolCall(
  call: ToolCallPart,
  tools: Tool[],
): Promise<ToolResultPart> {
  try {
    const tool = tools.find((candidate) => candidate.name === call.tool);
    if (!tool) throw new Error(`Unknown tool: ${call.tool}`);

    const output = await tool.execute(call.args);
    validateOutput(tool.outputSchema, output.structured);
    return {
      type: "toolResult",
      callId: call.id,
      result: { status: "success", ...output },
    };
  } catch (error) {
    return {
      type: "toolResult",
      callId: call.id,
      result: {
        status: "error",
        error: error instanceof Error ? error.message : String(error),
      },
    };
  }
}
```

Every invocation now resolves to its own result, so one failure does not cancel the other concurrent calls. The dispatch does not retry automatically: an operation may have changed something before failing, and repeating it could repeat that effect.

## When a model call fails

A failed model call produces no completed response for the next step. Earlier iterations in the same turn may nevertheless have executed tools and produced messages. If `runTurn` only throws, its caller loses access to that progress.

We return the collected messages together with the outcome:

```ts
type RunResult =
  | { status: "completed"; messages: Message[] }
  | { status: "failed"; messages: Message[]; error: unknown };
```

The completed messages remain available even when a later call fails:

```ts
async function runTurn(
  adapter: ModelAdapter,
  state: State,
  tools: Tool[],
  systemInstruction?: string,
): Promise<RunResult> {
  let newMessages: Message[] = [];

  try {
    while (true) {
      const messages = await runStep(adapter, [...state, ...newMessages], tools, systemInstruction);
      newMessages = [...newMessages, ...messages];

      if (!messages.some((message) => message.role === "tool")) {
        return { status: "completed", messages: newMessages };
      }
    }
  } catch (error) {
    return { status: "failed", messages: newMessages, error };
  }
}
```

The caller can retain `messages` in either case. Because this version waits for a complete model response before starting its calls, a failed generation contributes neither a model message nor an unfinished tool execution.

## Putting the loop into a conversation

The command-line chat appends each user message before starting its turn, then appends the messages returned by that turn:

```ts
import { createInterface } from "node:readline/promises";

async function chat(
  adapter: ModelAdapter,
  tools: Tool[],
  systemInstruction?: string,
): Promise<void> {
  const input = createInterface({ input: process.stdin, output: process.stdout });
  let conversation: State = [];

  try {
    while (true) {
      const text = await input.question("> ");
      if (!text) break;

      conversation = [...conversation, { role: "user", parts: [{ type: "text", text }] }];

      const result = await runTurn(adapter, conversation, tools, systemInstruction);
      conversation = [...conversation, ...result.messages];

      for (const message of result.messages) {
        if (message.role !== "model") continue;

        for (const part of message.parts) {
          if (part.type === "text") console.log(part.text);
        }
      }

      if (result.status === "failed") {
        console.error(result.error);
        break;
      }
    }
  } finally {
    input.close();
  }
}
```

The state connects two repetitions: the chat accepts successive user inputs, while each turn repeats model calls until the model requests no further action.

## Appendix: representing output schemas

`ToolDefinition` retains an output schema independently of any provider. Not every provider accepts an output schema as a separate tool field, so a model adapter chooses one of two representations through its configuration:

```ts
type OutputSchemaMode = "native" | "description";
```

When the provider supports a dedicated field, `native` mode places `outputSchema` there. Otherwise, `description` mode appends a rendered form of the schema to the description sent to the provider. This changes only the provider request; it does not modify the description stored in `ToolDefinition`.

## Appendix: preserving model reasoning

Almost all current agentic models use reasoning. A model works through a task before producing an answer or requesting a tool. That reasoning can contain intermediate conclusions, assumptions, and plans that are not repeated in the answer or tool call.

When the tool returns its result, the model can use its earlier reasoning to continue. The same reasoning can also remain useful across user turns. For a model trained to continue with that information present, removing it changes the conditions under which it operates: earlier outputs remain in the conversation, while part of the context they depended on is gone. The model may need to reconstruct that context, repeat work, or reach a different conclusion.

Our state therefore needs to preserve the reasoning information the provider makes available for later calls.

Providers represent this information differently. Some expose readable reasoning; others return encrypted content or structures containing signatures. We need a common place to preserve that information while allowing its data type to vary:

```ts
type ReasoningPart<T> = {
  type: "reasoning";
  data: T;
};

type ModelPart<T> = TextPart | ToolCallPart | ReasoningPart<T>;
```

`T` is the reasoning data type used by the chosen model adapter. It can be a string or a structured value containing everything needed to return the reasoning on later calls. A readable summary alone is insufficient when the provider requires additional information for continuation.

The type parameter `T` also passes through the message and state types, the model adapter, and the corresponding function signatures. The loop’s behavior remains unchanged.

The model adapter converts the provider's reasoning representation into a `ReasoningPart<T>` and reconstructs the required representation when sending the conversation back. This includes preserving any signatures and associations with other response parts. The loop keeps the reasoning part in its original position in the model message without interpreting or modifying `data`.

The loop already preserves the complete model response and executes only tool calls. Reasoning therefore remains in the state without requiring another branch in the loop.

Presentation remains separate from preservation. The chat displays `TextPart` values as answer text. A user interface may also display readable reasoning, but showing or hiding it does not change what the harness retains for the next model call.
