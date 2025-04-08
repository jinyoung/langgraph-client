<template>
  <v-card class="chat-container">
    <v-card-text class="chat-messages" ref="messagesContainer">
      <template v-for="(message, index) in messages" :key="index">
        <v-card :class="['message-card', message.role]" :color="message.role === 'user' ? 'primary' : 'grey-lighten-4'" :elevation="1">
          <v-card-text :class="message.role === 'user' ? 'text-white' : ''">
            {{ message.content }}
          </v-card-text>
        </v-card>
      </template>
    </v-card-text>

    <v-divider></v-divider>

    <v-card-actions class="chat-input">
      <v-textarea
        v-model="newMessage"
        rows="3"
        hide-details
        density="comfortable"
        variant="outlined"
        placeholder="Type your message here..."
        @keydown.enter.prevent="onEnterPress"
      ></v-textarea>
      <v-btn
        color="primary"
        :disabled="isProcessing || !newMessage.trim()"
        :loading="isProcessing"
        @click="sendMessage"
      >
        Send
        <v-icon end icon="mdi-send"></v-icon>
      </v-btn>
    </v-card-actions>
  </v-card>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue'

const serverUrl = "http://localhost:2024"
const assistantId = "agent"

const messages = ref([])
const newMessage = ref('')
const currentThreadId = ref(null)
const isProcessing = ref(false)
const messagesContainer = ref(null)

const scrollToBottom = async () => {
  await nextTick()
  if (messagesContainer.value) {
    messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight
  }
}

const createThread = async () => {
  try {
    const threadRes = await fetch(`${serverUrl}/threads`, {
      method: "POST",
      mode: "cors",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({})
    })
    const threadData = await threadRes.json()
    currentThreadId.value = threadData.thread_id
    console.log("Created thread:", currentThreadId.value)
  } catch (error) {
    console.error("Error creating thread:", error)
    messages.value.push({
      role: 'assistant',
      content: "Error: Could not connect to LangGraph server. Make sure it's running on localhost:2024"
    })
  }
}

const sendMessage = async () => {
  if (!currentThreadId.value || !newMessage.value.trim()) return

  const userMessage = newMessage.value.trim()
  messages.value.push({ role: 'user', content: userMessage })
  newMessage.value = ''
  isProcessing.value = true
  
  await scrollToBottom()

  try {
    // Create a new run with the user's message
    const runRes = await fetch(`${serverUrl}/threads/${currentThreadId.value}/runs`, {
      method: "POST",
      mode: "cors",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        assistant_id: assistantId,
        input: {
          messages: [{ role: "user", content: userMessage }]
        }
      })
    })
    
    const runData = await runRes.json()
    const runId = runData.run_id
    console.log("Started run:", runId)

    // Add placeholder for assistant's message
    const assistantMessageIndex = messages.value.length
    messages.value.push({ role: 'assistant', content: '' })
    await scrollToBottom()

    // Stream the assistant's reply
    const streamRes = await fetch(`${serverUrl}/threads/${currentThreadId.value}/runs/${runId}/stream`, {
      method: "GET",
      mode: "cors"
    })

    const reader = streamRes.body.getReader()
    const decoder = new TextDecoder("utf-8")
    
    while (true) {
      const { value, done } = await reader.read()
      if (done) break
      
      if (value) {
        const chunk = decoder.decode(value, { stream: true })
        messages.value[assistantMessageIndex].content += chunk
        await scrollToBottom()
      }
    }

    console.log("Stream complete")
  } catch (error) {
    console.error("Error:", error)
    messages.value.push({
      role: 'assistant',
      content: "Error: Something went wrong while processing your message."
    })
  } finally {
    isProcessing.value = false
    await scrollToBottom()
  }
}

const onEnterPress = (e) => {
  if (!e.shiftKey) {
    sendMessage()
  }
}

onMounted(() => {
  createThread()
})
</script>

<style scoped>
.chat-container {
  height: calc(100vh - 150px);
  display: flex;
  flex-direction: column;
}

.chat-messages {
  flex-grow: 1;
  overflow-y: auto;
  padding: 16px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.chat-input {
  padding: 16px;
  display: flex;
  gap: 16px;
  background-color: white;
}

.message-card {
  max-width: 80%;
  width: fit-content;
}

.message-card.user {
  align-self: flex-end;
}

.message-card.assistant {
  align-self: flex-start;
}

.v-textarea {
  flex-grow: 1;
}
</style> 