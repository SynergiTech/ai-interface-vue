<template>
    <section class="ai-interface" :class="props.classes.root" aria-label="AI conversation" aria-live="polite">
        <div v-if="messages.length === 0 && !thinking" class="ai-interface__empty" :class="props.classes.empty">
            <div :class="props.classes.emptyContent">
                <h2 class="ai-interface__empty-title" :class="props.classes.emptyTitle">No messages yet</h2>
                <p class="ai-interface__empty-copy" :class="props.classes.emptyCopy">
                    Your conversation will appear here
                </p>
            </div>
        </div>

        <div v-else class="ai-interface__messages" :class="props.classes.messages">
            <article
                v-for="message in messages"
                :key="message.uuid"
                class="ai-interface__message"
                :class="[
                    {
                        'ai-interface__message--user': message.type === 'user',
                        'ai-interface__message--assistant': message.type === 'assistant',
                        'ai-interface__message--error': message.status === 'error',
                        'ai-interface__message--avatars-hidden': !props.avatars,
                    },
                    props.classes.message,
                    message.type === 'user' ? props.classes.userMessage : props.classes.assistantMessage,
                    message.status === 'error' ? props.classes.errorMessage : null,
                    !props.avatars ? props.classes.avatarsHiddenMessage : null,
                ]"
            >
                <div
                    v-if="props.avatars"
                    class="ai-interface__avatar"
                    :class="[
                        props.classes.avatar,
                        message.type === 'user' ? props.classes.userAvatar : props.classes.assistantAvatar,
                    ]"
                    aria-hidden="true"
                >
                    <img
                        v-if="message.type === 'assistant' && props.assistantAvatarUrl"
                        :src="props.assistantAvatarUrl"
                        alt=""
                        class="ai-interface__avatar-image"
                        :class="props.classes.avatarImage"
                    />
                    <span v-else :class="props.classes.avatarInitials">
                        {{ message.type === 'user' ? props.userInitials : props.assistantInitials }}
                    </span>
                </div>

                <div class="ai-interface__message-body" :class="props.classes.messageBody">
                    <div class="ai-interface__message-heading" :class="props.classes.messageHeading">
                        <span class="ai-interface__message-author" :class="props.classes.messageAuthor">
                            {{ message.type === 'user' ? props.userName : props.assistantName }}
                        </span>
                        <span
                            v-if="message.status === 'streaming'"
                            class="ai-interface__message-status"
                            :class="[props.classes.messageStatus, props.classes.streamingStatus]"
                        >
                            Working
                        </span>
                        <span
                            v-else-if="message.status === 'error'"
                            class="ai-interface__message-status"
                            :class="[props.classes.messageStatus, props.classes.errorStatus]"
                        >
                            Error
                        </span>
                    </div>

                    <div
                        v-if="message.type === 'user'"
                        class="ai-interface__bubble ai-interface__bubble--user"
                        :class="[props.classes.bubble, props.classes.userBubble]"
                    >
                        {{ message.content }}
                    </div>

                    <div
                        v-else
                        class="ai-interface__bubble ai-interface__bubble--assistant"
                        :class="[props.classes.bubble, props.classes.assistantBubble]"
                    >
                        <div
                            v-if="message.content"
                            class="ai-interface__content"
                            :class="props.classes.content"
                            v-html="markdown.render(message.content)"
                        ></div>

                        <div
                            v-if="message.metadata.parts.length"
                            class="ai-interface__parts"
                            :class="props.classes.parts"
                        >
                            <template
                                v-for="(part, partIndex) in message.metadata.parts"
                                :key="`${message.uuid}-${partIndex}`"
                            >
                                <div
                                    v-if="part.type === 'thinking'"
                                    class="ai-interface__part ai-interface__part--thinking"
                                    :class="[props.classes.part, props.classes.thinkingPart]"
                                >
                                    <div class="ai-interface__part-heading" :class="props.classes.partHeading">
                                        <span :class="props.classes.partTitle">Thinking</span>
                                        <span class="ai-interface__part-label" :class="props.classes.partLabel">
                                            {{ part.content ? 'Complete' : 'In progress' }}
                                        </span>
                                    </div>
                                    <div
                                        v-if="part.content"
                                        class="ai-interface__part-content"
                                        :class="props.classes.partContent"
                                    >
                                        {{ part.content }}
                                    </div>
                                </div>

                                <div
                                    v-else-if="part.type === 'tool_call' && props.tools"
                                    class="ai-interface__part"
                                    :class="[props.classes.part, props.classes.toolCallPart]"
                                >
                                    <div class="ai-interface__part-heading" :class="props.classes.partHeading">
                                        <span :class="props.classes.partTitle">Tool calls</span>
                                        <span class="ai-interface__part-label" :class="props.classes.partLabel">
                                            {{ part.data.calls.length }}
                                        </span>
                                    </div>
                                    <div class="ai-interface__tool-list" :class="props.classes.toolList">
                                        <div
                                            v-for="call in part.data.calls"
                                            :key="call.id"
                                            class="ai-interface__tool-item"
                                            :class="props.classes.toolItem"
                                        >
                                            <strong class="ai-interface__tool-name" :class="props.classes.toolName">
                                                {{ call.name || 'Unnamed tool' }}
                                            </strong>
                                            <pre class="ai-interface__code" :class="props.classes.code">{{
                                                formatValue(call.arguments)
                                            }}</pre>
                                        </div>
                                    </div>
                                </div>

                                <div
                                    v-else-if="part.type === 'tool_result' && props.tools"
                                    class="ai-interface__part"
                                    :class="[props.classes.part, props.classes.toolResultPart]"
                                >
                                    <div class="ai-interface__part-heading" :class="props.classes.partHeading">
                                        <span :class="props.classes.partTitle">Tool results</span>
                                        <span class="ai-interface__part-label" :class="props.classes.partLabel">
                                            {{ part.data.results.length }}
                                        </span>
                                    </div>
                                    <div class="ai-interface__tool-list" :class="props.classes.toolList">
                                        <div
                                            v-for="result in part.data.results"
                                            :key="result.tool_call_id"
                                            class="ai-interface__tool-item"
                                            :class="props.classes.toolItem"
                                        >
                                            <div
                                                class="ai-interface__tool-result-heading"
                                                :class="props.classes.toolResultHeading"
                                            >
                                                <strong class="ai-interface__tool-name" :class="props.classes.toolName">
                                                    {{ result.name || 'Tool result' }}
                                                </strong>
                                                <span
                                                    class="ai-interface__result-status"
                                                    :class="[
                                                        result.success
                                                            ? 'ai-interface__result-status--success'
                                                            : 'ai-interface__result-status--error',
                                                        props.classes.resultStatus,
                                                        result.success
                                                            ? props.classes.resultSuccess
                                                            : props.classes.resultError,
                                                    ]"
                                                >
                                                    {{ result.success ? 'Success' : 'Failed' }}
                                                </span>
                                            </div>
                                            <pre class="ai-interface__code" :class="props.classes.code">{{
                                                result.result
                                            }}</pre>
                                            <p
                                                v-if="result.error"
                                                class="ai-interface__tool-error"
                                                :class="props.classes.toolError"
                                            >
                                                {{ formatValue(result.error) }}
                                            </p>
                                        </div>
                                    </div>
                                </div>

                                <div
                                    v-else-if="part.type === 'provider_tool' && props.tools"
                                    class="ai-interface__part"
                                    :class="[props.classes.part, props.classes.providerToolPart]"
                                >
                                    <div class="ai-interface__part-heading" :class="props.classes.partHeading">
                                        <span :class="props.classes.partTitle">Provider tool</span>
                                    </div>
                                    <pre class="ai-interface__code" :class="props.classes.code">{{
                                        formatValue(part.data)
                                    }}</pre>
                                </div>
                            </template>
                        </div>

                        <div
                            v-if="message.metadata.citations?.length"
                            class="ai-interface__citations"
                            :class="props.classes.citations"
                        >
                            <div class="ai-interface__part-heading" :class="props.classes.partHeading">
                                <span :class="props.classes.partTitle">Citations</span>
                                <span class="ai-interface__part-label" :class="props.classes.partLabel">
                                    {{ message.metadata.citations.length }}
                                </span>
                            </div>
                            <ol class="ai-interface__citation-list" :class="props.classes.citationList">
                                <li
                                    v-for="(citation, citationIndex) in message.metadata.citations"
                                    :key="citationIndex"
                                    :class="props.classes.citationItem"
                                >
                                    {{ formatValue(citation) }}
                                </li>
                            </ol>
                        </div>

                        <div
                            v-if="
                                message.type === 'assistant' &&
                                ((message.metadata.model && props.model) ||
                                    (message.metadata.provider && props.provider) ||
                                    (hasTokenUsage(message.metadata.usage) && props.tokens))
                            "
                            class="ai-interface__metadata"
                            :class="props.classes.metadata"
                        >
                            <div class="ai-interface__metadata-details" :class="props.classes.metadataDetails">
                                <span v-if="message.metadata.provider && props.provider">
                                    {{ message.metadata.provider }}
                                </span>
                                <span
                                    v-if="
                                        message.metadata.provider &&
                                        props.provider &&
                                        message.metadata.model &&
                                        props.model
                                    "
                                    aria-hidden="true"
                                >
                                    ·
                                </span>
                                <span v-if="message.metadata.model && props.model">{{ message.metadata.model }}</span>
                            </div>
                            <span
                                v-if="hasTokenUsage(message.metadata.usage) && props.tokens"
                                class="ai-interface__tokens"
                                :class="props.classes.tokens"
                                :title="tokenUsageLabel(message.metadata.usage)"
                            >
                                {{ totalTokens(message.metadata.usage).toLocaleString() }} tokens
                            </span>
                        </div>

                        <div
                            v-if="!message.content && !message.metadata.parts.length"
                            class="ai-interface__placeholder"
                            :class="props.classes.placeholder"
                        >
                            No response content.
                        </div>
                    </div>
                </div>
            </article>
        </div>

        <div v-if="thinking" class="ai-interface__thinking" :class="props.classes.thinking" role="status">
            <span class="ai-interface__thinking-dots" :class="props.classes.thinkingDots" aria-hidden="true">
                <i :class="props.classes.thinkingDot"></i>
                <i :class="props.classes.thinkingDot"></i>
                <i :class="props.classes.thinkingDot"></i>
            </span>
            <span :class="props.classes.thinkingLabel">Thinking…</span>
        </div>
    </section>
