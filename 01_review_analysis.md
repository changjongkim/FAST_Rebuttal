# FAST '27 #165 — 리뷰 분석

원문: `00_reviews_raw.md`. 작성 2026-11-17. 코드와 결과 파일로 확인한 사실은 **[확인]**, 아직 확인 중인 것은 **[검증중]**으로 표시했다.

태그: **[FACT]** 리뷰어의 사실 오류 정정, **[CLAR]** 논문에 있는데 잘못 읽힌 것, **[ANSWER]** 질문에 직접 답, **[ADMIT]** 인정하고 수정 약속, **[NEW]** 새 측정이 필요, **[PRES]** 표현과 그림 수정.

---

## 0. 판세

| 리뷰 | 점수 | 전문성 | 핵심 불만 | 돌릴 수 있나 |
|---|---|---|---|---|
| A | 2 WR | 3 | 이득의 출처(transport), 동시성 프로토콜, INT8 품질, 낮은 공유율 | 가능. 대조 실험이나 범위 한정을 받아들이겠다고 직접 씀 |
| B | 2 WR | 4 Expert | 이득의 출처(압축, block size), 인덱스 이해 불가 | 가능성 큼. 핵심 전제 두 개가 E2E에는 사실이 아님 |
| C | 3 WA | 3 | HPC 한정 범위, DeepSeek KV 축소, agentic 데이터셋, salting | 유지가 목표 |
| D | 4 A | 2 | 읽기 어려움, 통계, 그림 | 유지. **D가 @A1 코멘트로 답변 우선순위를 정함** |
| E | 3 WA | 2 | transport와 압축 비대칭, INT8 품질과 decode, GC, 장애 | 유지 또는 상향 |

평균 2.8로 경계선이다. 결과는 A나 B 중 한 명을 움직이느냐에 달렸고, B가 Expert라 무게가 가장 크다. D의 코멘트는 E의 질문 넷에 B의 block size와 C의 salting을 합친 것이고 사실상 PC가 본 핵심 쟁점 목록이다. 답변은 이 순서를 그대로 따른다.

**공통 1순위 쟁점:** A, B, E와 D#1 모두 같은 질문이다. 이득이 설계에서 왔는가, 빠른 네트워크(CXI 대 TCP)에서 왔는가, 압축(INT8)에서 왔는가, block size에서 왔는가.

---

## 1. 새로 확인한 사실 (답변의 축)

### F1. E2E 실험은 압축이 꺼져 있다 [확인]
- `cascade-vllm/scripts/run_direct_llm.py:775`, `run_open_loop.py:205`, `cascade-sglang/cascade_sglang/backend.py:124`, `radix_cache.py:105`가 모두 `"kv_compression": False`를 넣는다. C++ 기본값도 false다(`cascade_distributed.hpp:209`).
- 실제 실행 로그에서 `KV compression: OFF`가 vLLM 20회, SGLang 19회 찍혔다. ON은 0회다.
- 따라서 **헤드라인 E2E 수치(9.39-16.75x TTFT, 7.17-12.42x throughput, SGLang 2.95-3.03x)는 BF16 페이지를 그대로 옮긴 무손실 결과**다. B의 압축이 불공정하다는 지적과 E의 데이터 압축으로 전송량을 줄였다는 지적은 E2E에는 해당하지 않는다.
- 단, **논문 §4.6-4.8은 E2E를 compressed RDMA retrieval이라고 잘못 썼다** (Weak/Strong vLLM, SGLang, Breakdown, Load Stress 문단). 리뷰어 오독의 원인이 우리 문장이므로 [ADMIT]로 함께 인정해야 한다.
- 반면 **trace-driven 실험(§4.2-4.5)은 INT8을 켰다**. `script_example/benchmark/adapters/cascade_adapter.py:71`에 `cfg.kv_compression = True`가 하드코딩돼 있다. 이 구간은 압축이 저장 계층 baseline 대비 이득에 섞여 있다.

### F2. E2E의 block 크기는 baseline보다 작다 [확인]
- vLLM E2E에서 CASCADE의 단위는 vLLM 16-token 페이지다. TP rank당 `page_bytes` 1,310,720 = 80 layers × K,V × 16 tokens × 2 heads × 128 × 2 bytes이고, 8K prefix 요청 하나에 512 블록이 쓰인다(32n 결과의 `load_blocks`).
- SGLang E2E는 64-token 페이지다(`cascade-sglang/scripts/run_two_pass.py:452`, `--page-size` 기본 64).
- LMCache(Disk, Redis, P2P)와 Mooncake는 `chunk_size` 256을 썼다(`run_direct_llm.py:589`, `scripts/mooncake_config.py:66`). vLLM의 Mooncake도 LMCache의 Mooncake backend로 붙어 있다(결과 폴더의 `lmcache_mooncake_rank*.yaml`).
- 따라서 B의 지적(CASCADE는 1024 token이고 baseline은 256)은 **trace-driven에만 해당**한다(Llama 512, Qwen 1024 KV pairs). E2E에서는 CASCADE 쪽이 더 잘게 쪼갠다. E의 블록이 너무 크다는 지적도 E2E에는 해당하지 않는다.
- [검증중] §4.7 Table 3(E2E dedup)은 320MB blocks라고 쓰여 있고 블록 수도 1024-token 기준과 맞다. 어느 harness에서 나온 수치인지 확인이 필요하다.

