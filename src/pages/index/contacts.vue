<template>
  <q-page class="q-pa-md">
    <!-- Cabeçalho da Lista -->
    <div class="row items-center justify-between q-mb-md">
      <div class="text-h6 text-weight-bold text-primary">Seus Contatos</div>
      <q-chip outline color="primary" icon="fingerprint">
        Seu ID: {{ myPeerId ? myPeerId.substring(0, 8) + '...' : 'Carregando...' }}
      </q-chip>
    </div>

    <!-- Lista de Contatos -->
    <q-list separator class="bg-white rounded-borders shadow-1">
      <q-item v-for="contact in contacts" :key="contact.id" clickable v-ripple>
        <q-item-section avatar>
          <q-avatar color="primary" text-color="white" icon="person" />
        </q-item-section>

        <q-item-section>
          <q-item-label class="text-weight-bold">{{ contact.nickname }}</q-item-label>
          <q-item-label caption class="text-mono"> ID: {{ contact.id }} </q-item-label>
          <q-item-label caption class="text-mono text-grey-7">
            Sala P2P: {{ contact.roomId }}
          </q-item-label>
        </q-item-section>

        <q-item-section side>
          <div class="row q-gutter-xs">
            <q-btn flat round color="primary" icon="edit" @click="openEditDialog(contact)" />
            <q-btn flat round color="negative" icon="delete" @click="deleteContact(contact.id)" />
          </div>
        </q-item-section>
      </q-item>

      <q-item v-if="contacts.length === 0">
        <q-item-section class="text-center text-grey q-py-md">
          Nenhum contato adicionado ainda.
        </q-item-section>
      </q-item>
    </q-list>

    <!-- Botão Flutuante (FAB) para Adicionar Contato -->
    <q-page-sticky position="bottom-right" :offset="[18, 18]">
      <q-btn fab icon="add" color="primary" @click="openAddDialog" />
    </q-page-sticky>

    <!-- Modal/Diálogo para Adicionar ou Editar Contato -->
    <q-dialog v-model="isDialogOpen" persistent>
      <q-card style="min-width: 320px">
        <q-card-section class="row items-center q-pb-none">
          <div class="text-h6">
            {{ isEditing ? 'Editar Contato' : 'Novo Contato' }}
          </div>
          <q-space />
          <q-btn icon="close" flat round dense v-close-popup />
        </q-card-section>

        <q-card-section class="q-pt-md">
          <q-form @submit="saveContact" class="q-gutter-md">
            <q-input
              v-model="form.nickname"
              label="Apelido / Nickname"
              outlined
              dense
              :rules="[(val) => !!val || 'Por favor digite um apelido']"
            />

            <q-input
              v-model="form.id"
              label="ID do Contato (Peer ID)"
              outlined
              dense
              hint="ID exclusivo gerado pelo Trystero do seu amigo"
              :rules="[(val) => !!val || 'Por favor informe o ID']"
            />

            <div class="row justify-end q-mt-md">
              <q-btn label="Cancelar" flat color="grey" v-close-popup />
              <q-btn
                :label="isEditing ? 'Atualizar' : 'Salvar'"
                type="submit"
                color="primary"
                class="q-ml-sm"
              />
            </div>
          </q-form>
        </q-card-section>
      </q-card>
    </q-dialog>
  </q-page>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { useQuasar } from 'quasar'
import localforage from 'localforage'

const $q = useQuasar()

const myPeerId = ref('')
const contacts = ref([])

const isDialogOpen = ref(false)
const isEditing = ref(false)
const currentEditId = ref(null)

const form = ref({
  nickname: '',
  id: '',
})

onMounted(async () => {
  localforage.config({
    name: 'WhisperP2P',
    storeName: 'contacts_store',
  })

  let savedMyId = await localforage.getItem('myPeerId')
  if (!savedMyId) {
    savedMyId = 'my_peer_' + Math.random().toString(36).substring(2, 10)
    await localforage.setItem('myPeerId', savedMyId)
  }
  myPeerId.value = savedMyId

  const savedContacts = await localforage.getItem('contacts')
  if (savedContacts) {
    contacts.value = savedContacts
  }
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

// CORREÇÃO: Converte o Array reativo do Vue em objeto simples antes de salvar
const persistContacts = async () => {
  const plainContacts = JSON.parse(JSON.stringify(contacts.value))
  await localforage.setItem('contacts', plainContacts)
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

  const contactData = {
    nickname: form.value.nickname,
    id: form.value.id,
    roomId: roomId,
  }

  if (isEditing.value) {
    const index = contacts.value.findIndex((c) => c.id === currentEditId.value)
    if (index !== -1) {
      contacts.value[index] = contactData
    }
  } else {
    const exists = contacts.value.some((c) => c.id === form.value.id)
    if (exists) {
      $q.notify({
        type: 'warning',
        message: 'Este ID de contato já foi adicionado!',
        position: 'top',
      })
      return
    }
    contacts.value.push(contactData)
  }

  await persistContacts()
  isDialogOpen.value = false

  $q.notify({
    type: 'positive',
    message: isEditing.value ? 'Contato atualizado!' : 'Contato adicionado!',
    position: 'top',
  })
}

const deleteContact = async (id) => {
  contacts.value = contacts.value.filter((c) => c.id !== id)
  await persistContacts()

  $q.notify({
    type: 'negative',
    message: 'Contato removido.',
    position: 'top',
  })
}
</script>

<style scoped>
.text-mono {
  font-family: monospace;
}
</style>
