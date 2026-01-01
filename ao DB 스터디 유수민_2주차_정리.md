# 2주차 — 서버에서 DB로 가는 길

**핵심 질문**

- 서버는 DB랑 어떻게 “연결”되는가?

**내용**

- TCP/IP, Socket
- DB Connection이란 정확히 뭔가 (구체적을 어디서 어디로?)
- 요청 1번 = 커넥션 1번?
- 왜 Connection Pool이 필요한가

**키워드**

`TCP`, `Socket`, `DB Connection`, `Latency`

---

### 1) 서버와 DB의 연결 메커니즘 (TCP/IP & Socket)

서버(Client)가 DB(Server)에 접속한다는 것은 단순한 함수 호출이 아니라 네트워크를 통한 프로세스 간 통신(IPC)입니다.

- **Socket (소켓):**
    - 네트워크 상에서 두 프로그램이 데이터를 주고받기 위한 논리적인 단말기(Endpoint)입니다.
    - 서버(WAS)는 DB의 **IP 주소**와 **Port 번호**(예: MySQL 3306)를 타겟으로 소켓을 생성(Open)합니다.
- **TCP 3-Way Handshake:**
    - 신뢰성 있는 연결을 위해 TCP 프로토콜은 데이터를 보내기 전 3단계의 확인 과정을 거칩니다.
    - `SYN` (연결 요청) → `SYN-ACK` (요청 수락 및 확인) → `ACK` (확인)
    - **중요:** 이 과정은 물리적인 네트워크 왕복 시간(RTT)이 소요되므로 **비용이 비싼 작업**입니다.
- **DB 인증 및 세션 생성:**
    - TCP 연결 후, DB 레벨에서 ID/PW 인증(Authentication)을 수행합니다.
    - 인증이 성공하면 DB는 내부 메모리에 해당 연결을 위한 세션(Session)을 생성하고 상태를 관리합니다.

### 2) DB Connection이란 정확히 무엇인가?

"Connection을 맺었다"는 것은 아래의 상태(State)가 유지되고 있음을 의미합니다.

- **네트워크 레벨:** 클라이언트(WAS)와 DB 서버 간의 TCP 소켓이 `ESTABLISHED` 상태로 유지됨.
- **서버(WAS) 레벨:** 소켓을 관리하는 객체가 메모리에 할당됨.
- **DB 레벨:** 해당 클라이언트를 전담하는 **스레드(Thread) 혹은 프로세스**가 할당되고, 쿼리 처리를 위한 **전용 메모리 공간**(예: Work Memory, Sort Buffer)이 확보됨.

### 3) 요청 1번 = 커넥션 1번? (The Anti-Pattern)

과거의 CGI 방식이나 잘못된 설계에서는 요청(Request)이 들어올 때마다 커넥션을 맺고 끊었습니다.

- **프로세스:**
    1. Client 요청 진입
    2. DB 접속 (`connect()`: Handshake + 인증 → **약 0.1~0.5초 소요**)
    3. SQL 실행 (`select ...`: **약 0.005초 소요**)
    4. DB 접속 해제 (`close()`)
    5. Client 응답
- **문제점 (Latency 오버헤드):**
    - 실제 데이터를 가져오는 시간(SQL 실행)보다 **연결을 맺는 시간(Handshake)이 훨씬 깁니다.**
    - 트래픽이 증가하면 DB는 연결/해제 처리에 CPU를 낭비하여 정작 쿼리를 처리할 여력이 없어집니다.

### 4) Connection Pool (커넥션 풀)의 필요성

위의 비효율을 해결하기 위해, **미리 일정 개수의 연결(Connection)을 맺어두고 재사용하는 기법**입니다.

- **동작 원리:**
    1. **초기화:** WAS가 켜질 때 DB와 미리 N개의 연결을 맺어 Pool(웅덩이)에 보관.
    2. **대여 (Borrow):** 요청이 들어오면 Pool에서 노는 커넥션을 하나 꺼내 줌.
    3. **사용:** SQL 실행.
    4. **반납 (Return):** 연결을 끊지 않고(`close` 하지 않고) 다시 Pool에 돌려놓음.
