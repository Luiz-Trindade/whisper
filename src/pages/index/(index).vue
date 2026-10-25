<template>
    <q-page class="column no-wrap window-height overflow-hidden bg-dark text-white">
        <!-- VIEW 1: Chat List -->
        <div v-if="!activeChat" class="column fit">
            <!-- Toolbar -->
            <q-toolbar class="bg-dark text-white q-py-sm">
                <q-toolbar-title class="text-h5 text-weight-bold">Whisper</q-toolbar-title>
                <q-avatar size="36px" color="primary" text-color="white" class="cursor-pointer">
                    <img v-if="myAvatar" :src="myAvatar" style="object-fit: cover" />
                    <span v-else>{{ myNickname ? myNickname.charAt(0).toUpperCase() : 'U' }}</span>
                </q-avatar>
            </q-toolbar>

            <!-- Search Bar -->
            <div class="q-px-md q-pb-sm">
                <q-input
                    v-model="searchQuery"
                    placeholder="Search chats..."
                    dense
                    outlined
                    rounded
                    dark
                    bg-color="grey-10"
                    class="full-width"
                >
                    <template #prepend>
                        <q-icon name="search" color="grey-6" />
                    </template>
                </q-input>
            </div>

            <!-- Chats List (Responsiva com Ellipsis) -->
            <q-scroll-area class="col">
                <q-list separator dark class="bg-transparent q-px-sm">
                    <q-item
                        v-for="chat in filteredChatList"
                        :key="chat.id"
                        clickable
                        v-ripple
                        class="q-py-md rounded-borders row items-center no-wrap"
                        @click="openChat(chat)"
                    >
                        <q-item-section avatar class="col-auto">
                            <q-avatar color="grey-8" text-color="white" size="50px">
                                <img
                                    v-if="chat.avatar"
                                    :src="chat.avatar"
                                    style="object-fit: cover"
                                />
                                <span v-else class="text-h6">
                                    {{
                                        chat.nickname ? chat.nickname.charAt(0).toUpperCase() : 'C'
                                    }}
                                </span>
                            </q-avatar>
                        </q-item-section>

                        <!-- min-width: 0 é crucial para o ellipsis funcionar no Flexbox -->
                        <q-item-section class="col overflow-hidden" style="min-width: 0">
                            <q-item-label
                                class="text-weight-bold text-subtitle1 text-white ellipsis"
                            >
                                {{ chat.nickname }}
                            </q-item-label>
                            <q-item-label caption class="text-grey-5 text-body2 ellipsis">
                                {{ truncateMessage(chat.lastMessage) }}
                            </q-item-label>
                        </q-item-section>

                        <q-item-section side class="col-auto text-right q-pl-sm">
                            <div class="text-caption text-grey-6">{{ chat.time }}</div>
                            <q-badge v-if="chat.unread" color="primary" rounded class="q-mt-xs">
                                {{ chat.unread }}
                            </q-badge>
                        </q-item-section>
                    </q-item>
                </q-list>

                <!-- Empty State -->
                <div
                    v-if="chatList.length === 0"
                    class="text-center text-grey-6 q-py-xl column items-center"
                >
                    <q-icon name="chat_bubble_outline" size="48px" class="q-mb-md opacity-50" />
                    <div class="text-body1 text-weight-medium text-white">No chats found.</div>
                    <div class="text-caption q-mt-xs">Add a contact to get started.</div>
                </div>
            </q-scroll-area>
        </div>

        <!-- VIEW 2: Open Chat Screen -->
        <div v-else class="column fit bg-black">
            <!-- Chat Header -->
            <q-toolbar class="bg-dark text-white q-py-xs" style="z-index: 10">
                <q-btn
                    flat
                    round
                    dense
                    icon="arrow_back"
                    color="white"
                    @click="closeChat"
                    class="q-mr-xs"
                />

                <q-avatar
                    color="grey-8"
                    text-color="white"
                    size="40px"
                    class="q-mr-sm cursor-pointer"
                >
                    <img
                        v-if="activeChat.avatar"
                        :src="activeChat.avatar"
                        style="object-fit: cover"
                    />
                    <span v-else>
                        {{
                            activeChat.nickname ? activeChat.nickname.charAt(0).toUpperCase() : 'C'
                        }}
                    </span>
                </q-avatar>

                <q-toolbar-title class="q-pa-none">
                    <div class="text-subtitle1 text-weight-bold leading-tight">
                        {{ activeChat.nickname }}
                    </div>
                    <div
                        class="row items-center text-caption text-weight-regular"
                        style="margin-top: -2px"
                    >
                        <q-badge
                            :color="peerConnected ? 'positive' : 'grey-6'"
                            rounded
                            style="width: 8px; height: 8px"
                            class="q-mr-xs"
                        />
                        <span :class="peerConnected ? 'text-positive' : 'text-grey-5'">
                            {{ peerConnected ? 'Online' : 'Offline' }}
                        </span>
                        <q-icon name="lock" size="10px" color="grey-6" class="q-ml-sm q-mr-xs" />
                        <span class="text-grey-6" style="font-size: 10px">E2EE</span>
                    </div>
                </q-toolbar-title>
            </q-toolbar>

            <!-- Message List -->
            <div ref="chatScrollRef" class="col scroll q-pa-md bg-black">
                <q-chat-message
                    v-for="msg in activeChat.messages"
                    :key="msg.id"
                    :name="msg.sentBy === myPeerId ? 'You' : activeChat.nickname"
                    :text="[msg.text]"
                    :sent="msg.sentBy === myPeerId"
                    :bg-color="msg.sentBy === myPeerId ? 'green-10' : 'grey-9'"
                    text-color="white"
                    dark
                    class="text-body1"
                >
                    <template #stamp>
                        <div class="row items-center q-gutter-xs text-caption text-grey-5 q-pt-xs">
                            <span>{{ formatFullTime(msg.timestamp) }}</span>
                            <q-icon
                                v-if="msg.sentBy === myPeerId"
                                name="done_all"
                                size="15px"
                                :color="peerConnected ? 'blue-3' : 'grey-5'"
                                class="q-ml-xs"
                            />
                        </div>
                    </template>
                </q-chat-message>
            </div>

            <!-- Input Area -->
            <div class="row items-end q-pa-sm bg-dark">
                <q-input
                    v-model="newMessage"
                    placeholder="Message"
                    dark
                    dense
                    outlined
                    rounded
                    bg-color="grey-10"
                    class="col q-mr-sm"
                    @keyup.enter="sendMessage"
                    autogrow
                />
                <q-btn
                    round
                    color="primary"
                    icon="send"
                    @click="sendMessage"
                    :disable="!newMessage.trim()"
                    class="q-mb-xs"
                />
            </div>
        </div>
    </q-page>
