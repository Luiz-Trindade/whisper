<template>
  <q-page class="q-pa-md column">
    <!-- VISÃO 1: Lista de Conversas -->
    <div v-if="!activeChat" class="full-width">
      <div class="row items-center justify-between q-mb-md">
        <div class="text-h6 text-weight-bold text-primary">Conversas</div>
        <q-chip color="green-13" text-color="black" icon="wifi">P2P Ativo</q-chip>
      </div>

      <q-list separator class="bg-white rounded-borders shadow-1">
        <q-item
          v-for="chat in chatList"
          :key="chat.id"
          clickable
          v-ripple
          @click="openChat(chat)"
        >
          <q-item-section avatar>
            <q-avatar color="primary" text-color="white" icon="person" />
          </q-item-section>

          <q-item-section>
            <q-item-label class="text-weight-bold">{{ chat.nickname }}</q-item-label>
            <q-item-label caption lines="1">
              {{ chat.lastMessage || 'Nenhuma mensagem ainda' }}
            </q-item-label>
          </q-item-section>

          <q-item-section side class="text-caption">
            <div>{{ chat.time }}</div>
            <q-badge v-if="chat.unread" color="primary" rounded class="q-mt-xs">
              {{ chat.unread }}
            </q-badge>
          </q-item-section>
        </q-item>
      </q-list>

      <div v-if="chatList.length === 0" class="text-center text-grey q-py-xl">
        Nenhum contacto encontrado. Adicione um contacto na aba "Contacts"!
      </div>
    </div>

    <!-- VISÃO 2: Tela da Conversa Aberta -->
    <div v-else class="column col justify-between full-width">
      <!-- Cabeçalho do Chat -->
      <div class="row items-center q-pb-sm q-mb-sm border-bottom">
        <q-btn flat round icon="arrow_back" color="primary" @click="closeChat" />
        <q-avatar color="primary" text-color="white" icon="person" class="q-mx-sm" size="36px" />
        <div class="col">
          <div class="text-weight-bold">{{ activeChat.nickname }}</div>
          <div class="text-caption text-grey row items-center">
            <q-badge :color="peerConnected ? 'positive' : 'grey-5'" class="q-mr-xs" rounded />
            {{ peerConnected ? 'Conectado P2P' : 'Aguardando contacto entrar...' }}
          </div>
        </div>
      </div>

      <!-- Lista de Mensagens -->
      <div class="col scroll q-pa-sm bg-grey-2 rounded-borders" style="height: 60vh">
        <q-chat-message
          v-for="(msg, index) in activeChat.messages"
          :key="index"
          :name="msg.sent ? 'Você' : activeChat.nickname"
          :text="[msg.text]"
          :sent="msg.sent"
          :stamp="msg.time"
          :bg-color="msg.sent ? 'primary' : 'white'"
          :text-color="msg.sent ? 'white' : 'black'"
        />
      </div>

      <!-- Input de Envio -->
      <div class="row items-center q-pt-sm q-gutter-xs">
        <q-input
          v-model="newMessage"
          placeholder="Escreva uma mensagem..."
          outlined
          dense
          class="col"
          @keyup.enter="sendMessage"
        />
        <q-btn icon="send" color="primary" round @click="sendMessage" />
      </div>
    </div>
  </q-page>
</template>

<script setup>
import { ref, inject, onMounted, onUnmounted } from 'vue'
import localforage from 'localforage'
import { joinRoom } from 'trystero'

const hideFooter = inject('hideFooter', null)

const chatList = ref([])
const activeChat = ref(null)
const newMessage = ref('')
const peerConnected = ref(false)

let roomInstance = null
let chatAction = null

onMounted(async () => {
  localforage.config({
    name: 'WhisperP2P',
    storeName: 'contacts_store'
  })

  await loadChats()
})

// Carrega os contactos do localforage e os seus respetivos históricos de mensagens
const loadChats = async () => {
  const savedContacts = (await localforage.getItem('contacts')) || []
  
  const loadedList = []
  for (const contact of savedContacts) {
    const messages = (await localforage.getItem(`messages_${contact.roomId}`)) || []
    const lastMsg = messages.length > 0 ? messages[messages.length - 1] : null
    
    loadedList.push({
      ...contact,
      messages,
      lastMessage: lastMsg ? lastMsg.text : '',
      time: lastMsg ? lastMsg.time : '',
      unread: 0
    })
  }

  chatList.value = loadedList
}

// Abrir bate-papo e conectar na sala P2P no Trystero
const openChat = async (chat) => {
  chat.unread = 0
  activeChat.value = chat
  if (hideFooter) hideFooter.value = true

  // Entra na sala P2P específica deste contacto
  const config = { appId: 'whisper-p2p-app' }
  roomInstance = joinRoom(config, chat.roomId)

  // Cria a ação de chat
  chatAction = roomInstance.makeAction('chatMessage')

  // Status da ligação
  roomInstance.onPeerJoin = () => {
    peerConnected.value = true
  }

  roomInstance.onPeerLeave = () => {
    peerConnected.value = false
  }

  // Escuta mensagens recebidas em tempo real
  chatAction.onMessage = (data) => {
    receiveMessage(data)
  }
}

// Receber mensagem P2P
const receiveMessage = async (text) => {
  if (!activeChat.value) return

  const now = new Date()
  const timeStr = `${now.getHours().toString().padStart(2, '0')}:${now.getMinutes().toString().padStart(2, '0')}`

  const msgObj = {
    text,
    sent: false,
    time: timeStr
  }

  activeChat.value.messages.push(msgObj)
  activeChat.value.lastMessage = text
  activeChat.value.time = timeStr

  // Guarda as mensagens atualizadas no localforage
  await localforage.setItem(`messages_${activeChat.value.roomId}`, activeChat.value.messages)
}

// Enviar mensagem via Trystero e guardar localmente
const sendMessage = async () => {
  if (!newMessage.value.trim() || !activeChat.value || !chatAction) return

  const now = new Date()
  const timeStr = `${now.getHours().toString().padStart(2, '0')}:${now.getMinutes().toString().padStart(2, '0')}`
  const text = newMessage.value

  // Transmite a mensagem diretamente ao par
  chatAction.send(text)

  const msgObj = {
    text,
    sent: true,
    time: timeStr
  }

  activeChat.value.messages.push(msgObj)
  activeChat.value.lastMessage = text
  activeChat.value.time = timeStr

  // Guarda no localforage
  await localforage.setItem(`messages_${activeChat.value.roomId}`, activeChat.value.messages)

  newMessage.value = ''
}

// Fechar conversa e desconectar da sala
const closeChat = () => {
  if (roomInstance) {
    roomInstance.leave()
    roomInstance = null
    chatAction = null
  }
  peerConnected.value = false
  activeChat.value = null
  if (hideFooter) hideFooter.value = false
}

onUnmounted(() => {
  closeChat()
})
</script>

<style scoped>
.border-bottom {
  border-bottom: 1px solid #e0e0e0;
}
</style>