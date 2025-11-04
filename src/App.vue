const sendMessage = () => {
  const content = messageInput.value.trim()

  if (!content || !currentRoomId.value) {
    return
  }

  // 서버 DTO에 맞게 필드 구성 (timestamp 제거 - 서버에서 생성)
  const message = {
    roomId: currentRoomId.value,
    senderId: currentMemberId.value,
    content: content
  }

  console.log('📤 메시지 전송:', message)

  // STOMP SEND에 content-type 헤더 추가 (필수!)
  stompClient.send(
    `/publish/${currentRoomId.value}`, 
    JSON.stringify(message), 
    {
      'content-type': 'application/json;charset=UTF-8'
    }
  )

  messageInput.value = ''
}