</template>

<script setup>
import { ref, shallowRef, computed, inject, onMounted, onUnmounted, nextTick } from 'vue'
import localforage from 'localforage'
import { joinRoom } from '@trystero-p2p/mqtt'
import mqtt from 'mqtt'

const hideFooter = inject('hideFooter', null)

const TRYSTERO_CONFIG = {
    appId: 'whisper-p2p-mqtt-v1',
}

const MQTT_BROKER_WS = 'wss://broker.emqx.io:8084/mqtt'

const myPeerId = ref('')
const myNickname = ref('')
const myAvatar = ref('')
const searchQuery = ref('')
const chatList = ref([])
const activeChat = ref(null)
const newMessage = ref('')
const peerConnected = ref(false)
const chatScrollRef = ref(null)

// Usando shallowRef para evitar reatividade profunda acidental e erros no-undef do ESLint
const roomInstance = shallowRef(null)
const chatAction = shallowRef(null)
const profileAction = shallowRef(null)
const disconnectTimer = shallowRef(null)
const mqttClient = shallowRef(null)

// Função auxiliar para truncar as mensagens longas na lista de chats
const truncateMessage = (text, maxLength = 15) => {
    if (!text) return 'Tap to start chatting'
    return text.length > maxLength ? text.substring(0, maxLength) + '...' : text
}

