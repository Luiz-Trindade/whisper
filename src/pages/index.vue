<template>
  <q-layout view="lHh Lpr lFf">
    <!-- Cabeçalho -->
    <q-header elevated class="bg-primary text-white">
      <q-toolbar>
        <q-toolbar-title class="text-weight-bold"> Whisper </q-toolbar-title>
      </q-toolbar>
    </q-header>

    <q-page-container>
      <router-view />
    </q-page-container>

    <!-- Rodapé exibido apenas se hideFooter for falso -->
    <q-footer v-if="!hideFooter" elevated class="bg-primary text-white">
      <q-tabs align="justify" active-color="yellow" indicator-color="transparent">
        <q-route-tab to="/" name="chats" icon="chat" label="Chats" />
        <q-route-tab to="/contacts" name="contacts" icon="people" label="Contacts" />
        <q-route-tab to="/settings" name="settings" icon="settings" label="Settings" />
      </q-tabs>
    </q-footer>
  </q-layout>
</template>

<script setup>
import { ref, provide } from 'vue'

// Estado global para controlar a visibilidade das abas do rodapé
const hideFooter = ref(false)
provide('hideFooter', hideFooter)
</script>