</template>
<script setup lang="ts">
import MarkdownIt from 'markdown-it';
import { ref, type PropType } from 'vue';

type ClassValue = string | string[] | Record<string, boolean> | null | undefined;

interface AiInterfaceClasses {
    root?: ClassValue;
    empty?: ClassValue;
    emptyContent?: ClassValue;
    emptyTitle?: ClassValue;
    emptyCopy?: ClassValue;
    messages?: ClassValue;
    message?: ClassValue;
    userMessage?: ClassValue;
    assistantMessage?: ClassValue;
    errorMessage?: ClassValue;
    avatarsHiddenMessage?: ClassValue;
    avatar?: ClassValue;
    userAvatar?: ClassValue;
    assistantAvatar?: ClassValue;
    avatarImage?: ClassValue;
    avatarInitials?: ClassValue;
    messageBody?: ClassValue;
    messageHeading?: ClassValue;
    messageAuthor?: ClassValue;
    messageStatus?: ClassValue;
    streamingStatus?: ClassValue;
    errorStatus?: ClassValue;
    bubble?: ClassValue;
    userBubble?: ClassValue;
    assistantBubble?: ClassValue;
    content?: ClassValue;
    parts?: ClassValue;
    part?: ClassValue;
    thinkingPart?: ClassValue;
    toolCallPart?: ClassValue;
    toolResultPart?: ClassValue;
    providerToolPart?: ClassValue;
    partHeading?: ClassValue;
    partTitle?: ClassValue;
    partLabel?: ClassValue;
    partContent?: ClassValue;
    toolList?: ClassValue;
    toolItem?: ClassValue;
    toolName?: ClassValue;
    toolResultHeading?: ClassValue;
    resultStatus?: ClassValue;
    resultSuccess?: ClassValue;
    resultError?: ClassValue;
    code?: ClassValue;
    toolError?: ClassValue;
    citations?: ClassValue;
    citationList?: ClassValue;
    citationItem?: ClassValue;
    metadata?: ClassValue;
    metadataDetails?: ClassValue;
    tokens?: ClassValue;
    placeholder?: ClassValue;
    thinking?: ClassValue;
    thinkingDots?: ClassValue;
    thinkingDot?: ClassValue;
    thinkingLabel?: ClassValue;
}

const props = defineProps({
    classes: {
        type: Object as PropType<AiInterfaceClasses>,
        default: () => ({}),
    },
    avatars: {
        type: Boolean,
        default: true,
    },
    assistantAvatarUrl: {
        type: String,
        default: null,
    },
    assistantName: {
        type: String,
        default: 'Assistant',
    },
    assistantInitials: {
        type: String,
        default: 'AI',
    },
    userName: {
        type: String,
        default: 'You',
    },
    userInitials: {
        type: String,
        default: 'You',
    },
    tools: {
        type: Boolean,
        default: true,
    },
    tokens: {
        type: Boolean,
        default: true,
    },
    provider: {
        type: Boolean,
        default: true,
    },
    model: {
        type: Boolean,
        default: true,
    },
});