// --- Criptografia Ponta a Ponta (E2EE) - Web Crypto API Nativa ---
const hexToBuf = (hex) => {
    const match = hex.match(/.{1,2}/g)
    return new Uint8Array(match ? match.map((b) => parseInt(b, 16)) : [])
}

const bufToHex = (buf) => {
    return Array.from(new Uint8Array(buf))
        .map((b) => b.toString(16).padStart(2, '0'))
        .join('')
}

const getAesKey = async (roomId) => {
    const encoder = new TextEncoder()
    const keyMaterial = await window.crypto.subtle.importKey(
        'raw',
        encoder.encode(roomId),
        { name: 'PBKDF2' },
        false,
        ['deriveKey'],
    )
    return window.crypto.subtle.deriveKey(
        {
            name: 'PBKDF2',
            salt: encoder.encode('whisper-secure-salt'),
            iterations: 100000,
            hash: 'SHA-256',
        },
        keyMaterial,
        { name: 'AES-GCM', length: 256 },
        false,
        ['encrypt', 'decrypt'],
    )
}

const encryptMessage = async (roomId, msgObj) => {
    const key = await getAesKey(roomId)
    const iv = window.crypto.getRandomValues(new Uint8Array(12))
    const encoder = new TextEncoder()
    const data = encoder.encode(JSON.stringify(msgObj))

    const cipherBuffer = await window.crypto.subtle.encrypt({ name: 'AES-GCM', iv }, key, data)

    return {
        id: msgObj.id,
        sentBy: msgObj.sentBy,
        e: true,
        iv: bufToHex(iv),
        c: bufToHex(cipherBuffer),
    }
}

const decryptMessage = async (roomId, payload) => {
    if (!payload.e) return payload
    try {
        const key = await getAesKey(roomId)
        const iv = hexToBuf(payload.iv)
        const cipherBuffer = hexToBuf(payload.c)

        const decryptedBuffer = await window.crypto.subtle.decrypt(
            { name: 'AES-GCM', iv },
            key,
            cipherBuffer,
        )

        const decoder = new TextDecoder()
        return JSON.parse(decoder.decode(decryptedBuffer))
    } catch (error) {
        console.error('Erro ao descriptografar E2EE:', error)
        return null
    }
}
// ----------------------------------------------------------------

const filteredChatList = computed(() => {
    if (!searchQuery.value.trim()) return chatList.value
    return chatList.value.filter((chat) =>
        chat.nickname.toLowerCase().includes(searchQuery.value.toLowerCase()),
    )
})

onMounted(async () => {
    localforage.config({
        name: 'WhisperP2P',
        storeName: 'contacts_store',
    })

    let savedMyId = await localforage.getItem('myPeerId')
    myPeerId.value = savedMyId || ''

    const savedNick = await localforage.getItem('myNickname')
    myNickname.value = savedNick || ''

    const savedAvatar = await localforage.getItem('myAvatar')
    myAvatar.value = savedAvatar || ''

    await loadChats()
})

const formatFullTime = (msTimestamp) => {
    if (!msTimestamp) return ''
    const date = new Date(msTimestamp)
    const hours = date.getHours().toString().padStart(2, '0')
    const minutes = date.getMinutes().toString().padStart(2, '0')
    const seconds = date.getSeconds().toString().padStart(2, '0')
    return `${hours}:${minutes}:${seconds}`
}

const scrollToBottom = () => {
    nextTick(() => {
        if (chatScrollRef.value) {
            chatScrollRef.value.scrollTop = chatScrollRef.value.scrollHeight
        }
    })
}

const sortMessagesByTimestamp = (messages) => {
    return messages.sort((a, b) => (a.timestamp || 0) - (b.timestamp || 0))
}

