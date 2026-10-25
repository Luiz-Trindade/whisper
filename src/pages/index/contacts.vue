<template>
  <q-page class="column no-wrap window-height overflow-hidden bg-dark text-white">
    <!-- Toolbar -->
    <q-toolbar class="q-py-md col-auto">
      <q-toolbar-title class="text-h5 text-weight-bold">Contacts</q-toolbar-title>
    </q-toolbar>

    <!-- Flat Action Buttons (Responsive) -->
    <div class="row q-col-gutter-sm q-px-md q-pb-md col-auto">
      <div class="col-12 col-sm-6">
        <q-btn
          unelevated
          color="primary"
          icon="qr_code_scanner"
          label="Scan QR"
          class="full-width text-weight-bold"
          style="border-radius: 12px"
          @click="openQuickAddDialog"
        />
      </div>
      <div class="col-12 col-sm-6">
        <q-btn
          unelevated
          color="grey-10"
          text-color="white"
          icon="add"
          label="Manual Add"
          class="full-width text-weight-bold"
          style="border-radius: 12px"
          @click="openAddDialog"
        />
      </div>
    </div>

    <!-- Simplified Contacts List -->
    <q-scroll-area class="col">
      <q-list separator dark class="bg-transparent q-px-sm">
        <q-item v-for="contact in contacts" :key="contact.id" clickable v-ripple class="q-py-md">
          <q-item-section avatar>
            <q-avatar color="grey-10" text-color="white" size="52px">
              <img v-if="contact.avatar" :src="contact.avatar" style="object-fit: cover" />
              <span v-else class="text-h6 text-weight-medium">{{
                contact.nickname ? contact.nickname.charAt(0).toUpperCase() : 'C'
              }}</span>
            </q-avatar>
          </q-item-section>

          <q-item-section class="overflow-hidden">
            <q-item-label class="text-weight-bold text-white text-subtitle1 ellipsis">{{
              contact.nickname
            }}</q-item-label>
            <q-item-label caption class="text-grey-5 ellipsis">ID: {{ contact.id }}</q-item-label>
          </q-item-section>

          <q-item-section side>
            <div class="row q-gutter-xs">
              <q-btn flat round dense color="grey-5" icon="edit" @click="openEditDialog(contact)" />
              <q-btn
                flat
                round
                dense
                color="negative"
                icon="delete"
                @click="confirmDelete(contact)"
              />
            </div>
          </q-item-section>
        </q-item>

        <!-- Empty State -->
        <div
          v-if="contacts.length === 0"
          class="text-center text-grey-6 q-py-xl column items-center"
        >
          <q-icon name="people_outline" size="48px" class="q-mb-md opacity-50" />
          <div class="text-body1 text-weight-medium text-white">No contacts added yet.</div>
          <div class="text-caption q-mt-xs">Use the buttons above to get started.</div>
        </div>
      </q-list>
    </q-scroll-area>

    <!-- Maximized Dialog: Manual Add / Edit -->
    <q-dialog
      v-model="isDialogOpen"
      maximized
      transition-show="slide-up"
      transition-hide="slide-down"
    >
      <q-card class="bg-dark text-white column no-wrap no-shadow">
        <q-toolbar class="q-py-md col-auto">
          <q-btn flat round dense icon="close" v-close-popup />
          <q-toolbar-title class="text-weight-bold">
            {{ isEditing ? 'Edit Contact' : 'New Contact' }}
          </q-toolbar-title>
        </q-toolbar>

        <q-card-section class="col q-pt-lg">
          <q-form @submit="saveContact" class="column q-gutter-y-md">
            <div>
              <div class="text-subtitle2 text-grey-5 q-ml-sm q-mb-xs">Contact Name</div>
              <q-input
                v-model="form.nickname"
                placeholder="e.g., John Doe"
                dark
                rounded
                standout="bg-grey-9 text-white"
                bg-color="grey-10"
                :rules="[(val) => !!val || 'Please enter a nickname']"
              />
            </div>

            <div>
              <div class="text-subtitle2 text-grey-5 q-ml-sm q-mb-xs">P2P ID (Peer ID)</div>
              <q-input
                v-model="form.id"
                placeholder="Paste the unique ID here"
                dark
                rounded
                standout="bg-grey-9 text-white"
                bg-color="grey-10"
                hint="The unique ID generated in your friend's app"
                :rules="[(val) => !!val || 'Please enter the ID']"
              />
            </div>

            <div class="q-mt-xl">
              <q-btn
                :icon="isEditing ? 'update' : 'save'"
                :label="isEditing ? 'Update Contact' : 'Save Contact'"
                type="submit"
                unelevated
                rounded
                size="lg"
                color="primary"
                class="full-width text-weight-bold"
              />
            </div>
          </q-form>
        </q-card-section>
      </q-card>
    </q-dialog>

    <!-- Maximized Dialog: QR Code -->
    <q-dialog
      v-model="isQuickAddOpen"
      maximized
      transition-show="slide-up"
      transition-hide="slide-down"
      @hide="stopScanner"
    >
      <q-card class="bg-dark text-white column no-wrap no-shadow">
        <q-toolbar class="q-py-md col-auto">
          <q-btn flat round dense icon="close" v-close-popup />
          <q-toolbar-title class="text-weight-bold">Add via QR Code</q-toolbar-title>
        </q-toolbar>

        <q-tabs
          v-model="quickTab"
          active-color="primary"
          indicator-color="primary"
          align="justify"
          dense
          no-caps
          class="text-grey-5 col-auto"
          @update:model-value="onTabChange"
        >
          <q-tab name="qr" label="My QR Code" icon="qr_code" />
          <q-tab name="scan" label="Scan" icon="qr_code_scanner" />
        </q-tabs>

        <q-tab-panels v-model="quickTab" animated class="bg-transparent col">
          <!-- TAB 1: My QR -->
          <q-tab-panel name="qr" class="column items-center justify-center q-pa-none">
            <div class="text-caption text-grey-5 text-center q-mb-xl">
              Ask your friend to scan this code.
            </div>

            <div class="q-pa-md bg-white rounded-borders">
              <qrcode-vue v-if="qrPayload" :value="qrPayload" :size="220" level="H" />
            </div>

            <div
              class="row items-center q-mt-xl bg-grey-10 q-pa-md"
              style="border-radius: 16px; min-width: 250px"
            >
              <q-avatar size="48px" color="primary" text-color="white" class="q-mr-md">
                <img v-if="myAvatar" :src="myAvatar" style="object-fit: cover" />
                <span v-else>{{ myNickname ? myNickname.charAt(0).toUpperCase() : 'U' }}</span>
              </q-avatar>
              <div class="column">
                <span class="text-h6 text-weight-bold text-white leading-tight">{{
                  myNickname
                }}</span>
                <span class="text-caption text-grey-5 leading-tight">Your Public Profile</span>
              </div>
            </div>
          </q-tab-panel>

          <!-- TAB 2: Scan -->
          <q-tab-panel name="scan" class="column q-pa-md">
            <div class="text-caption text-grey-5 text-center q-mb-md">
              Point your camera at your friend's QR Code.
            </div>

            <div
              id="qr-reader"
              style="width: 100%; border-radius: 12px"
              class="col overflow-hidden bg-black"
            ></div>
          </q-tab-panel>
        </q-tab-panels>
      </q-card>
    </q-dialog>
  </q-page>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, nextTick } from 'vue'
