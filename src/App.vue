<template>
  <div class="container">
    <div class="header">
      <h1>💬 채팅 테스트 클라이언트</h1>
      <p>Spring Boot 채팅 서버 테스트용 프론트엔드</p>
    </div>

    <div class="login-section">
      <div class="login-form">
        <label>서버 URL:</label>
        <input 
          v-model="serverUrl" 
          type="text" 
          placeholder="http://localhost:8080" 
          style="width: 200px;"
        >
        <label>WebSocket Endpoint:</label>
        <select v-model="wsEndpoint" style="width: 150px;">
          <option value="/connect">/connect</option>
          <option value="/ws">/ws</option>
          <option value="/stomp">/stomp</option>
          <option value="/websocket">/websocket</option>
        </select>
        <label>이메일:</label>
        <input 
          v-model="email" 
          type="email" 
          placeholder="test@example.com" 
          style="width: 180px;"
        >
        <label>비밀번호:</label>
        <input 
          v-model="password" 
          type="password" 
          placeholder="password" 
          style="width: 150px;"
          @keypress.enter="login"
        >
        <button @click="login">로그인</button>
        <button @click="showSignupModal = true" class="btn-signup">회원가입</button>
        <button @click="disconnect" class="btn-logout">로그아웃</button>
      </div>
      <div :class="['status', isConnected ? 'connected' : 'disconnected']">
        {{ statusText }}
      </div>
    </div>

    <div class="main-content">
      <div class="sidebar">
        <div class="room-controls">
          <h3>새 채팅방</h3>
          <input 
            v-model="otherMemberId" 
            type="number" 
            placeholder="상대방 ID"
          >
          <button @click="createPrivateRoom">1:1 채팅방 만들기</button>
          <button @click="loadRooms" class="btn-secondary">채팅방 목록 새로고침</button>
        </div>
        <div class="room-list">
          <div v-if="rooms.length === 0" class="empty-state">
            {{ isConnected ? '채팅방이 없습니다' : '로그인 후 채팅방을 만들어보세요' }}
          </div>
          <div 
            v-for="room in rooms" 
            :key="room.roomId"
            :class="['room-item', { active: room.roomId === currentRoomId }]"
            @click="selectRoom(room)"
          >
            <div class="room-header">
              <h4>채팅방 {{ room.roomId }}</h4>
              <span v-if="room.unreadCount > 0" class="unread-badge">
                {{ room.unreadCount }}
              </span>
            </div>
            <p class="room-type">Room ID: {{ room.roomId }} ({{ getRoomTypeLabel(room.type) }})</p>
            <p :class="['room-preview', { empty: !room.lastMessagePreview }]">
              {{ room.lastMessagePreview || '메시지가 없습니다' }}
            </p>
            <p v-if="room.lastMessageAt" class="room-time">
              {{ formatLastMessageTime(room.lastMessageAt) }}
            </p>
          </div>
        </div>
      </div>

      <div class="chat-area">
        <div v-if="!currentRoomId" class="no-room-selected">
          채팅방을 선택해주세요
        </div>
        <template v-else>
          <div class="chat-header">
            <h3>{{ currentRoomName }}</h3>
            <button @click="leaveRoom" class="btn-danger">나가기</button>
          </div>
          <div class="messages" ref="messagesContainer" @scroll="handleScroll">
            <div v-if="isLoadingMore" class="loading-indicator">
              <div class="spinner"></div>
              <span>이전 메시지 불러오는 중...</span>
            </div>
            
            <div v-if="!hasMoreMessages && messages.length > 0" class="no-more-messages">
              처음 메시지입니다
            </div>
            
            <div 
              v-for="(message, index) in messages" 
              :key="message.seq || index"
              :class="['message', message.senderId == currentMemberId ? 'mine' : 'others']"
            >
              <div v-if="message.senderId != currentMemberId" class="message-sender">
                사용자 {{ message.senderId }}
              </div>
              <div class="message-content">
                <div v-if="getMessageType(message) === 'TEXT'" class="message-text">
                  {{ message.content || '(내용 없음)' }}
                </div>
                
                <div v-else-if="getMessageType(message) === 'IMAGE'" class="message-image">
                  <img 
                    v-if="message.fileUrl" 
                    :src="message.fileUrl" 
                    :alt="message.fileName || '이미지'" 
                    @click="openImageModal(message.fileUrl)"
                    @error="handleImageError"
                  >
                  <div v-else class="error-message">❌ 이미지 URL이 없습니다</div>
                  <div v-if="message.content" class="image-caption">{{ message.content }}</div>
                </div>
                
                <div v-else-if="getMessageType(message) === 'FILE'" class="message-file">
                  <div class="file-icon">📄</div>
                  <div class="file-info">
                    <div class="file-name">{{ message.fileName || '파일명 없음' }}</div>
                    <div class="file-size">{{ formatFileSize(message.fileSize) }}</div>
                  </div>
                  <a v-if="message.fileUrl" :href="message.fileUrl" target="_blank" class="file-download">다운로드</a>
                  <span v-else class="error-message">❌ URL 없음</span>
                </div>
                
                <div v-else-if="getMessageType(message) === 'VIDEO'" class="message-video">
                  <video v-if="message.fileUrl" controls :src="message.fileUrl" class="video-player"></video>
                  <div v-else class="error-message">❌ 비디오 URL이 없습니다</div>
                  <div v-if="message.content" class="video-caption">{{ message.content }}</div>
                </div>
                
                <div v-else-if="getMessageType(message) === 'AUDIO'" class="message-audio">
                  <div class="audio-icon">🎵</div>
                  <audio v-if="message.fileUrl" controls :src="message.fileUrl" class="audio-player"></audio>
                  <div v-else class="error-message">❌ 오디오 URL이 없습니다</div>
                </div>
                
                <div v-else-if="getMessageType(message) === 'SYSTEM'" class="message-system">
                  {{ message.content || '(시스템 메시지)' }}
                </div>
                
                <div v-else class="message-unknown">
                  ⚠️ 알 수 없는 메시지 타입: {{ getMessageType(message) }}
                </div>
                
                <div class="message-time">{{ formatTime(message.timestamp) }}</div>
              </div>
            </div>
          </div>
          <div class="message-input">
            <div class="attachment-buttons">
              <button @click="triggerFileInput('image')" class="btn-attachment" title="이미지">
                🖼️
              </button>
              <button @click="triggerFileInput('file')" class="btn-attachment" title="파일">
                📎
              </button>
              <button @click="triggerFileInput('video')" class="btn-attachment" title="비디오">
                🎬
              </button>
              <button @click="triggerFileInput('audio')" class="btn-attachment" title="오디오">
                🎵
              </button>
            </div>
            <input 
              v-model="messageInput" 
              type="text" 
              placeholder="메시지를 입력하세요..." 
              @keypress.enter="sendMessage"
            >
            <button @click="sendMessage">전송</button>
            
            <input 
              ref="imageInput"
              type="file" 
              accept="image/*" 
              @change="handleFileSelect"
              style="display: none;"
            >
            <input 
              ref="fileInput"
              type="file" 
              @change="handleFileSelect"
              style="display: none;"
            >
            <input 
              ref="videoInput"
              type="file" 
              accept="video/*" 
              @change="handleFileSelect"
              style="display: none;"
            >
            <input 
              ref="audioInput"
              type="file" 
              accept="audio/*" 
              @change="handleFileSelect"
              style="display: none;"
            >
          </div>
          
          <div v-if="uploadProgress > 0" class="upload-progress">
            <div class="progress-bar">
              <div class="progress-fill" :style="{ width: uploadProgress + '%' }"></div>
            </div>
            <div class="progress-text">업로드 중... {{ uploadProgress }}%</div>
          </div>
        </template>
      </div>
    </div>
  </div>

  <!-- 회원가입 모달 -->
  <div :class="['modal', { show: showSignupModal }]" @click.self="showSignupModal = false">
    <div class="modal-content">
      <div class="modal-header">
        <h2>회원가입</h2>
        <span class="close" @click="showSignupModal = false">&times;</span>
      </div>
      <div class="modal-body">
        <div class="form-group">
          <label>회원 유형</label>
          <select v-model="signupForm.type">
            <option value="individual">일반 회원</option>
            <option value="company">기업 회원</option>
          </select>
        </div>
        <div v-if="signupForm.type === 'company'" class="form-group">
          <label>기업 ID</label>
          <input v-model="signupForm.companyId" type="number" placeholder="기업 ID를 입력하세요">
        </div>
        <div class="form-group">
          <label>이름</label>
          <input v-model="signupForm.name" type="text" placeholder="홍길동">
        </div>
        <div class="form-group">
          <label>이메일</label>
          <input v-model="signupForm.email" type="email" placeholder="test@example.com">
        </div>
        <div class="form-group">
          <label>비밀번호</label>
          <input v-model="signupForm.password" type="password" placeholder="비밀밀번호">
        </div>
        <div class="form-group">
          <label>닉네임</label>
          <input v-model="signupForm.nickname" type="text" placeholder="닉네임">
        </div>
        <div class="form-group">
          <label>전화번호</label>
          <input v-model="signupForm.phone" type="tel" placeholder="010-1234-5678">
        </div>
      </div>
      <div class="modal-footer">
        <button @click="showSignupModal = false" style="background: #6c757d; color: white;">
          취소
        </button>
        <button @click="signup" style="background: #25d366; color: white;">
          가입하기
        </button>
      </div>
    </div>
  </div>
  
  <!-- 이미지 확대 모달 -->
  <div :class="['modal', { show: showImageModal }]" @click.self="showImageModal = false">
    <div class="image-modal-content">
      <span class="close" @click="showImageModal = false">&times;</span>
      <img :src="currentImage" alt="확대 이미지">
    </div>
  </div>
