<template>
    <v-container fluid :class="smAndDown ? 'pa-0' : ''">
        <v-card flat class="w-100">
            <v-card-title>
                <div class="d-flex align-center justify-center">
                    <div class="text-h6 text-center">{{ user?.username }}</div>
                    <v-btn icon="share" variant="text" size="small" @click="dialog = true" :ripple="false" />
                </div>
            </v-card-title>
            <v-card-subtitle class="pa-0 mt-n4" :class="smAndDown ? 'text-center' : 'text-end'">
                <v-btn class="text-caption" :href="`${repo}/fork`" target="_blank" rel="noopener" text="update this index" :variant="smAndDown ? 'text' : 'outlined'" size="small" :append-icon="GitHubIcon" />
            </v-card-subtitle>
        </v-card>
        <v-list>
            <v-list-item :id="`u.${user.username} ${link.uuid}`" v-for="(link, i) in socialLinks" :key="i" :href="link.url" :title="link.title || link.url" :subtitle="link.subtitle" @mouseenter="hovered[i] = true" @mouseleave="hovered[i] = false" @focus="hovered[i] = true" @blur="hovered[i] = false" tabindex="0" :class="smAndDown ? 'pa-0' : ''">
                <template v-slot:prepend>
                    <component v-if="link.icon" class="mr-8" :is="link.icon" />
                    <v-img style="border-radius: 25%" v-else-if="link.favicon" :src="link.favicon || link.svg" class="mr-2" width="24" height="24" />
                    <v-icon v-else-if="/mailto:[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}(\?[a-zA-Z0-9=&%+-]*)?/.test(link.url)" icon="mail" size="24"></v-icon>
                    <v-icon v-else icon="link" size="24"></v-icon>
                </template>
                <template v-slot:title="{ title }">
                    <span :class="smAndDown ? 'text-body-2' : ''">{{ title }}</span>
                </template>
                <template v-slot:append>
                    <flip-board ref="flip" class="pa-0" title="Redirecting" v-model="timer" :paused="hovered[i] || dialog" :timeout="link.redirect.timeout" v-if="link.redirect?.timeout && (timer === undefined || timer > -1)" :class="timer < 1 ? 'animate__animated animate__fadeOut' : ''" />
                    <flip-board-clicks :ref="`flip-clicks-${i}`" class="pa-0 animate__animated animate__fadeIn" tooltip="clicks" :modelValue="link.clicks" v-if="link.clicks" />
                </template>
            </v-list-item>
        </v-list>
        <v-dialog v-model="dialog" class="d-flex justify-center align-center mx-1" @close="dialog = false" max-width="500" width="100%">
            <v-card rounded="xl" class="my-1 pb-3">
                <v-card-title class="d-flex align-center text-body-1 font-weight-bold">QR Code
                    <v-spacer />
                    <v-icon icon="close" size="x-small" @click="dialog = false"></v-icon>
                </v-card-title>
                <v-divider />
                <v-card-text>
                    <div class="d-flex align-center justify-center" style="width: 250px">
                        <v-img :src="qrcode"></v-img>
                        <v-img src="/ai.logo.svg" width="35" class="qr-logo"></v-img>
                    </div>
                </v-card-text>
                <v-divider />
                <v-card-actions>
                    <v-btn variant="outlined" rounded size="small" @click="downloadHandler">download image</v-btn>
                    <v-btn variant="tonal" rounded size="small" class="mr-4" @click="copyHandler('qrcode')">copy to clipboard
                        <v-tooltip v-model="tooltips['qrcode']" class="justify-center align-center" attach location="top">
                            <span class="text-caption font-weight-bold">Copied</span>
                        </v-tooltip>
                    </v-btn>
                </v-card-actions>
            </v-card>
            <v-card rounded="xl" class="my-1 pb-3">
                <v-card-title class="d-flex align-center text-body-1 font-weight-bold">Binary</v-card-title>
                <v-card-subtitle>
                    {{ binary }}
                </v-card-subtitle>
                <v-card-actions>
                    <v-btn variant="tonal" rounded size="small" @click="copyHandler('binary')" class="mr-4">copy to clipboard
                        <v-tooltip v-model="tooltips['binary']" class="justify-center align-center" attach location="top">
                            <span class="text-caption font-weight-bold">Copied</span>
                        </v-tooltip>
                    </v-btn>
                </v-card-actions>
            </v-card>
            <v-card rounded="xl" class="my-1 pb-3">
                <v-card-title class="d-flex align-center text-body-1 font-weight-bold">URL</v-card-title>
                <v-card-subtitle>
                    {{ qcLink }}
                </v-card-subtitle>
                <v-card-actions>
                    <v-btn variant="tonal" rounded size="small" @click="copyHandler('link')" class="mr-4">copy to clipboard
                        <v-tooltip v-model="tooltips['link']" class="justify-center align-center" attach location="top">
                            <span class="text-caption font-weight-bold">Copied</span>
                        </v-tooltip>
                    </v-btn>
                </v-card-actions>
            </v-card>
        </v-dialog>
    </v-container>