type Message = UserMessage | AssistantMessage;
type MessageStatus = 'streaming' | 'completed' | 'error';

interface BaseMessage {
    uuid: string;
    content: string;
    status: MessageStatus;
    created_at: string; // ISO 8601
}

interface UserMessage extends BaseMessage {
    type: 'user';
    metadata: null;
}

interface AssistantMessage extends BaseMessage {
    type: 'assistant';
    metadata: AssistantMetadata;
}

interface AssistantMetadata {
    model: string;
    parts: AssistantPart[];
    usage: Usage;
    provider: string;
    citations: unknown[] | null;
    response_id: string;
    finish_reason: string;
}

type AssistantPart = TextPart | ThinkingPart | ToolCallPart | ToolResultPart | ProviderToolPart;

interface TextPart {
    type: 'text';
    content: string;
}

interface ThinkingPart {
    type: 'thinking';
    id: string;
    content: string;
    summary: Record<string, unknown> | null;
}

interface ToolCallPart {
    type: 'tool_call';
    data: {
        calls: ToolCall[];
    };
}

interface ToolCall {
    id: string;
    name: string;
    arguments: Record<string, unknown>;
}

interface ToolResultPart {
    type: 'tool_result';
    data: {
        results: ToolResult[];
    };
}

interface ToolResult {
    name: string;
    error: unknown | null;
    result: string;
    success: boolean;
    arguments: Record<string, unknown>;
    tool_call_id: string;
}

interface ProviderToolPart {
    type: 'provider_tool';
    data: Record<string, unknown>;
}

interface Usage {
    promptTokens: number;
    thoughtTokens: number | null;
    completionTokens: number;
    cacheReadInputTokens: number | null;
    cacheWriteInputTokens: number | null;
}

interface ActiveStream {
    id: string | null;
    messageId: string | null;
    temporaryMessageId: string | null;
    reasoningId: string | null;
    model: string;
    provider: string;
    responseId: string;
}

interface NormalizedStreamEvent {
    type: string;
    originalType: string;
    payload: Record<string, unknown>;
}

const thinking = ref<boolean>(false);
const messages = ref<Message[]>([]);

const activeTextPart = ref<TextPart | null>(null);
let activeTextMessageId: string | null = null;
let activeStream: ActiveStream | null = null;

const markdown = new MarkdownIt({
    html: false,
});

const createActiveStream = (): ActiveStream => ({
    id: null,
    messageId: null,
    temporaryMessageId: null,
    reasoningId: null,
    model: '',
    provider: '',
    responseId: '',
});

const ensureActiveStream = (): ActiveStream => {
    if (activeStream === null) {
        activeStream = createActiveStream();
    }

    return activeStream;
};

const processMessages = (newMessages: Message[]): void => {
    messages.value.push(...newMessages);
};

const clearMessages = (): void => {
    messages.value = [];
    thinking.value = false;
    activeTextPart.value = null;
    activeTextMessageId = null;
    activeStream = null;
};

const isRecord = (value: unknown): value is Record<string, unknown> => {
    return typeof value === 'object' && value !== null && !Array.isArray(value);
};

const parseJson = (value: unknown): unknown => {
    if (typeof value !== 'string') {
        return value;
    }

    try {
        return JSON.parse(value);
    } catch {
        return value;
    }
};

const readValue = (payload: Record<string, unknown>, ...keys: string[]): unknown => {
    for (const key of keys) {
        if (key in payload) {
            return payload[key];
        }
    }

    return undefined;
};

const readString = (payload: Record<string, unknown>, ...keys: string[]): string | undefined => {
    const value = readValue(payload, ...keys);

    return typeof value === 'string' && value !== '' ? value : undefined;
};

const readNumber = (payload: Record<string, unknown>, ...keys: string[]): number | undefined => {
    const value = readValue(payload, ...keys);

    if (typeof value === 'number' && Number.isFinite(value)) {
        return value;
    }

    if (typeof value === 'string' && value.trim() !== '') {
        const parsed = Number(value);

        return Number.isFinite(parsed) ? parsed : undefined;
    }

    return undefined;
};

const readBoolean = (payload: Record<string, unknown>, ...keys: string[]): boolean | undefined => {
    const value = readValue(payload, ...keys);

    return typeof value === 'boolean' ? value : undefined;
};

const readRecord = (payload: Record<string, unknown>, ...keys: string[]): Record<string, unknown> | undefined => {
    const value = readValue(payload, ...keys);

    return isRecord(value) ? value : undefined;
};

const readArray = (payload: Record<string, unknown>, ...keys: string[]): unknown[] | undefined => {
    const value = readValue(payload, ...keys);

    return Array.isArray(value) ? value : undefined;
};

const toIsoDate = (value: unknown): string => {
    let date: Date;

    if (typeof value === 'number' && Number.isFinite(value)) {
        date = new Date(value < 1_000_000_000_000 ? value * 1_000 : value);
    } else if (typeof value === 'string' && !Number.isNaN(Date.parse(value))) {
        date = new Date(value);
    } else {
        date = new Date();
    }

    return date.toISOString();
};

const randomId = (): string => {
    if (typeof crypto !== 'undefined' && typeof crypto.randomUUID === 'function') {
        return crypto.randomUUID();
    }

    return `assistant-${Date.now()}-${Math.random().toString(36).slice(2)}`;
};

const normalizeEventType = (eventType: string): string | null => {
    const normalized = eventType
        .replace(/[^a-zA-Z0-9]+/g, '_')
        .toLowerCase()
        .replace(/^_+|_+$/g, '');

    const aliases: Record<string, string> = {
        start: 'stream_start',
        stream_start: 'stream_start',
        stream_started: 'stream_start',
        stream_finished: 'stream_end',
        stream_completed: 'stream_end',
        text_start: 'text_start',
        text_started: 'text_start',
        text_delta: 'text_delta',
        text_end: 'text_complete',
        text_complete: 'text_complete',
        text_completed: 'text_complete',
        reasoning_start: 'thinking_start',
        reasoning_delta: 'thinking_delta',
        reasoning_end: 'thinking_complete',
        thinking_start: 'thinking_start',
        thinking_started: 'thinking_start',
        thinking_delta: 'thinking_delta',
        thinking_complete: 'thinking_complete',
        thinking_completed: 'thinking_complete',
        tool_call: 'tool_call',
        tool_input_available: 'tool_call',
        tool_result: 'tool_result',
        tool_output_available: 'tool_result',
        provider_tool_event: 'provider_tool_event',
        citation: 'citation',
        data_citation: 'citation',
        error: 'error',
        finish: 'stream_end',
        stream_end: 'stream_end',
    };

    return aliases[normalized] ?? null;
};