</template>

<script setup>
import { ref, computed, nextTick, onUnmounted } from 'vue'
import SockJS from 'sockjs-client'
import webstomp from 'webstomp-client'

// MessageType enum
const MessageType = {
  TEXT: 'TEXT',
  IMAGE: 'IMAGE',
  FILE: 'FILE',
  VIDEO: 'VIDEO',
  AUDIO: 'AUDIO',
  SYSTEM: 'SYSTEM'
}

const serverUrl = ref('http://localhost:8080')
const wsEndpoint = ref('/connect')
const email = ref('')
const password = ref('')
const isConnected = ref(false)
const currentMemberId = ref(null)
const accessToken = ref(null)
const otherMemberId = ref('')
const rooms = ref([])
const currentRoomId = ref(null)
const currentRoomName = ref('')
const messages = ref([])
const messageInput = ref('')
const messagesContainer = ref(null)
const showSignupModal = ref(false)
const showImageModal = ref(false)
const currentImage = ref('')
const uploadProgress = ref(0)

// 무한 스크롤 관련 상태
const isLoadingMore = ref(false)
const hasMoreMessages = ref(true)
const nextBeforeSeq = ref(null)
const isFirstLoad = ref(true)

// 파일 입력 refs
const imageInput = ref(null)
const fileInput = ref(null)
const videoInput = ref(null)
const audioInput = ref(null)
const currentFileType = ref('')

