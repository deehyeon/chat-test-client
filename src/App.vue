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
          </div>
        </div>
      </div>

      <div class="chat-area">
        <div v-if="!currentRoomId" class="no-room-selected">
          채팅방을 선택해주세요
        </div>
        <div v-else style="flex: 1; display: flex; flex-direction: column;">
          <div class="chat-header">
            <h3>{{ currentRoomName }}</h3>
            <button @click="leaveRoom" class="btn-danger">나가기</button>
          </div>
          <div class="messages" ref="messagesContainer">
            <div 
              v-for="(message, index) in messages" 
              :key="index"
              :class="['message', message.senderId == currentMemberId ? 'mine' : 'others']"
            >
              <div v-if="message.senderId != currentMemberId" class="message-sender">
                사용자 {{ message.senderId }}
              </div>
              <div class="message-content">
                <div class="message-text">{{ message.content }}</div>
                <div class="message-time">{{ formatTime(message.timestamp) }}</div>
              </div>
            </div>
          </div>
          <div class="message-input">
            <input 
              v-model="messageInput" 
              type="text" 
              placeholder="메시지를 입력하세요..." 
              @keypress.enter="sendMessage"
            >
            <button @click="sendMessage">전송</button>
          </div>
        </div>
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
          <input v-model="signupForm.password" type="password" placeholder="비밀번호">
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
</template>

<script setup>
import { ref, computed, nextTick, onUnmounted } from 'vue'
import SockJS from 'sockjs-client'
import webstomp from 'webstomp-client'

const serverUrl = ref('http://localhost:8080')
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

const login = async () => {
  if (!email.value || !password.value) {
    alert('이메일과 비밀번호를 입력해주세요.')
    return
  }

  try {
    console.log('========== 로그인 시작 ==========')
    console.log('로그인 시도:', serverUrl.value + '/v1/auth/login')
    console.log('요청 데이터:', { email: email.value, password: '***' })

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

    console.log('응답 상태:', response.status)

    if (!response.ok) {
      const errorData = await response.json().catch(() => ({ message: '응답 파싱 실패' }))
      console.error('로그인 실패 응답:', errorData)
      alert('로그인 실패: ' + (errorData.message || '이메일 또는 비밀번호를 확인해주세요.'))
      return
    }

    const data = await response.json()
    console.log('========== 로그인 응답 전체 ==========')
    console.log(JSON.stringify(data, null, 2))
    
    // 다양한 응답 구조 시도
    console.log('========== 토큰 추출 시도 ==========')
    console.log('data.result:', data.result)
    console.log('data.result?.accessToken:', data.result?.accessToken)
    console.log('data.accessToken:', data.accessToken)
    console.log('data.token:', data.token)
    console.log('data.result?.token:', data.result?.token)
    
    // 토큰 추출 (다양한 경로 시도)
    let token = null
    let memberId = null
    
    if (data.result) {
      token = data.result.accessToken || data.result.token
      memberId = data.result.memberId || data.result.id || data.result.userId
    } else {
      token = data.accessToken || data.token
      memberId = data.memberId || data.id || data.userId
    }
    
    console.log('========== 추출된 값 ==========')
    console.log('추출된 토큰:', token)
    console.log('추출된 memberId:', memberId)
    
    if (!token) {
      console.error('❌ 토큰을 찾을 수 없습니다!')
      alert('로그인 응답에서 토큰을 찾을 수 없습니다.\n\n서버 응답을 확인해주세요.\n\n콘솔(F12)을 열어 전체 응답을 확인하세요.')
      return
    }
    
    accessToken.value = token
    currentMemberId.value = memberId

    console.log('✅ 토큰 저장 완료:', accessToken.value.substring(0, 30) + '...')
    console.log('✅ 회원 ID:', currentMemberId.value)
    console.log('========== WebSocket 연결 시도 ==========')
    
    connectWebSocket()
  } catch (error) {
    console.error('로그인 오류 상세:', error)

    if (error.name === 'TypeError' && error.message.includes('fetch')) {
      alert(`서버 연결 실패!\n\n확인 사항:\n1. 서버가 실행 중인가요? (${serverUrl.value})\n2. 서버 URL이 올바른가요?\n3. CORS 설정이 되어 있나요?\n\n브라우저 콘솔(F12)을 열어 자세한 에러를 확인하세요.`)
    } else {
      alert('로그인 중 오류가 발생했습니다: ' + error.message)
    }
  }
}