const loadChats = async () => {
    const savedContacts = (await localforage.getItem('contacts')) || []

    const loadedList = []
    for (const contact of savedContacts) {
        let savedMessages = (await localforage.getItem(`messages_${contact.roomId}`)) || []
        savedMessages = sortMessagesByTimestamp(savedMessages)

        const lastMsg = savedMessages.length > 0 ? savedMessages[savedMessages.length - 1] : null

        loadedList.push({
            ...contact,
            messages: savedMessages,
            lastMessage: lastMsg ? lastMsg.text : '',
            time: lastMsg ? formatFullTime(lastMsg.timestamp) : '',
            lastTimestamp: lastMsg ? lastMsg.timestamp : 0,
            unread: 0,
        })
    }

    loadedList.sort((a, b) => (b.lastTimestamp || 0) - (a.lastTimestamp || 0))
    chatList.value = loadedList
}

const saveMessagesToStorage = async (roomId, messages) => {
    const plainMessages = JSON.parse(JSON.stringify(messages))
    await localforage.setItem(`messages_${roomId}`, plainMessages)
}

const updateContactProfile = async (peerId, nickname, avatar) => {
    const contacts = (await localforage.getItem('contacts')) || []
    const index = contacts.findIndex((c) => c.id === peerId)

    if (index !== -1) {
        if (nickname) contacts[index].nickname = nickname
        if (avatar !== undefined) contacts[index].avatar = avatar
        await localforage.setItem('contacts', contacts)

        if (activeChat.value && activeChat.value.id === peerId) {
            if (nickname) activeChat.value.nickname = nickname
            if (avatar !== undefined) activeChat.value.avatar = avatar
        }
    }
}

const syncMqttCloudQueue = (roomId) => {
    if (mqttClient.value) {
        mqttClient.value.end()
        mqttClient.value = null
    }

    mqttClient.value = mqtt.connect(MQTT_BROKER_WS, {
        clientId: 'whisper_' + myPeerId.value + '_' + Math.random().toString(36).substring(2, 6),
        clean: true,
    })

    const topicPattern = `whisper/v1/store/${roomId}/+`

    mqttClient.value.on('connect', () => {
        mqttClient.value.subscribe(topicPattern, { qos: 1 })
    })

    mqttClient.value.on('message', async (topic, payload) => {
        if (!payload || payload.length === 0) return

        try {
            const rawMsg = JSON.parse(payload.toString())

            if (rawMsg && rawMsg.sentBy !== myPeerId.value && activeChat.value) {
                const incomingMsg = await decryptMessage(roomId, rawMsg)

                if (incomingMsg) {
                    const exists = activeChat.value.messages.some((m) => m.id === incomingMsg.id)

                    if (!exists) {
                        activeChat.value.messages.push(incomingMsg)
                        activeChat.value.messages = sortMessagesByTimestamp(
                            activeChat.value.messages,
                        )

                        activeChat.value.lastMessage = incomingMsg.text
                        activeChat.value.time = formatFullTime(incomingMsg.timestamp)

                        await saveMessagesToStorage(
                            activeChat.value.roomId,
                            activeChat.value.messages,
                        )
                        scrollToBottom()
                    }
                }

                mqttClient.value.publish(topic, '', { retain: true, qos: 1 })
            }
        } catch {
            // Ignora payloads malformados
        }
    })
}

const publishToMqttCloud = (roomId, payloadObj) => {
    if (!mqttClient.value || !mqttClient.value.connected) return

    const topic = `whisper/v1/store/${roomId}/${payloadObj.id}`
    const payload = JSON.stringify(payloadObj)

    mqttClient.value.publish(topic, payload, { retain: true, qos: 1 })
}