const signupForm = ref({
  type: 'individual',
  companyId: '',
  name: '',
  email: '',
  password: '',
  nickname: '',
  phone: ''
})

let stompClient = null
let subscription = null

const statusText = computed(() => {
  if (isConnected.value) {
    return `연결됨 (${email.value} / ID: ${currentMemberId.value})`
  }
  return '연결 안됨'
})

const getMessageType = (message) => {
  return message.messageType || message.type || 'TEXT'
}

const handleImageError = (e) => {
  console.error('❌ 이미지 로드 실패:', e.target.src)
}

const isNearBottom = () => {
  if (!messagesContainer.value) return false
  const { scrollTop, scrollHeight, clientHeight } = messagesContainer.value
  return scrollHeight - scrollTop - clientHeight < 150
}

const login = async () => {
  if (!email.value || !password.value) {
    alert('이메일과 비밀번호를 입력해주세요.')
    return
  }

  try {
    console.log('========== 로그인 시작 ==========')

    const response = await fetch(`${serverUrl.value}/v1/auth/login`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        email: email.value,
        password: password.value
      })
    })

    if (!response.ok) {
      const errorData = await response.json().catch(() => ({ message: '응답 파싱 실패' }))
      alert('로그인 실패: ' + (errorData.message || '이메일 또는 비밀번호를 확인해주세요.'))
      return
    }

    const responseData = await response.json()
    const data = responseData.data
    
    if (!data || !data.tokenInfo || !data.memberInfo) {
      alert('로그인 응답 형식이 올바르지 않습니다.')
      return
    }
    
    accessToken.value = data.tokenInfo.accessToken
    currentMemberId.value = data.memberInfo.memberId

    console.log('✅ 로그인 성공')
    
    await connectWebSocket()
  } catch (error) {
    console.error('로그인 오류:', error)
    alert('로그인 중 오류가 발생했습니다: ' + error.message)
  }
}

