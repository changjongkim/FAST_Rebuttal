# FAST '27 #165 — Response 구성안

- 분량 **1000 words** (HotCRP 안내: Try to stay within 1000 words, short and to the point). Markdown과 LaTeX 수식 지원.
- 마감 **2026-11-20 20:59 KST**. 내부 목표 11-20 15:00.
- 안내문에 the conference deadline has passed라고 되어 있으므로 논문 수정본은 낼 수 없다. 새 측정치는 명시적으로 금지되지 않았고 D가 직접 요청했다.
- 구성: **Q1-Q5 (D가 지정, 필수)** → **G1-G4 (나머지 리뷰의 치명적인 것만 묶음)** → 수정 약속 한 줄.
- 상태 표시: ✅ 확인된 근거, 🔄 코드 검증 중, 🧪 실험이 필요하고 결과에 따라 문장이 바뀜.

## 단어 배분

| 절 | 주제 | 출처 | 단어 |
|---|---|---|---|
| 서두 | 감사 한 줄 + 핵심 정정 한 줄 | | 30 |
| **Q1** | 설계, 네트워크, 압축의 분리 + block size | D1, A1, B(핵심), E1 | **290** |
| **Q2** | 압축과 품질, decode, 전체 요청 시간 | D2, A4, B1-2, E2 | **140** |
| **Q3** | 조기 분기 시 partial chunk, Lustre 파일 정리 | D3, E3, A+ | **100** |
| **Q4** | 장애, 동적 확장 | D4, E4 | **80** |
| **Q5** | Salting | D5, C | **60** |
| G1 | 인덱스와 Dedup Map의 의미 | A2, B4, D(3.2) | 90 |
| G2 | 동시성, 일관성, 메타데이터 비용 | A3, A6, B8, B(p9) | 90 |
| G3 | 워크로드와 범위 | A5, B3, C1-C3 | 70 |
| G4 | 통계와 표현 | D, B(smaller), A+ | 40 |
| 합계 | | | **≈990** |

---

## Q1. Controlled breakdown: 설계 대 네트워크 대 압축, block size (290)

리뷰어 넷이 같은 질문을 했으므로 답변의 무게중심이다. 변수를 하나씩 제거하는 순서로 쓴다.

1. **압축: E2E에서는 0이다.** ✅
   - 헤드라인 E2E 결과(9.39-16.75x TTFT, 7.17-12.42x throughput, SGLang 2.95-3.03x)는 모두 `kv_compression` OFF로 측정했다. BF16 페이지를 그대로 옮겼다(실행 로그 39개에서 OFF 확인).
   - §4.6-4.8이 이를 compressed RDMA라고 쓴 것은 우리 서술 오류이고 고치겠다.
   - INT8은 trace-driven(§4.2-4.5)에만 켰다. 그 구간의 baseline(HDF5, PDC, LMCache Disk)은 CASCADE와 같은 Slingshot 패브릭으로 Lustre에 접근한다. INT8은 바이트를 절반으로 줄일 뿐이라 5.7-28.5x 중 최대 2x만 설명할 수 있다. 🧪 X8(압축 OFF 재측정)을 하면 수치로 대체한다.
2. **Block size: E2E에서 CASCADE가 더 잘다.** ✅
   - CASCADE는 vLLM 16-token, SGLang 64-token 페이지를 쓰고 LMCache와 Mooncake는 256-token chunk(기본값)를 썼다.
   - 1024-token(320MB) 블록은 trace-driven에서만 쓰였고, 거기서는 모든 시스템이 같은 블록을 받는다 🔄(trace baseline 블록 크기 재확인).
   - block size 영향은 trace-driven 결과가 보여 준다. 160MB에서 320MB로 가면 모든 시스템이 느려지고(Fig 7), 1MB-1GB 혼합 블록에서도 순위가 유지된다(Fig 10).
3. **Hit rate: 같다.** ✅
   - 32노드에서 pass 2의 hit rate는 CASCADE 99.4-99.9%, Mooncake 99.1-99.9%, LMCache P2P 98.7%다. 남는 차이는 hit 이후의 경로 비용뿐이다.
4. **네트워크: 남는 변수는 하나다.**
   - Fig 16(64K, 32노드)의 KV retrieval+GPU transfer는 2,008ms 대 3,385/3,617ms(1.7-1.8x)다. 이것이 전송 경로 몫의 상한이다.
   - 나머지 TTFT 격차는 scheduler queueing(1,824 대 7,070/8,252ms)에서 온다. 조회가 노드 안에서 끝나 prefill admission이 빨리 풀리기 때문이다.
   - 🧪 **X1(CASCADE over TCP)**이 되면 같은 셀의 TCP 수치로 설계 몫을 직접 보인다.
   - X1이 안 되면 A가 제안한 대로 주장을 평가 플랫폼으로 한정한다고 명시하고, 위 1-3으로 변수 셋이 제거되었음을 강조한다.

## Q2. 압축과 품질, decode, 전체 요청 시간 (140)

1. **품질:** E2E는 무손실이라 복원된 KV가 바이트 단위로 같다. greedy 출력은 정의상 같다. 🧪 X3: LongBench v2 요청에서 vanilla vLLM과 출력 일치율, 정답률.
2. **INT8:** granularity와 scale을 명시한다 🔄. INT8은 trace-driven의 선택 사항이다. 🧪 X3에서 INT8 ON 정답률을 내거나, 없으면 측정 약속.
3. **Decode:** CASCADE는 prefill 이전에만 관여하고 decode는 HBM의 KV를 vLLM이 그대로 읽으므로 TPOT에 개입하지 않는다. E2E는 출력 1 token으로 TTFT를 쟀다는 점을 인정한다. 🧪 X4: max_tokens 128 셀의 TPOT와 전체 요청 시간. 없으면 TPOT가 같으므로 전체 요청 시간 이득은 TTFT 이득과 같다고 논리로만 답한다.

