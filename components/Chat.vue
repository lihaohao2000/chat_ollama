<script setup lang="ts">
import { useMutationObserver, useLocalStorage } from '@vueuse/core'
import MarkdownIt from "markdown-it";
import MarkdownItAbbr from "markdown-it-abbr";
import MarkdownItAnchor from "markdown-it-anchor";
import MarkdownItFootnote from "markdown-it-footnote";
import MarkdownItHighlightjs from "markdown-it-highlightjs";
import MarkdownItSub from "markdown-it-sub";
import MarkdownItSup from "markdown-it-sup";
import MarkdownItTasklists from "markdown-it-task-lists";
import MarkdownItTOC from "markdown-it-toc-done-right";
import {
  fetchHeadersOllama,
  fetchHeadersThirdApi,
} from '@/utils/settings';
import { type ChatBoxFormData } from '@/components/ChatInputBox.vue'

const props = defineProps({
  knowledgebase: Object
});

let chatBef = true

const markdown = new MarkdownIt()
  .use(MarkdownItAbbr)
  .use(MarkdownItAnchor)
  .use(MarkdownItFootnote)
  .use(MarkdownItHighlightjs)
  .use(MarkdownItSub)
  .use(MarkdownItSup)
  .use(MarkdownItTasklists)
  .use(MarkdownItTOC);

const chatInputBoxRef = shallowRef()

const model = useLocalStorage('model', "qwen:latest")

const messages = ref<Array<{ role: 'system' | 'assistant' | 'user', content: string, type?: 'loading' | 'canceled' }>>([]);
const sending = ref(false);
let abortHandler: (() => void) | null = null;

const visibleMessages = computed(() => {
  return messages.value.filter((message) => message.role !== 'system');
});

watch(model, async (newModel) => {
  messages.value = [];
})

const fetchStream = async (url: string, options: RequestInit) => {
  const response = await fetch(url, options);

  if (response.body) {
    messages.value = messages.value.filter((message) => message.type !== 'loading');
    const reader = response.body.getReader();
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      const chunk = new TextDecoder().decode(value);
      chunk.split("\n\n").forEach(async (line) => {
        if (line) {
          console.log('line: ', line);
          const chatMessage = JSON.parse(line);
          const content = chatMessage?.message?.content;
          if (content) {
            if (messages.value.length > 0 && messages.value[messages.value.length - 1].role === 'assistant') {
              messages.value[messages.value.length - 1].content += content;
            } else {
              messages.value.push({ role: 'assistant', content });
            }
          }
        }
      });
    }
  } else {
    console.log("The browser doesn't support streaming responses.");
  }
}

const onSend = async (data: ChatBoxFormData) => {
  chatBef = false
  const input = data.content.trim()
  if (sending.value || !input) {
    return;
  }

  sending.value = true;
  chatInputBoxRef.value?.reset()

  messages.value.push({
    role: "user",
    content: input
  });

  messages.value.push({
    role: "assistant",
    content: "",
    type: 'loading'
  })

  const body = JSON.stringify({
    // knowledgebaseId: props.knowledgebase?.id,
    knowledgebaseId: 1,
    model: model.value,
    messages: [...messages.value.filter(m => m.type !== 'loading')],
    stream: true,
  })
  const controller = new AbortController();
  abortHandler = () => controller.abort();
  await fetchStream('/api/models/chat', {
    method: 'POST',
    body: body,
    headers: {
      ...fetchHeadersOllama.value,
      ...fetchHeadersThirdApi.value,
      'Content-Type': 'application/json',
    },
    signal: controller.signal,
  });

  sending.value = false;
}

const messageListEl = shallowRef<HTMLElement>();
useMutationObserver(messageListEl, () => {
  messageListEl.value?.scrollTo({
    top: messageListEl.value.scrollHeight,
    behavior: 'smooth'
  });
}, { childList: true, subtree: true });

function onAbortChat() {
  abortHandler?.()
  if (messages.value.length > 0) {
    const lastOne = messages.value[messages.value.length - 1];
    if (lastOne.type === 'loading') {
      messages.value.pop();
    } else if (lastOne.role === 'assistant') {
      lastOne.type = 'canceled';
    }
  }
  sending.value = false
}
</script>

<template>
  <div class="flex flex-col flex-1 w-full items-center">
    <div class="flex flex-col flex-1 box-border dark:text-gray-300 -mx-4 w-full max-w-full sm:max-w-2xl">
      <div ref="messageListEl" class="relative flex-1 overflow-auto px-4">

        <div class="flex h-full flex-col items-center justify-center text-token-text-primary" v-if="chatBef">
          <div class="relative">
            <img src="..\public\CUlogo_mid.png" alt="Logo" class="mb-3 h-12 w-12" />
          </div>
          <h2 class="mb-5 text-2xl font-bold">您需要什么帮助？</h2>
        </div>

        <div v-for="(message, index) in visibleMessages" :key="index" :class="{ 'text-right': message.role === 'user' }">
          <div class="text-gray-500 dark:text-gray-400">{{ message.role }}</div>
          <div class="mb-4 leading-6" :class="{ 'text-gray-400 dark:text-gray-500': message.type === 'canceled' }">
            <div
              :class="`inline-flex ${message.role == 'assistant' ? 'bg-gray-50 dark:bg-gray-800' : 'bg-primary-50 dark:bg-primary-400/60'} border border-primary/20 rounded-lg px-3 py-2`">
              <div v-if="message.type === 'loading'"
                class="text-xl text-primary animate-spin i-heroicons-arrow-path-solid">
              </div>
              <div v-else v-html="markdown.render(message.content)" />
            </div>
          </div>
        </div>
      </div>
      <div class="shrink-0 pt-4 px-4">
        <ChatInputBox ref="chatInputBoxRef" :disabled="!model" :loading="sending" @submit="onSend" @stop="onAbortChat" />
      </div>
    </div>
  </div>
</template>

<style>
code {
  color: rgb(31, 64, 226);
  white-space: pre-wrap;
  margin: 0 0.4em;
}

.dark code {
  color: rgb(125, 179, 250);
}
</style>
