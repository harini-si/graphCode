<template>
  <div class="chatbox">
    <h1 class="chat-heading">Chat</h1>
    <div>
      <div class="Testing div">
        <div class="chatline">
          <input
            type="text"
            ref="chatbot_input"
            id="chatbot_input"
            v-model="message"
            @keyup.enter="displayMessage"
            class="input-text"
            placeholder="Type a message..."
          />
          <button class="go-button" @click="displayMessage">Go</button>
        </div>
        <div>
          <ul class="chat-box-list">
            <li
              v-for="(message, index) in messages"
              :key="index"
              :class="{
                'user-message': message.role === 'user',
                'bot-message': message.role === 'chatbot'
              }"
            >
              <span>{{ message.content }}</span>
            </li>
          </ul>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { proxyRefs, ref } from 'vue'
import { Configuration, OpenAIApi } from 'openai'

export const latestBotResponse = ref('')

export default {
  name: 'Chat',
  setup() {
    const chatbot_output = ref(null)
    const message = ref('')
    const messages = ref([])

    const get_openai_response = async message => {
      const configuration = new Configuration({
        apiKey: process.env.VUE_APP_OPENAI_API_KEY
      })
      const openai = new OpenAIApi(configuration)
      const user_input = message

      try {
        const completion = await openai.createChatCompletion({
          model: 'gpt-3.5-turbo',
          messages: [
            {
              role: 'user',
              content: user_input
            }
          ]
        })

        const completion_text = completion.data.choices[0].message.content
        const response = { role: 'chatbot', content: completion_text }

        return response
      } catch (error) {
        if (error.response) {
          console.error(error.response.status)
          console.error(error.response.data)
        } else {
          console.error(error.message)
        }
      }
      return null
    }

    const displayMessage = async () => {
      const user_message = message.value
      const bot_response = await get_openai_response(user_message)

      // When the function is called, emit the latest bot response

      latestBotResponse.value = bot_response.content

      messages.value.push({ role: 'user', content: user_message })
      messages.value.push(bot_response)

      chatbot_output.value = bot_response.content
      message.value = ''
    }

    return {
      displayMessage,
      chatbot_output,
      messages,
      message,
      latestBotResponse
    }
  }
}
</script>

<style>
.chatbox {
  background: aliceblue;
  font-family: Arial, Helvetica, sans-serif;
  max-height: 84vh;
}

.chatline {
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: space-between;
  padding: 0.5rem;
}

.chat-heading {
  font-size: 1.5rem;
  font-weight: bold;
  text-align: center;
  margin: 0;
}

.chat-box-list {
  max-height: 37.5rem; /* Set the maximum height of the chat box */
  overflow-y: auto; /* Enable vertical scrolling */
}

.user-message {
  background-color: lightblue;
  padding: 10px;
  margin-bottom: 10px;
  border-radius: 10px;
  max-width: 70%;
  align-self: flex-start;
  font-size: 0.8rem;
  text-align: left;
}

.bot-message {
  background-color: greenyellow;
  padding: 10px;
  margin-bottom: 10px;
  border-radius: 10px;
  max-width: 70%;
  align-self: flex-end;
  font-size: 0.8rem;
  text-align: left;
}

.go-button {
}
</style>