const connectWebSocket = () => {
  return new Promise((resolve, reject) => {
    if (!accessToken.value) {
      alert('토큰이 없어서 WebSocket 연결을 할 수 없습니다.')
      reject(new Error('No access token'))
      return
    }

    const wsUrl = serverUrl.value + wsEndpoint.value
    const socket = new SockJS(wsUrl)
    stompClient = webstomp.over(socket)
    
    const connectHeaders = {
      'Authorization': 'Bearer ' + accessToken.value
    }
    
    stompClient.connect(
      connectHeaders,
      function(frame) {
        console.log('✅✅✅ WebSocket CONNECT 성공! ✅✅✅')
        isConnected.value = true
        
        // ✅ 개인 큐 구독
        subscribeRoomSummary()
        
        loadRooms()
        resolve(frame)
      },
      function(error) {
        console.error('❌ WebSocket 연결 실패:', error)
        isConnected.value = false
        alert('WebSocket 연결 실패')
        reject(error)
      }
    )
    
    socket.onclose = function(e) {
      console.log('⚠️ WebSocket 연결 종료')
      isConnected.value = false
    }
  })
}

// ✅ 개인 큐 구독 - lastMessageAt 포함, 최신순 정렬 추가
function subscribeRoomSummary() {
  if (!stompClient) {
    console.error('❌ STOMP 클라이언트가 없습니다.')
    return
  }

  console.log('📡 /user/queue/room-summary 구독 시도...')

  stompClient.subscribe('/user/queue/room-summary', (frame) => {
    try {
      const summary = JSON.parse(frame.body)
      console.log('📬 [room-summary 수신]', summary)

      const roomId = summary.roomId
      const preview = summary.lastMessagePreview || summary.preview || ''
      const unread = summary.unreadCount || summary.unread || 0
      const lastMessageAt = summary.lastMessageAt || new Date().toISOString()
      const type = summary.type || 'PRIVATE'

      // rooms 배열 갱신
      const idx = rooms.value.findIndex(r => r.roomId === roomId)
      if (idx !== -1) {
        rooms.value[idx].lastMessagePreview = preview
        rooms.value[idx].unreadCount = unread
        rooms.value[idx].lastMessageAt = lastMessageAt
        rooms.value[idx].type = type
      } else {
        // 새 방 추가
        rooms.value.push({
          roomId,
          type,
          lastMessagePreview: preview,
          unreadCount: unread,
          lastMessageAt
        })
      }

      // ✅ 최신순 정렬 (lastMessageAt 기준)
      rooms.value.sort((a, b) => {
        const timeA = new Date(a.lastMessageAt || 0).getTime()
        const timeB = new Date(b.lastMessageAt || 0).getTime()
        return timeB - timeA
      })

      console.log(`✅ 방 ${roomId} 요약 업데이트: preview="${preview}", unread=${unread}, time=${lastMessageAt}`)
      console.log('📋 현재 방 목록:', rooms.value.length, '개')
    } catch (e) {
      console.error('room-summary 파싱 오류:', e, frame.body)
    }
  })

  console.log('✅ /user/queue/room-summary 구독 완료')
}

