<template>
  <div class="flex flex-col h-screen">
    <div
      class="flex flex-nowrap fixed w-full items-baseline top-0 px-6 py-4 bg-gray-100"
    >
      <div class="text-2xl font-bold">ChatGPT</div>
      <div class="ml-4 text-sm text-gray-500">
        基于 OpenAI 的 ChatGPT 自然语言模型人工智能对话
      </div>
      <div
        class="ml-auto px-3 py-2 text-sm cursor-pointer hover:bg-white rounded-md"
        @click="clickConfig()"
      >
        设置Key
      </div>
    </div>

    <div class="flex-1 mx-2 mt-20 mb-2" ref="chatListDom">
      <div
        class="group flex flex-col px-4 py-3 hover:bg-slate-100 rounded-lg"
        v-for="(item,i) of messageList.filter((v) => v.role !== 'system')" :key="i"
      >
        <div class="flex justify-between items-center mb-2">
          <!-- 设置的别名 -->
          <div class="font-bold">{{ roleAlias[item.role] }}：</div>
          <Copy class="invisible group-hover:visible" :content="item.content" />
        </div>
      <div>
        <!-- 将Markdown 格式的文本解析为HTML -->
          <div
            class="prose text-sm text-slate-600 leading-relaxed"
            v-if="item.content"
            v-html="md.render(item.content)"
          ></div>
          <Loding v-else />
        </div>
      </div>
    </div>

    <div class="sticky bottom-0 w-full p-6 pb-8 bg-gray-100">
      <div class="-mt-2 mb-2 text-sm text-gray-500" v-if="isConfig">
        请输入 API Key：
      </div>
      <div class="flex">
        <!-- 已经在对话不做发送 isTalking || sendOrSave()-->
        <input
          class="input"
          :type="isConfig ? 'password' : 'text'"
          :placeholder="isConfig ? 'sk-xxxxxxxxxx' : '请输入'"
          v-model="messageContent"
          @keydown.enter="isTalking || sendOrSave()"
        />
        <button class="btn" :disabled="isTalking" @click="sendOrSave()">
          {{ isConfig ? "保存" : "发送" }}
        </button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import type { ChatMessage } from "@/types";
import { ref, watch, nextTick, onMounted } from "vue";
import { chat } from "@/libs/gpt";
import cryptoJS from "crypto-js";
import Loding from "@/components/Loding.vue";
import Copy from "@/components/Copy.vue";
import { md } from "@/libs/markdown";
 
let apiKey = ""; // key
let isConfig = ref(true); // isConfig key 默认需要配置key
let isTalking = ref(false);// is a talk
let messageContent = ref("");// input value
const chatListDom = ref<HTMLDivElement>(); // contentList Dom
const decoder = new TextDecoder("utf-8"); // 解码器
const roleAlias = { user: "ME", assistant: "ChatGPT", system: "System" };
const messageList = ref<ChatMessage[]>([
  {
    role: "system",
    content: "你是 ChatGPT，OpenAI 训练的大型语言模型，尽可能简洁地回答。",
  },
  {
    role: "assistant",
    content: `你好，我是AI语言模型，我可以提供一些常用服务和信息，例如：

1. 翻译：我可以把中文翻译成英文，英文翻译成中文，还有其他一些语言翻译，比如法语、日语、西班牙语等。

2. 咨询服务：如果你有任何问题需要咨询，例如健康、法律、投资等方面，我可以尽可能为你提供帮助。

3. 闲聊：如果你感到寂寞或无聊，我们可以聊一些有趣的话题，以减轻你的压力。

请告诉我你需要哪方面的帮助，我会根据你的需求给你提供相应的信息和建议。`,
  },
]);

/**
 * 挂载之后检查key 是否存在 change input框状态
 * 
 */
onMounted(() => {
  if (getAPIKey()) {
    switchConfigStatus();
  }
});

const sendChatMessage = async (content: string = messageContent.value) => {
  try {
    isTalking.value = true;
    // messageList 去掉页面显示的tip
    if (messageList.value.length === 2) {
      messageList.value.pop();
    }
    // 为了联想 上下文 将所有msg push
    messageList.value.push({ role: "user", content });
    clearMessageContent();
    // chat 回答占位
    messageList.value.push({ role: "assistant", content: "" });
    // body.getReader() 是在可读流（ReadableStream）上调用的方法。
    //可读流通常用于表示从某个数据源（比如网络请求、文件读取等）获取的数据流。
    // Fetch API 返回的响应体就是一个可读流。
    const { body, status } = await chat(messageList.value, getAPIKey());
    if (body) {
      const reader = body.getReader();
      // 处理reader 流
      await readStream(reader, status);
    }
  } catch (error: any) {
    console.log(error,'error');
    
    appendLastMessageContent(error);
    
  } finally {
    // 会话结束
    isTalking.value = false;
  }
};


/**
 * 处理可读流 reader
 * 
 */