### F3. 라우팅은 일부러 노드를 건너게 했다 [확인]
- pass 2에서 node r은 node (r-1) mod N이 만든 prefix를 요청한다(`run_direct_llm.py:882`, `:901`). cache-aware routing이 없는 최악의 경우를 재는 설계다.
- 32노드 pass 2의 hit rate(cached_tokens/input_tokens)는 다음과 같다(기존 결과로 계산, 새 실험 아님).

| 시스템 | 8K weak | 64K weak |
|---|---|---|
| CASCADE | 99.4% | 99.9% |
| Mooncake | 99.4% | 99.1-99.9% (4회) |
| LMCache (P2P) | 98.7% | (확인 필요) |
| vLLM CPU offload | 1.2% | |
| LMCache (Disk/local) | 1.2% | 0.0% |
| vLLM (no connector) | 0.2% | |

- B의 추측(라우팅 때문에 CPU offload hit가 거의 없다)은 정확하다. 동시에 **peer 시스템끼리는 hit rate가 같으므로 TTFT 격차는 hit rate 차이가 아니다**. 격차는 조회, 전송, 그리고 그 비용이 키우는 scheduler queueing에서 온다(Fig 16: 64K에서 KV retrieval+GPU transfer 2,008 vs 3,385/3,617ms, queueing 1,824 vs 7,070/8,252ms).
- 주의: Fig 16에서 전송 구간만 보면 격차는 1.7-1.8배다. 큰 격차는 queueing 증폭에서 나온다. A가 이 점을 이미 짚었다(transfer costs can amplify queueing). 이 증폭 자체가 transport에 좌우되는지는 TCP 대조 실험 없이는 증명할 수 없다.

### F4. E2E는 decode를 재지 않았다 [확인]
- 결과 파일의 `max_tokens`가 1이다(드라이버 기본값은 8). E의 decode와 full request time 질문은 [ADMIT] 또는 [NEW]다.
- 설계상 CASCADE는 decode 경로에 관여하지 않는다(KV가 이미 HBM에 있음). TPOT는 vLLM과 같아야 하지만 측정치는 없다.

### F5. 그 밖의 실행 사실 [확인]
- slurm 런처는 `CASCADE_PREFIX_REPLICATION=0`으로 실행한다(`run_direct_compare.slurm:170`, sglang `point.slurm:183`). 따라서 E2E에서 prefix를 모든 노드로 미리 복제하는 기능은 꺼져 있었다.
- CASCADE의 local DRAM hit 대부분은 `prefetched_blocks`와 같다. 원격 블록을 조회 시점에 local로 미리 당겨 오는 것으로 보이며, 이것이 설계 이득(전송과 scheduling의 겹침)인지 정확한 의미를 확인해야 한다 [검증중].

---

## 2. 리뷰어별 분석

### A (Weak reject, Knowledgeable): 가장 꼼꼼하고 공정함
| # | 지적 | 태그 | 대응 |
|---|---|---|---|
| A1 | Transport confounding. CASCADE over TCP를 요구하고, 안 되면 주장을 플랫폼으로 한정하라고 함 | [NEW]/[ADMIT] | F1으로 압축은 제거된다. TCP 대조는 실험 X1. 불가능하면 A가 제시한 범위 한정을 수용한다 |
| A1' | 압축, prefix protection, aggregated I/O 각각의 ablation | [NEW] | 압축은 F1. prefix protection은 Fig 12(이미 있음). aggregated I/O는 trace 데이터 확인 |
| A2 | sharded인데 전 노드가 모든 shard를 보유? Dedup Map은 왜 따로? | [ANSWER] | 코드 확인 후 정확히 [검증중] |
| A3 | 동시 중복 할당, 충돌 해소, RDMA 진행 중 버퍼 재사용 | [ANSWER] | 코드 확인 [검증중] |
| A4 | INT8 granularity, LongBench v2 품질, 비양자화 성능 | [FACT]+[NEW] | 비양자화 성능 = 이미 보고된 E2E 수치(F1). 품질은 E2E가 무손실. INT8 품질 측정은 X3 |
| A5 | 낮은 공유율, 바뀌는 hotspot, Fig 12의 tier 용량, hit ratio, PFS 트래픽 | [CLAR]/[NEW] | E2E는 실제 LongBench 요청이고 공유율 80-95% 설정이 아님. trace 로그에서 수치 확인 |
| A6 | 동기화 시간, CPU와 네트워크 사용, 전파 지연, fallback 비율 | [ANSWER]/[NEW] | counter가 있는지 [검증중]. Table 3의 재계산 관찰이 부분 근거 |
| A+ | zero-copy 범위, block ID namespace, TP 블록 회계, partial prefix, PUT 회피와 연산 회피 구분, 논리와 물리 바이트 구분, 버전, warm-up, 반복, 완료율 | [ANSWER]/[PRES] | 한 줄씩. 버전은 vLLM 0.18.1 등 |