const signup = async () => {
  const form = signupForm.value

  if (!form.name || !form.email || !form.password || !form.nickname || !form.phone) {
    alert('모든 필드를 입력해주세요.')
    return
  }

  if (form.type === 'company' && !form.companyId) {
    alert('기업 ID를 입력해주세요.')
    return
  }

  const requestBody = {
    name: form.name,
    email: form.email,
    password: form.password,
    nickname: form.nickname,
    phone: form.phone
  }

  try {
    let url = `${serverUrl.value}/v1/auth/signup/${form.type}`
    if (form.type === 'company') {
      url += `/${form.companyId}`
    }

    const response = await fetch(url, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(requestBody)
    })

    if (!response.ok) {
      const errorData = await response.json()
      alert('회원가입 실패: ' + (errorData.message || '오류가 발생했습니다.'))
      return
    }

    const responseData = await response.json()
    const data = responseData.data
    
    if (!data || !data.tokenInfo || !data.memberInfo) {
      alert('회원가입 응답 형식이 올바르지 않습니다.')
      return
    }

    accessToken.value = data.tokenInfo.accessToken
    currentMemberId.value = data.memberInfo.memberId
    email.value = form.email

    alert('회원가입 성공! 자동으로 로그인됩니다.')
    showSignupModal.value = false
    await connectWebSocket()
  } catch (error) {
    console.error('회원가입 오류:', error)
    alert('회원가입 중 오류: ' + error.message)
  }
}

const disconnect = () => {
  if (stompClient) {
    stompClient.disconnect()
    stompClient = null
  }
  isConnected.value = false
  rooms.value = []
  messages.value = []
  currentRoomId.value = null
  currentMemberId.value = null
  accessToken.value = null
  console.log('🔴 WebSocket 연결 종료 및 구독 해제 완료')
}

const createPrivateRoom = async () => {
  if (!otherMemberId.value) {
    alert('상대방 ID를 입력해주세요.')
    return
  }

  if (!accessToken.value) {
    alert('먼저 로그인해주세요.')
    return
  }

  try {
    const response = await fetch(`${serverUrl.value}/v1/chat/private?otherMemberId=${otherMemberId.value}`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer ' + accessToken.value
      }
    })

    if (!response.ok) {
      const errorData = await response.json()
      alert('채팅방 생성 실패: ' + (errorData.message || '오류 발생'))
      return
    }

    const data = await response.json()
    const roomId = data.data || data.result || data
    
    await loadRooms()
    
    setTimeout(() => {
      const newRoom = rooms.value.find(r => r.roomId === roomId)
      if (newRoom) {
        selectRoom(newRoom)
      }
    }, 150)
    
  } catch (error) {
    console.error('Error:', error)
    alert('채팅방 생성 중 오류가 발생했습니다.')
  }
}

const loadRooms = async () => {
  if (!accessToken.value) return

  try {
    const response = await fetch(`${serverUrl.value}/v1/chat/rooms/me?page=0&size=20`, {
      headers: {
        'Authorization': 'Bearer ' + accessToken.value
      }
    })

    if (response.ok) {
      const responseData = await response.json()
      const roomList = responseData.data?.content || responseData.result?.content || responseData.content || []
      
      // ✅ 서버 DTO 필드명 매핑 + lastMessageAt 포함
      rooms.value = roomList.map(r => ({
        roomId: r.roomId,
        type: r.type,
        lastMessagePreview: r.lastMessagePreview ?? r.preview ?? '',
        unreadCount: r.unreadCount ?? r.unread ?? 0,
        lastMessageAt: r.lastMessageAt
      }))
      
      // ✅ 최신순 정렬
      rooms.value.sort((a, b) => {
        const timeA = new Date(a.lastMessageAt || 0).getTime()
        const timeB = new Date(b.lastMessageAt || 0).getTime()
        return timeB - timeA
      })
      
      console.log(`📋 방 목록 로드: ${rooms.value.length}개`)
    }
  } catch (error) {
    console.error('Error:', error)
  }
}

