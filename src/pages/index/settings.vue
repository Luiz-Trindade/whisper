<template>
    <q-page class="column no-wrap window-height overflow-hidden bg-dark text-white">
        <!-- Toolbar -->
        <q-toolbar class="bg-dark text-white q-py-sm col-auto">
            <q-toolbar-title class="text-h5 text-weight-bold">Profile & Settings</q-toolbar-title>
        </q-toolbar>

        <q-scroll-area class="col">
            <div class="q-pa-md column items-center">
                <!-- Profile Picture Section (Flat) -->
                <div class="relative-position q-mb-lg q-mt-md">
                    <q-avatar size="120px" color="grey-10" text-color="white">
                        <img v-if="avatar" :src="avatar" style="object-fit: cover" />
                        <span v-else class="text-h2 text-weight-bold">
                            {{ nickname ? nickname.charAt(0).toUpperCase() : 'U' }}
                        </span>
                    </q-avatar>

                    <!-- Overlay Button for Changing Photo -->
                    <q-btn
                        round
                        unelevated
                        color="primary"
                        icon="photo_camera"
                        size="md"
                        class="absolute-bottom-right"
                        style="transform: translate(10%, 10%)"
                        @click="triggerFileInput"
                    >
                        <q-tooltip>Change photo</q-tooltip>
                    </q-btn>
                </div>

                <q-btn
                    v-if="avatar"
                    flat
                    dense
                    color="negative"
                    label="Remove Photo"
                    class="q-mb-md"
                    @click="removeAvatar"
                />

                <!-- Hidden File Input -->
                <input
                    ref="fileInputRef"
                    type="file"
                    accept="image/*"
                    style="display: none"
                    @change="handleFileChange"
                />

                <!-- Profile Form -->
                <div class="full-width q-gutter-y-md q-mt-sm" style="max-width: 400px">
                    <div class="text-subtitle2 text-grey-5 q-ml-sm q-mb-xs">Display Name</div>
                    <q-input
                        v-model="nickname"
                        placeholder="How you appear on Whisper"
                        outlined
                        dense
                        dark
                        bg-color="grey-10"
                    >
                        <template #prepend>
                            <q-icon name="person" color="grey-6" />
                        </template>
                    </q-input>

                    <div class="text-subtitle2 text-grey-5 q-ml-sm q-mb-xs q-mt-lg">
                        Your P2P Identity
                    </div>
                    <q-input v-model="peerId" outlined dense dark readonly bg-color="grey-10">
                        <template #prepend>
                            <q-icon name="fingerprint" color="grey-6" />
                        </template>
                        <template #append>
                            <q-btn
                                flat
                                round
                                icon="content_copy"
                                color="grey-4"
                                @click="copyId"
                                size="sm"
                            />
                            <q-btn
                                flat
                                round
                                icon="refresh"
                                color="warning"
                                @click="generateNewId"
                                size="sm"
                            />
                        </template>
                    </q-input>
                    <div class="text-caption text-grey-6 q-px-sm text-center">
                        Share this ID in the Contacts tab to chat securely.
                    </div>

                    <q-btn
                        icon="save"
                        label="Save Changes"
                        unelevated
                        rounded
                        size="lg"
                        color="primary"
                        class="full-width text-weight-bold"
                        @click="saveSettings"
                    />

                    <!-- About Section (Flat Card) -->
                    <div class="q-mt-xl bg-grey-10 q-pa-md" style="border-radius: 12px">
                        <div class="row items-center q-mb-sm">
                            <q-icon name="info" color="primary" size="20px" class="q-mr-xs" />
                            <div class="text-subtitle1 text-weight-bold">About Whisper</div>
                        </div>

                        <div class="text-caption text-grey-4 q-gutter-y-xs">
                            <p class="q-mb-xs">
                                Whisper is an open-source decentralized messaging app built with a
                                strong focus on
                                <strong>privacy</strong>, <strong>security</strong>, and
                                peer-to-peer networking.
                            </p>
                            <p class="q-mb-xs">
                                Developed as an academic study and software development project by a
                                <strong>Computer Science student</strong>.
                            </p>
                            <p class="q-mb-xs text-grey-5">
                                Licensed under the <strong>AGPL-3.0 License</strong>.
                            </p>
                            <p class="q-mb-none text-grey-5">
                                <strong>Disclaimer:</strong> The developer accepts no responsibility
                                or liability for user conduct, infractions, or illegal activities
                                committed through this application.
                            </p>
                        </div>
                    </div>
                </div>
            </div>
        </q-scroll-area>
    </q-page>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { useQuasar } from 'quasar'