const normalizeStreamEvent = (stream: unknown, eventType?: string): NormalizedStreamEvent | null => {
    const parsedStream = parseJson(stream);

    if (!isRecord(parsedStream)) {
        return null;
    }

    const hasExplicitEvent = typeof eventType === 'string';
    const outerEventType = eventType ?? readString(parsedStream, 'event');
    const hasWebsocketEnvelope =
        (hasExplicitEvent && typeof parsedStream.type !== 'string') || typeof parsedStream.event === 'string';

    let payload = parsedStream;

    if (hasWebsocketEnvelope && 'data' in parsedStream) {
        const parsedData = parseJson(parsedStream.data);

        if (isRecord(parsedData)) {
            payload = parsedData;
        }
    }

    const originalType = readString(payload, 'type') ?? outerEventType ?? '';
    const normalizedType = normalizeEventType(originalType);

    if (normalizedType === null || originalType.startsWith('pusher:')) {
        return null;
    }

    return {
        type: normalizedType,
        originalType,
        payload,
    };
};

const createUsage = (): Usage => ({
    promptTokens: 0,
    thoughtTokens: null,
    completionTokens: 0,
    cacheReadInputTokens: null,
    cacheWriteInputTokens: null,
});

const createAssistantMetadata = (): AssistantMetadata => ({
    model: activeStream?.model ?? '',
    parts: [],
    usage: createUsage(),
    provider: activeStream?.provider ?? '',
    citations: null,
    response_id: activeStream?.responseId ?? '',
    finish_reason: '',
});

const normalizeToolCall = (value: unknown): ToolCall | null => {
    if (!isRecord(value)) {
        return null;
    }

    const argumentsValue = parseJson(readValue(value, 'arguments', 'input'));

    return {
        id: readString(value, 'id', 'tool_id', 'toolId', 'toolCallId') ?? randomId(),
        name: readString(value, 'name', 'tool_name', 'toolName') ?? '',
        arguments: isRecord(argumentsValue) ? argumentsValue : {},
    };
};

const normalizeToolResult = (value: unknown): ToolResult | null => {
    if (!isRecord(value)) {
        return null;
    }

    const toolCallId = readString(value, 'tool_call_id', 'toolCallId', 'tool_id', 'toolId', 'id') ?? randomId();
    const resultValue = readValue(value, 'result', 'output');
    const errorValue = readValue(value, 'error');
    const argumentsValue = parseJson(readValue(value, 'arguments', 'input'));

    return {
        name: readString(value, 'name', 'tool_name', 'toolName') ?? '',
        error: errorValue === undefined ? null : errorValue,
        result: typeof resultValue === 'string' ? resultValue : (JSON.stringify(resultValue) ?? ''),
        success:
            readBoolean(value, 'success') ??
            (errorValue === undefined || errorValue === null),
        arguments: isRecord(argumentsValue) ? argumentsValue : {},
        tool_call_id: toolCallId,
    };
};

const normalizeAssistantPart = (value: unknown): AssistantPart | null => {
    if (!isRecord(value) || typeof value.type !== 'string') {
        return null;
    }

    switch (value.type) {
        case 'text':
            return {
                type: 'text',
                content: typeof value.content === 'string' ? value.content : '',
            };

        case 'thinking':
            return {
                type: 'thinking',
                id: readString(value, 'id', 'reasoning_id', 'reasoningId') ?? randomId(),
                content: typeof value.content === 'string' ? value.content : '',
                summary: readRecord(value, 'summary') ?? null,
            };

        case 'tool_call': {
            const data = readRecord(value, 'data') ?? {};
            const calls = (readArray(data, 'calls') ?? [])
                .map((call) => normalizeToolCall(call))
                .filter((call): call is ToolCall => call !== null);

            return {
                type: 'tool_call',
                data: { calls },
            };
        }

        case 'tool_result': {
            const data = readRecord(value, 'data') ?? {};
            const results = (readArray(data, 'results') ?? [])
                .map((result) => normalizeToolResult(result))
                .filter((result): result is ToolResult => result !== null);

            return {
                type: 'tool_result',
                data: { results },
            };
        }

        case 'provider_tool':
            return {
                type: 'provider_tool',
                data: readRecord(value, 'data') ?? {},
            };

        default:
            return null;
    }
};

const normalizeAssistantMetadata = (value: unknown): AssistantMetadata => {
    const metadata = isRecord(value) ? value : {};
    const parts = (readArray(metadata, 'parts') ?? [])
        .map((part) => normalizeAssistantPart(part))
        .filter((part): part is AssistantPart => part !== null);
    const citationsValue = readValue(metadata, 'citations');

    return {
        model: readString(metadata, 'model') ?? '',
        parts,
        usage: normalizeUsage(readRecord(metadata, 'usage') ?? {}),
        provider: readString(metadata, 'provider') ?? '',
        citations: Array.isArray(citationsValue) ? citationsValue : null,
        response_id: readString(metadata, 'response_id', 'responseId') ?? '',
        finish_reason: readString(metadata, 'finish_reason', 'finishReason') ?? '',
    };
};

const normalizeMessage = (value: unknown): Message | null => {
    if (!isRecord(value) || (value.type !== 'user' && value.type !== 'assistant')) {
        return null;
    }

    const status = value.status;

    if (
        typeof value.uuid !== 'string' ||
        typeof value.content !== 'string' ||
        (status !== 'streaming' && status !== 'completed' && status !== 'error')
    ) {
        return null;
    }

    if (value.type === 'user') {
        return {
            uuid: value.uuid,
            type: 'user',
            content: value.content,
            status,
            created_at: toIsoDate(readValue(value, 'created_at', 'createdAt', 'timestamp')),
            metadata: null,
        };
    }

    return {
        uuid: value.uuid,
        type: 'assistant',
        content: value.content,
        status,
        created_at: toIsoDate(readValue(value, 'created_at', 'createdAt', 'timestamp')),
        metadata: normalizeAssistantMetadata(readValue(value, 'metadata')),
    };
};

const findAssistantMessage = (messageId: string | null): AssistantMessage | undefined => {
    if (messageId === null) {
        return undefined;
    }

    return messages.value.find(
        (message): message is AssistantMessage => message.type === 'assistant' && message.uuid === messageId
    );
};

const mergeAssistantMessages = (target: AssistantMessage, source: AssistantMessage): void => {
    if (source.content !== '') {
        target.content = `${source.content}${target.content}`;
    }

    target.metadata.parts.unshift(...source.metadata.parts);
};