- **효과:** TCP Handshake 및 DB 인증 비용을 **0**으로 만듭니다.
    
    ![스크린샷 2025-12-31 15.21.14.png](2%EC%A3%BC%EC%B0%A8%20%E2%80%94%20%EC%84%9C%EB%B2%84%EC%97%90%EC%84%9C%20DB%EB%A1%9C%20%EA%B0%80%EB%8A%94%20%EA%B8%B8/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-12-31_15.21.14.png)
    

### 5) DBCP 설정 방법 (MySQL)

DB connection은 backend server와 DB 서버 사이의 연결을 의미하기 때문에 backend server와 DB 서버 각각에서의 설정 방법을 잘 알고 있어야함. 양쪽의 설정을 맞춰주는 것이 핵심.

- DB 서버 설정 (MySQL) - 중요한 두가지 파라미터
    1. max_connections : client와 맺을 수 있는 최대 connection 수
        
        →이 수치를 넘어가면 새로운 연결 요청은 거부됨. (Too many connections 에러)
        
        ex) 만약 max_connections 수가 4, DBCP의 최대 connection 수가 4라면? 
        
        → 트래픽이 많이 몰려오면 그 요청들을 다 처리하기 위해 백엔드 서버가 바쁘게 일을 함. 그러면 DB 서버로 오는 요청들도 많아지고 그만큼 connection도 바쁘게 사용이 됨. 따라서 백엔드서버가 과부화가 옴. 이걸 해결하기 위해 이때 새로운 동일한 백엔드 어플리케이션 서버를 추가하여 새 connection을 추가하려고 하면(트래픽 분산 목적) max_connections 수가 4이기때문에 이미 connection 수가 4라서 더 이상의 connection은 불가능함. 따라서 파라미터를 잘 설정해야 신규 connection 추가나 DBCP 내 connection 늘리는 것에도 에러가 발생하지 않고 정상적으로 동작함.
        
    2. wait_timeout : connection이 inactive 할 때 다시 요청이 오기까지 얼마의 시간을 기다린 뒤에 close할 것인지를 결정
        
        → 비정상적인 connection 종료, connection 다 쓰고 반환이 안됨, 네트워크 단절 등의 이유로 connection상태가 비정상적임에도 DB 서버는 connection이 비정상적인 줄 모르고 하염없이 요청을 기다리게 됨. 이런 상태가 많아지다 보면 리소스도 점유하고 DB 서버에 안좋은 영향을 끼치기 때문에 적절한 시점에 이 문제를 해결해줘야함. 그때 사용하는게 이 파라미터인것. 정한 시간 내에 요청이 도착하면 0으로 초기화.
        
        → 비정상적으로 종료된 커넥션이 계속 자원을 점유하는 것을 방지.
        
        → 이 시간은 뒤에 나올 DBCP의 maxLifetime 보다 반드시 길어야 함.
        