### B (Weak reject, Expert): 가장 중요. 전제 두 개가 E2E에선 틀림
| # | 지적 | 태그 | 대응 |
|---|---|---|---|
| B1 | 손실 압축 대 비압축 비교는 불공정. 압축 없는 버전을 보여라 | **[FACT]** | F1: E2E는 압축 없음. 우리 문장 오류도 인정 |
| B2 | 이미 양자화된 모델에는 압축이 무의미 | [FACT] | 같은 답. 압축은 선택 사항이고 E2E는 꺼져 있음 |
| B3 | 라우팅이 무작위라 전송 메커니즘에 의존하나? | [ANSWER] | F3: 일부러 노드를 건너게 함. 그 이유와 hit rate 표 |
| B4 | 인덱스: GPU별 shard? HBM이 관여? Lustre도 매핑? 영속? | [ANSWER] | 코드 확인 [검증중] |
| B5 | 방법별 cache hit rate | [ANSWER] | F3 표(기존 데이터) |
| B6 | LMCache P2P와 Mooncake의 전송 지연을 직접 비교 | [NEW]/[ANSWER] | Fig 16 구간값. 순수 전송 microbench는 X2 |
| B7 | 같은 block size로 | **[FACT]** | F2: E2E에서는 우리가 16/64, baseline이 256 |
| B8 | eventually consistent 인덱스가 hit에 주는 영향, 갱신이 잦은 워크로드 | [ANSWER]/[NEW] | Table 3의 재계산 관찰. 갱신률 sweep은 X7 |
| B-s | Table 1 잡다함, must be offloaded 과장, Data Layer 문단, workload 설명, Fig 9 Hot, Variable-sized 이름, burst 중 metadata 갱신 | [PRES] | 수정 약속 한 줄 |

**B 공략:** B가 직접 쓴 질문은 이득의 출처와 압축, block size의 역할이다. F1과 F2가 이 두 질문에 대한 사실 답이다. B는 Fig 16을 칭찬했으므로 Fig 16과 hit rate로 이득을 설명하면 B의 언어로 답하는 셈이다.

### C (Weak accept)
| # | 지적 | 태그 | 대응 |
|---|---|---|---|
| C1 | HPC 전용이라 범위가 좁다. LMCache/HiCache/Mooncake에 PR하면 될 일 | [CLAR] | CXI 실패가 설정이 아니라 프로토콜 변경이라는 점(§4.1). MPI에 위임하면 패브릭이 바뀌어도 코드가 안 바뀜 |
| C2 | DeepSeek-V4 류의 KV 축소로 PFS가 과잉 | [ANSWER] | 요청당 KV가 줄면 보관할 문맥 수가 늘 뿐. agentic 장문맥으로 총량은 계속 증가. 압축 KV에도 같은 계층 적용 |
| C3 | ShareGPT와 LongBench는 chat 시대 데이터. agentic 데이터셋 | [ANSWER]/[ADMIT] | agentic은 턴마다 증분 문맥을 재사용해 prefix 공유가 더 높아 CASCADE에 유리한 방향. 새 데이터셋 실험은 약속만 |
| C4 | Salting으로 tenant 간 timing side channel을 막을 수 있나 | [ANSWER] | Block ID 체인에 tenant salt를 넣으면 tenant 간 공유만 끊긴다. vLLM 경로가 vLLM block hash(cache_salt 포함)를 쓰는지 확인 [검증중] |

