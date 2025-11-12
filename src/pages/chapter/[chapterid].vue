<template>
    <v-app-bar :elevation="2">
        <template v-slot:prepend>
            <v-btn icon="mdi-arrow-left" class="ml-2" @click="router.back()"></v-btn>
        </template>

        <v-app-bar-title>{{ chapter.title }}</v-app-bar-title>
    </v-app-bar>

    <AuroraBackground v-if="chapter && videos.length > 0 && images.length > 0">
        <LetterPullup v-if="chapter.title" :words="chapter.title" :delay="0.05" class="text-black dark:text-white" />
        <v-container class="my-8" max-width="90%" min-width="1280px" style=" align-content: center;">
            <div class="d-flex ga-8 justify-center">
                <s-avatar class="flex-1-0" :data="videos" :current="current" @ended="nextPPT()"></s-avatar>
                <s-docs ref="docsRef" :data="images" :current="current" @change="handlePPTChange"></s-docs>
            </div>
        </v-container>
    </AuroraBackground>

    <v-empty-state v-else-if="!loading"
        image="https://cdn.vuetifyjs.com/docs/images/components/v-empty-state/teamwork.png">
        <template v-slot:title>
            <div class="text-subtitle-2 mt-8">
                没有相应的课程内容
            </div>
        </template>

        <template v-slot:actions>
            <v-btn class="text-none" prepend-icon="mdi-arrow-left" elevation="1" rounded="lg" size="small" text="back"
                width="96" @click="router.back()"></v-btn>
        </template>
    </v-empty-state>
    <Loading></Loading>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue';
import SAvatar from './avatar_ai.vue'
import SDocs from './docs.vue'
import { useRoute, useRouter } from 'vue-router'

import { PPTApi } from '@/api/ppt';
import { ChapterApi } from '@/api/chapter';

const route = useRoute()

const router = useRouter()

const docsRef = ref()

const chapter = ref({})

const ppts = ref([])

const images = computed(() => ppts.value.map(item => item.imagePath))

const videos = computed(() => ppts.value.map(item => item.videoPath))

const current = ref(0)

const loading = ref(true)

const nextPPT = () => {
    if (current.value + 1 < images.value.length) {
        current.value++
    } else {
        current.value = 0
    }
}

const loadChapter = async () => {
    chapter.value = await ChapterApi.info(route.params.chapterid)
}

const loadItmes = async () => {
    loading.value = true

    ppts.value = await PPTApi.list(route.params.chapterid)

    loading.value = false
}

const handlePPTChange = (index) => {
    current.value = index
}

onMounted(() => {
    loadChapter()
    loadItmes()
})
</script>

<style lang="scss" scoped></style>