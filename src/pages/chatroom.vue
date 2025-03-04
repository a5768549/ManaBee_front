<template>
    <v-container fluid class="chat-container pa-0">
        <v-row no-gutters>
            <!-- 左側聊天室列表 -->
            <v-col cols="3" class="chat-sidebar">
                <v-card outlined>
                    <v-card-title>聊天室</v-card-title>
                    <v-divider></v-divider>
                    <v-list dense>
                        <v-list-item
                            v-for="room in chatRooms"
                            :key="room.roomId"
                            @click="selectRoom(room)"
                            :class="{ active: room.roomId === currentRoomId }"
                            link
                        >
                            <v-list-item-avatar>
                                <v-img :src="room.avatar || defaultAvatar"></v-img>
                            </v-list-item-avatar>
                            <v-list-item-content>
                                <v-list-item-title>{{ room.roomName }}</v-list-item-title>
                                <v-list-item-subtitle>{{ room.lastMessage || '' }}</v-list-item-subtitle>
                            </v-list-item-content>
                        </v-list-item>
                    </v-list>
                </v-card>
            </v-col>

            <!-- 右側聊天區域 -->
            <v-col cols="9" class="chat-main">
                <v-card class="chat-card" outlined>
                    <v-card-title class="chat-header">
                        <span>{{ currentRoomName || '請選擇聊天室' }}</span>
                    </v-card-title>
                    <v-divider></v-divider>
                    <v-card-text class="chat-messages" ref="chatMessages">
                        <div
                            v-for="(msg, index) in messages"
                            :key="index"
                            :class="msg.senderId === currentUserId ? 'message-sent' : 'message-received'"
                        >
                            <div class="message-content">{{ msg.content }}</div>
                            <div class="message-time">{{ formatTimestamp(msg.timestamp) }}</div>
                        </div>
                    </v-card-text>
                    <v-divider></v-divider>
                    <v-card-actions class="chat-input">
                        <v-text-field
                            v-model="newMessage"
                            label="輸入訊息"
                            outlined
                            dense
                            hide-details
                            class="flex-grow-1"
                            @keyup.enter="sendMessage"
                        ></v-text-field>
                        <v-btn color="primary" @click="sendMessage">發送</v-btn>
                    </v-card-actions>
                </v-card>
            </v-col>
        </v-row>
    </v-container>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue'
import { io } from 'socket.io-client'
import { useUserStore } from '@/stores/user'
import { useRoute, useRouter } from 'vue-router'
import { useAxios } from '@/composables/axios'
import dayjs from 'dayjs'

const { apiAuth } = useAxios()
const userStore = useUserStore()
const router = useRouter()
const route = useRoute()

// 預設頭像
const defaultAvatar = 'https://cdn.vuetifyjs.com/images/cards/docks.jpg'

// 目前使用者ID（假設 userStore.userId 為字串）
const currentUserId = ref(userStore.userId)

// 從網址 query 中取得 targetUserId（代表點選「聯絡他」）
const targetUserId = route.query.userId || ''

// 儲存聊天室列表 (格式必須包含 roomId, roomName, avatar, lastMessage 與 users[ { _id, username } ])
const chatRooms = ref([])
// rooms 傳給聊天區的資料 (此處與 chatRooms 可以一致)
const rooms = ref([])
// 當前聊天室ID與名稱
const currentRoomId = ref(null)
const currentRoomName = ref('')
// 當前聊天室訊息
const messages = ref([])
// 訊息輸入框
const newMessage = ref('')

// Socket.io 連線設定
let socket = io(import.meta.env.VITE_BACKEND_URL)
socket.on('receiveMessage', (msg) => {
    if (msg.roomId !== currentRoomId.value) return
    messages.value.push(msg)
    nextTick(() => scrollToBottom())
})

// 取得當前聊天室訊息
async function fetchMessages(roomId) {
    try {
        const res = await apiAuth.get('/chat/messages', { params: { roomId } })
        if (res.data.success) {
            messages.value = res.data.result
            nextTick(() => scrollToBottom())
        }
    } catch (error) {
        console.error('fetchMessages error:', error)
    }
}

// 切換聊天室：更新當前聊天室，加入 socket 房間，並取得訊息
function selectRoom(room) {
    currentRoomId.value = room.roomId
    currentRoomName.value = room.roomName
    socket.emit('joinRoom', room.roomId)
    fetchMessages(room.roomId)
}