const connectWebSocket = () => {
  // 토큰 확인
  if (!accessToken.value) {
    console.error('❌ accessToken이 없습니다!')
    alert('토큰이 없어서 WebSocket 연결을 할 수 없습니다.')
    return
  }

  console.log('🔌 WebSocket 연결 시작...')
  console.log('서버 URL:', serverUrl.value + '/connect')
  console.log('토큰 (앞 30자):', accessToken.value.substring(0, 30) + '...')

  const socket = new SockJS(serverUrl.value + '/connect')
  stompClient = webstomp.over(socket)
  
  // 디버그 로그 활성화
  stompClient.debug = (msg) => {
    console.log('🔍 STOMP:', msg)
  }
  
  // STOMP CONNECT 헤더에 Authorization 추가
  const connectHeaders = {
    'Authorization': 'Bearer ' + accessToken.value
  }
  
  console.log('📤 전송할 헤더:', {
    'Authorization': 'Bearer ' + accessToken.value.substring(0, 30) + '...'
  })
  
  stompClient.connect(
    connectHeaders,
    function(frame) {
      console.log('✅ WebSocket 연결 성공!')
      console.log('Frame:', frame)
      isConnected.value = true
      loadRooms()
    },
    function(error) {
      console.error('❌ WebSocket 연결 오류:', error)
      isConnected.value = false
      alert('WebSocket 연결에 실패했습니다.\n\n에러: ' + (error.headers?.message || error))
    }
  )
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

    const data = await response.json()
    console.log('회원가입 응답:', data)
    
    // 토큰 추출 (로그인과 동일한 방식)
    let token = null
    let memberId = null
    
    if (data.result) {
      token = data.result.accessToken || data.result.token
      memberId = data.result.memberId || data.result.id || data.result.userId
    } else {
      token = data.accessToken || data.token
      memberId = data.memberId || data.id || data.userId
    }

    if (!token) {
      alert('회원가입 응답에서 토큰을 찾을 수 없습니다.')
      return
    }

    alert('회원가입 성공! 자동으로 로그인됩니다.')

    accessToken.value = token
    currentMemberId.value = memberId
    email.value = form.email

    showSignupModal.value = false
    connectWebSocket()
  } catch (error) {
    console.error('회원가입 오류:', error)
    alert('회원가입 중 오류가 발생했습니다: ' + error.message)
  }
}

const disconnect = () => {
  if (stompClient !== null) {
    if (subscription) {
      subscription.unsubscribe()
      subscription = null
    }
    stompClient.disconnect()
    stompClient = null
  }
  isConnected.value = false
  currentMemberId.value = null
  currentRoomId.value = null
  accessToken.value = null
  rooms.value = []
  messages.value = []
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

    if (response.ok) {
      const data = await response.json()
      const roomId = data.result || data
      alert(`채팅방이 생성되었습니다. Room ID: ${roomId}`)
      loadRooms()
    } else {
      const errorData = await response.json()
      alert('채팅방 생성 실패: ' + (errorData.message || '오류 발생'))
    }
  } catch (error) {
    console.error('Error:', error)
    alert('채팅방 생성 중 오류가 발생했습니다.')
  }
}

const loadRooms = async () => {
  if (!accessToken.value) {
    return
  }

  try {
    const response = await fetch(`${serverUrl.value}/v1/chat/rooms/me?page=0&size=20`, {
      headers: {
        'Authorization': 'Bearer ' + accessToken.value
      }
    })

    if (response.ok) {
      const data = await response.json()
      rooms.value = data.result?.content || data.content || []
    } else {
      console.error('채팅방 목록을 불러올 수 없습니다.')
    }
  } catch (error) {
    console.error('Error:', error)
  }
}

const selectRoom = (room) => {
  currentRoomId.value = room.roomId
  currentRoomName.value = `채팅방 ${room.roomId}`

  if (subscription) {
    subscription.unsubscribe()
  }

  subscription = stompClient.subscribe(`/topic/chat/room/${room.roomId}`, (message) => {
    const chatMessage = JSON.parse(message.body)
    messages.value.push(chatMessage)
    nextTick(() => scrollToBottom())
  })

  loadMessages(room.roomId)
}

const loadMessages = async (roomId) => {
  if (!accessToken.value) {
    return
  }

  try {
    const response = await fetch(`${serverUrl.value}/v1/chat/rooms/${roomId}/messages?size=50`, {
      headers: {
        'Authorization': 'Bearer ' + accessToken.value
      }
    })

    if (response.ok) {
      const data = await response.json()
      const messageList = data.result?.content || data.content || []

      messages.value = messageList.reverse()

      nextTick(() => scrollToBottom())
      markAsRead(roomId)
    } else {
      console.error('메시지 불러오기 실패')
    }
  } catch (error) {
    console.error('Error:', error)
  }
}

const sendMessage = () => {
  const content = messageInput.value.trim()

  if (!content || !currentRoomId.value) {
    return
  }

  const message = {
    roomId: currentRoomId.value,
    senderId: currentMemberId.value,
    content: content,
    timestamp: new Date().toISOString()
  }

  stompClient.send(`/publish/${currentRoomId.value}`, JSON.stringify(message), {})

  messageInput.value = ''
}

const markAsRead = async (roomId) => {
  if (!accessToken.value) {
    return
  }

  try {
    await fetch(`${serverUrl.value}/v1/chat/rooms/${roomId}/read`, {
      method: 'POST',
      headers: {
        'Authorization': 'Bearer ' + accessToken.value
      }
    })
  } catch (error) {
    console.error('읽음 처리 실패:', error)
  }
}

const leaveRoom = async () => {
  if (!currentRoomId.value || !accessToken.value) return

  if (!confirm('정말 이 채팅방을 나가시겠습니까?')) {
    return
  }

  try {
    const response = await fetch(`${serverUrl.value}/v1/chat/rooms/${currentRoomId.value}`, {
      method: 'DELETE',
      headers: {
        'Authorization': 'Bearer ' + accessToken.value
      }
    })

    if (response.ok) {
      alert('채팅방을 나갔습니다.')
      currentRoomId.value = null
      currentRoomName.value = ''
      messages.value = []
      loadRooms()
    } else {
      alert('채팅방 나가기 실패')
    }
  } catch (error) {
    console.error('Error:', error)
    alert('채팅방 나가기 중 오류가 발생했습니다.')
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

const getRoomTypeLabel = (type) => {
  if (type === 'PRIVATE') return '개인'
  if (type === 'GROUP') return '그룹'
  return type
}

onUnmounted(() => {
  disconnect()
})
</script>

<style scoped>
/* App.vue의 스코프 스타일은 전역 style.css를 사용하므로 비워둡니다 */
</style>
