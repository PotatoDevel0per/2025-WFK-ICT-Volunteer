# N8N으로 GitHub 자동 코드 리뷰 만들기 🤖

## 📚 이론설명

### 이 문서는 무엇인가요?

이 가이드는 **N8N**이라는 도구를 사용해서 GitHub에 올라온 코드를 **AI가 자동으로 검토**해주는 시스템을 만드는 방법을 설명합니다. 마치 선생님이 숙제를 자동으로 채점해주는 것처럼요!

### 우리가 만들 것

- **다른 브랜치에서 main 브랜치로 Pull Request(병합 요청)를 보낼 때**
- AI가 자동으로 코드를 검토하고
- 리뷰 댓글을 자동으로 남겨주는 시스템

> **중요!** 단순히 코드를 수정하는 것이 아니라, **브랜치를 나누어서 작업한 후 main으로 병합을 요청할 때** 작동합니다!

### 필요한 것들
1. **GitHub 계정** - 코드를 저장하는 곳 (**무료**)
2. **GitHub Access Token** - N8N이 GitHub에 접근하기 위한 열쇠 (**무료**)
3. **컴퓨터 또는 서버** - N8N을 돌릴 곳 (셀프호스팅)
4. **Docker** - N8N을 쉽게 설치하기 위한 도구 (**무료**)
5. **CloudFlared** - 외부에서 접근할 수 있게 해주는 터널 (**무료**)
6. **Google Gemini API** - AI 리뷰어 역할 (**무료**)
7. **Claude Pro 계정** (선택사항) - 더 좋은 AI 리뷰

### N8N 셀프호스팅이란?
**중요!** N8N은 클라우드 서비스를 사용하지 않고 직접 컴퓨터에 설치해서 사용해요.