import { useQuasar } from 'quasar'
import localforage from 'localforage'
import QrcodeVue from 'qrcode.vue'
import { Html5Qrcode } from 'html5-qrcode'

const $q = useQuasar()

const myPeerId = ref('')
const myNickname = ref('')
const myAvatar = ref('')
const contacts = ref([])

const isDialogOpen = ref(false)
const isEditing = ref(false)
const currentEditId = ref(null)
const form = ref({ nickname: '', id: '' })

const isQuickAddOpen = ref(false)
const quickTab = ref('qr')
let html5QrcodeScanner = null

const qrPayload = computed(() => {
  if (!myPeerId.value) return ''
  return JSON.stringify({
    v: 'whisper_v1',
    id: myPeerId.value,
    nickname: myNickname.value,
  })
})

onMounted(async () => {
  localforage.config({ name: 'WhisperP2P', storeName: 'contacts_store' })

  let savedMyId = await localforage.getItem('myPeerId')
  if (!savedMyId) {
    savedMyId = 'my_peer_' + Math.random().toString(36).substring(2, 10)
    await localforage.setItem('myPeerId', savedMyId)
  }
  myPeerId.value = savedMyId

  const savedNick = await localforage.getItem('myNickname')
  myNickname.value = savedNick || 'User_' + Math.floor(Math.random() * 1000)

  const savedAvatar = await localforage.getItem('myAvatar')
  if (savedAvatar) myAvatar.value = savedAvatar

  const savedContacts = await localforage.getItem('contacts')
  if (savedContacts) contacts.value = savedContacts
})

const generateRoomHash = async (idA, idB) => {
  const sortedPair = [idA, idB].sort().join('_')
  const encoder = new TextEncoder()
  const data = encoder.encode(sortedPair)
  const hashBuffer = await crypto.subtle.digest('SHA-256', data)
  const hashArray = Array.from(new Uint8Array(hashBuffer))
  return hashArray
    .map((b) => b.toString(16).padStart(2, '0'))
    .join('')
    .substring(0, 24)
}

const persistContacts = async () => {
  const plainContacts = JSON.parse(JSON.stringify(contacts.value))
  await localforage.setItem('contacts', plainContacts)
}