const readStream = async (
  reader: ReadableStreamDefaultReader<Uint8Array>,
  status: number
) => {
  // 部分line
  let partialLine = "";
  // while true  持续从流中读取数据
  while (true) {
    // eslint-disable-next-line no-await-in-loop
    // 通过 await 异步等待从流中读取的数据，其中 value 包含读取到的数据块，而 done 是一个布尔值，表示是否已经读取完毕。
    // if (done) break;：如果已经读取完毕，退出循环。
    const { value, done } = await reader.read();
    if (done) break;
   
    const decodedText = decoder.decode(value, { stream: true }); // text string 
    // 可能读取失败
    if (status !== 200) {
      const json = JSON.parse(decodedText); // start with "data: "
      const content = json.error.message ?? decodedText;
      appendLastMessageContent(content);
      return;
    }

    const chunk = partialLine + decodedText;
    const newLines = chunk.split(/\r?\n/);
    //  newLines 数组中取出最后一个元素，并将其赋值给变量 partialLine。
    partialLine = newLines.pop() ?? "";
    // ['data: {"id":"chatcmpl-8TpUD0CS6K5vqrpOY5DTsWNigSgR…0,"delta":{"content":"愿"},"finish_reason":null}]}', '', 'data: {"id":"chatcmpl-8TpUD0CS6K5vqrpOY5DTsWNigSgR…0,"delta":{"content":"你"},"finish_reason":null}]}', '', 'data: {"id":"chatcmpl-8TpUD0CS6K5vqrpOY5DTsWNigSgR…0,"delta":{"content":"幸"},"finish_reason":null}]}', '']
    console.log(newLines);
    
    for (const line of newLines) {
      if (line.length === 0) continue; // ignore empty message
      if (line.startsWith(":")) continue; // ignore sse comment message
      if (line === "data: [DONE]") return; //

      const json = JSON.parse(line.substring(6)); // start with "data: "
      const content =
        status === 200
          ? json.choices[0].delta.content ?? ""
          : json.error.message;
      appendLastMessageContent(content);
    }
  }
};

/**
 * 
 * 
 */
const appendLastMessageContent = (content: string) =>
{
   // 将之前展位 assitants 的content  替换为api 返回的内容
  
  return (messageList.value[messageList.value.length - 1].content += content);

}

/**
 * 保存key 或者发送chat msg 
 */

const sendOrSave = () => {
  // 输入的问题string length 为空 或者输入的key 
  if (!messageContent.value.length) return;
  // 配置key isConfig = true
  if (isConfig.value) {
    // 保存key 到 localstroage change isConfig
    if (saveAPIKey(messageContent.value.trim())) {
      switchConfigStatus();
    }
    clearMessageContent();
  } else {
    // 不是配置key 发送 chat msg
    sendChatMessage();
  }
};

/**
 * 设置key 
 */
const clickConfig = () => {
  if (!isConfig.value) {
    messageContent.value = getAPIKey();
  } else {
    clearMessageContent();
  }
  switchConfigStatus();
};

const getSecretKey = () => "lsztest";

/** 
 * 保存key 到 localstorage
 */ 
const saveAPIKey = (apiKey: string) => {
  // 校验key 
  if (apiKey.slice(0, 3) !== "sk-" || apiKey.length !== 51) {
    alert("API Key 错误，请检查后重新输入！");
    return false;
  }
  // 加密存到浏览器
  const aesAPIKey = cryptoJS.AES.encrypt(apiKey, getSecretKey()).toString();
  localStorage.setItem("apiKey", aesAPIKey);
  return true;
};

  const getAPIKey = () => {
    if (apiKey) return apiKey;
    // ?? 如果 localStorage.getItem("apiKey") 的结果为 null 或 undefined，则使用空字符串代替。
    const aesAPIKey = localStorage.getItem("apiKey") ?? "";
    // 解密
    apiKey = cryptoJS.AES.decrypt(aesAPIKey, getSecretKey()).toString(
      cryptoJS.enc.Utf8
    );
    return apiKey;
  };

const switchConfigStatus = () => (isConfig.value = !isConfig.value);

const clearMessageContent = () => (messageContent.value = "");

const scrollToBottom = () => {
  if (!chatListDom.value) return;
  scrollTo(0, chatListDom.value.scrollHeight);
};

/**
 * 检测watch messageList.value dom更新完成跳转到底部
 */
watch(messageList.value, () => nextTick(() => scrollToBottom()));
</script>

<style scoped>
pre {
  font-family: -apple-system, "Noto Sans", "Helvetica Neue", Helvetica,
    "Nimbus Sans L", Arial, "Liberation Sans", "PingFang SC", "Hiragino Sans GB",
    "Noto Sans CJK SC", "Source Han Sans SC", "Source Han Sans CN",
    "Microsoft YaHei", "Wenquanyi Micro Hei", "WenQuanYi Zen Hei", "ST Heiti",
    SimHei, "WenQuanYi Zen Hei Sharp", sans-serif;
}
</style>