- DBCP 설정 (HikariCP) - 중요한 네가지 파라미터
    1. minimumIdle : pool에서 유지하는 최소한의 idle connection(요청이 올때까지 대기하고 있는 connection, 유휴자원) 수
        
        →idle connection 수가 minimumIdle보다 작고 전체 connection 수도 maximumPoolSize보다 작다면 신속하게 추가로 connection을 만듦. 기본 값은 maximumPoolSize와 동일(= pool size 고정)
        
        →과거(Commons DBCP)에는 메모리 아끼려고 최소 유휴 자원(`minimumIdle`)을 작게 잡았지만, **HikariCP(최신 표준)는 `minimumIdle` = `maximumPoolSize`로 똑같이 맞추는 것을 권장**합니다. (Fixed Size)
        
        →**이유:** 트래픽이 튈 때 커넥션을 새로 만드는 비용보다 그냥 메모리 좀 더 쓰고 미리 다 만들어두는 게 성능상 훨씬 이득이기 때문입니다.
        
    2. maximumPoolSize : pool이 가질 수 있는 최대 connection(idle과 active(in-use) connection 합친 것) 수
        
        → ex) mininumIdle이 2, maximumPoolSize가 4라면? 
        
    3. maxLifetime : pool에서 connection의 최대 수명
        
        →maxLifetime을 넘기면 idle일 경우 pool에서 바로 제거, active인 경우 pool로 반환된 후 제거. pool로 반환이 안되면 maxLifetime 동작 안함. 즉, pool로 반환을 잘 시켜주는 것이 중요함. DB의 connection time limit보다 몇 초 짧게 설정해야 함.
        
        ex) 만약 DB의 wait_timeout이 60초, DBCP의 maxLifetime이 60초라면?
        
        →반드시 **`maxLifetime` < `wait_timeout`** 이어야 함. (보통 30~60초 정도 짧게 설정) 
        
        둘의 시간이 똑같으면 WAS가 커넥션을 쓰려고 막 꺼내는 순간(0.001초 차이로) DB가 "시간 다 됐네?" 하고 끊어버리는 Race Condition(경쟁 상태)이 발생할 수 있음. 이러면 WAS에서 갑자기 에러가 터짐.
        
    4. connectionTimeout : pool에서 connection을 받기 위한 대기 시간
        
        ex) 만약 connectionTimeout이 30초라면? 
        
        → 트래픽이 많이 몰려올때 요청을 무한정 기다리게 할 수 없으니 30초가 넘어도 connection을 못받은 요청은 끊어줌. 그 과정을 수행하는 파라미터. 실무에선 client는 10초 이상 기다리지 않으므로 적절하게 이 파라미터를 설정하여 사용해야 함.
        
    
    ### ***파라미터를 적절하게 설정하는 것이 중요!!***
    

### 6) 적절한 connection 수를 찾기 위한 과정

- **모니터링과 지표:** DB 커넥션 풀의 상태를 실시간으로 관찰하기 위해 다음 지표들을 추적해야 합니다.
    - **Active Connections:** 현재 사용 중인 커넥션 수
    - **Idle Connections:** 대기 중인 커넥션 수
    - **Wait Count:** 커넥션을 얻기 위해 대기한 요청 수
    - **Wait Time:** 평균 대기 시간
- **성능 테스트:** 실제 서비스 환경과 유사한 부하 테스트(naver의 nGrinder)를 통해 최적의 커넥션 수를 찾아야 합니다.
    - 단계적으로 풀 사이즈를 늘려가며 TPS(Transaction Per Second)와 응답 시간을 측정
    - 특정 지점 이후로는 커넥션을 늘려도 성능이 개선되지 않거나 오히려 저하되는 구간 확인
- **경험적 공식:** `connections = ((core_count * 2) + effective_spindle_count)` (HikariCP 공식 문서 권장)
    - 예: 4코어 DB 서버 + SSD(spindle=1) → 약 9~10개가 적정선
    - 다만 워크로드 특성(CPU-bound vs I/O-bound)에 따라 달라질 수 있으므로 반드시 실측 필요

---

## 2. 관점별 심화 분석

### 1) 🌐 네트워크 (Network) 관점

- **TCP Keep-Alive:** 연결을 맺어두고 오랫동안 데이터 전송이 없으면, 중간의 방화벽이나 라우터가 연결을 끊어버릴 수 있습니다. 이를 방지하기 위해 주기적으로 패킷을 보내 연결을 유지해야 합니다.
- **대역폭(Bandwidth)과 패킷:** 커넥션 생성 시 발생하는 패킷은 작지만 빈도가 높으면 네트워크 스위치에 부하를 줄 수 있습니다. Pooling은 이를 방지합니다.

### 2) 💻 운영체제 (OS) 관점