### D (Accept, Some familiarity): 표현과 통계
- 통계: 반복 횟수, 평균, 표준편차가 없음 → 기존 반복 실행으로 계산이 가능한지 X6.
- 설명 요청: 1-byte Dedup Map의 용도, local path에서 압축하는 이유, validation 실패 시 재계산 방법, Alg 1 문장과 알고리즘 불일치, GPU 블록마다 DRAM shadow를 두는 이유, Alg 2의 Memory Pool[GPU] 표기, weak와 strong 차이, 128 requests의 의미, Fig 9b 갈색 막대, mixed-length의 의미, burst recovery 측정 시점, prefix sharing의 중요성, Fig 13 log scale, **TTFT 정의 누락**, 8K와 64K의 의미, DeepCAM streaming의 안전성.
- 대부분 [PRES]이고 한 줄 약속으로 처리한다. 단 TTFT 정의, local path 압축 이유, validation 실패, DRAM shadow 이유는 짧게 답한다.

### E (Weak accept)
| # | 질문 | 태그 | 대응 |
|---|---|---|---|
| E1 | 설계, 네트워크, 압축을 분리한 breakdown | [FACT]+[NEW] | D#1과 같음 |
| E2 | 압축이 장문 품질에 주는 영향, decode 속도, 전체 요청 시간 | [FACT]+[NEW] | F1(무손실), F4(decode 미측정) |
| E3 | 조기 분기 시 partial chunk 재사용, Lustre 공유 파일의 obsolete 정리 | [ANSWER] | E2E는 16/64 token 단위라 분기 지점까지 재사용. GC는 코드 확인 [검증중] |
| E4 | 장애와 동적 확장 | [ANSWER]/[ADMIT] | 캐시는 재구성 가능하므로 손실 = miss. 고정 communicator는 인정 [검증중] |
| E-s | E의 요약에 trace-driven에서 Mooncake보다 빠르다고 적힘 | 무시 | trace-driven에는 Mooncake가 없지만 정정할 가치가 낮음 |

---

## 3. 쟁점 묶음과 답변 배분 (1000 단어 가정)

| 순서 | 쟁점 | 근거 리뷰 | 핵심 메시지 | 근거 | 단어 |
|---|---|---|---|---|---|
| 1 | 이득의 출처 | D#1, A1, B, E1 | E2E는 무손실이고 block도 더 작다. 남는 변수는 transport 하나이고, 그것을 이렇게 분리했다 | F1, F2, F3, X1 | 280 |
| 2 | 품질과 decode | D#2, A4, E2 | E2E 품질 변화 없음. INT8은 trace에만 적용. decode는 측정치나 범위 | F1, F4, X3, X4 | 130 |
| 3 | partial chunk와 GC | D#3, E3, A+ | 분기 지점까지 페이지 단위 재사용. Lustre 정리 방식 | 코드 | 100 |
| 4 | 장애와 동적 확장 | D#4, E4 | 캐시이므로 손실 = miss. 고정 world는 한계로 인정 | 코드 | 80 |
| 5 | Salting | D#5, C4 | Block ID 체인에 salt. tenant 내부 공유만 유지 | 코드 | 60 |
| 6 | 인덱스, Dedup Map, 동시성 | A2, A3, B4, D | 정확한 의미를 짧게 | 코드 | 150 |
| 7 | 메타데이터 비용과 일관성 | A6, B8 | counter 수치 또는 Table 3 근거 | 코드/로그 | 80 |
| 8 | 범위(HPC, 데이터셋, KV 축소) | C1-C3, A5 | 짧게 | | 70 |
| 9 | 통계와 표현 | D, B-s | 반복 통계 한 줄, 나머지는 수정 약속 | X6 | 50 |

SC26 rebuttal에서 재활용할 수 있는 논리: 캐시는 재구성 가능하므로 stale이나 손실은 miss일 뿐이라는 점(B, F 절), hot-block skew를 미래 과제로 둔 점(D 절), MPI 위임으로 패브릭을 이식한다는 점.

---

## 4. 인정해야 할 것

1. 논문이 E2E를 compressed라고 잘못 서술했다.
2. CASCADE 자체를 TCP로 돌린 대조 실험이 없다. X1이 성공하지 못하면 A가 제안한 범위 한정을 받아들인다.
3. INT8 품질을 직접 재지 않았다(trace-driven에만 적용).
4. decode와 전체 요청 시간을 재지 않았다.
5. 반복 통계가 없다.
6. 고정 MPI world (코드 확인 후 표현 확정).

---

## 5. 코드 검증 결과 [검증중]

백그라운드 에이전트가 C++과 connector 코드에서 다음을 확인하고 있다. 도착하면 이 절을 채운다.
Global Shard Index 구조와 소유권과 영속성, Dedup Map의 존재 이유, 동시 PUT 충돌과 RDMA 버퍼 회수, metadata sync counter, Aggregated File Writer의 GC, 장애 처리, Block ID와 cache_salt와 partial block, INT8 granularity, GPU tier와 DRAM shadow, TCP 경로나 CC 변형의 존재 여부.