const ensureAssistantMessage = (
    messageId: string,
    payload: Record<string, unknown>,
    isTemporary = false
): AssistantMessage => {
    const temporaryMessage = findAssistantMessage(activeStream?.temporaryMessageId ?? null);
    const existingMessage = findAssistantMessage(messageId);

    if (existingMessage !== undefined) {
        if (temporaryMessage !== undefined && temporaryMessage !== existingMessage && activeStream !== null) {
            mergeAssistantMessages(existingMessage, temporaryMessage);

            const temporaryIndex = messages.value.indexOf(temporaryMessage);

            if (temporaryIndex !== -1) {
                messages.value.splice(temporaryIndex, 1);
            }

            activeStream.messageId = messageId;
            activeStream.temporaryMessageId = null;
        }

        return existingMessage;
    }

    if (temporaryMessage !== undefined && activeStream !== null) {
        temporaryMessage.uuid = messageId;
        activeStream.messageId = isTemporary ? null : messageId;
        activeStream.temporaryMessageId = isTemporary ? messageId : null;

        return temporaryMessage;
    }

    const message: AssistantMessage = {
        uuid: messageId,
        type: 'assistant',
        content: '',
        status: 'streaming',
        created_at: toIsoDate(readValue(payload, 'timestamp', 'created_at')),
        metadata: createAssistantMetadata(),
    };

    messages.value.push(message);

    if (activeStream !== null) {
        if (isTemporary) {
            activeStream.temporaryMessageId = messageId;
        } else {
            activeStream.messageId = messageId;
        }
    }

    return message;
};

const activeMessageIdForEvent = (event: NormalizedStreamEvent): string | null => {
    const explicitMessageId = readString(event.payload, 'message_id', 'messageId', 'message_uuid', 'messageUuid', 'uuid');

    if (explicitMessageId !== undefined) {
        return explicitMessageId;
    }

    if (
        ['text_start', 'text_delta', 'text_complete'].includes(event.type) &&
        ['text-start', 'text-delta', 'text-end'].includes(event.originalType)
    ) {
        return readString(event.payload, 'id') ?? null;
    }

    return activeStream?.messageId ?? activeStream?.temporaryMessageId ?? null;
};

const ensureMessageForEvent = (event: NormalizedStreamEvent): AssistantMessage => {
    ensureActiveStream();

    const messageId = activeMessageIdForEvent(event) ?? randomId();
    const hasExplicitMessageId =
        readString(event.payload, 'message_id', 'messageId', 'message_uuid', 'messageUuid', 'uuid') !== undefined ||
        (['text_start', 'text_delta', 'text_complete'].includes(event.type) &&
            ['text-start', 'text-delta', 'text-end'].includes(event.originalType) &&
            readString(event.payload, 'id') !== undefined);
    const isTemporary =
        !hasExplicitMessageId && (activeStream?.messageId === null || activeStream?.messageId === undefined);

    return ensureAssistantMessage(messageId, event.payload, isTemporary);
};

const findToolCall = (message: AssistantMessage, toolCallId: string): ToolCall | undefined => {
    for (const part of message.metadata.parts) {
        if (part.type !== 'tool_call') {
            continue;
        }

        const toolCall = part.data.calls.find((call) => call.id === toolCallId);

        if (toolCall !== undefined) {
            return toolCall;
        }
    }

    return undefined;
};

const appendTextDelta = (message: AssistantMessage, delta: string): void => {
    if (activeTextPart.value === null || activeTextMessageId !== message.uuid) {
        const lastPart = message.metadata.parts.at(-1);

        if (lastPart?.type === 'text') {
            activeTextPart.value = lastPart;
        } else {
            activeTextPart.value = { type: 'text', content: '' };
            message.metadata.parts.push(activeTextPart.value);
        }

        activeTextMessageId = message.uuid;
    }

    const textPart = activeTextPart.value;

    if (textPart === null) {
        return;
    }

    textPart.content += delta;
    message.content += delta;
};

const appendThinkingDelta = (
    message: AssistantMessage,
    reasoningId: string,
    delta: string,
    summary: Record<string, unknown> | null = null
): void => {
    const existingPart = message.metadata.parts.find(
        (part): part is ThinkingPart => part.type === 'thinking' && part.id === reasoningId
    );
    const thinkingPart =
        existingPart ??
        (() => {
            const part: ThinkingPart = {
                type: 'thinking',
                id: reasoningId,
                content: '',
                summary: null,
            };

            message.metadata.parts.push(part);

            return part;
        })();

    thinkingPart.content += delta;

    if (summary !== null) {
        thinkingPart.summary = summary;
    }
};

const addToolCallEntry = (message: AssistantMessage, payload: Record<string, unknown>): void => {
    const toolCall: ToolCall = {
        id: readString(payload, 'id', 'tool_id', 'toolId', 'toolCallId') ?? randomId(),
        name: readString(payload, 'name', 'tool_name', 'toolName') ?? '',
        arguments: (() => {
            const value = parseJson(readValue(payload, 'arguments', 'input'));

            return isRecord(value) ? value : {};
        })(),
    };
    const existingCall = findToolCall(message, toolCall.id);

    if (existingCall !== undefined) {
        Object.assign(existingCall, toolCall);
        return;
    }

    const existingPart = [...message.metadata.parts]
        .reverse()
        .find((part): part is ToolCallPart => part.type === 'tool_call');

    if (existingPart !== undefined) {
        existingPart.data.calls.push(toolCall);
    } else {
        message.metadata.parts.push({
            type: 'tool_call',
            data: { calls: [toolCall] },
        });
    }
};

const addToolCall = (message: AssistantMessage, payload: Record<string, unknown>): void => {
    const calls = readArray(payload, 'calls');

    if (calls !== undefined) {
        calls.forEach((call) => {
            if (isRecord(call)) {
                addToolCallEntry(message, call);
            }
        });

        return;
    }

    addToolCallEntry(message, payload);
};

const addToolResultEntry = (message: AssistantMessage, payload: Record<string, unknown>): void => {
    const toolCallId =
        readString(payload, 'tool_call_id', 'toolCallId', 'tool_id', 'toolId', 'id') ?? randomId();
    const toolCall = findToolCall(message, toolCallId);
    const resultValue = readValue(payload, 'result', 'output');
    const result: ToolResult = {
        name: readString(payload, 'tool_name', 'toolName', 'name') ?? toolCall?.name ?? '',
        error: readValue(payload, 'error') === undefined ? null : readValue(payload, 'error'),
        result: typeof resultValue === 'string' ? resultValue : (JSON.stringify(resultValue) ?? ''),
        success:
            readBoolean(payload, 'success') ??
            (readValue(payload, 'error') === undefined || readValue(payload, 'error') === null),
        arguments: (() => {
            const value = parseJson(readValue(payload, 'arguments', 'input') ?? toolCall?.arguments);

            return isRecord(value) ? value : {};
        })(),
        tool_call_id: toolCallId,
    };
    const existingPart = [...message.metadata.parts]
        .reverse()
        .find((part): part is ToolResultPart => part.type === 'tool_result');
    const existingResult = existingPart?.data.results.find(
        (existingToolResult) => existingToolResult.tool_call_id === toolCallId
    );

    if (existingResult !== undefined) {
        Object.assign(existingResult, result);
        return;
    }

    if (existingPart !== undefined) {
        existingPart.data.results.push(result);
    } else {
        message.metadata.parts.push({
            type: 'tool_result',
            data: { results: [result] },
        });
    }
};