- **File Descriptor (FD):** 리눅스(Unix)에서 소켓은 **파일**로 취급됩니다.
    - OS는 프로세스당 열 수 있는 파일의 개수(`ulimit -n`)를 제한합니다.
    - 커넥션이 반납되지 않고 누수(Leak)되어 계속 늘어나면 `Too many open files` 에러가 발생하며 서버가 멈춥니다.
- **Context Switching:** DB 서버 입장에서 커넥션 하나는 보통 하나의 스레드/프로세스입니다. 커넥션이 너무 많으면(수천 개), CPU가 스레드를 교체하는 비용(Context Switching)이 커져 성능이 오히려 저하됩니다.

### 3) 🗄️ 데이터베이스 (DB) 관점

- **Max Connections 설정:** DB마다 허용 가능한 최대 연결 수가 정해져 있습니다. (예: MySQL `max_connections` 기본값 151).
    - WAS가 여러 대로 늘어날 경우, (WAS 수 × WAS당 풀 사이즈)가 DB의 `max_connections`를 초과하지 않도록 설계해야 합니다.
- **메모리 점유:** 연결이 맺어져 있는 것만으로도 DB의 RAM을 차지합니다. 사용하지 않는 과도한 Idle Connection은 메모리 낭비입니다.

### 4) ☁️ 클라우드 (Cloud/Infra) 관점

- **Scale-out 문제:** 클라우드 환경에서 WAS는 오토스케일링으로 쉽게 늘어납니다.
    - WAS가 갑자기 10대에서 100대가 되면, DB 커넥션 요청이 폭주(Thundering Herd)하여 DB가 다운될 수 있습니다.
- **Proxy Layer (예: AWS RDS Proxy):** 이를 해결하기 위해 WAS와 DB 사이에 **커넥션 풀링 전용 미들웨어(Proxy)**를 둡니다. WAS가 많아져도 DB로 들어가는 실제 연결 수는 일정하게 조절(Multiplexing)합니다.

---

## 3. 다양한 상황별 동작 및 대응 (Scenario)

### 1) ⚠️ 장애 상황: Connection Pool 고갈 (Exhaustion)

- **현상:** 모든 커넥션이 사용 중(`Active`)이라, 새로운 요청이 커넥션을 얻지 못하고 대기하다가 `Timeout` 에러 발생.
- **원인:**
    1. 트래픽 폭주.
    2. **Slow Query:** 특정 쿼리가 3초씩 걸린다면, 그동안 커넥션을 붙잡고 있게 되어 회전율이 급감함. (가장 흔한 원인)
    3. **Transaction Lock:** DB 락 때문에 커넥션이 반납되지 않음.
- **대응:** 풀 사이즈를 늘리는 것은 임시방편. 근본적으로 **Slow Query 튜닝** 및 트랜잭션 범위를 최소화해야 함.

### 2) 🔌 네트워크 단절 및 재연결 (Stale Connection)

- **현상:** DB 서버가 재부팅되거나 네트워크가 일시적으로 끊김. WAS의 Pool에 있는 커넥션은 끊긴 줄 모르고 "유효하다"고 생각함. 요청 시 `Broken Pipe` 또는 `Connection Reset` 에러 발생.
- **대응 (자동화):**
    - **Validation Query:** 커넥션을 빌려주기 전에 `SELECT 1` 같은 가벼운 쿼리를 날려 연결 상태를 확인 (구식 방식).
    - **Connection Test (최신):** JDBC 드라이버 자체의 `isValid()` 기능을 사용하여 소켓 레벨에서 생존 여부를 체크. (HikariCP 기본 동작).

### 3) 📈 확장 상황: WAS Scale-out 시 설정

- **상황:** WAS 1대(Pool 50개) → WAS 10대 확장.
- **계산:** 총 커넥션 500개. DB `max_connections`가 300이라면 장애 발생.
- **대응:** WAS 대수가 늘어나면 **개별 WAS의 Pool Size를 줄여야 함** (예: 50개 → 20개). 혹은 앞서 언급한 RDS Proxy 같은 중간 계층 도입.