const openChat = async (chat) => {
    let storedMessages = (await localforage.getItem(`messages_${chat.roomId}`)) || []
    storedMessages = sortMessagesByTimestamp(storedMessages)

    chat.messages = storedMessages
    chat.unread = 0
    activeChat.value = chat
    peerConnected.value = false

    if (hideFooter) hideFooter.value = true

    roomInstance.value = joinRoom(TRYSTERO_CONFIG, chat.roomId)
    chatAction.value = roomInstance.value.makeAction('chatMessage')
    profileAction.value = roomInstance.value.makeAction('profileSync')

    syncMqttCloudQueue(chat.roomId)

    roomInstance.value.onPeerJoin = () => {
        if (disconnectTimer.value) {
            clearTimeout(disconnectTimer.value)
            disconnectTimer.value = null
        }

        if (!peerConnected.value) {
            peerConnected.value = true

            if (profileAction.value) {
                profileAction.value.send({
                    nickname: myNickname.value,
                    avatar: myAvatar.value,
                })
            }
        }
    }

    roomInstance.value.onPeerLeave = () => {
        if (disconnectTimer.value) clearTimeout(disconnectTimer.value)

        disconnectTimer.value = setTimeout(() => {
            peerConnected.value = false
            disconnectTimer.value = null
        }, 4000)
    }

    profileAction.value.onMessage = async (profileData) => {
        if (profileData && activeChat.value) {
            await updateContactProfile(
                activeChat.value.id,
                profileData.nickname,
                profileData.avatar,
            )
        }
    }

    chatAction.value.onMessage = async (rawMsg) => {
        if (!activeChat.value || !rawMsg) return

        const incomingMsg = await decryptMessage(activeChat.value.roomId, rawMsg)
        if (!incomingMsg) return

        const exists = activeChat.value.messages.some((m) => m.id === incomingMsg.id)
        if (!exists) {
            activeChat.value.messages.push(incomingMsg)
            activeChat.value.messages = sortMessagesByTimestamp(activeChat.value.messages)

            activeChat.value.lastMessage = incomingMsg.text
            activeChat.value.time = formatFullTime(incomingMsg.timestamp)

            await saveMessagesToStorage(activeChat.value.roomId, activeChat.value.messages)
            scrollToBottom()

            if (mqttClient.value && mqttClient.value.connected) {
                const topic = `whisper/v1/store/${activeChat.value.roomId}/${incomingMsg.id}`
                mqttClient.value.publish(topic, '', { retain: true, qos: 1 })
            }
        }
    }

    scrollToBottom()
}

const sendMessage = async () => {
    if (!newMessage.value.trim() || !activeChat.value) return

    const nowMs = Date.now()
    const timeFormatted = formatFullTime(nowMs)

    const msgObj = {
        id: `${myPeerId.value}_${nowMs}_${Math.random().toString(36).substring(2, 7)}`,
        text: newMessage.value.trim(),
        sentBy: myPeerId.value,
        time: timeFormatted,
        timestamp: nowMs,
    }

    const encryptedPayload = await encryptMessage(activeChat.value.roomId, msgObj)

    publishToMqttCloud(activeChat.value.roomId, encryptedPayload)

    if (peerConnected.value && chatAction.value) {
        chatAction.value.send(encryptedPayload)
    }

    activeChat.value.messages.push(msgObj)
    activeChat.value.messages = sortMessagesByTimestamp(activeChat.value.messages)

    activeChat.value.lastMessage = msgObj.text
    activeChat.value.time = timeFormatted

    await saveMessagesToStorage(activeChat.value.roomId, activeChat.value.messages)

    newMessage.value = ''
    scrollToBottom()
}

const closeChat = async () => {
    if (disconnectTimer.value) {
        clearTimeout(disconnectTimer.value)
        disconnectTimer.value = null
    }
    if (mqttClient.value) {
        mqttClient.value.end()
        mqttClient.value = null
    }
    if (roomInstance.value) {
        roomInstance.value.leave()
        roomInstance.value = null
        chatAction.value = null
        profileAction.value = null
    }
    peerConnected.value = false
    activeChat.value = null
    if (hideFooter) hideFooter.value = false

    await loadChats()
}

onUnmounted(() => {
    if (disconnectTimer.value) clearTimeout(disconnectTimer.value)
    if (mqttClient.value) {
        mqttClient.value.end()
    }
    if (roomInstance.value) {
        roomInstance.value.leave()
    }
})
</script>

<style scoped>
.leading-tight {
    line-height: 1.2;
}

.ellipsis {
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}
</style>
