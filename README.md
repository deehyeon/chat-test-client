# 채팅 테스트 클라이언트 (Vue.js)

Spring Boot 채팅 서버 테스트용 Vue.js 프론트엔드 클라이언트입니다.

## 🚀 기술 스택

- **Vue 3** - Composition API 사용
- **Vite** - 빠른 개발 서버 및 빌드 도구
- **SockJS** - WebSocket 폴백 지원
- **STOMP** - 메시징 프로토콜

## 📦 설치 방법

```bash
# 의존성 설치
npm install
```

## 🎯 실행 방법

```bash
# 개발 서버 실행 (http://localhost:3000)
npm run dev

# 프로덕션 빌드
npm run build

# 빌드 미리보기
npm run preview
```

## 🔧 사용 방법

### 1. 서버 연결
- 서버 URL에 백엔드 서버 주소 입력 (기본값: `http://localhost:8080`)
- 이메일과 비밀번호 입력 후 로그인

### 2. 회원가입
- "회원가입" 버튼 클릭
- 일반 회원 또는 기업 회원 선택
- 필수 정보 입력 후 가입
- 가입 성공 시 자동 로그인

### 3. 채팅방 생성
- 좌측 사이드바에서 "상대방 ID" 입력
- "1:1 채팅방 만들기" 버튼 클릭
- 생성된 채팅방이 목록에 표시됨

### 4. 메시지 전송
- 채팅방 목록에서 원하는 채팅방 선택
- 하단 입력창에 메시지 입력 후 Enter 또는 "전송" 버튼 클릭
- 실시간으로 메시지 송수신

### 5. 채팅방 나가기
- 우측 상단 "나가기" 버튼 클릭
- 확인 후 채팅방 목록에서 제거됨

## 📝 주요 기능

- ✅ 실시간 WebSocket 통신 (SockJS + STOMP)
- ✅ JWT 기반 인증
- ✅ 1:1 개인 채팅방 생성
- ✅ 읽지 않은 메시지 카운트 표시
- ✅ 메시지 히스토리 로딩
- ✅ 자동 스크롤
- ✅ 반응형 UI (WhatsApp 스타일)
- ✅ 일반/기업 회원가입 지원

## 🔗 백엔드 API 엔드포인트

### 인증
- `POST /v1/auth/login` - 로그인
- `POST /v1/auth/signup/individual` - 일반 회원가입
- `POST /v1/auth/signup/company/{companyId}` - 기업 회원가입

### 채팅
- `POST /v1/chat/private?otherMemberId={id}` - 1:1 채팅방 생성
- `GET /v1/chat/rooms/me` - 내 채팅방 목록 조회
- `GET /v1/chat/rooms/{roomId}/messages` - 채팅방 메시지 조회
- `POST /v1/chat/rooms/{roomId}/read` - 메시지 읽음 처리
- `DELETE /v1/chat/rooms/{roomId}` - 채팅방 나가기

### WebSocket
- **연결**: `/connect`
- **구독**: `/topic/chat/room/{roomId}`
- **발행**: `/publish/{roomId}`

## 🎨 프로젝트 구조

```
chat-test-client/
├── index.html              # HTML 엔트리 포인트
├── package.json            # 프로젝트 의존성
├── vite.config.js          # Vite 설정
└── src/
    ├── main.js             # Vue 앱 초기화
    ├── App.vue             # 메인 컴포넌트
    └── style.css           # 전역 스타일
```

## 🔒 환경 변수

필요시 `.env` 파일을 생성하여 기본 서버 URL을 설정할 수 있습니다:

```env
VITE_API_URL=http://localhost:8080
```

## 🐛 트러블슈팅

### CORS 오류
백엔드 서버에서 CORS 설정을 확인하세요:
```java
@Configuration
public class WebConfig {
    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/**")
                        .allowedOrigins("http://localhost:3000")
                        .allowedMethods("*")
                        .allowedHeaders("*")
                        .allowCredentials(true);
            }
        };
    }
}
```

### WebSocket 연결 실패
1. 백엔드 서버가 실행 중인지 확인
2. 서버 URL이 올바른지 확인
3. 브라우저 콘솔(F12)에서 에러 로그 확인

## 📄 라이선스

MIT License

## 👨‍💻 개발자

- GitHub: [@deehyeon](https://github.com/deehyeon)
