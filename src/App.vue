<template>
  <router-view />
</template>

<script setup>
import { useQuasar } from 'quasar'

const $q = useQuasar()
$q.dark.set(true) // Ativa o modo dark na inicialização
</script>
