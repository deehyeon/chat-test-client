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
const wsEndpoint = ref('/connect')  // ⭐ 서버 엔드포인트 선택 가능
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

    const responseData = await response.json()
    console.log('========== 로그인 응답 ==========')
    console.log(JSON.stringify(responseData, null, 2))
    
    const data = responseData.data
    if (!data) {
      console.error('❌ data가 없습니다!')
      alert('로그인 응답 형식이 올바르지 않습니다.')
      return
    }
    
    const tokenInfo = data.tokenInfo
    const memberInfo = data.memberInfo
    
    if (!tokenInfo || !tokenInfo.accessToken) {
      console.error('❌ tokenInfo 또는 accessToken이 없습니다!')
      console.log('tokenInfo:', tokenInfo)
      alert('로그인 응답에서 토큰을 찾을 수 없습니다.')
      return
    }
    
    if (!memberInfo || !memberInfo.memberId) {
      console.error('❌ memberInfo 또는 memberId가 없습니다!')
      console.log('memberInfo:', memberInfo)
      alert('로그인 응답에서 회원 정보를 찾을 수 없습니다.')
      return
    }
    
    accessToken.value = tokenInfo.accessToken
    currentMemberId.value = memberInfo.memberId

    console.log('✅ 토큰 저장 완료')
    console.log('✅ Access Token (전체):', accessToken.value)
    console.log('✅ 회원 ID:', currentMemberId.value)
    console.log('✅ 회원 이름:', memberInfo.memberName)
    console.log('========== WebSocket 연결 시작 ==========')
    
    await connectWebSocket()
  } catch (error) {
    console.error('로그인 오류:', error)
    alert('로그인 중 오류가 발생했습니다: ' + error.message)
  }
}

const connectWebSocket = () => {
  return new Promise((resolve, reject) => {
    if (!accessToken.value) {
      console.error('❌ accessToken이 없습니다!')
      alert('토큰이 없어서 WebSocket 연결을 할 수 없습니다.')
      reject(new Error('No access token'))
      return
    }

    // ⭐ 서버 엔드포인트 URL 생성
    const wsUrl = serverUrl.value + wsEndpoint.value
    
    console.log('\n========== WebSocket 연결 시도 ==========')
    console.log('🎯 WebSocket URL:', wsUrl)
    console.log('🎯 엔드포인트:', wsEndpoint.value)
    console.log('🔑 Access Token (전체):', accessToken.value)
    console.log('👤 회원 ID:', currentMemberId.value)
    console.log('\n💡 브라우저 Network 탭에서 실제 URL을 확인하세요!')
    console.log('예상 URL: ws://localhost:8080' + wsEndpoint.value + '/XXX/XXX/websocket')

    const socket = new SockJS(wsUrl)
    stompClient = webstomp.over(socket)
    
    stompClient.debug = (msg) => {
      console.log('🔍 STOMP:', msg)
    }
    
    const connectHeaders = {
      'Authorization': 'Bearer ' + accessToken.value
    }
    
    console.log('\n📤 CONNECT 헤더:')
    console.log('  Authorization: Bearer ' + accessToken.value.substring(0, 20) + '...')
    console.log('  (전체 길이: ' + connectHeaders['Authorization'].length + ')')
    
    console.log('\n🔌 STOMP.connect() 호출...')
    
    stompClient.connect(
      connectHeaders,
      function(frame) {
        console.log('\n✅✅✅ WebSocket CONNECT 성공! ✅✅✅')
        console.log('Frame:', frame)
        isConnected.value = true
        loadRooms()
        resolve(frame)
      },
      function(error) {
        console.log('\n❌❌❌ WebSocket CONNECT 실패 ❌❌❌')
        console.error('Error:', error)
        console.error('Error message:', error.headers?.message || error.body || 'Unknown')
        
        isConnected.value = false
        
        let errorMessage = 'WebSocket CONNECT 실패'
        if (error && error.headers && error.headers.message) {
          errorMessage += ': ' + error.headers.message
        }
        
        console.error('\n🔍 확인할 사항:')
        console.error('1. 서버 엔드포인트가 "' + wsEndpoint.value + '"가 맞는지?')
        console.error('2. 서버 StompWebSocketConfig에서 registerStompEndpoints() 확인')
        console.error('3. 브라우저 Network 탭에서 실제 연결되는 URL 확인')
        
        alert(errorMessage + '\n\n엔드포인트를 변경해보세요: /connect, /ws, /stomp, /websocket')
        reject(error)
      }
    )
    
    socket.onclose = function(e) {
      console.log('🔴 SockJS 연결 종료:', e.code, e.reason)
      isConnected.value = false
    }
    
    socket.onerror = function(e) {
      console.error('🔴 SockJS 에러:', e)
    }
  })
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
    console.log('회원가입 응답:', responseData)
    
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

  if (!isConnected.value || !stompClient) {
    alert('WebSocket이 연결되지 않았습니다. 다시 로그인해주세요.')
    return
  }

  try {
    console.log('💬 방 생성 시작...')
    
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
    console.log('✅ 방 생성 성공! Room ID:', roomId)
    
    await loadRooms()
    
    console.log('⏱️ 150ms 딜레이 후 방 선택...')
    setTimeout(() => {
      const newRoom = rooms.value.find(r => r.roomId === roomId)
      if (newRoom) {
        console.log('🎯 새 방 선택:', newRoom)
        selectRoom(newRoom)
      } else {
        console.warn('⚠️ 새로 만들어진 방을 찾을 수 없습니다.')
        alert(`채팅방이 생성되었습니다. Room ID: ${roomId}`)
      }
    }, 150)
    
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
      const responseData = await response.json()
      rooms.value = responseData.data?.content || responseData.result?.content || responseData.content || []
      console.log('📋 방 목록 로드 완료:', rooms.value.length, '개')
    } else {
      console.error('채팅방 목록을 불러올 수 없습니다.')
    }
  } catch (error) {
    console.error('Error:', error)
  }
}

