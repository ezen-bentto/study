- Redis
    
    <aside>
    
    💡 레디스란?
    
    메모리 기반의 키-값 데이터베이스
    
    </aside>
    
    ### Redis 설치
    
    **설치 방법**
    
    1. 로컬 (무료)
    2. 클라우드 (프리티어 무료)
        
        ⇒ 우선 로컬에 설치 후 클라우드로 전환 예정
        
    
    **로컬 설치 방법**
    
    공식적으로 윈도우는 지원하지 않음
    
    1. WSL2 (Windows Subsystem for Linux) 방법 → 리눅스 설치 후 레디스 설치
    2.  [Microsoft Archive Redis Github](https://github.com/microsoftarchive/redis)에서 배포된 zip 또는 msi파일 다운 후 설치
        
        → 2016년 1월에 배포된 버전이라 오래됨 (2025년 기준 7.x)
        
    
    **msi 파일 다운 후 설치 방법**
    
    1. https://github.com/microsoftarchive/redis/releases **접속**
    2. [**Redis-x64-3.0.504.msi](https://github.com/microsoftarchive/redis/releases/download/win-3.0.504/Redis-x64-3.0.504.msi) 다운받기**
    3. Next > 폴더 확인 > 포트번호 6379(기본) > Set the Max Memory Limit 체크 해제 
    4. 설치된 폴더로 들어가 redis-server.exe 실행
    5. redis-cli.exe  실행 후 ping 입력 시 pong 출력 되면 성공
    
    ### Node.js에서 Redis 모듈 사용 방법
    
    <aside>
    
    `yarn add redis@^4.6.13` // 그냥 최신 5.x로 설치하면 호환성 문제 발생
    
    `yarn add -D @types/redis`
    
    </aside>
    
    ```jsx
    .env
    # ────────────── REDIS ──────────────
    REDIS_HOST=localhost
    REDIS_PORT=6379
    
    # 클라우드용(아직은X)
    REDIS_USERNAME=default
    REDIS_PASSWORD=abcdefghijklmnopqrstuvwsyz
    ```
    
    ```
    // config > env.config.ts
    redis: {
      host: process.env.REDIS_HOST!,
      port: Number(process.env.REDIS_PORT!),
    }
    ```
    
    ```
    // config > redis.config.ts
    import { createClient } from 'redis';
    import { ENV } from './env.config';
    
    export const client = createClient({
      socket: {
        host: ENV.redis.host,
        port: ENV.redis.port,
      },
      legacyMode: true, // v3, v4 문법 호환
    });
    
    client.on('error', (error) => {
      console.error('Redis error:', error);
    });
    
    client.on('connect', () => {
      console.log('Redis connected');
    });
    
    // Redis 연결 초기화 함수
    export const connectRedis = async () => {
      try {
        if (!client.isOpen) {
          await client.connect();
        }
      } catch (error) {
        console.error('Redis connection failed:', error);
        throw error;
      }
    };
    
    /
    connectRedis();
    ```
    
- 캐싱 예시 코드
    
    ### 2. 조회수 처리 로직 (중복 방지 + 캐시 저장)
    
    ```tsx
    // viewController.ts
    import redisClient from './redisClient';
    // 사용자 IP를 식별하는 유틸 >> 이거 그냥 우리 ip 하면 받아짐
    import { getUserIP } from './utils'; 
    
    export const handlePostView = async (postId: string, req: any) => {
      const userIP = getUserIP(req); // 사용자 식별 (예: IP 또는 로그인 유저 ID)
      const viewKey = `post:${postId}:views`;
      const userViewKey = `post:${postId}:user:${userIP}`;
    
      // 짧은 시간(예: 1시간) 내 중복 조회 방지
      const alreadyViewed = await redisClient.exists(userViewKey);
      if (alreadyViewed) return;
    
      // 유저 조회 기록 설정 (TTL 1시간)
      await redisClient.set(userViewKey, '1', { EX: 60 * 60 });
    
      // 전체 조회수 증가 (Redis에 임시로 저장)
      await redisClient.incr(viewKey);
    };
    
    ```
    
    ---
    
    ### 3. 일정 시간마다 Redis 데이터를 DB로 옮기는 작업 (예: 5분 간격)
    
    ```tsx
    import { client } from '@/config/redis.config';
    import { ContestModel } from '@/models/contest.model';
    // 필요하다면 CommunityModel, PolicyModel도 import
    import { CommunityModel } from '@/models/community.model';
    import { PolicyModel } from '@/models/policy.model';
    
    export const syncViewsToDb = async (target: 'contest' | 'community' | 'policy') => {
      try {
        const keys = await client.v4.keys(`${target}:*:views`);
    
        for (const key of keys) {
          const id = key.split(':')[1]; // 예: contest:123:views
          const views = await client.v4.get(key);
          if (!views) continue;
    
          const viewsCount = parseInt(views);
          if (isNaN(viewsCount)) continue;
    
          switch (target) {
            case 'contest':
              await ContestModel.incrementViews(id, viewsCount);
              break;
            case 'community':
              await CommunityModel.incrementViews(id, viewsCount);
              break;
            case 'policy':
              await PolicyModel.incrementViews(id, viewsCount);
              break;
            default:
              console.warn(`[syncViewsToDb] 알 수 없는 매개변수: ${매개변수}`);
              continue;
          }
    
          await client.v4.del(key); // Redis 키 삭제
        }
    
        console.log(`[syncViewsToDb] ${keys.length}건 (${매개변수}) 동기화 완료`);
      } catch (err) {
        console.error(`[syncViewsToDb] ${매개변수} 조회수 동기화 실패:`, err);
      }
    };
    
    ```
    
    ```sql
    CREATE TABLE contest (
      id SERIAL PRIMARY KEY,
      writer_id VARCHAR(255) NOT NULL,
      title VARCHAR(255) NOT NULL,
      img TEXT NOT NULL,
      organizer VARCHAR(255) NOT NULL,
      prize TEXT NOT NULL,
      start_date DATE NOT NULL,
      end_date DATE NOT NULL,
      homepage TEXT NOT NULL,
      participants TEXT NOT NULL,
      benefits TEXT NOT NULL,
      contest_tag VARCHAR(255) NOT NULL,
      article TEXT NOT NULL,
      views INTEGER DEFAULT 0
    );
    ```
    
    ```sql
    INSERT INTO contest (
      writer_id, title, img, organizer, prize,
      start_date, end_date, homepage, participants,
      benefits, contest_tag, article, views
    ) VALUES (
      'user_001',
      '2025 청년 디자인 공모전',
      'https://example.com/images/design2025.jpg',
      '한국디자인진흥원',
      '최우수상 500만원, 총 상금 1,000만원',
      '2025-07-01',
      '2025-08-15',
      'https://designcontest2025.or.kr',
      '만 19세 이상 34세 이하 누구나',
      '입상작 전시 및 포트폴리오 제공',
      '디자인,청년,공모전',
      '청년 디자이너를 위한 전국 단위의 공모전입니다. 창의적인 아이디어를 기다립니다.',
      152
    );
    
    INSERT INTO contest (
      writer_id, title, img, organizer, prize,
      start_date, end_date, homepage, participants,
      benefits, contest_tag, article, views
    ) VALUES (
      'user_002',
      '2025 환경 아이디어 공모전',
      'https://example.com/images/environment2025.jpg',
      '환경부',
      '총 상금 2,000만원 및 채용 가산점',
      '2025-06-10',
      '2025-07-20',
      'https://envidea2025.kr',
      '전 국민 누구나 참여 가능',
      '우수 아이디어 실현 기회 제공',
      '환경,아이디어,친환경',
      '기후 위기를 극복할 창의적인 환경 아이디어를 모집합니다.',
      234
    );
    
    INSERT INTO contest (
      writer_id, title, img, organizer, prize,
      start_date, end_date, homepage, participants,
      benefits, contest_tag, article, views
    ) VALUES (
      'user_003',
      '2025 글로벌 스타트업 챌린지',
      'https://example.com/images/startup2025.jpg',
      '중소벤처기업부',
      '우승 상금 1억원 및 글로벌 진출 기회',
      '2025-09-01',
      '2025-10-31',
      'https://globalstartup2025.go.kr',
      '예비 창업자 및 스타트업',
      '해외 투자자 미팅 및 창업 공간 제공',
      '스타트업,창업,글로벌',
      '전 세계 창업가들이 경쟁하는 글로벌 무대, 지금 도전하세요!',
      387
    );
    
    ```
    

## ✅ EC2에 Redis 설치 순서 (Ubuntu 기준)

### 1. 필수 패키지 설치

```bash
bash
복사편집
sudo apt update
sudo apt install build-essential tcl -y

```

- `build-essential`: `make` 및 컴파일 도구
- `tcl`: Redis 테스트용 도구 (선택 사항이지만 권장)

---

### 2. Redis 소스 코드 다운로드

```bash
bash
복사편집
wget http://download.redis.io/releases/redis-4.0.13.tar.gz

```

---

### 3. 압축 해제

```bash
bash
복사편집
tar -xzf redis-4.0.13.tar.gz
cd redis-4.0.13

```

---

### 4. 빌드 (컴파일)

```bash
bash
복사편집
make

```

- 컴파일 완료 후, 실행파일은 `src/` 디렉터리에 생성됨

---

### 5. Redis 서버 실행

```bash
bash
복사편집
src/redis-server

```

- 정상 실행 시, Redis ASCII 아트와 함께 포트 6379에서 대기 상태 표시됨

---

### 6. 동작 확인

새 터미널에서:

```bash
bash
복사편집
cd redis-4.0.13
src/redis-cli ping

```

- 결과: `PONG` → 정상 작동 확인

---

필요하면 `--daemonize yes` 옵션으로 백그라운드 실행도 가능해요:

```bash
bash
복사편집
src/redis-server --daemonize yes

```