const addToolResult = (message: AssistantMessage, payload: Record<string, unknown>): void => {
    const results = readArray(payload, 'results');

    if (results !== undefined) {
        results.forEach((result) => {
            if (isRecord(result)) {
                addToolResultEntry(message, result);
            }
        });

        return;
    }

    addToolResultEntry(message, payload);
};

const normalizeUsage = (usage: Record<string, unknown>): Usage => ({
    promptTokens: readNumber(usage, 'prompt_tokens', 'promptTokens') ?? 0,
    thoughtTokens: readNumber(usage, 'thought_tokens', 'thoughtTokens') ?? null,
    completionTokens: readNumber(usage, 'completion_tokens', 'completionTokens') ?? 0,
    cacheReadInputTokens: readNumber(usage, 'cache_read_input_tokens', 'cacheReadInputTokens') ?? null,
    cacheWriteInputTokens: readNumber(usage, 'cache_write_input_tokens', 'cacheWriteInputTokens') ?? null,
});

const processCompleteMessage = (value: Record<string, unknown>): boolean => {
    const message = normalizeMessage(value);

    if (message === null) {
        return false;
    }

    const existingIndex = messages.value.findIndex((existingMessage) => existingMessage.uuid === message.uuid);

    if (existingIndex === -1) {
        messages.value.push(message);
    } else {
        messages.value[existingIndex] = message;
    }

    if (message.type === 'assistant' && message.status !== 'streaming') {
        setThinking(false);
    }

    return true;
};

const formatValue = (value: unknown): string => {
    if (typeof value === 'string') {
        return value;
    }

    try {
        return JSON.stringify(value, null, 2) ?? String(value);
    } catch {
        return String(value);
    }
};

const hasTokenUsage = (usage: Usage): boolean => {
    return [
        usage.promptTokens,
        usage.completionTokens,
        usage.thoughtTokens,
        usage.cacheReadInputTokens,
        usage.cacheWriteInputTokens,
    ].some((value) => typeof value === 'number' && value > 0);
};

const totalTokens = (usage: Usage): number => {
    return [usage.promptTokens, usage.completionTokens].reduce(
        (total, value) => total + (typeof value === 'number' ? value : 0),
        0
    );
};

const tokenUsageLabel = (usage: Usage): string => {
    return `${usage.promptTokens.toLocaleString()} prompt + ${usage.completionTokens.toLocaleString()} completion tokens`;
};

const processStream = (stream: unknown, eventType?: string): void => {
    const parsedStream = parseJson(stream);

    if (Array.isArray(parsedStream)) {
        parsedStream.forEach((item) => processStream(item, eventType));

        return;
    }

    if (isRecord(parsedStream) && processCompleteMessage(parsedStream)) {
        return;
    }

    const event = normalizeStreamEvent(parsedStream, eventType);

    if (event === null) {
        return;
    }

    switch (event.type) {
        case 'stream_start': {
            activeStream = {
                id: readString(event.payload, 'id') ?? null,
                messageId:
                    readString(event.payload, 'message_id', 'messageId', 'message_uuid', 'messageUuid') ?? null,
                temporaryMessageId: null,
                reasoningId: null,
                model: readString(event.payload, 'model') ?? '',
                provider: readString(event.payload, 'provider') ?? '',
                responseId: readString(event.payload, 'response_id', 'responseId') ?? '',
            };
            activeTextPart.value = null;
            activeTextMessageId = null;
            setThinking(true);
            break;
        }

        case 'text_start': {
            const message = ensureMessageForEvent(event);
            const textPart: TextPart = { type: 'text', content: '' };

            activeTextPart.value = textPart;
            activeTextMessageId = message.uuid;
            message.metadata.parts.push(textPart);
            setThinking(false);
            break;
        }

        case 'text_delta': {
            const message = ensureMessageForEvent(event);
            const delta = readString(event.payload, 'delta') ?? '';

            appendTextDelta(message, delta);
            setThinking(false);
            break;
        }

        case 'text_complete':
            activeTextPart.value = null;
            activeTextMessageId = null;
            break;

        case 'thinking_start': {
            const reasoningId =
                readString(event.payload, 'reasoning_id', 'reasoningId') ??
                (event.originalType === 'reasoning-start' ? readString(event.payload, 'id') : undefined) ??
                randomId();
            const message = ensureMessageForEvent(event);
            const stream = ensureActiveStream();

            stream.reasoningId = reasoningId;
            activeStream = stream;
            appendThinkingDelta(message, reasoningId, '', readRecord(event.payload, 'summary') ?? null);
            setThinking(true);
            break;
        }

        case 'thinking_delta': {
            const reasoningId =
                readString(event.payload, 'reasoning_id', 'reasoningId') ??
                (event.originalType === 'reasoning-delta' ? readString(event.payload, 'id') : undefined) ??
                activeStream?.reasoningId ??
                randomId();
            const message = ensureMessageForEvent(event);
            const stream = ensureActiveStream();

            stream.reasoningId = reasoningId;
            activeStream = stream;
            appendThinkingDelta(
                message,
                reasoningId,
                readString(event.payload, 'delta') ?? '',
                readRecord(event.payload, 'summary') ?? null
            );
            setThinking(true);
            break;
        }

        case 'thinking_complete':
            setThinking(false);
            break;

        case 'tool_call': {
            const message = ensureMessageForEvent(event);

            addToolCall(message, event.payload);
            activeTextPart.value = null;
            activeTextMessageId = null;
            setThinking(true);
            break;
        }

        case 'tool_result': {
            const message = ensureMessageForEvent(event);

            addToolResult(message, event.payload);
            setThinking(true);
            break;
        }

        case 'provider_tool_event': {
            const message = ensureMessageForEvent(event);

            message.metadata.parts.push({
                type: 'provider_tool',
                data: event.payload,
            });
            setThinking(true);
            break;
        }

        case 'citation': {
            const message = ensureMessageForEvent(event);
            const citation = readValue(event.payload, 'citation') ?? event.payload;

            message.metadata.citations = [...(message.metadata.citations ?? []), citation];
            break;
        }

        case 'error': {
            const message = ensureMessageForEvent(event);
            const errorMessage = readString(event.payload, 'message', 'errorText') ?? 'The AI stream failed.';

            message.status = 'error';
            message.metadata.finish_reason = 'error';

            if (message.content === '') {
                message.content = errorMessage;
            }

            setThinking(false);
            break;
        }

        case 'stream_end': {
            const messageId = activeStream?.messageId ?? activeStream?.temporaryMessageId;

            if (messageId !== null && messageId !== undefined) {
                const message = ensureAssistantMessage(messageId, event.payload);
                const metadata = readRecord(event.payload, 'messageMetadata', 'message_metadata') ?? event.payload;
                const usage = readRecord(metadata, 'usage');
                const citations = readArray(event.payload, 'citations');

                if (usage !== undefined) {
                    message.metadata.usage = normalizeUsage(usage);
                }

                message.metadata.finish_reason = readString(metadata, 'finish_reason', 'finishReason') ?? '';
                message.metadata.response_id =
                    readString(event.payload, 'response_id', 'responseId') ?? activeStream?.responseId ?? '';

                if (citations !== undefined) {
                    message.metadata.citations = citations;
                }

                message.status = 'completed';
            }

            activeTextPart.value = null;
            activeTextMessageId = null;
            activeStream = null;
            setThinking(false);
            break;
        }
    }
};