## Q3. Partial chunk와 Lustre 정리 (100)

1. **Partial reuse:** Block ID는 앞 Block ID를 포함한 체인이라 분기 직전 마지막 완전 페이지까지 재사용하고 꼬리만 재계산한다(vLLM APC와 같은 방식). E2E 단위는 16/64 token이다. 🔄 partial block 처리 코드 확인.
2. **Lustre 정리:** 🔄 코드 확인 후 확정. 확인할 것은 세 가지다. 할당 범위의 캐시라 job 종료 시 파일을 지우는지, 실행 중 compaction이 있는지, 모든 항목이 무효가 된 파일을 지우는지. compaction이 없으면 append-only 파일을 통째로 회수하는 방식과 그 비용을 설명하고, compaction은 향후 과제로 둔다.

## Q4. 장애와 동적 확장 (80)

1. **캐시 의미론:** 블록을 잃으면 miss이고, 위치 정보가 낡았으면 검증에 실패해 재계산한다. 정확성은 깨지지 않는다(SC26 rebuttal 논리 재사용). 🔄 validation 경로 확인.
2. **고정 world:** batch HPC 할당은 크기가 고정이라 고정 communicator와 맞는다. rank 장애 시 동작과 동적 확장 미지원은 인정한다 🔄. 향후 과제로 ULFM, MPI Sessions, 할당 재시작 시 Lustre 계층에서 재구성하는 방법을 든다.

## Q5. Salting (60)

- **가능하다.** 체인의 첫 Block ID에 tenant salt를 넣으면 공유는 같은 salt 안에서만 일어나고 tenant 간 timing 채널이 닫힌다. 대가는 tenant 간 dedup을 잃는 것이다.
- 🔄 vLLM 경로가 vLLM block hash(cache_salt 포함)를 그대로 쓰는지 확인. 그렇다면 vLLM의 salting이 바뀌지 않고 그대로 적용된다고 쓸 수 있다.
- HPC 할당은 보통 한 사용자나 한 프로젝트 범위다.

---

## G1. 인덱스와 Dedup Map (90) — A2, B4, D
- 무엇이 shard 단위이고 무엇이 전 노드에 복제되는지, 갱신 소유자는 누구인지, HBM이 관여하는지, Lustre 위치도 담는지, 영속성이 있는지를 정확히 쓴다. 🔄
- Dedup Map을 따로 두는 이유(빠른 PUT 경로? 위치 인덱스 갱신과의 분리?)가 코드에 근거가 없으면 단순화 가능성을 인정한다. 🔄
- 다중 노드 PUT/GET 예시는 수정본에 넣겠다고 약속한다.

## G2. 동시성, 일관성, 메타데이터 비용 (90) — A3, A6, B8
- 동시 중복 PUT, 복제 인덱스 병합 시 충돌 해소, RDMA 읽기가 진행 중인 버퍼의 회수 순서를 쓴다. 🔄
- 일관성이 느슨해서 생기는 비용은 Table 3에 이미 있다. 원격 블록이 보이기 전에 재계산이 일어나고, 같은 Block ID라 추가 할당 없이 흡수된다. 그래도 57-66%의 쓰기를 피했다. ✅
- 동기화 시간과 fallback 비율은 counter가 있으면 수치로 쓴다. 🔄/🧪 X7

## G3. 워크로드와 범위 (70) — A5, B3, C1-C3
- 라우팅은 일부러 노드를 건너게 했다. cache-aware routing이 노드 하나에 몰 수 없는 경우(부하 분산, 한 노드를 넘는 hot prefix)를 겨냥한다. ✅
- E2E는 80-95% 공유 설정이 아니라 실제 LongBench 요청이다. ✅
- agentic 워크로드는 턴마다 증분 문맥을 재사용하므로 공유가 더 많다.
- KV 축소 모델에도 같은 계층이 적용된다. 요청당 KV가 줄면 보관할 문맥 수가 늘 뿐이다.
- 상용 offloader에 PR하는 대신 MPI에 위임한 이유: CXI 대응은 각 시스템의 프로토콜 변경이 필요하다(§4.1).

## G4. 통계와 표현 (40) — D, B, A+
- 🧪 X6: 기존 반복 실행의 변동계수 한 줄.
- TTFT 정의, 소프트웨어 버전, 출력 길이, 그림 눈금과 이름(mixed block-size), Table 1과 2 정리를 수정본에서 고치겠다고 한 문장으로 약속한다.

---

## 근거 현황과 필요한 일

| 근거 | 상태 | 쓰이는 곳 |
|---|---|---|
| E2E 압축 OFF, 16/64 token 페이지, baseline 256 | ✅ | Q1, Q2 |
| peer 시스템 hit rate 99% | ✅ | Q1, G3 |
| Fig 16 구간값, Table 3 | ✅ (논문) | Q1, G2 |
| 인덱스, 동시성, GC, 장애, salt, partial block, INT8 granularity | 🔄 에이전트 검증 중 | Q2-Q5, G1, G2 |
| X1 CASCADE over TCP | 🧪 미착수 | Q1 |
| X3 출력 일치와 INT8 정답률 | 🧪 미착수 | Q2 |
| X4 decode | 🧪 미착수 | Q2 |
| X6 반복 통계 | 🧪 분석만 | G4 |
| X8 trace 압축 OFF | 🧪 미착수 | Q1 |
