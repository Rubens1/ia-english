<script setup>
import { ref, computed, nextTick, onMounted } from 'vue';
import { Icon } from '@iconify/vue';
import { GoogleGenerativeAI } from '@google/generative-ai'

// Inicializando a API do Google Generative AI
const genAI = new GoogleGenerativeAI("AIzaSyDMFQEfDkG4iWpZbxaNeDgjRcCyVaoxQqQ");
const model = genAI.getGenerativeModel({ 
  model: 'gemini-1.5-flash',  // Novo nome do modelo
  apiVersion: 'v1'         // Versão mais recente da API
});
const mainGoogle = async (texto) => {
  try {
    const userMessage = { role: "user", parts: [{ text: texto }] };
    const modelResponse = { role: "model", parts: [{ text: "Eu estou aprendnedo inglês e preciso que você me ajuda com a pronuncia." }] };
    
    const chat = model.startChat({
      history: [userMessage, modelResponse],
      generationConfig: {
        maxOutputTokens: 100,
      },
    });

    const result = await chat.sendMessage(texto);
    const response = await result.response;
    const text = response.text();

    return text;

  } catch (error) {
    console.error("Erro ao iniciar a conversa com a AI: ", error);
    throw error;
  }
};

const isRecording = ref(false);
const recognition = ref(null);
const messages = ref([]);
const inputMessage = ref('');
const isTyping = ref(false);
const selectedLanguage = ref('en-US');
const voices = ref([]);
const synth = ref(null);

const SUPPORTED_LANGUAGES = [
  { code: 'en-US', name: 'English' },
  { code: 'pt-BR', name: 'Português' },
  { code: 'es-ES', name: 'Español' },
  { code: 'fr-FR', name: 'Français' }
];

// Configurações do chat
const CHAT_DELAY_PER_CHAR = 30;

onMounted(() => {
  if ('speechSynthesis' in window) {
    synth.value = window.speechSynthesis;
    const updateVoices = () => {
      voices.value = synth.value.getVoices();
    };
    synth.value.onvoiceschanged = updateVoices;
    updateVoices();
  }
});

function speakText(text) {
    if (!synth.value) return;

const utterance = new SpeechSynthesisUtterance(text);
const voice = voices.value.find(v => v.lang === selectedLanguage.value);

if (voice) {
  utterance.voice = voice;
  utterance.lang = selectedLanguage.value;
  synth.value.speak(utterance);
}

}

function sendMessage() {
    if (inputMessage.value.trim()) {
        addUserMessage(inputMessage.value);
        simulateResponse(inputMessage.value);
        inputMessage.value = '';
    }
}

function addUserMessage(text) {
    messages.value.push({
        id: Date.now(),
        sender: 'user',
        text: text.trim()
    });
    scrollToBottom();
}

async function simulateResponse(userMessage) {
    isTyping.value = true;
    scrollToBottom();
    
    const delay = Math.max(1000, userMessage.length * CHAT_DELAY_PER_CHAR);
    await new Promise(resolve => setTimeout(resolve, delay));
    
    const responseFromAI = await mainGoogle(userMessage);
    messages.value.push({
        id: Date.now() + 1,
        sender: 'assistant',
        text: responseFromAI
    });
    
    isTyping.value = false;
    scrollToBottom();
    speakText(responseFromAI);
}


const microphoneIcon = computed(() => 
    isRecording.value ? 'mdi:microphone-off' : 'mdi:microphone'
);

function startRecording() {
    if (!('webkitSpeechRecognition' in window)) {
        alert('Seu navegador não suporta reconhecimento de voz!');
        return;
    }

    recognition.value = new webkitSpeechRecognition();
    recognition.value.lang = selectedLanguage.value;
    recognition.value.continuous = true; // Permite múltiplas gravações
    recognition.value.interimResults = false;

    recognition.value.onresult = (event) => {
        const transcript = event.results[event.results.length - 1][0].transcript;
        addUserMessage(transcript);
        simulateResponse(transcript);
    };

    recognition.value.onerror = (event) => {
        //console.error('Erro:', event.error);
        isRecording.value = false;
    };

    recognition.value.onend = () => {
        isRecording.value = false;
    };

    recognition.value.start();
    isRecording.value = true;
}

function stopRecording() {
    if (recognition.value) {
        recognition.value.stop();
        isRecording.value = false;
    }
}

function scrollToBottom() {
    nextTick(() => {
        const chatMessages = document.querySelector('.chat-messages');
        if (chatMessages) {
            chatMessages.scrollTop = chatMessages.scrollHeight;
        }
    });
}

</script>

<template>
    <div class="chat-container">
        <div class="language-selector">
      <select v-model="selectedLanguage">
        <option 
          v-for="lang in SUPPORTED_LANGUAGES" 
          :key="lang.code" 
          :value="lang.code"
        >
          {{ lang.name }}
        </option>
      </select>
    </div>
        <div class="chat-messages">
            <div v-for="message in messages" :key="message.id"
                :class="message.sender === 'user' ? 'chat-message user' : 'chat-message assistant'">
                <div>{{ message.text }}</div>
            </div>
        </div>

        <div class="chat-input">
            <input type="text" v-model="inputMessage" @keyup.enter="sendMessage" placeholder="Digite sua mensagem...">
            <button @click="sendMessage">
                <Icon icon="tabler:send" width="24" height="24" />
            </button>
            <button @click="isRecording ? stopRecording() : startRecording()" 
                    :class="{ 'recording': isRecording }">
                <Icon :icon="microphoneIcon" width="24" height="24" />
                {{ isRecording ? 'Parar' : 'Falar' }}
            </button>
        </div>
    </div>
</template>

<style scoped>
.language-selector {
  margin-bottom: 1rem;
}

.language-selector select {
  padding: 8px 12px;
  border-radius: 4px;
  border: 1px solid #ccc;
  background-color: white;
}
.audio-preview{
    display: none;
}

.voice-recorder {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 1rem;
}

button.recording {
    background: #f44336;
    animation: pulse 1.5s infinite;
}

@keyframes pulse {
    0% {
        transform: scale(1);
    }

    50% {
        transform: scale(1.05);
    }

    100% {
        transform: scale(1);
    }
}

audio {
    margin-top: 1rem;
    width: 250px;
}

.chat-container {
    max-width: 600px;
    margin: 0 auto;
    padding: 20px;
}

.chat-messages {
    height: 700px;
    overflow-y: scroll;
    padding: 10px;
    background-color: #fff;
    border-radius: 5px;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.1);
}

.chat-message {
    margin-bottom: 10px;
    padding: 10px;
    border-radius: 5px;
}

.chat-message.user {
    background-color: #e8f5ff;
    text-align: right;
    color: black;
}

.chat-message.assistant {
    background-color: #f5f5f5;
    text-align: left;
    color: black;
}

.chat-input {
    margin-top: 10px;
    display: flex;
}

.chat-input input {
    flex-grow: 1;
    padding: 10px;
    border: none;
    border-radius: 5px;
    outline: none;
}

.chat-input button {
    padding: 10px 15px;
    border: none;
    border-radius: 5px;
    background-color: #4caf50;
    color: #fff;
    cursor: pointer;
    outline: none;
    margin: 0 10px;
    display: flex;
    align-items: center;
    justify-content: center;
}
</style>