const selectRoom = (room) => {
  if (!stompClient || !isConnected.value) {
    alert('WebSocket 연결이 끊어졌습니다.')
    return
  }

  console.log('🚪 방 입장:', room.roomId)

  currentRoomId.value = room.roomId
  currentRoomName.value = `채팅방 ${room.roomId}`
  
  // ✅ 방 입장 시 해당 방의 unreadCount 즉시 0으로 설정
  const idx = rooms.value.findIndex(r => r.roomId === room.roomId)
  if (idx !== -1) {
    rooms.value[idx].unreadCount = 0
  }
  
  messages.value = []
  nextBeforeSeq.value = null
  hasMoreMessages.value = true
  isFirstLoad.value = true

  if (subscription) {
    subscription.unsubscribe()
    subscription = null
  }

  const subscriptionPath = `/topic/chat/room/${room.roomId}`
  
  try {
    subscription = stompClient.subscribe(subscriptionPath, (message) => {
      const chatMessage = JSON.parse(message.body)
      console.log('📩 실시간 메시지:', chatMessage)
      
      messages.value.push(chatMessage)
      nextTick(() => {
        scrollToBottom()
      })
    })
  } catch (error) {
    console.error('❌ 방 구독 실패:', error)
    alert('채팅방 구독에 실패했습니다.')
    return
  }
  
  loadMessages(room.roomId)
}

const loadMessages = async (roomId, beforeSeq = null) => {
  if (!accessToken.value || isLoadingMore.value) return

  try {
    isLoadingMore.value = true
    
    let url = `${serverUrl.value}/v1/chat/rooms/${roomId}/messages?size=50`
    if (beforeSeq) {
      url += `&beforeSeq=${beforeSeq}`
    }

    const response = await fetch(url, {
      headers: {
        'Authorization': 'Bearer ' + accessToken.value
      }
    })

    if (response.ok) {
      const responseData = await response.json()
      const messageList = responseData.data?.content || responseData.result?.content || responseData.content || []
      const hasNext = responseData.data?.hasNext ?? responseData.result?.hasNext ?? false

      if (isFirstLoad.value) {
        messages.value = messageList
        isFirstLoad.value = false
        nextTick(() => {
          scrollToBottom()
        })
      } else {
        const scrollHeight = messagesContainer.value.scrollHeight
        messages.value = [...messageList, ...messages.value]
        
        nextTick(() => {
          const newScrollHeight = messagesContainer.value.scrollHeight
          messagesContainer.value.scrollTop = newScrollHeight - scrollHeight
        })
      }
      
      hasMoreMessages.value = hasNext
      if (hasNext && messageList.length > 0) {
        nextBeforeSeq.value = messageList[0].seq
      }
    }
  } catch (error) {
    console.error('Error:', error)
  } finally {
    isLoadingMore.value = false
  }
}

const handleScroll = () => {
  if (!messagesContainer.value || isLoadingMore.value || !hasMoreMessages.value) {
    return
  }
  
  if (messagesContainer.value.scrollTop < 100) {
    loadMessages(currentRoomId.value, nextBeforeSeq.value)
  }
}

const sendMessage = () => {
  const content = messageInput.value.trim()

  if (!content || !currentRoomId.value || !stompClient || !isConnected.value) {
    return
  }

  const message = {
    roomId: currentRoomId.value,
    senderId: currentMemberId.value,
    type: MessageType.TEXT,
    content: content,
    fileUrl: null,
    fileName: null,
    fileSize: null
  }

  try {
    stompClient.send(
      `/publish/${currentRoomId.value}`,
      JSON.stringify(message),
      { 'content-type': 'application/json' }
    )

    messageInput.value = ''
  } catch (error) {
    console.error('❌ 메시지 전송 실패:', error)
    alert('메시지 전송에 실패했습니다.')
  }
}

