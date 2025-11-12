<template>
    <v-card class="rounded-lg" max-width="500px">
        <template #text>
            <v-responsive :aspect-ratio="3 / 4" width="450px" height="600px">
                <video ref="videoRef" class="h-100 w-100" autoplay="true" playsinline="true"></video>
            </v-responsive>
            <audio ref="audioRef" autoplay="true"></audio>
        </template>
        <template #actions>
            <App class="w-100">
                <Sender v-model:value="message" submitType="shiftEnter"
                    placeholder="Press Shift + Enter to send message" class="w-100" @submit="ai" />
            </App>
        </template>
    </v-card>

    <!-- <v-card min-width="400px" title="Settings">
                <template #text>
                    <v-checkbox v-if="false" v-model="useStun" label="Use STUN server"></v-checkbox>

                    <div class="d-flex ga-2 mb-4">
                        <v-btn :disabled="starting" :loading="loading" loading
                            @click="exchange_signaling()">Start</v-btn>
                        <v-btn :disabled="!starting" @click="stop()">Stop</v-btn>
                    </div>

                    <p class="text-subtitle-2 my-2">Input Message</p>

                    <v-textarea v-model="message" label="Message"></v-textarea>

                    <v-btn :disabled="message.length == 0" @click="general">General Send</v-btn>

                    <v-btn :disabled="message.length == 0" class="ml-2" @click="ai">AI Chat Send</v-btn>

                    <v-btn class="ml-2" @click="prefabricated">历史的魅力</v-btn>
                </template>
            </v-card> -->


</template>

<script setup>
import { onMounted, onUnmounted, ref, watch } from 'vue';
import { useWebSocket } from '@vueuse/core'
import { App } from 'ant-design-vue';
import { Sender } from 'ant-design-x-vue';
import { useRoute } from 'vue-router'
import { useLoadingStore } from '@/stores/loading';
import spacegt from 'spacegt';

const emit = defineEmits(['loadImg'])

const loadingStore = useLoadingStore()

const route = useRoute()

const authorizationStore = spacegt.stores.useAuthorizationStore()

const webSocket = ref({})

const starting = ref(false)

const loading = ref(false)

const useStun = ref(false)

const sessionid = ref(0)

const message = ref("")

var config = { sdpSemantics: 'unified-plan' };

var pc = null;

const audioRef = ref()
const videoRef = ref()

watch(() => webSocket.value.data, (value) => {
    try {
        const data = JSON.parse(value)

        if (data.event_type == 'ai_chat' && data.type == 'answer') {
            console.log(data.message)
        }
        if (data.event_type == 'message' && data.type == 'answer') {
            console.log(data.message)
            if (data.image) {
                emit('loadImg', data.image)
            }
        }
    } catch (e) { }
})

function stop() {
    starting.value = false
    pc.close()
}

const general = () => {
    sendWait({
        event_type: 'content_offer',
        messages: [{ text: message.value, interrupt: false, type: 'echo' }],
    })

    message.value = ''
}

const prefabricated = () => {
    console.log(route)
    sendWait({
        event_type: 'content_offer',
        course: route.params.chapterid,
        // or
        // chapter: route.params.chapterid,
    })
}

const ai = () => {
    sendWait({
        event_type: 'ai_chat',
        message: message.value,
        interrupt: false,
        type: 'echo'
    })

    message.value = ''
}

// 会话初始化
const initialization = async () => {
    await sendWait({ event_type: 'initialization' }, true)
}

// 交换信令,开启直播
const exchange_signaling = async () => {
    loading.value = true

    if (useStun.value) config.iceServers = [{ urls: ['stun:stun.l.google.com:19302'] }];

    pc = new RTCPeerConnection(config);

    // connect audio / video
    pc.addEventListener('track', (evt) => {
        if (evt.track.kind == 'video') {
            videoRef.value.srcObject = evt.streams[0];
        } else {
            audioRef.value.srcObject = evt.streams[0];
        }
    });

    pc.addTransceiver('video', { direction: 'recvonly' });
    pc.addTransceiver('audio', { direction: 'recvonly' });

    pc.createOffer().then((offer) => {
        return pc.setLocalDescription(offer);
    }).then(() => {
        // wait for ICE gathering to complete
        return new Promise((resolve) => {
            if (pc.iceGatheringState === 'complete') {
                resolve();
            } else {
                const checkState = () => {
                    if (pc.iceGatheringState === 'complete') {
                        pc.removeEventListener('icegatheringstatechange', checkState);
                        resolve();
                    }
                };
                pc.addEventListener('icegatheringstatechange', checkState);
            }
        });
    }).then(() => {
        var offer = pc.localDescription;
        return sendWait({
            event_type: 'exchange_signaling',
            sdp: offer.sdp,
            type: offer.type,
        })
    }).then((answer) => {
        sessionid.value = answer.sessionid
        loading.value = false
        starting.value = true
        pc.setRemoteDescription(answer);

        prefabricated()
        loadingStore.loadCompleted()
    }).catch((e) => {
        console.error(e);
    });
}

// 用来做等待队列
const messageQueue = []
const sendWait = async (data, wait = true) => {
    webSocket.value.send(JSON.stringify(data))

    if (wait) {
        const timeout = 10000
        return new Promise((resolve, reject) => {
            let count = 0
            const intervalId = setInterval(() => {
                const wsdata = messageQueue.pop()
                if (wsdata) {
                    if (wsdata.type == 'answer' && data.event_type == wsdata.event_type) {
                        clearInterval(intervalId)
                        resolve(wsdata)
                    }
                }
                count += 100
                if (count > timeout) {
                    clearInterval(intervalId)
                    reject()
                }
            }, 100)
        })
    }
}

// close peer connection
const closeAll = () => {
    setTimeout(() => {
        webSocket.value.close()
        pc.close()
    }, 500);
}

loadingStore.loadStart()

onMounted(async () => {
    webSocket.value = useWebSocket(import.meta.env.VITE_APP_AI_LIVE_SERVICE_WSS + '/api/session?token=' + authorizationStore.token,
        {
            onMessage(ws, event) {
                try {
                    messageQueue.push(JSON.parse(event.data))
                } catch (e) { console.error(e) }
            }
        }
    )

    await initialization()

    exchange_signaling()

    // 页面离开时关闭连接
    window.onbeforeunload = function (e) {
        closeAll()

        e = e || window.event
        // 兼容IE8和Firefox 4之前的版本
        if (e) {
            e.returnValue = '关闭提示'
        }
        // Chrome, Safari, Firefox 4+, Opera 12+ , IE 9+
        return '关闭提示'
    }

    window.onunload = function (event) {
        // 在这里执行你想要的操作
        closeAll()
    };
})

onUnmounted(() => {
    closeAll()
})

</script>

<style scoped>
:deep(.ant-btn) {
    display: flex;
    justify-content: center;
    align-items: center;
    color: #fff;
}
</style>
