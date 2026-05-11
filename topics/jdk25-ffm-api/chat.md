# 1:1 Chat — JDK 25 FFM API

사용자와 메인 Claude 의 가벼운 대화 로그. 시간순 누적.

---

## 2026-05-10

- 사용자: 유튜브 영상 "Native Interoperability with JDK 25 and the FFM API" (https://www.youtube.com/watch?v=_4bEHwJ4Bdk) 를 봤음. 영상 요약을 공유하면서, 앞으로 이 주제로 이것저것 물어볼 예정이라고 알림. 요약 핵심:
  - FFM = JNI/ByteBuffer 대체, 100% Java
  - Memory API: `MemorySegment` + `Arena`
  - Layout / VarHandle: C struct 대응 + 안전 접근
  - Function API: `Linker` 로 네이티브 함수 호출
  - jextract: C 헤더 → Java 바인딩 자동 생성
  - 응용: ONNX Runtime 같은 네이티브 라이브러리 활용
- Claude: 새 주제 폴더 `topics/jdk25-ffm-api/` 시작. README/chat 초기화 완료. 다음 질문 받을 준비됨. 1:1 외래로 진행, 깊이 들어가는 비교(JNI vs FFM 성능 트레이드오프 등)나 결정이 필요한 시점에 협진/다학제 전환 제안 예정.

### Q1. 힙 메모리 vs 네이티브 메모리 차이
- 사용자: FFM 맥락에서 힙/네이티브 메모리 구분이 잘 이해 안 됨.
- Claude: 핵심 정리.
  - **힙**: JVM이 OS로부터 받아 자기 안에서 쪼개 쓰는 영역. `new` 로 만든 Java 객체가 거주. GC가 관리·청소·**이동(compaction)** 까지 함. `-Xmx` 적용.
  - **네이티브**: JVM 바깥, OS가 `malloc`/`mmap` 으로 직접 내주는 메모리. C/C++ 가 쓰는 그 영역과 동일. **GC가 손대지 않음 → 주소 고정**. 해제 책임은 개발자(FFM에선 `Arena` 가 대행).
  - **왜 FFM에서 구분이 중요?** 네이티브 함수에 메모리를 넘기려면 주소가 안정적이어야 함. 힙은 GC가 옮길 수 있어서 위험. → 네이티브 함수와 통신할 데이터는 네이티브 세그먼트에 둬야 안전.
  - **MemorySegment 의 통합 추상화**: `MemorySegment.ofArray(arr)` = 힙, `arena.allocate(n)` = 네이티브. 읽고 쓰는 API는 동일.
  - **결정적 vs 비결정적 해제**: GC는 비결정적, `Arena.close()` 는 결정적. FFM이 "deterministic lifecycle" 이라 부르는 이유.

### Q2. 네이티브 메모리를 정밀하게 쓰면 얻는 이점
- 사용자: OS 물리 메모리를 더 정밀하게 다뤄서 얻는 이점이 뭐냐?
- Claude: 7가지로 정리.
  1. **GC 압박/STW 감소**: 큰 버퍼가 힙 밖에 있으면 GC가 스캔·이동 안 함. 저지연 시스템에 유리.
  2. **`-Xmx` 무관**: 힙 작게 두고도 수십 GB 데이터 다룸 (Cassandra/Lucene/Netty 패턴).
  3. **Zero-copy**: 네이티브 함수에 주소만 넘김. JNI의 pin/copy 오버헤드 사라짐. ONNX/OpenSSL 등 통합에 결정적.
  4. **결정적 해제**: try-with-resources + Arena → 큰 자원 즉시 반납, 메모리 사용량 예측 가능.
  5. **mmap**: 거대 파일을 메모리처럼. DB·검색엔진·로그 분석.
  6. **레이아웃 통제 → 캐시 친화성**: C struct와 동일한 바이트 배치, 객체 헤더 오버헤드 0, flat 버퍼.
  7. **외부 메모리 직접 접근**: IPC 공유메모리, GPU/FPGA, DMA 버퍼 — 힙으론 불가.
- 균형: 평범한 비즈니스 코드는 힙 그대로가 정답. FFM은 "필요할 때 꺼내 쓰는 칼".

### Q3. Spring MVC에서 3GB 데이터 받으면 "네이티브에 담아"라는 order인가?
- 사용자: Spring MVC 예시. 3GB 데이터를 받으면 무거우니까 특정 영역에선 "네이티브에 담아"라는 명령이 되는 거냐?
- Claude: 방향은 맞지만 두 가지 정정.
  1. **자동 아님, 명시적**: JVM이 자동으로 옮기지 않음. 개발자가 `Arena.allocate(3GB)` 라고 코드에 쓰는 순간 거기로 감. `byte[]` 는 항상 힙. 또한 `byte[]` 는 `int` 인덱스라 2GB 한도, 네이티브는 `long` 이라 한도 없음.
  2. **트리거는 "크기"가 아니라 "용도"**: Spring MVC에서 3GB 데이터의 1순위 해법은 **스트리밍**(청크 처리)이지 네이티브가 아님. 디스크 임시 저장이 2순위. 네이티브는 다음 경우에만:
     - ONNX/네이티브 모델에 zero-copy로 넘겨야 함
     - mmap으로 파일을 메모리처럼 다룰 때
     - 요청 간 공유되는 거대 캐시 (GC 부담 회피)
     - 네이티브 코덱·암호화·압축 라이브러리 사용
- 핵심 질문: **그 데이터가 네이티브 세계와 만나야 하는가, GC 바깥에서 살아야 하는가** — Yes 일 때만 FFM.

### Q4. "어쩔 수 없이" 3GB를 통째로 들고 있어야 한다면? (전제 안에서)
- 사용자: 일반적 대안(스트리밍 등)은 알고 있음. 그 가정 안에서 힙 vs 네이티브 차이를 보고 싶음.
- Claude: (이전 답변 우회 해법 제시한 것 사과하고 가정 안에서 재답변)
  - **힙에 두면**:
    1. `byte[]` 는 int 인덱스라 통짜 3GB 불가, `byte[][]` 로 쪼개야 함 → 코드 더러워짐
    2. GC가 스캔·이동 시도 (Young→Old 승격, compaction). STW 초 단위로 튐 → 같은 JVM의 다른 요청 얼어붙음
    3. `-Xmx` 4–5GB 이상 잡아야 함. 컨테이너 메모리도 같이 올려야
    4. 동시 두 요청 시 6GB → GC 스파이럴, OOM. 요청 단위 격리 실패
    5. heap dump 무거워짐
  - **네이티브에 두면**:
    1. long 오프셋, 한도 없음. 단일 세그먼트로 단순한 코드
    2. GC 무관 → 다른 요청의 pause 시간 영향 0
    3. `-Xmx512m` 짜리 JVM이 3GB 데이터 다룸. 힙=비즈니스, 네이티브=페이로드 분리
    4. try-with-resources로 즉시 반환, 다음 요청이 바로 자리 씀
    5. 한 요청의 거대 페이로드가 다른 요청에 영향 안 줌 → **테일 레이턴시 안정**
  - 핵심 차이: **거대 데이터 보유 중에 같은 인스턴스의 다른 요청들이 영향받느냐**. Cassandra/Lucene/Netty가 일찍 off-heap 간 이유, FFM은 이걸 표준 API로 만든 것.

### Q5. 우회책이 원리적으로 안 통하는, 진짜 어쩔 수 없는 시나리오는?
- 사용자: 그럼 일반적 대안 다 안 통하는 진짜 사용처가 뭐냐?
- Claude: 6가지 카테고리.
  1. **네이티브 API가 통짜 버퍼 요구** — ML 추론(ONNX/TF Lite/llama.cpp), BLAS/LAPACK, 비디오 코덱. 함수 시그니처 자체가 청크 못 받음.
  2. **mmap이 유일한 답** — 메모리보다 큰 데이터셋(50GB 인덱스), 거대 모델 가중치(Llama 70B = 140GB), 임베디드 DB(RocksDB/LMDB).
  3. **요청 간 공유 hot 캐시** — 수 GB 룩업 테이블·임베딩 사전. 힙=GC 망가짐, Redis=네트워크 왕복으로 µs→ms. 같은 프로세스 + GC 회피 둘 다 필요.
  4. **외부 주소 그 자체** — GPU 버퍼(CUDA), 공유메모리 IPC, 메모리 매핑 디바이스(PCI/FPGA), DPDK ring buffer. "스트리밍 우회"라는 개념 자체가 성립 안 함.
  5. **저지연 SLA** — HFT, 게임 서버 tick, RT 신호처리. ZGC/Shenandoah로도 한계.
  6. **데이터 포맷 자체가 네이티브** — Apache Arrow, FlatBuffers, Cap'n Proto, Parquet zero-copy. 힙 복사하는 순간 설계 목적 사라짐.
- 핵심 규칙: **JVM 밖과 만나는 순간** 네이티브 메모리 등장. JVM 안에서 끝나면 대부분 우회 가능.

### Q6. Java가 AI 생태계와 엮이려는 흐름이 보이는데?
- 사용자: 직관적으로 그게 느껴짐.
- Claude: 정확. JDK 22~25의 큰 변화가 한 방향("Java에서 ML 일급으로")을 가리킴.
  - **JDK 차원**:
    - FFM API (JEP 454, JDK 22 final, 25 유지) — motivation에 ONNX/TF 명시
    - **Project Babylon** — Code Reflection + HAT. Java 람다를 GPU 커널로 컴파일. JAX/Triton의 Java 판
    - Vector API (JDK 25 = 9th Incubator) — SIMD로 행렬 연산 가속
    - Virtual Threads + ScopedValue — LLM 호출처럼 외부 IO·고동시성에 적합
  - **생태계**:
    - Spring AI 1.0 정식 (2025) — 타이밍 우연 아님
    - LangChain4j, Quarkus LangChain4j
    - DJL (AWS) — JNI에서 FFM으로 이동 중
    - Tribuo (Oracle), llama.cpp Java bindings
  - **왜 지금**:
    1. 엔터프라이즈 코어는 여전히 Java. LLM 기능이 그 위로 밀려들어옴
    2. 마이크로서비스 분리보다 인-프로세스 추론이 매력적 (네트워크 오버헤드 회피)
    3. 거의 모든 백엔드가 임베딩/RAG/요약 호출자가 됨
    4. Python venv·GPU 드라이버 의존성 지옥에서 운영 측 도피처
  - 부제: JDK 17 = "현대화", JDK 25 = "AI/네이티브 시대 준비".

### 📌 2026-05-11 Notion 등재
- AI-Meeting DB에 회의록(잠정)으로 등록
- URL: https://www.notion.so/35dc2a937798816b8fa9d529449810cb
- DB 스키마 확장: Topic에 `jdk25-ffm-api`, Tags에 `java/jdk25/ffm/native-memory/ai-integration/study-note` 추가
- 본문 구성: 한줄요약 → 출처 → 인덱스 → Q1~Q6(각 💡한줄핵심+표/코드) → 큰그림 → 다음 주제
- 후속 수정: "빠른 인덱스" → "목차"로 리네임, 수동 목록을 `<table_of_contents/>` 블록으로 교체 (Q1만 다른 페이지로 잘못 자동 멘션되던 문제도 같이 해결)