const triggerFileInput = (type) => {
  currentFileType.value = type
  if (type === 'image') imageInput.value.click()
  else if (type === 'file') fileInput.value.click()
  else if (type === 'video') videoInput.value.click()
  else if (type === 'audio') audioInput.value.click()
}

const handleFileSelect = async (event) => {
  const file = event.target.files[0]
  if (!file) return

  const maxSize = 10 * 1024 * 1024
  if (file.size > maxSize) {
    alert('파일 크기는 10MB를 초과할 수 없습니다.')
    return
  }

  try {
    uploadProgress.value = 0
    const fileUrl = await uploadFile(file)
    
    let messageType
    if (currentFileType.value === 'image') messageType = MessageType.IMAGE
    else if (currentFileType.value === 'video') messageType = MessageType.VIDEO
    else if (currentFileType.value === 'audio') messageType = MessageType.AUDIO
    else messageType = MessageType.FILE

    const message = {
      roomId: currentRoomId.value,
      senderId: currentMemberId.value,
      type: messageType,
      content: messageInput.value.trim() || null,
      fileUrl: fileUrl,
      fileName: file.name,
      fileSize: file.size
    }

    stompClient.send(
      `/publish/${currentRoomId.value}`,
      JSON.stringify(message),
      { 'content-type': 'application/json' }
    )

    messageInput.value = ''
    uploadProgress.value = 0
    event.target.value = ''
  } catch (error) {
    console.error('❌ 파일 전송 실패:', error)
    alert('파일 전송에 실패했습니다.')
    uploadProgress.value = 0
  }
}

const uploadFile = async (file) => {
  return new Promise((resolve) => {
    let progress = 0
    const interval = setInterval(() => {
      progress += 10
      uploadProgress.value = progress
      if (progress >= 100) {
        clearInterval(interval)
        const blobUrl = URL.createObjectURL(file)
        resolve(blobUrl)
      }
    }, 100)
  })
}

const leaveRoom = async () => {
  if (!currentRoomId.value || !accessToken.value) return
  if (!confirm('정말 이 채팅방을 나가시겠습니까?')) return

  try {
    const response = await fetch(`${serverUrl.value}/v1/chat/rooms/${currentRoomId.value}`, {
      method: 'DELETE',
      headers: { 'Authorization': 'Bearer ' + accessToken.value }
    })

    if (response.ok) {
      if (subscription) {
        subscription.unsubscribe()
        subscription = null
      }
      
      currentRoomId.value = null
      currentRoomName.value = ''
      messages.value = []
      loadRooms()
      alert('채팅방을 나갔습니다.')
    }
  } catch (error) {
    console.error('Error:', error)
  }
}

const scrollToBottom = () => {
  if (messagesContainer.value) {
    messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight
  }
}

const formatTime = (timestamp) => {
  if (!timestamp) return ''
  const date = new Date(timestamp)
  return date.toLocaleTimeString('ko-KR', {
    hour: '2-digit',
    minute: '2-digit'
  })
}

// ✅ 시간 표시를 HH:MM 형식으로만 표시 (09:10, 19:00)
const formatLastMessageTime = (timestamp) => {
  if (!timestamp) return ''
  const date = new Date(timestamp)
  return date.toLocaleTimeString('ko-KR', {
    hour: '2-digit',
    minute: '2-digit'
  })
}

const formatFileSize = (bytes) => {
  if (!bytes) return ''
  if (bytes < 1024) return bytes + ' B'
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + ' KB'
  return (bytes / (1024 * 1024)).toFixed(1) + ' MB'
}

const getRoomTypeLabel = (type) => {
  if (type === 'PRIVATE') return '개인'
  if (type === 'GROUP') return '그룹'
  return type
}

const openImageModal = (imageUrl) => {
  currentImage.value = imageUrl
  showImageModal.value = true
}

onUnmounted(() => {
  disconnect()
})
</script>

<style scoped>
/* App.vue의 스코프 스타일은 전역 style.css를 사용하므로 비워둡니다 */
</style>