const addOrUpdateContact = async (peerId, nickname, avatar = '') => {
  const roomId = await generateRoomHash(myPeerId.value, peerId)
  const existingIndex = contacts.value.findIndex((c) => c.id === peerId)

  const contactData = {
    nickname: nickname || 'P2P Contact',
    id: peerId,
    roomId: roomId,
    avatar: avatar || (existingIndex !== -1 ? contacts.value[existingIndex].avatar || '' : ''),
  }

  if (existingIndex !== -1) {
    contacts.value[existingIndex] = contactData
  } else {
    contacts.value.push(contactData)
  }

  await persistContacts()
  return contactData
}

const openQuickAddDialog = () => {
  quickTab.value = 'qr'
  isQuickAddOpen.value = true
}

const onTabChange = (newTab) => {
  if (newTab === 'scan') {
    startScanner()
  } else {
    stopScanner()
  }
}

const startScanner = () => {
  nextTick(async () => {
    try {
      if (html5QrcodeScanner) {
        await stopScanner()
      }

      html5QrcodeScanner = new Html5Qrcode('qr-reader')
      await html5QrcodeScanner.start(
        { facingMode: 'environment' },
        { fps: 10, qrbox: { width: 220, height: 220 } },
        async (decodedText) => {
          await stopScanner()
          try {
            const data = JSON.parse(decodedText)

            if (data && data.v === 'whisper_v1' && data.id) {
              if (data.id === myPeerId.value) {
                $q.notify({
                  type: 'warning',
                  message: 'You scanned your own QR Code!',
                  position: 'top',
                })
                return
              }

              const added = await addOrUpdateContact(data.id, data.nickname)
              $q.notify({
                type: 'positive',
                message: `Contact "${added.nickname}" added successfully!`,
                icon: 'person_add',
                position: 'top',
              })
              isQuickAddOpen.value = false
            } else {
              $q.notify({ type: 'warning', message: 'Invalid QR Code.', position: 'top' })
            }
          } catch {
            $q.notify({ type: 'negative', message: 'Error processing QR Code.', position: 'top' })
          }
        },
        () => {},
      )
    } catch {
      $q.notify({
        type: 'negative',
        message: 'Unable to access the camera.',
        position: 'top',
      })
    }
  })
}

const stopScanner = async () => {
  if (html5QrcodeScanner) {
    try {
      if (html5QrcodeScanner.isScanning) {
        await html5QrcodeScanner.stop()
      }
      html5QrcodeScanner.clear()
    } catch {
      // Scanner cleanup errors can be safely ignored.
    }
    html5QrcodeScanner = null
  }
}

const openAddDialog = () => {
  isEditing.value = false
  currentEditId.value = null
  form.value = { nickname: '', id: '' }
  isDialogOpen.value = true
}

const openEditDialog = (contact) => {
  isEditing.value = true
  currentEditId.value = contact.id
  form.value = { nickname: contact.nickname, id: contact.id }
  isDialogOpen.value = true
}

const saveContact = async () => {
  if (!form.value.nickname || !form.value.id) return

  const roomId = await generateRoomHash(myPeerId.value, form.value.id)

  if (isEditing.value) {
    const index = contacts.value.findIndex((c) => c.id === currentEditId.value)
    if (index !== -1) {
      contacts.value[index].nickname = form.value.nickname
      contacts.value[index].id = form.value.id
      contacts.value[index].roomId = roomId
    }
  } else {
    const exists = contacts.value.some((c) => c.id === form.value.id)
    if (exists) {
      $q.notify({ type: 'warning', message: 'This ID has already been added!', position: 'top' })
      return
    }
    contacts.value.push({
      nickname: form.value.nickname,
      id: form.value.id,
      roomId: roomId,
      avatar: '',
    })
  }

  await persistContacts()
  isDialogOpen.value = false
  $q.notify({
    type: 'positive',
    message: isEditing.value ? 'Contact updated!' : 'Contact added!',
    position: 'top',
  })
}

// Flat Confirmation Dialog
const confirmDelete = (contact) => {
  $q.dialog({
    title: 'Delete Contact',
    message: `Are you sure you want to delete "${contact.nickname}"?`,
    dark: true,
    class: 'bg-grey-10 text-white no-shadow',
    ok: {
      label: 'Delete',
      color: 'negative',
      unelevated: true,
      rounded: true,
    },
    cancel: {
      label: 'Cancel',
      color: 'grey-5',
      flat: true,
      rounded: true,
    },
    persistent: true,
  }).onOk(async () => {
    contacts.value = contacts.value.filter((c) => c.id !== contact.id)
    await persistContacts()
    $q.notify({ type: 'negative', message: 'Contact removed.', position: 'top' })
  })
}

onUnmounted(() => stopScanner())
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
