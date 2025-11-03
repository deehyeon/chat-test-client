# 채팅 서버 테스트 클라이언트

Spring Boot 기반 채팅 서버를 테스트하기 위한 웹 클라이언트입니다.

## 📁 프로젝트 구조

```
.
├── chat-test.html              # 메인 테스트 클라이언트
├── serve.py                    # 간단한 HTTP 서버
├── README.md                   # 프로젝트 설명
├── QUICKSTART.md               # 빠른 시작 가이드
├── CORS_TROUBLESHOOTING.md     # CORS 문제 해결
├── DIAGNOSIS.md                # 문제 진단 체크리스트
├── DTO_STRUCTURE.md            # DTO 구조 문서
├── server/                     # 서버 설정 파일들
│   ├── WebConfig.java
│   ├── SecurityConfig.java
│   ├── StompWebSocketConfig.java
│   └── ChatSaverExample.java
└── .gitignore
```

## 🚀 빠른 시작

### 1. 서버 설정

`server/` 폴더의 Java 파일들을 Spring Boot 프로젝트에 적용하세요.

**중요 파일:**
- `WebConfig.java` - CORS 설정 (필수)
- `StompWebSocketConfig.java` - WebSocket 설정 수정
- `SecurityConfig.java` - Spring Security 통합

### 2. HTTP 서버 실행

```bash
python3 serve.py
```

### 3. 브라우저에서 열기

```
http://localhost:8000/chat-test.html
```

### 4. 테스트

1. "회원가입" 버튼으로 계정 생성
2. 로그인
3. 채팅방 생성
4. 다른 브라우저 창에서 다른 계정으로 접속
5. 실시간 채팅 테스트

## 📖 자세한 가이드

- **5분 안내:** [QUICKSTART.md](QUICKSTART.md)
- **CORS 문제:** [CORS_TROUBLESHOOTING.md](CORS_TROUBLESHOOTING.md)
- **진단 가이드:** [DIAGNOSIS.md](DIAGNOSIS.md)
- **DTO 구조:** [DTO_STRUCTURE.md](DTO_STRUCTURE.md)

## ✨ 주요 기능

### 프론트엔드
- ✅ 이메일/비밀번호 기반 회원가입/로그인
- ✅ JWT 토큰 인증
- ✅ WebSocket 실시간 채팅
- ✅ 1:1 채팅방 생성
- ✅ 메시지 읽음 처리
- ✅ 채팅방 목록 조회
- ✅ 이전 메시지 불러오기

### 서버 연동
- ✅ REST API 연동 (로그인, 채팅방 관리)
- ✅ STOMP over WebSocket
- ✅ JWT 토큰 기반 인증

## 🔧 트러블슈팅

### 로그인 실패 (Failed to fetch)

**원인:** CORS 설정 문제

**해결:**
1. `server/WebConfig.java`를 프로젝트에 추가
2. `server/SecurityConfig.java` 참고하여 수정
3. 서버 재시작
4. HTTP 서버로 HTML 파일 열기

👉 [CORS_TROUBLESHOOTING.md](CORS_TROUBLESHOOTING.md) 참고

### WebSocket 연결 실패

**원인:** WebSocket CORS 설정 문제

**해결:**
1. `StompWebSocketConfig.java`에서 `.setAllowedOriginPatterns("*")` 설정
2. 서버 재시작

### 메시지 수신 안 됨

**원인:** 서버 destination 경로 불일치

**해결:**
1. `ChatSaver.sendMessage()`의 destination 확인
2. 프론트엔드 구독 경로와 일치하는지 확인

👉 [DIAGNOSIS.md](DIAGNOSIS.md) 참고

## 📋 API 엔드포인트

### REST API
```
POST   /v1/auth/signup/individual       # 일반 회원가입
POST   /v1/auth/signup/company/{id}     # 기업 회원가입
POST   /v1/auth/login                   # 로그인
POST   /v1/chat/private                 # 채팅방 생성
GET    /v1/chat/rooms/me                # 채팅방 목록
GET    /v1/chat/rooms/{id}/messages     # 메시지 조회
POST   /v1/chat/rooms/{id}/read         # 읽음 처리
DELETE /v1/chat/rooms/{id}              # 채팅방 나가기
```

### WebSocket
```
WS     /connect                         # WebSocket 연결
STOMP  /publish/{roomId}                # 메시지 전송
STOMP  /topic/chat/room/{roomId}        # 메시지 구독
```

## 🛠 기술 스택

### 프론트엔드
- HTML5 + CSS3 + Vanilla JavaScript
- SockJS (WebSocket polyfill)
- STOMP.js (WebSocket 메시징)

### 서버 (예상)
- Spring Boot
- Spring WebSocket
- Spring Security + JWT
- STOMP over WebSocket

## 📝 설정 체크리스트

서버 측:
- [ ] `WebConfig.java` 추가
- [ ] `SecurityConfig.java` 수정
- [ ] `StompWebSocketConfig.java` 수정
- [ ] `ChatSaver` destination 경로 확인
- [ ] 서버 재시작

클라이언트 측:
- [ ] HTTP 서버로 HTML 서빙
- [ ] 브라우저 캐시 클리어
- [ ] F12 콘솔에서 에러 확인

## 🔐 보안 고려사항

⚠️ 이 프로젝트는 **개발/테스트 목적**으로 설계되었습니다.

**프로덕션 환경에서는:**
- CORS를 특정 origin으로 제한
- HTTPS 사용
- 토큰을 안전하게 저장 (HttpOnly 쿠키 등)
- 입력 값 검증 강화
- Rate limiting 적용

## 📄 라이선스

이 프로젝트는 테스트 목적으로 만들어졌습니다.

## 🤝 기여

버그 리포트나 개선 제안은 이슈로 등록해주세요.

## 📞 지원

문제가 발생하면:
1. [DIAGNOSIS.md](DIAGNOSIS.md)로 자가 진단
2. [CORS_TROUBLESHOOTING.md](CORS_TROUBLESHOOTING.md)로 CORS 문제 해결
3. 브라우저 콘솔(F12)과 서버 로그 확인

---

**Made for testing Spring Boot chat server** 🚀