<template>
  <div class="flex flex-col h-screen">
    <div
      class="flex flex-nowrap fixed w-full items-baseline top-0 px-6 py-4 bg-gray-100"
    >
      <div class="text-2xl font-bold">ChatBot</div>
      <div class="ml-4 text-sm text-gray-500">for the aged🧓</div>
      <div
        class="ml-auto px-3 py-2 text-sm cursor-pointer hover:bg-white rounded-md"
        @click="clickConfig()"
      >
        Setting
      </div>
    </div>

    <div class="flex-1 mx-2 mt-20 mb-2" ref="chatListDom">
      <div
        class="group flex flex-col px-4 py-3 hover:bg-slate-100 rounded-lg"
        v-for="item of messageList.filter((v) => v.role !== 'system')"
      >
        <div class="flex justify-between items-center mb-2">
          <div class="font-bold">{{ roleAlias[item.role] }}：</div>
          <div class="flex items-center">
            <Copy
              class="invisible group-hover:visible"
              :content="item.content"
            />
            <button @click="speak(item.content)" class="speak">
              <img src="/img/Speaker.png" alt="Play" />
            </button>
          </div>
        </div>
        <div>
          <div
            class="prose text-sm text-slate-600 leading-relaxed"
            v-if="item.content"
            v-html="md.render(item.content)"
          ></div>
          <Loding v-else />
        </div>
      </div>
    </div>
    <div v-if="isRecording" class="recording-overlay">
      <div>
        <img src="/img/Microphone2.png" alt="Microphone" class="record" />
        Recording: {{ recordingTime }}
      </div>
      <button @click="stopRecordingAndTranscribe">Save</button>
      <button @click="cancelRecording">Cancel</button>
    </div>
    <div class="sticky bottom-0 w-full p-6 pb-8 bg-gray-100">
      <div class="-mt-2 mb-2 text-sm text-gray-500" v-if="isConfig">
        Please enter the API Key：
      </div>
      <div class="flex">
        <input
          class="input"
          :type="isConfig ? 'password' : 'text'"
          :placeholder="
            isConfig ? 'sk-xxxxxxxxxx' : 'Please enter your message'
          "
          v-model="messageContent"
          @keydown.enter="isTalking || sendOrSave()"
        />
        <button
          id="start-record-btn"
          class="record-btn"
          @click="startRecording"
        >
          <img src="/img/microphone.png" alt="Microphone" />
        </button>

        <button class="btn" :disabled="isTalking" @click="sendOrSave()">
          {{ isConfig ? "Save" : "Send" }}
        </button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import type { ChatMessage } from "@/types";
import { ref, watch, nextTick, onMounted, onUnmounted } from "vue";
import { chat } from "@/libs/gpt";
import cryptoJS from "crypto-js";
import Loding from "@/components/Loding.vue";
import Copy from "@/components/Copy.vue";
import { md } from "@/libs/markdown";

let apiKey = "";
let isConfig = ref(true);
let isTalking = ref(false);
let messageContent = ref("");
const chatListDom = ref<HTMLDivElement>();
const decoder = new TextDecoder("utf-8");
const roleAlias = { user: "ME", assistant: "ChatBot", system: "System" };
const messageList = ref<ChatMessage[]>([
  {
    role: "system",
    content:
      "You are ChatBot, a large language model trained by OpenAI, to answer as concisely as possible.",
  },
  {
    role: "assistant",
    content: `Hello, I am ChatBot, I can provide some common services and information. 
    If you're feeling lonely or bored, you can chat with me.`,
  },
]);

const reminders = [
  "Remember to drink water",
  "Let's chat",
  "It's time to take a break",
  "Remember to take medicine",
];

let reminderIndex = 0;
let reminderInterval = null;

const setReminder = () => {
  const reminderMessage = reminders[reminderIndex];
  appendLastMessageContent(reminderMessage);
  // optional
  //speak(reminderMessage);
  reminderIndex = (reminderIndex + 1) % reminders.length;
};

onMounted(() => {
  if (getAPIKey()) {
    switchConfigStatus();
  }
  reminderInterval = setInterval(setReminder, 7200000); // 3600000 毫秒 = 1 小时
});

onUnmounted(() => {
  clearInterval(reminderInterval);
});

const sendChatMessage = async (content: string = messageContent.value) => {
  try {
    isTalking.value = true;
    if (messageList.value.length === 2) {
      messageList.value.pop();
    }
    messageList.value.push({ role: "user", content });
    clearMessageContent();

    const { body, status } = await chat(messageList.value, getAPIKey());
    if (body) {
      const reader = body.getReader();
      await readStream(reader, status);
    }
  } catch (error: any) {
    appendLastMessageContent("An error occurred: " + error, "assistant");
  } finally {
    isTalking.value = false;
  }
};

const readStream = async (
  reader: ReadableStreamDefaultReader<Uint8Array>,
  status: number
) => {
  let partialLine = "";
  let accumulatedContent = ""; 

  while (true) {
    const { value, done } = await reader.read();
    if (done) break;

    const decodedText = decoder.decode(value, { stream: true });

    if (status !== 200) {
      return;
    }

    const chunk = partialLine + decodedText;
    const newLines = chunk.split(/\r?\n/);

    partialLine = newLines.pop() ?? "";

    for (const line of newLines) {
      if (line.length === 0 || line.startsWith(":")) continue;
      if (line === "data: [DONE]") {
        if (accumulatedContent.trim()) {
          appendLastMessageContent(accumulatedContent, "assistant");
          speak(accumulatedContent);
        }
        accumulatedContent = ""; 
        continue;
      }
      const json = JSON.parse(line.substring(6)); // start with "data: "
      const content = json.choices[0].delta.content ?? "";
      accumulatedContent += content; 
    }
  }
};

