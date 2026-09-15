# AI Interface Vue

A dependency-light Vue 3 component for displaying AI conversations, including streamed assistant responses, reasoning, tool calls, tool results, citations, token usage, and Markdown-formatted assistant content.

The component is display-only. It does not open a WebSocket, send messages, make HTTP requests, or manage a conversation API. The consuming application owns transport and passes complete messages or stream events to the component.

## Features

- Vue 3 component with TypeScript support.
- Pure CSS styling with no Tailwind or UI framework dependency.
- User and assistant message bubbles with configurable names, initials, and assistant avatar image.
- Optional avatars, tool details, provider/model metadata, and token usage.
- Markdown rendering for assistant message content only.
- Raw HTML disabled in Markdown output.
- Streaming support for text, thinking/reasoning, tool calls, tool results, provider tools, citations, errors, and completion events.
- Exposed methods for processing data, controlling the thinking indicator, and clearing the conversation.
- Additive class overrides for every meaningful rendered element.
- Responsive layout, dark-mode support through `prefers-color-scheme`, and reduced-motion support.

## Requirements

- Vue 3.5 or later.
- Node.js and npm.

`vue` is a peer dependency. `markdown-it` is installed by this package and is used to render assistant content.

## Installation

This package is not currently published to the npm registry. Install it directly from the public [SynergiTech/ai-interface-vue](https://github.com/SynergiTech/ai-interface-vue) GitHub repository instead.

The recommended installation command tracks the `main` branch:

```bash
npm install github:SynergiTech/ai-interface-vue#main
```

Because the repository is public, no GitHub authentication is needed when using the shorthand above. The equivalent HTTPS URL is:

```bash
npm install git+https://github.com/SynergiTech/ai-interface-vue.git#main
```

You can also use SSH if your environment already has GitHub SSH authentication configured:

```bash
npm install git+ssh://git@github.com/SynergiTech/ai-interface-vue.git#main
```

For reproducible installations, pin the dependency to a release tag or commit. The current release tag is `0.1.0`:

```bash
npm install github:SynergiTech/ai-interface-vue#0.1.0
```

The package has an npm `prepare` script, so npm builds the package from source during a GitHub installation before adding it to the consuming project. The `"private": true` setting prevents accidental publication to npm; it does not prevent installation from GitHub.

### Refreshing a cached GitHub installation

When installing from a branch such as `main`, npm may use the commit recorded in the consuming application's `package-lock.json`. Re-run the explicit install command with `--force` to fetch the current remote branch and update the lockfile:

```bash
npm install github:SynergiTech/ai-interface-vue#main --force
```

If the old package is still present in `node_modules`, remove only this dependency and install it again:

```bash
npm uninstall ai-interface-vue
npm install github:SynergiTech/ai-interface-vue#main --force
```

Do not use `npm ci` to update a branch-based Git dependency. `npm ci` intentionally installs the exact revision already recorded in the lockfile. Update the dependency with the explicit `npm install` command first, then commit the resulting `package.json` and `package-lock.json` changes. Clearing the entire npm cache is normally unnecessary; use `npm cache clean --force` only as a last resort for a confirmed cache problem.

## Basic usage

Import the component and its generated stylesheet:

```vue
<script setup lang="ts">
import { ref } from 'vue';
import AiInterface from 'ai-interface-vue';
import 'ai-interface-vue/style.css';

const aiInterface = ref<InstanceType<typeof AiInterface> | null>(null);

const showConversation = (): void => {
    aiInterface.value?.processMessages([
        {
            uuid: 'user-1',
            type: 'user',
            content: 'What can you tell me about Demo Event?',
            status: 'completed',
            created_at: new Date().toISOString(),
            metadata: null,
        },
        {
            uuid: 'assistant-1',
            type: 'assistant',
            content: '**Demo Event** is currently **open**.',
            status: 'completed',
            created_at: new Date().toISOString(),
            metadata: {
                model: 'gpt-5.6-luna',
                provider: 'openai',
                parts: [],
                usage: {
                    promptTokens: 553,
                    thoughtTokens: 52,
                    completionTokens: 189,
                    cacheReadInputTokens: 0,
                    cacheWriteInputTokens: null,
                },
                citations: null,
                response_id: 'response-1',
                finish_reason: 'stop',
            },
        },
    ]);
};
</script>

<template>
    <AiInterface ref="aiInterface" />
    <button type="button" @click="showConversation">Load conversation</button>
</template>
```

`processMessages` appends the supplied messages to the current list. It does not replace or deduplicate messages.

## Props

All props are optional. The defaults below are used when a prop is omitted.

| Prop | Type | Default | Description |
| --- | --- | --- | --- |
| `classes` | `AiInterfaceClasses` | `{}` | Additional Vue class values for the rendered elements. See [Class overrides](#class-overrides). |
| `avatars` | `boolean` | `true` | Shows the circular avatar or initials element for each message. When `false`, the avatar element is removed and the message body uses the full available width. |
| `assistantAvatarUrl` | `string \| null` | `null` | URL for the assistant avatar image. The assistant initials are used when this is `null` or empty. User messages never use this image. |
| `assistantName` | `string` | `'Assistant'` | Label displayed above assistant messages. |
| `assistantInitials` | `string` | `'AI'` | Text displayed inside the assistant avatar when no assistant image is available. |
| `userName` | `string` | `'You'` | Label displayed above user messages. |
| `userInitials` | `string` | `'You'` | Text displayed inside the user avatar. |
| `tools` | `boolean` | `true` | Shows thinking, tool-call, tool-result, and provider-tool details. The data is still stored when this is `false`; only its display is hidden. |
| `tokens` | `boolean` | `true` | Shows assistant token usage when usage values are available. |
| `provider` | `boolean` | `true` | Shows the provider name in assistant metadata. |
| `model` | `boolean` | `true` | Shows the model name in assistant metadata. |

The `provider`, `model`, and `tokens` props independently control the corresponding sections of the assistant metadata row. For example, set `:provider="false"` to hide the provider while leaving the model and token count visible.

## Exposed component API

Because the component uses `<script setup>`, these values and methods are available through a template ref:

```vue
<script setup lang="ts">
import { ref } from 'vue';
import AiInterface from 'ai-interface-vue';

const aiInterface = ref<InstanceType<typeof AiInterface> | null>(null);

const clearConversation = (): void => {
    aiInterface.value?.clearMessages();
};
</script>

<template>
    <AiInterface ref="aiInterface" />
    <button type="button" @click="clearConversation">Clear</button>
</template>
```

| Exposed value or method | Signature | Description |
| --- | --- | --- |
| `messages` | `Message[]` | Reactive array containing the messages currently displayed. |
| `thinking` | `boolean` | Reactive state for the global “Thinking…” indicator. |
| `processMessages` | `(messages: Message[]) => void` | Appends complete user or assistant messages. |
| `processStream` | `(stream: unknown, eventType?: string) => void` | Processes one stream event, a JSON-encoded event, an array of events, a WebSocket-style `{ event, data }` envelope, or a complete message object. |
| `setThinking` | `(value: boolean) => void` | Explicitly sets the global thinking indicator. |
| `clearMessages` | `() => void` | Clears all messages, hides thinking, and resets the active stream state. |

## Message format

### User message

```ts
interface UserMessage {
    uuid: string;
    type: 'user';
    content: string;
    status: 'streaming' | 'completed' | 'error';
    created_at: string;
    metadata: null;
}
```

### Assistant message

```ts
interface AssistantMessage {
    uuid: string;
    type: 'assistant';
    content: string;
    status: 'streaming' | 'completed' | 'error';
    created_at: string;
    metadata: {
        model: string;
        provider: string;
        parts: AssistantPart[];
        usage: Usage;
        citations: unknown[] | null;
        response_id: string;
        finish_reason: string;
    };
}

interface Usage {
    promptTokens: number;
    thoughtTokens: number | null;
    completionTokens: number;
    cacheReadInputTokens: number | null;
    cacheWriteInputTokens: number | null;
}
```

The component expects assistant metadata to contain `parts`, `usage`, and `citations`, even when those values are empty. A safe empty assistant metadata object is:

```ts
const emptyAssistantMetadata = {
    model: '',
    provider: '',
    parts: [],
    usage: {
        promptTokens: 0,
        thoughtTokens: null,
        completionTokens: 0,
        cacheReadInputTokens: null,
        cacheWriteInputTokens: null,
    },
    citations: null,
    response_id: '',
    finish_reason: '',
};
```

## Stream processing

`processStream` is intentionally transport-agnostic. The parent can pass data from Laravel Echo, Pusher, a native WebSocket, an HTTP stream, or any other source after receiving it.

The method accepts these input shapes:

```ts
// Direct event
aiInterface.value?.processStream({
    type: 'text_delta',
    message_id: 'assistant-1',
    delta: 'Hello',
});

// JSON-encoded event
aiInterface.value?.processStream(JSON.stringify({
    type: 'text_delta',
    message_id: 'assistant-1',
    delta: ' world',
}));

// Event envelope
aiInterface.value?.processStream({
    event: 'text_delta',
    data: {
        message_id: 'assistant-1',
        delta: '!',
    },
});

// Multiple events
aiInterface.value?.processStream([
    { type: 'text_start', message_id: 'assistant-1' },
    { type: 'text_delta', message_id: 'assistant-1', delta: 'Hello' },
]);
```

The optional `eventType` argument can be used when the event name is supplied separately:

```ts
aiInterface.value?.processStream(
    { message_id: 'assistant-1', delta: 'Hello' },
    'text_delta'
);
```

### Supported event types

| Normalized event | Accepted aliases | Main effect |
| --- | --- | --- |
| `stream_start` | `start`, `stream_started` | Starts a stream, records model/provider identifiers, and shows the thinking indicator. |
| `text_start` | `text_started` | Starts a text part and hides the thinking indicator. |
| `text_delta` | — | Appends the `delta` text to the assistant message and active text part. |
| `text_complete` | `text_end`, `text_completed` | Closes the active text part. |
| `thinking_start` | `reasoning_start`, `thinking_started` | Starts a thinking part and shows the thinking indicator. |
| `thinking_delta` | `reasoning_delta` | Appends reasoning text to the matching thinking part. |
| `thinking_complete` | `reasoning_end`, `thinking_completed` | Hides the thinking indicator. |
| `tool_call` | `tool_input_available` | Adds or updates a tool call and shows the thinking indicator. |
| `tool_result` | `tool_output_available` | Adds or updates a tool result and shows the thinking indicator. |
| `provider_tool_event` | — | Stores the complete event payload as a provider-tool part. |
| `citation` | `data_citation` | Adds the supplied citation to the assistant metadata. |
| `error` | — | Marks the active assistant message as errored and displays an error message when no content exists. |
| `stream_end` | `finish`, `stream_finished`, `stream_completed` | Applies usage, finish reason, response ID, and citations, marks the message completed, and resets stream state. |

Event names are normalized by replacing punctuation with underscores and converting to lowercase. Unknown events are ignored. Pusher lifecycle events beginning with `pusher:` are also ignored.

### Stream payload fields

The parser accepts the following common field aliases:

| Event | Fields |
| --- | --- |
| `stream_start` | `id`, `message_id` or `messageId`, `model`, `provider`, `response_id` or `responseId` |
| `text_start` | `message_id`, `messageId`, `message_uuid`, `messageUuid`, or `uuid`; a direct `text-start` event can also use `id` |
| `text_delta` | `delta`, plus the message ID fields above |
| `thinking_start` | `reasoning_id` or `reasoningId`, optional `summary`; a direct `reasoning-start` event can use `id` |
| `thinking_delta` | `reasoning_id` or `reasoningId`, `delta`, optional `summary`; a direct `reasoning-delta` event can use `id` |
| `tool_call` | Either one tool's `id`, `tool_id`, `toolId`, or `toolCallId`; name fields; and `arguments` or `input`, or a `calls` array containing those objects |
| `tool_result` | Either one result's `tool_call_id`, `toolCallId`, `tool_id`, `toolId`, or `id`, result/output, optional `error`, optional `success`, and optional arguments/input, or a `results` array containing those objects |
| `citation` | `citation`; if absent, the complete event payload is stored as the citation |
| `error` | `message` or `errorText` |
| `stream_end` | `messageMetadata` or `message_metadata`, nested `usage`, `finish_reason` or `finishReason`, `response_id` or `responseId`, and top-level `citations` |

Usage fields accept both camelCase and snake_case forms:

| Internal field | Accepted input fields |
| --- | --- |
| `promptTokens` | `promptTokens`, `prompt_tokens` |
| `thoughtTokens` | `thoughtTokens`, `thought_tokens` |
| `completionTokens` | `completionTokens`, `completion_tokens` |
| `cacheReadInputTokens` | `cacheReadInputTokens`, `cache_read_input_tokens` |
| `cacheWriteInputTokens` | `cacheWriteInputTokens`, `cache_write_input_tokens` |

### Complete messages passed to `processStream`

When `processStream` receives an object with a valid message `type`, `uuid`, `content`, and `status`, it treats the object as a complete message rather than a stream event. A new UUID is appended; an existing UUID is replaced. A completed or errored assistant message also hides the thinking indicator.

## Markdown rendering

Only `assistant` message `content` is rendered with `markdown-it`. User messages, thinking content, tool arguments, tool results, citations, and metadata are rendered as text or code blocks.

Raw HTML is disabled in the Markdown renderer. Markdown features such as headings, emphasis, links, lists, blockquotes, inline code, fenced code blocks, horizontal rules, and tables are styled by the component's pure CSS.

```ts
aiInterface.value?.processMessages([
    {
        uuid: 'assistant-1',
        type: 'assistant',
        content: '# Result\n\nThe value is **£100**.\n\n- Revenue\n- Costs',
        status: 'completed',
        created_at: new Date().toISOString(),
        metadata: emptyAssistantMetadata,
    },
]);
```

## Class overrides

The `classes` prop adds class values to the component's existing semantic classes. It does not remove or replace the built-in classes, so default styling remains available unless your CSS overrides it.

Each value accepts the same forms supported by Vue's `:class` binding:

```ts
type ClassValue =
    | string
    | string[]
    | Record<string, boolean>
    | null
    | undefined;
```

Example:

```vue
<AiInterface
    :classes="{
        root: 'conversation-panel',
        assistantBubble: ['conversation-panel__bubble', 'conversation-panel__bubble--assistant'],
        tokens: { 'conversation-panel__tokens': true },
        avatarsHiddenMessage: 'conversation-panel__message--no-avatar',
    }"
    :avatars="false"
/>
```

### Complete override table

| Override key | Applied to | Typical use |
| --- | --- | --- |
| `root` | Root `<section>` | Set the outer layout, theme, width, or spacing. |
| `empty` | Empty-state wrapper | Style the no-messages state. |
| `emptyContent` | Empty-state content container | Control empty-state inner layout. |
| `emptyTitle` | Empty-state heading | Style “No messages yet”. |
| `emptyCopy` | Empty-state description | Style the empty-state supporting text. |
| `messages` | Messages list container | Control list layout and spacing. |
| `message` | Every message `<article>` | Apply shared message styles. |
| `userMessage` | User message `<article>` | Target user rows. |
| `assistantMessage` | Assistant message `<article>` | Target assistant rows. |
| `errorMessage` | Message `<article>` with `status: 'error'` | Highlight errored messages. |
| `avatarsHiddenMessage` | Message `<article>` when `avatars` is `false` | Adjust layout when avatars are disabled. |
| `avatar` | Every avatar container | Set shared avatar sizing or appearance. |
| `userAvatar` | User avatar container | Style the user avatar separately. |
| `assistantAvatar` | Assistant avatar container | Style the assistant avatar separately. |
| `avatarImage` | Assistant avatar `<img>` | Control image styling. |
| `avatarInitials` | Initials `<span>` | Style fallback initials. |
| `messageBody` | Message body wrapper | Control message content width and layout. |
| `messageHeading` | Author/status row | Style the heading row. |
| `messageAuthor` | Author name | Style user or assistant names. |
| `messageStatus` | Status label | Style shared status labels. |
| `streamingStatus` | “Working” status label | Target streaming messages. |
| `errorStatus` | “Error” status label | Target errored messages. |
| `bubble` | Every message bubble | Apply shared bubble styles. |
| `userBubble` | User message bubble | Style user content bubbles. |
| `assistantBubble` | Assistant message bubble | Style assistant content bubbles. |
| `content` | Assistant Markdown content wrapper | Style rendered assistant Markdown. |
| `parts` | Assistant parts container | Control spacing between thinking/tools. |
| `part` | Every thinking/tool/provider-tool part | Apply shared part styling. |
| `thinkingPart` | Thinking part | Style reasoning details. |
| `toolCallPart` | Tool-call part | Style tool invocation details. |
| `toolResultPart` | Tool-result part | Style tool output details. |
| `providerToolPart` | Provider-tool part | Style provider-specific tool events. |
| `partHeading` | Thinking/tool/citation heading | Style part labels and counts. |
| `partTitle` | Part title text | Style “Thinking”, “Tool calls”, “Tool results”, or “Citations”. |
| `partLabel` | Part count/status label | Style counts or “Complete”/“In progress”. |
| `partContent` | Thinking text content | Style the rendered reasoning text. |
| `toolList` | Tool call/result list | Control tool item layout and spacing. |
| `toolItem` | Individual tool call/result item | Style each tool detail card. |
| `toolName` | Tool name | Style tool names. |
| `toolResultHeading` | Tool result name/status row | Control result header layout. |
| `resultStatus` | Tool result status | Apply shared success/failure status styles. |
| `resultSuccess` | Successful tool result status | Target successful results. |
| `resultError` | Failed tool result status | Target failed results. |
| `code` | Tool/provider `<pre>` blocks | Style serialized arguments, results, and provider data. |
| `toolError` | Tool error paragraph | Style tool execution errors. |
| `citations` | Citations wrapper | Style the citations section. |
| `citationList` | Citations `<ol>` | Control citation list layout. |
| `citationItem` | Individual citation `<li>` | Style each citation. |
| `metadata` | Assistant metadata row | Control provider/model/token layout. |
| `metadataDetails` | Provider/model group | Style the left metadata group. |
| `tokens` | Token usage label | Style the right-aligned token count. |
| `placeholder` | Empty assistant response placeholder | Style “No response content.” |
| `thinking` | Global thinking indicator | Style the indicator shown while a stream is working. |
| `thinkingDots` | Thinking dot group | Control the dot group layout. |
| `thinkingDot` | Each animated thinking dot | Set dot size, color, or animation. |
| `thinkingLabel` | “Thinking…” label | Style the thinking text. |

The component also keeps its built-in semantic classes, which can be targeted directly when a dedicated override key is not needed. These include classes such as `.ai-interface__message--user`, `.ai-interface__message--assistant`, `.ai-interface__bubble--user`, `.ai-interface__bubble--assistant`, `.ai-interface__part--thinking`, `.ai-interface__result-status--success`, and `.ai-interface__result-status--error`.

## Theming with CSS variables

The component defines these custom properties on the root element:

| Variable | Default |
| --- | --- |
| `--ai-background` | `#f8fafc` |
| `--ai-surface` | `#ffffff` |
| `--ai-border` | `#e2e8f0` |
| `--ai-text` | `#1e293b` |
| `--ai-muted` | `#64748b` |
| `--ai-accent` | `#2563eb` |
| `--ai-accent-soft` | `#eff6ff` |
| `--ai-warning` | `#b45309` |
| `--ai-warning-soft` | `#fffbeb` |
| `--ai-error` | `#b91c1c` |
| `--ai-error-soft` | `#fef2f2` |

For many projects, overriding the variables is enough to apply the host application's visual language:

```css
.conversation-panel {
    --ai-background: #f5f7fb;
    --ai-surface: #ffffff;
    --ai-border: #d8dee9;
    --ai-text: #172033;
    --ai-muted: #667085;
    --ai-accent: #7c3aed;
    --ai-accent-soft: #f5f3ff;
    --ai-warning: #b45309;
    --ai-warning-soft: #fffbeb;
    --ai-error: #b42318;
    --ai-error-soft: #fef3f2;
}
```

The component also provides a dark palette automatically when the user's system has a dark color scheme. Add a host class and override the variables yourself if the consuming application uses an explicit theme switch instead.

## Tool and citation display

Tool calls and results are shown in expandable-style visual sections within the assistant bubble's normal flow. The component currently displays:

- Tool names.
- Serialized tool arguments.
- Tool result status (`Success` or `Failed`).
- Serialized tool output.
- Tool error details when available.
- Provider-tool payloads.
- Citation values.

Set `tools` to `false` to hide these sections without changing the underlying assistant message data.

## Token display

When token usage is present and `tokens` is enabled, the assistant metadata row displays the total of prompt and completion tokens. The full prompt/completion breakdown is available as the browser tooltip on the token label.

The component recognizes thought and cache token fields when deciding whether usage exists, but the visible total is currently prompt tokens plus completion tokens.

## Local development

From the package repository:

```bash
npm install
npm run typecheck
npm run build
```

The build produces:

- `dist/ai-interface-vue.js` — ESM build.
- `dist/ai-interface-vue.umd.cjs` — UMD/CommonJS-compatible build.
- `dist/ai-interface-vue.css` — extracted component stylesheet.

The package's `prepare` script runs `npm run build`, which is useful when the package is installed directly from a Git repository. `dist/` and `node_modules/` are excluded from version control by `.gitignore`.

## Package exports

```ts
import AiInterface, { AiInterface as NamedAiInterface } from 'ai-interface-vue';
import 'ai-interface-vue/style.css';
```

The default and named exports both refer to the same component. The stylesheet is exposed through the `ai-interface-vue/style.css` subpath.

## Accessibility and behavior notes

- The root element uses `aria-label="AI conversation"` and `aria-live="polite"`.
- The global thinking indicator uses `role="status"`.
- Avatar images have an empty alt attribute because the adjacent author label provides the meaningful name.
- Reduced-motion users do not receive the animated thinking-dot effect.
- User content is interpolated as text.
- Assistant Markdown is rendered with raw HTML disabled.

## License

This project is released under the [MIT License](LICENSE).