// 發送訊息：先透過 socket 即時傳送，然後用 API 儲存到資料庫
async function sendMessage() {
    if (!newMessage.value.trim() || !currentRoomId.value) return
    const msgObj = {
        roomId: currentRoomId.value,
        content: newMessage.value,
        senderId: currentUserId.value,
        timestamp: Date.now(),
    }
    // 即時發送訊息
    socket.emit('sendMessage', msgObj)
    messages.value.push(msgObj)
    // 呼叫 API 將訊息儲存到資料庫
    try {
        await apiAuth.post('/chat/messages', msgObj)
    } catch (error) {
        console.error('儲存訊息到資料庫失敗:', error)
    }
    newMessage.value = ''
    nextTick(() => scrollToBottom())
}

// 自動捲動訊息區到底部
function scrollToBottom() {
    const chatMessagesEl = document.querySelector('.chat-messages')
    if (chatMessagesEl) {
        chatMessagesEl.scrollTop = chatMessagesEl.scrollHeight
    }
}

// 格式化時間
function formatTimestamp(ts) {
    return dayjs(ts).format('HH:mm')
}

// 取得當前使用者所有聊天室
async function getUserRooms() {
    try {
        const res = await apiAuth.get('/chat/rooms')
        if (res.data.success) {
            // 假設後端回傳資料格式為：{ _id, participants: [id1, id2], lastMessage, partnerName, partnerAvatar }
            const fetchedRooms = res.data.result.map((room) => {
                const partnerId = room.participants.find((id) => id !== currentUserId.value)
                return {
                    roomId: room._id,
                    roomName: room.partnerName || '對話 - ' + partnerId,
                    avatar: room.partnerAvatar || defaultAvatar,
                    lastMessage: room.lastMessage || '',
                    // users 格式符合 vue-advanced-chat 要求：每個使用者物件至少包含 _id 與 username
                    users: [
                        { _id: currentUserId.value, username: userStore.username },
                        { _id: partnerId, username: room.partnerName || '聊天對象' },
                    ],
                }
            })
            chatRooms.value = fetchedRooms
            rooms.value = fetchedRooms
        }
    } catch (error) {
        console.error('getUserRooms error:', error)
    }
}

// 建立或取得與 targetUserId 的聊天室
async function createOrGetRoom() {
    try {
        const res = await apiAuth.post('/chat/room', { targetUserId })
        if (res.data.success) {
            currentRoomId.value = res.data.room._id
            // 假設後端只回傳聊天室的 _id 與 participants，
            // 我們自行組裝符合 vue-advanced-chat 格式的房間物件
            const roomObj = {
                roomId: res.data.room._id,
                roomName: '對話', // 你可以用對方名稱組合，例如 "對話 - XXX"
                avatar: '', // 可根據需要從後端取得
                lastMessage: '',
                users: [
                    { _id: currentUserId.value, username: userStore.username },
                    { _id: targetUserId, username: '聊天對象' },
                ],
            }
            // 將新房間加入列表中
            chatRooms.value = [roomObj, ...chatRooms.value]
            rooms.value = [roomObj, ...rooms.value]
            socket.emit('joinRoom', currentRoomId.value)
            fetchMessages(currentRoomId.value)
        } else {
            console.error('取得房間失敗:', res.data.message)
        }
    } catch (error) {
        console.error('createOrGetRoom error:', error)
    }
}

// onMounted：如果網址帶 targetUserId 則建立或取得對話，否則取得所有聊天室
onMounted(() => {
    if (targetUserId && targetUserId !== '') {
        createOrGetRoom()
    }
    getUserRooms()
})
</script>

<style scoped>
.chat-container {
    height: 100vh;
}
.chat-sidebar {
    border-right: 1px solid #eee;
    padding: 16px;
}
.chat-main {
    padding: 16px;
    height: 100vh;
}
.chat-card {
    display: flex;
    flex-direction: column;
    height: 100%;
}
.chat-header {
    font-size: 20px;
    font-weight: bold;
}
.chat-messages {
    flex: 1;
    overflow-y: auto;
    padding: 16px;
    background-color: #fafafa;
}
.message-sent {
    text-align: right;
    margin-bottom: 12px;
}
.message-received {
    text-align: left;
    margin-bottom: 12px;
}
.message-content {
    display: inline-block;
    padding: 8px 12px;
    border-radius: 16px;
    background-color: #e0e0e0;
}
.message-sent .message-content {
    background-color: #1976d2;
    color: white;
}
.message-time {
    font-size: 12px;
    color: #666;
    margin-top: 4px;
}
.chat-input {
    display: flex;
    align-items: center;
}
</style>
