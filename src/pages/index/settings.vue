<template>
  <q-page class="q-pa-md">
    <!-- Título da Página -->
    <div class="row items-center justify-between q-mb-md">
      <div class="text-h6 text-weight-bold text-primary">Configurações do Perfil</div>
    </div>

    <q-card flat bordered class="q-pa-sm">
      <!-- Avatar com Inicial -->
      <q-card-section class="text-center">
        <q-avatar size="80px" color="primary" text-color="white" class="q-mb-sm text-h4">
          {{ nickname ? nickname.charAt(0).toUpperCase() : 'U' }}
        </q-avatar>
        <div class="text-subtitle1 text-weight-bold">{{ nickname || 'Sem Nome' }}</div>
      </q-card-section>

      <q-card-section class="q-gutter-y-md">
        <!-- Campo: Nickname -->
        <q-input
          v-model="nickname"
          label="Seu Nickname"
          outlined
          dense
          hint="Nome de exibição visível para os seus contactos"
        >
          <template #prepend>
            <q-icon name="person" />
          </template>
        </q-input>

        <!-- Campo: ID / Hash P2P -->
        <q-input
          v-model="peerId"
          label="Seu ID P2P (Peer ID)"
          outlined
          dense
          readonly
          hint="ID exclusivo usado pelo Trystero para o encontrar"
        >
          <template #prepend>
            <q-icon name="fingerprint" />
          </template>

          <template #append>
            <q-btn flat round icon="content_copy" @click="copyId" size="sm">
              <q-tooltip>Copiar ID</q-tooltip>
            </q-btn>
            <q-btn flat round icon="refresh" color="warning" @click="generateNewId" size="sm">
              <q-tooltip>Gerar Novo ID</q-tooltip>
            </q-btn>
          </template>
        </q-input>

        <!-- Botão Salvar -->
        <div class="row justify-end q-mt-lg">
          <q-btn label="Salvar Alterações" color="primary" icon="save" @click="saveSettings" />
        </div>
      </q-card-section>
    </q-card>
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

// Carrega os dados salvos no localforage
onMounted(async () => {
  localforage.config({
    name: 'WhisperP2P',
    storeName: 'contacts_store',
  })

  // Carrega o nickname salvo
  const savedNickname = await localforage.getItem('myNickname')
  nickname.value = savedNickname || 'Utilizador_' + Math.floor(Math.random() * 1000)

  // Carrega ou gera o próprio Peer ID
  let savedMyId = await localforage.getItem('myPeerId')
  if (!savedMyId) {
    savedMyId = selfId || 'peer_' + Math.random().toString(36).substring(2, 10)
    await localforage.setItem('myPeerId', savedMyId)
  }
  peerId.value = savedMyId
})

// Gerar novo Hash/ID aleatório e guardar no localforage
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
    message: 'Novo ID gerado e guardado com sucesso!',
    icon: 'refresh',
    position: 'top',
    timeout: 1500,
  })
}

// Copiar ID para a área de transferência
const copyId = () => {
  if (!peerId.value) return
  navigator.clipboard.writeText(peerId.value)
  $q.notify({
    type: 'positive',
    message: 'ID copiado para a área de transferência!',
    icon: 'content_copy',
    position: 'top',
    timeout: 1500,
  })
}

// Salvar dados no localforage
const saveSettings = async () => {
  await localforage.setItem('myNickname', nickname.value)
  await localforage.setItem('myPeerId', peerId.value)

  $q.notify({
    type: 'positive',
    message: 'Configurações guardadas!',
    icon: 'check_circle',
    position: 'top',
    timeout: 1500,
  })
}
</script>