</template>
<style scoped>
.qr-logo {
    position: absolute;
    border-radius: 25%;
    background: white;
    opacity: 0.95;
}
</style>
<script setup>
import 'animate.css'
import QRCode from 'qrcode'
import { ref, computed, inject, onMounted, watch, getCurrentInstance, nextTick } from 'vue'
import { useDisplay } from 'vuetify/lib/framework.mjs'
import { GitHubIcon } from 'vue3-simple-icons'
import { v5 as uuidv5 } from 'uuid'

import FlipBoard from '@/components/FlipBoard.vue'
import FlipBoardClicks from '@/components/FlipBoardClicks.vue'

const analytics = ref()
const clipboard = inject('clipboard')
const { $api, $keycloak, $getRepoForName } = getCurrentInstance().appContext.config.globalProperties
const binary = computed(() => qcLink.value?.split('').map(x => x.charCodeAt(0).toString(2)).join(' '))
const dialog = ref(false)
const { smAndDown } = useDisplay()
const props = defineProps({
    user: Object,
    auth: Object
})
const tooltips = ref({
    binary: false,
    qrcode: false,
    link: false
})
const links = computed(() => props.user?.links)
const qcLink = computed(() => `${window.location.origin}/u/${props.user.username}`)
const flip = ref()
const timer = ref()
const parseSocialLinks = inject('parseSocialLinks')

const hovered = ref(links.value?.reduce((acc, cur, i) => ({ ...acc, [i]: false }), {}) || {})
const socialLinks = computed(() => parseSocialLinks(links.value)?.map(link => {
    if (!analytics.value) return link
    link.uuid = uuidv5(link.url, uuidv5.URL)
    const matchingLinkClickAnalytics = analytics.value?.linkClicks && Object.entries(analytics.value.linkClicks).find(([key, value]) => link.uuid.includes(key))?.[1]

    if (matchingLinkClickAnalytics) {
        link.clicks = Number(matchingLinkClickAnalytics.count)
    }
    return link
}).sort((a, b) => !a?.clicks || a.clicks > b.clicks ? 1 : -1))
const qrcode = ref()
const repo = $getRepoForName(props.user.username)

// console.log(socialLinks.value)
function copyDataUrlToClipboard(dataUrl) {
    const byteString = atob(dataUrl.split(',')[1])
    const mimeString = dataUrl.split(',')[0].split(':')[1].split(';')[0]
    const ab = new ArrayBuffer(byteString.length)
    const ia = new Uint8Array(ab)

    for (let i = 0; i < byteString.length; i++) {
        ia[i] = byteString.charCodeAt(i)
    }

    clipboard.copy(new Blob([ab], { type: mimeString }))
}

function copyHandler(name) {
    tooltips.value[name] = true
    if (name === 'qrcode') {
        copyDataUrlToClipboard(qrcode.value)
    } else if (name === 'binary') {
        clipboard.copy(binary.value)
    } else if (name === 'link') {
        clipboard.copy(qcLink.value)
    }
    setTimeout(() => tooltips.value[name] = false, 1500)
    gtag('event', 'link_copy', { method: name })
}

async function asyncInit() {
    qrcode.value = await QRCode.toDataURL(qcLink.value, { type: 'image/webp' })
    await $keycloak.value.isLoaded
    if ($keycloak.value.isAuthenticated) {
        nextTick(async () => analytics.value = await $api.analytics(props.auth, props.user.username))
    }
}
function downloadHandler() {
    const link = document.createElement('a')
    link.href = qrcode.value
    link.download = `${window.location.hostname}-${props.user.username}.webp`
    link.click()
    gtag('event', 'link_download')
}

asyncInit()
onMounted(() => {
    watch(dialog, dialog => {
        if (!dialog) return
        if (!qrcode.value) {
            asyncInit()
        }
        gtag('event', 'qc_dialog')
    })
    watch(timer, timer => {
        if (timer !== undefined && timer < 0) {
            window.location.href = links.value.find(link => link.redirect).url
        }
    })
})
</script>