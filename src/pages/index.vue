<template>
    <q-layout view="lHh Lpr lFf" class="bg-dark text-white">
        <!-- ================= PAGE CONTAINER ================= -->
        <q-page-container class="page-container-wrapper bg-dark">
            <router-view #default="{ Component }">
                <transition :name="transitionName" mode="default">
                    <component :is="Component" :key="route.path" />
                </transition>
            </router-view>
        </q-page-container>

        <!-- ================= BOTTOM NAVIGATION ================= -->
        <q-footer v-if="!hideFooter" class="bg-grey-10 text-grey-6">
            <q-tabs
                align="justify"
                active-color="primary"
                indicator-color="transparent"
                class="text-weight-bold"
                no-caps
                dense
            >
                <q-route-tab to="/" name="chats" icon="chat" label="Chats" exact />
                <q-route-tab to="/contacts" name="contacts" icon="people" label="Contacts" exact />
                <q-route-tab
                    to="/settings"
                    name="settings"
                    icon="settings"
                    label="Settings"
                    exact
                />
            </q-tabs>
        </q-footer>
    </q-layout>
</template>

<script setup>
import { ref, provide, watch } from 'vue'
import { useRoute } from 'vue-router'

const route = useRoute()

// Global state to hide footer in open chat screen
const hideFooter = ref(false)
provide('hideFooter', hideFooter)

// --- ROUTE ORDER FOR PAGE ANIMATION ---
const routeGroups = {
    '/': 0,
    '/contacts': 1,
    '/settings': 2,
}

const transitionName = ref('q-transition--fade')

watch(
    () => route.path,
    (toPath, fromPath) => {
        const toIndex = routeGroups[toPath] !== undefined ? routeGroups[toPath] : 0
        const fromIndex = routeGroups[fromPath] !== undefined ? routeGroups[fromPath] : 0

        if (toIndex === fromIndex) {
            transitionName.value = 'q-transition--fade'
        } else if (toIndex > fromIndex) {
            transitionName.value = 'q-transition--slide-left'
        } else {
            transitionName.value = 'q-transition--slide-right'
        }
    },
)
</script>

<style scoped>
.page-container-wrapper {
    overflow-x: hidden;
    position: relative;
}

:deep(.q-tab--active) {
    color: var(--q-primary) !important;
}

/* Fade Transition */
.q-transition--fade-enter-active,
.q-transition--fade-leave-active {
    transition: opacity 0.2s ease;
}
.q-transition--fade-enter-from,
.q-transition--fade-leave-to {
    opacity: 0;
}

/* Slide Left / Right Transitions */
.q-transition--slide-left-enter-active,
.q-transition--slide-left-leave-active,
.q-transition--slide-right-enter-active,
.q-transition--slide-right-leave-active {
    transition:
        transform 0.25s cubic-bezier(0.4, 0, 0.2, 1),
        opacity 0.25s ease;
}
.q-transition--slide-left-enter-from {
    transform: translateX(100%);
    opacity: 0;
}
.q-transition--slide-left-leave-to {
    transform: translateX(-20%);
    opacity: 0;
}
.q-transition--slide-right-enter-from {
    transform: translateX(-100%);
    opacity: 0;
}
.q-transition--slide-right-leave-to {
    transform: translateX(20%);
    opacity: 0;
}
</style>