// const appendLastMessageContent = (content: string) =>
//   (messageList.value[messageList.value.length - 1].content += content);
//   console.log(messageList.value)
const appendLastMessageContent = (
  content: string,
  role: string = "assistant"
) => {
  if (content.trim()) {
    messageList.value.push({ role, content });
    console.log(messageList.value);
  }
};

const sendOrSave = () => {
  if (!messageContent.value.length) return;
  if (isConfig.value) {
    if (saveAPIKey(messageContent.value.trim())) {
      switchConfigStatus();
    }
    clearMessageContent();
  } else {
    sendChatMessage();
  }
};

const clickConfig = () => {
  if (!isConfig.value) {
    messageContent.value = getAPIKey();
  } else {
    clearMessageContent();
  }
  switchConfigStatus();
};

const getSecretKey = () => "lianginx";

const saveAPIKey = (apiKey: string) => {
  if (apiKey.slice(0, 3) !== "sk-" || apiKey.length !== 51) {
    alert("API Key error, please enter it again！");
    return false;
  }
  const aesAPIKey = cryptoJS.AES.encrypt(apiKey, getSecretKey()).toString();
  localStorage.setItem("apiKey", aesAPIKey);
  return true;
};

const getAPIKey = () => {
  /*
  if (apiKey) return apiKey;
  const aesAPIKey = localStorage.getItem("apiKey") ?? "";
  apiKey = cryptoJS.AES.decrypt(aesAPIKey, getSecretKey()).toString(
    cryptoJS.enc.Utf8
  );
  */
  return "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";
};

// const getAPIKey = () => {
//   if (apiKey) return apiKey;
//   const aesAPIKey = localStorage.getItem("apiKey") ?? "";
//   apiKey = cryptoJS.AES.decrypt(aesAPIKey, getSecretKey()).toString(
//     cryptoJS.enc.Utf8
//   );
//   return apiKey;
// };

const switchConfigStatus = () => (isConfig.value = !isConfig.value);

const clearMessageContent = () => (messageContent.value = "");

const scrollToBottom = () => {
  if (!chatListDom.value) return;
  scrollTo(0, chatListDom.value.scrollHeight);
};

const isRecording = ref(false);
const recordingTime = ref(0);
let recordingInterval = null;
let speechRecognition = null;

const startRecording = () => {
  const SpeechRecognition =
    window.SpeechRecognition || window.webkitSpeechRecognition;
  if (!SpeechRecognition) {
    alert("Sorry, your browser does not support speech recognition.");
    return;
  }

  speechRecognition = new SpeechRecognition();
  speechRecognition.continuous = true;
  speechRecognition.lang = "en-US"; // Set language to English
  speechRecognition.interimResults = false;

  speechRecognition.onstart = () => {
    isRecording.value = true;
    recordingTime.value = 0;
    recordingInterval = setInterval(() => {
      recordingTime.value++;
    }, 1000);
  };

  speechRecognition.onresult = (event) => {
    const last = event.results.length - 1;
    messageContent.value = event.results[last][0].transcript;
    console.log(event.results[last][0].transcript);
  };

  speechRecognition.onerror = (event) => {
    console.error("Speech recognition errors:", event.error);
  };

  speechRecognition.onend = () => {
    if (isRecording.value) {
      speechRecognition.start();
    }
  };

  speechRecognition.start();
};

const stopRecordingAndTranscribe = () => {
  if (speechRecognition) {
    speechRecognition.stop();
  }
  clearInterval(recordingInterval);
  isRecording.value = false;
};

const cancelRecording = () => {
  if (speechRecognition) {
    speechRecognition.stop();
  }
  clearInterval(recordingInterval);
  isRecording.value = false;
  recordingTime.value = 0;
  messageContent.value = "";
};

const speak = (text) => {
  if ("speechSynthesis" in window) {
    const utterance = new SpeechSynthesisUtterance(text);
    console.log(text);
    utterance.lang = "en-US"; // 设置语言为英语

    utterance.onstart = () => console.log("Speech synthesis started");
    utterance.onend = () => console.log("Speech synthesis finished");
    utterance.onerror = (event) =>
      console.error("Speech synthesis error:", event);

    window.speechSynthesis.speak(utterance);
  } else {
    console.error("Your browser does not support text-to-speech.");
    alert("Your browser does not support text-to-speech.");
  }
};

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
.speak {
  width: 20px;
  height: 20px;
  margin-left: 20px;
  padding: 0;
}
.record {
  width: 50px;
  display: block;
  margin: 0 auto;
  max-width: 100%;
  height: auto;
}

.record-btn {
  height: 40px;
  width: 30px;
  margin-right: 10px;
}

.recording-overlay {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  padding: 20px;
  border: 2px solid #000;
  background-color: #fff;
  text-align: center;
  z-index: 1000;
  border-radius: 10px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  text-align: center;
}

.recording-overlay button {
  margin: 10px;
  padding: 10px 20px;
  border: none;
  border-radius: 5px;
  background-color: #007bff;
  color: white;
  cursor: pointer;
  transition: background-color 0.2s;
}

button:hover {
  background-color: #0056b3;
}

button:active {
  background-color: #00408d;
}
.speak:hover {
  transform: scale(1.1);
  background-color: inherit;
}
.speak:active {
  transform: scale(1.2);
  background-color: inherit;
}
.prose {
  max-width: fit-content;
  margin-right: 80px;
}
</style>