const setThinking = (thinkingValue: boolean): void => {
    thinking.value = thinkingValue;
};

defineExpose({
    messages,
    thinking,
    processMessages,
    processStream,
    setThinking,
    clearMessages,
});
</script>

<style scoped>
.ai-interface {
    --ai-background: #f8fafc;
    --ai-surface: #ffffff;
    --ai-border: #e2e8f0;
    --ai-text: #1e293b;
    --ai-muted: #64748b;
    --ai-accent: #2563eb;
    --ai-accent-soft: #eff6ff;
    --ai-warning: #b45309;
    --ai-warning-soft: #fffbeb;
    --ai-error: #b91c1c;
    --ai-error-soft: #fef2f2;
    box-sizing: border-box;
    display: flex;
    flex-direction: column;
    gap: 1rem;
    width: 100%;
    min-width: 0;
    padding: 1rem;
    color: var(--ai-text);
    background: var(--ai-background);
    font-family:
        Inter,
        ui-sans-serif,
        system-ui,
        -apple-system,
        BlinkMacSystemFont,
        'Segoe UI',
        sans-serif;
    line-height: 1.5;
}

.ai-interface *,
.ai-interface *::before,
.ai-interface *::after {
    box-sizing: border-box;
}

.ai-interface__empty {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 0.75rem;
    min-height: 12rem;
    padding: 2rem;
    border: 1px dashed var(--ai-border);
    border-radius: 0.75rem;
    text-align: left;
}

.ai-interface__avatar {
    display: inline-flex;
    flex: 0 0 auto;
    align-items: center;
    justify-content: center;
    width: 2.25rem;
    height: 2.25rem;
    border-radius: 50%;
    color: #ffffff;
    background: var(--ai-accent);
    font-size: 0.75rem;
    font-weight: 700;
    letter-spacing: 0.02em;
}

.ai-interface__avatar-image {
    display: block;
    width: 100%;
    height: 100%;
    border-radius: inherit;
    object-fit: cover;
}

.ai-interface__empty-title {
    margin: 0;
    color: var(--ai-text);
    font-size: 0.95rem;
    font-weight: 600;
    text-align: center;
}

.ai-interface__empty-copy {
    margin: 0.15rem 0 0;
    color: var(--ai-muted);
    font-size: 0.85rem;
}

.ai-interface__messages {
    display: flex;
    flex-direction: column;
    gap: 1.25rem;
}

.ai-interface__message {
    display: flex;
    align-items: flex-start;
    gap: 0.75rem;
    min-width: 0;
}

.ai-interface__message--user {
    flex-direction: row-reverse;
}

.ai-interface__message--user .ai-interface__avatar {
    background: #475569;
}

.ai-interface__message-body {
    display: flex;
    flex: 1 1 auto;
    flex-direction: column;
    gap: 0.4rem;
    min-width: 0;
    max-width: min(48rem, calc(100% - 3rem));
}

.ai-interface__message--user .ai-interface__message-body {
    align-items: flex-end;
}

.ai-interface__message--avatars-hidden .ai-interface__message-body {
    max-width: 100%;
}

.ai-interface__message-heading,
.ai-interface__part-heading,
.ai-interface__tool-result-heading {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 0.75rem;
}

.ai-interface__message-heading {
    width: 100%;
    padding: 0 0.25rem;
}

.ai-interface__message--user .ai-interface__message-heading {
    justify-content: flex-end;
}

.ai-interface__message-author {
    color: var(--ai-muted);
    font-size: 0.75rem;
    font-weight: 600;
}

.ai-interface__message-status,
.ai-interface__part-label {
    color: var(--ai-muted);
    font-size: 0.7rem;
    font-weight: 500;
}

.ai-interface__message--error .ai-interface__message-status {
    color: var(--ai-error);
}

.ai-interface__bubble {
    width: fit-content;
    max-width: 100%;
    padding: 0.75rem 0.9rem;
    border: 1px solid var(--ai-border);
    border-radius: 0.75rem;
    overflow-wrap: anywhere;
}

.ai-interface__bubble--user {
    color: #ffffff;
    background: var(--ai-accent);
    border-color: var(--ai-accent);
    border-top-right-radius: 0.25rem;
    white-space: pre-wrap;
}

.ai-interface__bubble--assistant {
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
    width: 100%;
    color: var(--ai-text);
    background: var(--ai-surface);
    border-top-left-radius: 0.25rem;
}

.ai-interface__content {
    overflow-wrap: anywhere;
}

.ai-interface__part-content {
    overflow-wrap: anywhere;
    white-space: pre-wrap;
}

.ai-interface__content {
    min-height: 1.5rem;
}

.ai-interface__content > :first-child {
    margin-top: 0;
}

.ai-interface__content > :last-child {
    margin-bottom: 0;
}

.ai-interface__content p {
    margin: 0;
}

.ai-interface__content p + p,
.ai-interface__content p + ul,
.ai-interface__content p + ol,
.ai-interface__content ul + p,
.ai-interface__content ol + p {
    margin-top: 0.75rem;
}

.ai-interface__content ul,
.ai-interface__content ol {
    margin: 0.75rem 0 0;
    padding-left: 1.35rem;
}

.ai-interface__content li + li {
    margin-top: 0.25rem;
}

.ai-interface__content h1,
.ai-interface__content h2,
.ai-interface__content h3,
.ai-interface__content h4,
.ai-interface__content h5,
.ai-interface__content h6 {
    margin: 1rem 0 0.5rem;
    color: var(--ai-text);
    font-weight: 700;
    line-height: 1.25;
}

.ai-interface__content h1 {
    font-size: 1.35rem;
}