const selectRoom = (room) => {
  if (!stompClient || !isConnected.value) {
    console.error('❌ STOMP 클라이언트가 연결되지 않았습니다!')
    alert('WebSocket 연결이 끄어졌습니다. 다시 로그인해주세요.')
    return
  }

  console.log('👉 방 선택:', room.roomId)
  
  currentRoomId.value = room.roomId
  currentRoomName.value = `채팅방 ${room.roomId}`

  if (subscription) {
    console.log('🔴 기존 구독 해제')
    subscription.unsubscribe()
    subscription = null
  }

  const subscriptionPath = `/topic/chat/room/${room.roomId}`
  console.log('📡 SUBSCRIBE 시도:', subscriptionPath)
  
  try {
    subscription = stompClient.subscribe(subscriptionPath, (message) => {
      console.log('📩 메시지 수신:', message.body)
      const chatMessage = JSON.parse(message.body)
      messages.value.push(chatMessage)
      nextTick(() => scrollToBottom())
    })
    
    console.log('✅ SUBSCRIBE 성공!')
  } catch (error) {
    console.error('❌ SUBSCRIBE 실패:', error)
    alert('채팅방 구독에 실패했습니다: ' + error.message)
    return
  }

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
      const responseData = await response.json()
      const messageList = responseData.data?.content || responseData.result?.content || responseData.content || []

      messages.value = messageList.reverse()
      console.log('📨 메시지 로드 완료:', messages.value.length, '개')

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

  if (!stompClient || !isConnected.value) {
    alert('WebSocket 연결이 끄어졌습니다.')
    return
  }

  const message = {
    roomId: currentRoomId.value,
    senderId: currentMemberId.value,
    content: content
  }

  console.log('📤 메시지 SEND:', message)

  try {
    stompClient.send(
      `/publish/${currentRoomId.value}`,
      JSON.stringify(message),
      {
        'content-type': 'application/json'
      }
    )

    console.log('✅ 메시지 전송 완료')
    messageInput.value = ''
  } catch (error) {
    console.error('❌ 메시지 전송 실패:', error)
    alert('메시지 전송에 실패했습니다.')
  }
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
      
      if (subscription) {
        subscription.unsubscribe()
        subscription = null
      }
      
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