import localforage from 'localforage'
import { selfId } from 'trystero'

const $q = useQuasar()
const nickname = ref('')
const peerId = ref('')
const avatar = ref('')
const fileInputRef = ref(null)

onMounted(async () => {
    localforage.config({ name: 'WhisperP2P', storeName: 'contacts_store' })
    const savedNickname = await localforage.getItem('myNickname')
    nickname.value = savedNickname || 'User_' + Math.floor(Math.random() * 1000)
    const savedAvatar = await localforage.getItem('myAvatar')
    if (savedAvatar) avatar.value = savedAvatar
    let savedMyId = await localforage.getItem('myPeerId')
    if (!savedMyId) {
        savedMyId = selfId || 'peer_' + Math.random().toString(36).substring(2, 10)
        await localforage.setItem('myPeerId', savedMyId)
    }
    peerId.value = savedMyId
})

const triggerFileInput = () => {
    if (fileInputRef.value) fileInputRef.value.click()
}

const compressImage = (file) => {
    return new Promise((resolve) => {
        const reader = new FileReader()
        reader.onload = (e) => {
            const img = new Image()
            img.onload = () => {
                const canvas = document.createElement('canvas')
                const maxDim = 250
                let width = img.width
                let height = img.height

                if (width > height) {
                    if (width > maxDim) {
                        height = Math.round((height * maxDim) / width)
                        width = maxDim
                    }
                } else {
                    if (height > maxDim) {
                        width = Math.round((width * maxDim) / height)
                        height = maxDim
                    }
                }

                canvas.width = width
                canvas.height = height
                const ctx = canvas.getContext('2d')
                ctx.drawImage(img, 0, 0, width, height)
                resolve(canvas.toDataURL('image/jpeg', 0.8))
            }
            img.src = e.target.result
        }
        reader.readAsDataURL(file)
    })
}

const handleFileChange = async (event) => {
    const file = event.target.files && event.target.files[0]
    if (!file) return
    if (!file.type.startsWith('image/')) {
        $q.notify({ type: 'negative', message: 'Please select a valid image file.' })
        return
    }
    try {
        const compressedBase64 = await compressImage(file)
        avatar.value = compressedBase64
        await localforage.setItem('myAvatar', compressedBase64)
        $q.notify({
            type: 'positive',
            message: 'Profile picture updated!',
            icon: 'photo',
            position: 'top',
        })
    } catch {
        $q.notify({ type: 'negative', message: 'Error processing image.' })
    }
}

const removeAvatar = async () => {
    avatar.value = ''
    await localforage.removeItem('myAvatar')
    $q.notify({ type: 'info', message: 'Profile picture removed.', position: 'top' })
}

const generateNewId = async () => {
    const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789'
    let newHash = ''
    for (let i = 0; i < 20; i++) {
        newHash += chars.charAt(Math.floor(Math.random() * chars.length))
    }
    peerId.value = newHash
    await localforage.setItem('myPeerId', newHash)
    $q.notify({
        type: 'warning',
        message: 'New ID generated successfully!',
        icon: 'refresh',
        position: 'top',
        timeout: 1500,
    })
}

const copyId = () => {
    if (!peerId.value) return
    navigator.clipboard.writeText(peerId.value)
    $q.notify({
        type: 'positive',
        message: 'ID copied!',
        icon: 'content_copy',
        position: 'top',
        timeout: 1500,
    })
}

const saveSettings = async () => {
    await localforage.setItem('myNickname', nickname.value)
    await localforage.setItem('myPeerId', peerId.value)
    await localforage.setItem('myAvatar', avatar.value)
    $q.notify({
        type: 'positive',
        message: 'Settings saved!',
        icon: 'check_circle',
        position: 'top',
        timeout: 1500,
    })
}
</script>

<style scoped></style>