.ai-interface__content h2 {
    font-size: 1.2rem;
}

.ai-interface__content h3 {
    font-size: 1.05rem;
}

.ai-interface__content h4,
.ai-interface__content h5,
.ai-interface__content h6 {
    font-size: 0.95rem;
}

.ai-interface__content blockquote {
    margin: 0.75rem 0;
    padding-left: 0.8rem;
    border-left: 0.2rem solid var(--ai-border);
    color: var(--ai-muted);
}

.ai-interface__content a {
    color: var(--ai-accent);
    text-decoration: underline;
    text-underline-offset: 0.15em;
}

.ai-interface__content code {
    padding: 0.1rem 0.3rem;
    border-radius: 0.25rem;
    color: var(--ai-text);
    background: #f1f5f9;
    font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, 'Liberation Mono', monospace;
    font-size: 0.85em;
}

.ai-interface__content pre {
    margin: 0.75rem 0 0;
    padding: 0.7rem;
    overflow-x: auto;
    border-radius: 0.4rem;
    color: #334155;
    background: #f1f5f9;
}

.ai-interface__content pre code {
    display: block;
    padding: 0;
    color: inherit;
    background: transparent;
    font-size: 0.8rem;
    white-space: pre;
}

.ai-interface__content hr {
    margin: 1rem 0;
    border: 0;
    border-top: 1px solid var(--ai-border);
}

.ai-interface__content table {
    width: 100%;
    margin: 0.75rem 0 0;
    border-collapse: collapse;
    font-size: 0.82rem;
}

.ai-interface__content th,
.ai-interface__content td {
    padding: 0.45rem 0.6rem;
    border: 1px solid var(--ai-border);
    text-align: left;
}

.ai-interface__content th {
    color: var(--ai-text);
    font-weight: 600;
}

.ai-interface__parts {
    display: flex;
    flex-direction: column;
    gap: 0.6rem;
}

.ai-interface__part,
.ai-interface__citations {
    padding: 0.7rem;
    border: 1px solid var(--ai-border);
    border-radius: 0.55rem;
    background: var(--ai-background);
}

.ai-interface__part--thinking {
    border-color: #fde68a;
    background: var(--ai-warning-soft);
}

.ai-interface__part-heading {
    color: var(--ai-muted);
    font-size: 0.75rem;
    font-weight: 600;
}

.ai-interface__part--thinking .ai-interface__part-heading {
    color: var(--ai-warning);
}

.ai-interface__part-content {
    margin-top: 0.55rem;
    color: var(--ai-muted);
    font-size: 0.82rem;
}

.ai-interface__tool-list {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    margin-top: 0.55rem;
}

.ai-interface__tool-item {
    min-width: 0;
    padding: 0.6rem;
    border: 1px solid var(--ai-border);
    border-radius: 0.45rem;
    background: var(--ai-surface);
}

.ai-interface__tool-name {
    display: block;
    overflow-wrap: anywhere;
    color: var(--ai-text);
    font-size: 0.8rem;
}

.ai-interface__tool-result-heading {
    align-items: flex-start;
}

.ai-interface__result-status {
    flex: 0 0 auto;
    font-size: 0.7rem;
    font-weight: 600;
}

.ai-interface__result-status--success {
    color: #15803d;
}

.ai-interface__result-status--error,
.ai-interface__tool-error {
    color: var(--ai-error);
}

.ai-interface__code {
    max-width: 100%;
    margin: 0.45rem 0 0;
    padding: 0.6rem;
    overflow-x: auto;
    border-radius: 0.4rem;
    color: #334155;
    background: #f1f5f9;
    font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, 'Liberation Mono', monospace;
    font-size: 0.72rem;
    line-height: 1.55;
    white-space: pre-wrap;
    overflow-wrap: anywhere;
}

.ai-interface__tool-error {
    margin: 0.45rem 0 0;
    font-size: 0.75rem;
    white-space: pre-wrap;
}

.ai-interface__citation-list {
    display: flex;
    flex-direction: column;
    gap: 0.4rem;
    margin: 0.55rem 0 0;
    padding-left: 1.25rem;
    color: var(--ai-muted);
    font-size: 0.78rem;
}

.ai-interface__citation-list li {
    padding-left: 0.2rem;
    overflow-wrap: anywhere;
    white-space: pre-wrap;
}

.ai-interface__metadata {
    display: flex;
    align-items: center;
    color: var(--ai-muted);
    font-size: 0.68rem;
}

.ai-interface__metadata-details {
    display: inline-flex;
    align-items: center;
    gap: 0.35rem;
    min-width: 0;
}

.ai-interface__tokens {
    flex: 0 0 auto;
    margin-left: auto;
    font-variant-numeric: tabular-nums;
    white-space: nowrap;
}

.ai-interface__placeholder {
    color: var(--ai-muted);
    font-size: 0.85rem;
    font-style: italic;
}

.ai-interface__thinking {
    display: inline-flex;
    align-items: center;
    align-self: flex-start;
    gap: 0.5rem;
    padding: 0.5rem 0.75rem;
    color: var(--ai-muted);
    font-size: 0.8rem;
}

.ai-interface__thinking-dots {
    display: inline-flex;
    align-items: center;
    gap: 0.2rem;
}

.ai-interface__thinking-dots i {
    width: 0.3rem;
    height: 0.3rem;
    border-radius: 50%;
    background: var(--ai-accent);
    animation: ai-interface-pulse 1.2s infinite ease-in-out;
}

.ai-interface__thinking-dots i:nth-child(2) {
    animation-delay: 0.15s;
}

.ai-interface__thinking-dots i:nth-child(3) {
    animation-delay: 0.3s;
}

@keyframes ai-interface-pulse {
    0%,
    60%,
    100% {
        opacity: 0.35;
        transform: translateY(0);
    }

    30% {
        opacity: 1;
        transform: translateY(-0.15rem);
    }
}

@media (max-width: 40rem) {
    .ai-interface {
        padding: 0.75rem;
    }

    .ai-interface__message-body {
        max-width: calc(100% - 3rem);
    }
}

@media (prefers-color-scheme: dark) {
    .ai-interface {
        --ai-background: #0f172a;
        --ai-surface: #1e293b;
        --ai-border: #334155;
        --ai-text: #e2e8f0;
        --ai-muted: #94a3b8;
        --ai-accent-soft: #172554;
        --ai-warning: #fbbf24;
        --ai-warning-soft: #451a03;
        --ai-error: #fca5a5;
        --ai-error-soft: #450a0a;
    }

    .ai-interface__code {
        color: #cbd5e1;
        background: #0f172a;
    }
}

@media (prefers-reduced-motion: reduce) {
    .ai-interface__thinking-dots i {
        animation: none;
    }
}
</style>