상세한 설치 방법은 여기를 참고하세요:
👉 **[N8N 셀프호스팅 가이드](https://github.com/PotatoDevel0per/2025MAS/tree/main/n8n-self-hosting)**

**간단 요약:**
- Docker를 사용해서 N8N 설치 (**무료**)
- CloudFlared로 외부 접근 가능하게 설정 (**무료**)
- GitHub 웹훅이 여러분 컴퓨터의 N8N에 직접 연결

### GitHub Access Token 발급받기 🔑

N8N이 GitHub에 접근하려면 특별한 열쇠(Access Token)가 필요해요!

**단계별 발급 방법:**
1. **GitHub 프로필 클릭** (오른쪽 상단)
2. **Settings** 클릭
3. **Developer Settings** 클릭 (왼쪽 메뉴 맨 아래)
4. **Personal Access Tokens** 클릭
5. **Tokens (Classic)** 클릭
6. **Generate new token** → **Generate new token (classic)** 선택
7. **Confirm access** (비밀번호 입력)
8. **Note**에 이름 아무거나 쓰기 (예: "N8N Auto Review")
9. **Scopes**에서 **repo만 체크** ✅ (전체 저장소 권한)
10. **Generate token** 버튼 클릭
11. **생성된 Access Token 복사해서 안전한 곳에 저장!** 📝

> **⚠️ 중요**: Token은 한 번만 보여져요! 복사 후 바로 메모장에 저장하세요!

### 예상 비용
- N8N: **무료!** (셀프호스팅)
- Docker: **무료!**
- CloudFlared: **무료!**
- Google Gemini API: **무료!** (CLI 버전)
- 전기세: 약간 증가 (컴퓨터 24시간 가동시)
- **총 비용: 거의 무료! 💰**

### 전체 흐름 요약

```
1. 브랜치에서 코드 수정 → 
2. main으로 Pull Request 생성 → 
3. N8N이 PR 감지 → 
4. 변경된 파일 정보 수집 → 
5. AI가 이해할 수 있게 정리 → 
6. AI가 코드 검토 → 
7. 검토 결과를 PR에 댓글로 등록 → 
8. "AI 검토 완료" 라벨 추가
```

## 🛠️ 셀프호스팅 설정 방법

### N8N 셀프호스팅 설정 (필수!)

**먼저 이 단계들을 완료해야 해요:**

#### 1️⃣ CloudFlared 터널 생성
```bash
cloudflared tunnel --url localhost:5678
```
- 실행하면 `https://xxx-xxx-xxx.trycloudflare.com` 같은 도메인이 나와요
- **이 도메인을 복사해두세요!** 🔗

#### 2️⃣ Docker Compose 파일 준비
새 폴더를 만들고 `docker-compose.yml` 파일을 만드세요:

```yaml
version: '3.8'
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      # 기본 설정
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=admin123
      
      # 외부 접속 허용
      - N8N_HOST=0.0.0.0
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      
      # 웹훅 설정 (🔗 여기에 1단계에서 복사한 도메인 넣기!)
      - WEBHOOK_URL=https://voted-statutory-processor-matthew.trycloudflare.com
      
      # 데이터 저장
      - N8N_USER_FOLDER=/home/node/.n8n
      
      # AI 모델 연동 설정
      - N8N_AI_ENABLED=true
      
      # 시간대 설정
      - TZ=Asia/Bangladesh
      
    volumes:
      - n8n_data:/home/node/.n8n
      - /var/run/docker.sock:/var/run/docker.sock:ro
    
    extra_hosts:
      - "host.docker.internal:host-gateway"

volumes:
  n8n_data:
    driver: local
networks:
  default:
    driver: bridge
```

#### 3️⃣ N8N 컨테이너 실행
```bash
docker-compose up -d
```

#### 4️⃣ N8N 접속 확인
- 1단계에서 받은 도메인으로 접속 (예: `https://xxx-xxx-xxx.trycloudflare.com`)
- 로그인: `admin` / `admin123`

> **⚠️ 중요**: CloudFlared는 터미널을 종료하면 도메인이 사라져요! 항상 터미널을 켜두거나, 영구 터널을 만드세요.

## 📋 단계별 만들기

### 1단계: GitHub 연결 설정하기

**무엇을 하나요?**
브랜치 간 Pull Request가 생성되었을 때 N8N이 감지할 수 있게 연결해요.

**설정 방법:**
1. N8N에서 "GitHub Trigger" 노드 추가
2. GitHub 계정 연결 (위에서 발급받은 Access Token 사용!)
3. 감시할 저장소 선택 (예: "test" 저장소)
4. 이벤트 타입을 "pull_request"로 설정

**쉽게 말하면:** "야, N8N아! 누가 다른 브랜치에서 main으로 병합 요청하면 나한테 알려줘!"

### 2단계: 변경된 파일 정보 가져오기

**무엇을 하나요?**
어떤 파일이 어떻게 바뀌었는지 자세한 정보를 가져와요.

**설정 방법:**
1. "HTTP Request" 노드 추가
2. URL 설정: `https://api.github.com/repos/{{사용자명}}/{{저장소명}}/pulls/{{번호}}/files`
3. GitHub API 인증 설정 (Header Auth with Token 사용)

**쉽게 말하면:** "어떤 파일이 바뀌었는지 자세히 알려줘!"

### 3단계: AI가 이해할 수 있게 정리하기

**무엇을 하나요?**
GitHub에서 받은 복잡한 정보를 AI가 쉽게 이해할 수 있는 형태로 바꿔요.

**코드 예시:**
```javascript
// 파일들을 하나씩 확인해서
const files = $input.all().map(item => item.json);
let diffs = '';

for (const file of files) {
  // 파일 이름 추가
  diffs += `### 파일: ${file.filename}\n\n`;
  
  // 변경된 코드 추가 (안전하게 처리)
  if (file.patch) {
    const safePatch = file.patch.replace(/```/g, "''");
    diffs += "```diff\n" + safePatch + "\n```\n";
  }
}

// AI에게 보낼 메시지 만들기
const userMessage = `
당신은 시니어 iOS 개발자입니다.
다음 코드 변경사항을 검토해주세요:

${diffs}

임무:
- 파일별로 코드 변경사항을 검토하세요
- 관련된 코드 라인에 댓글을 작성하세요
- 패치가 없는 파일은 무시하세요
- 코드를 반복하지 말고 바로 댓글을 작성하세요
`;
```

**쉽게 말하면:** "AI야, 이 코드들 좀 봐줘. 이해하기 쉽게 정리해뒀어!"

### 4단계: AI에게 코드 검토 시키기

**무엇을 하나요?**
정리된 코드를 AI에게 보내서 전문적인 리뷰를 받아요.

**설정 방법:**
1. "AI Agent" 노드 추가
2. Google Gemini Chat Model 연결
3. 시스템 메시지: "당신은 전문 코드 리뷰어입니다."

**쉽게 말하면:** "AI 선생님, 이 코드 어떤가요? 문제점이나 개선점 알려주세요!"

### 5단계: 리뷰 결과를 GitHub에 올리기

**무엇을 하나요?**
AI가 작성한 리뷰를 실제 GitHub Pull Request에 댓글로 남겨요.

**설정 방법:**
1. "GitHub" 노드 추가 (Create a review)
2. 리뷰 타입을 "comment"로 설정
3. AI가 작성한 내용을 body에 입력

**쉽게 말하면:** "AI야, 이 결과를 GitHub PR에 올려서 모든 사람이 볼 수 있게 해줘!"

### 6단계: 라벨 자동 추가하기

**무엇을 하나요?**
리뷰가 완료되면 "ReviewedByAI" 라벨을 자동으로 붙여요.

**설정 방법:**
1. "GitHub" 노드 추가 (Edit an issue)
2. 라벨에 "ReviewedByAI" 추가

**쉽게 말하면:** "이 코드는 AI가 검토했다는 표시를 달아놔!"

## 🔧 트러블슈팅

### 자주 발생하는 문제들
1. **"터널이 연결되지 않아요"**
   - CloudFlared가 실행 중인지 확인
   - 방화벽에서 5678 포트 차단 여부 확인

2. **"N8N에 접속이 안 돼요"**
   - Docker 컨테이너가 실행 중인지 확인: `docker ps`
   - 포트가 올바른지 확인: `localhost:5678`

3. **"GitHub 웹훅이 작동하지 않아요"**
   - WEBHOOK_URL이 올바른 CloudFlare 도메인인지 확인
   - GitHub에서 웹훅 전송 로그 확인
   - **Access Token 권한 확인** (repo 권한 있는지)

4. **"AI 응답이 오지 않아요"**
   - Gemini API 키가 올바른지 확인
   - API 할당량 초과 여부 확인

5. **"GitHub API 인증 오류"**
   - Access Token이 올바른지 확인
   - Token이 만료되지 않았는지 확인

## 🧪 테스트 방법

### GitHub 자동 리뷰 시스템 테스트하기

위 설정이 완료되면, 이제 시스템 테스트를 할 수 있어요:

### 테스트 준비하기
1. **새 브랜치 만들기**
   ```bash
   git checkout -b feature/test-branch
   ```
   
2. **코드 수정하기**
   - 아무 파일이나 수정 (예: README.md에 한 줄 추가)
   
3. **변경사항 커밋하고 푸시하기**
   ```bash
   git add .
   git commit -m "테스트용 코드 수정"
   git push origin feature/test-branch
   ```
   
4. **GitHub에서 Pull Request 만들기**
   - GitHub 웹사이트에서 `feature/test-branch` → `main`으로 PR 생성
   - 이때 N8N 시스템이 작동합니다! 🎉

> **꿀팁**: main 브랜치에 직접 푸시하면 시스템이 작동하지 않아요! 반드시 별도 브랜치에서 작업 후 PR을 보내야 해요.

### 실제 사용 시나리오

#### 개발자 A의 하루
1. **오전**: `feature/login-page` 브랜치에서 로그인 페이지 개발
2. **오후**: 작업 완료 후 main으로 PR 생성
3. **5분 후**: AI가 자동으로 코드 검토 완료! 
   - "변수명이 더 명확했으면 좋겠어요"
   - "이 함수는 너무 길어요, 분리해보세요"
4. **수정**: AI 피드백을 반영해서 코드 개선
5. **병합**: 리뷰어가 최종 승인 후 main에 병합

## 🎉 완성 후 얻는 것

- **비용 절약**: 완전 무료로 자동 코드 검토! 💰
- **시간 절약**: 기본적인 코드 검토가 자동으로!
- **일관성**: AI는 24시간 같은 기준으로 검토
- **학습 도구**: AI 리뷰를 보며 코딩 실력 향상
- **효율성**: 사람은 더 중요한 검토에 집중 가능
- **실력 향상**: 직접 서버를 운영하며 DevOps 경험!

## 🚨 주의사항

1. **브랜치 작업 필수**: main에 직접 푸시하면 시스템이 작동하지 않아요
2. **컴퓨터 항상 켜두기**: N8N이 동작하려면 컴퓨터가 켜져있어야 해요
3. **보안**: API 키는 절대 공개하지 마세요  
4. **한계**: AI는 완벽하지 않으니 중요한 코드는 사람이 다시 확인하세요
5. **테스트**: 처음에는 작은 변경사항으로 테스트해보세요
6. **전기세**: 24시간 가동시 전기요금이 약간 증가할 수 있어요

## 💡 추가 개선 아이디어

- 특정 파일 타입만 검토하도록 필터링
- 심각도에 따라 다른 라벨 추가
- 슬랙이나 이메일로 알림 보내기
- 여러 AI 모델의 의견 비교하기
- **다른 팀원들과 같은 N8N 인스턴스 공유하기**

## 📚 추가 학습 자료

- **N8N 셀프호스팅 상세 가이드**: https://github.com/PotatoDevel0per/2025MAS/tree/main/n8n-self-hosting
- **CloudFlared 터널 관리**: https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/
- N8N 공식 문서: https://docs.n8n.io
- GitHub API 문서: https://docs.github.com/en/rest
- Docker 기초: https://docs.docker.com/get-started/
- Docker Compose 가이드: https://docs.docker.com/compose/

---

**🎊 축하해요! 이제 완전 무료로 AI 코드 리뷰어를 만들 수 있어요!**

*셀프호스팅이 어렵다면 위의 가이드 문서를 먼저 따라해보세요!*