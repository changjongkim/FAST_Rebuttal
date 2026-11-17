# FAST '27 #165 — Response 진행 계획

마감: **2026-11-20 20:59 KST** (HotCRP 표시). 내부 목표는 **11-20 15:00 KST 제출**이다.
분량: **1000 words** (확정). 형식: md 파일 → HotCRP에 붙여넣기(Markdown, LaTeX 지원).
구성: D가 지정한 Q1-Q5는 필수. 나머지는 치명적인 것만 G1-G4로 묶는다. 상세는 `03_response_outline.md`.
문체: 세미콜론 금지, 문장을 잇는 대시 금지, 큰따옴표 금지. 리뷰어 문구는 인용부호 없이 풀어 쓴다.

## 파일 구성 (`kcj/FAST_Rebuttal/`, GitHub `changjongkim/FAST_Rebuttal`)
| 파일 | 내용 |
|---|---|
| `00_reviews_raw.md` | 리뷰 원문 (완료) |
| `01_review_analysis.md` | 리뷰 분석, 확인된 사실 F1-F5, 쟁점 배분 (완료, §5 코드 검증 갱신 예정) |
| `02_plan.md` | 이 문서 |
| `03_evidence.md` | 답변에 쓸 수치와 출처(file:line, job id) 한곳에 모음 |
| `10_response_v1.md` / `10_response_v1_ko.md` | 영문 초안 / 한국어 대역 |
| `11_response_vN.md` | 수정본. 최종본은 `99_response_final.md` |

## 단계

### D0 — 11-17 밤 ~ 11-18 오전
- [x] 리뷰 원문 md 변환
- [x] 분석. 핵심 사실 확인: E2E 압축 OFF, E2E block 16/64 token 대 baseline 256, 라우팅은 일부러 노드를 건넘, peer 시스템 hit rate 99% 동일, decode 미측정
- [ ] 코드 검증 (에이전트 진행 중): 인덱스, Dedup Map, 동시성, GC, 장애, salt, partial block, INT8 granularity, TCP 경로 유무
- [ ] **사용자 결정** (아래 4개)
- [ ] 실험 착수 전 탐색: X1 TCP 탐색(30분), X3 품질 harness 준비

### D1 — 11-18
- [ ] 선택한 실험 실행
- [ ] `03_evidence.md` 작성
- [ ] v1 초안(영문과 한국어). 숫자가 아직 없는 자리는 `[X1]`처럼 표시

### D2 — 11-19
- [ ] 실험 수치 반영 → v2
- [ ] 리뷰어 B 입장에서 v2를 읽는 독립 검토(논문과 리뷰만 주고 결론은 주지 않음)
- [ ] 단어 수와 문체 자동 검사

### D3 — 11-20 오전
- [ ] 최종본, 사용자 최종 수정, HotCRP 입력은 사용자가 직접

## 실험 후보 (72시간 안)

| ID | 내용 | 답하는 질문 | 비용 | 위험 | 추천 |
|---|---|---|---|---|---|
| X1 | **CASCADE를 TCP로**: Cray MPICH에 libfabric tcp provider를 강제해 다른 설정은 그대로 두고 E2E 재실행(64K weak 8·32노드, Fig 16 셀) | D#1, A1, B, E1 (최우선 쟁점) | 탐색 30분, 성공 시 노드 몇 시간 | Cray MPICH가 cxi 외 provider를 거부하거나 RMA가 안 될 수 있음 | **탐색부터** |
| X3 | **품질**: LongBench v2 요청으로 vanilla vLLM과 CASCADE(무손실)의 greedy 출력 일치와 정답률, CASCADE INT8 ON 정답률 | D#2, A4, E2 | 1-2노드 몇 시간 | INT8 경로가 vLLM 페이지 경로에서 동작하는지 확인 필요 | **추천** |
| X6 | **통계**: 기존 반복 실행으로 평균과 표준편차 | D, A+ | 분석만 | 셀마다 반복 수가 다름 | **추천** |
| X4 | **Decode**: max_tokens 128로 몇 셀 재실행, TPOT와 전체 요청 시간 | D#2, E2 | 중간 | 시간 | 여유 있으면 |
| X8 | trace-driven을 압축 OFF로 8노드 재측정 | B1(trace 쪽) | 중간 | 낮음 | 여유 있으면 |
| X7 | metadata sync 시간과 fallback 비율 | A6, B8 | counter가 있으면 분석만 | counter가 없으면 계측 필요 | 코드 검증 결과로 결정 |
| X2 | 순수 전송 microbench (CASCADE MPI_Get 대 Mooncake TCP) | B6 | 중간 | 낮음 | 낮은 우선순위 |
| X5 | block size sweep | B7, D#1 | 중간 | 낮음 | F2로 답이 되므로 불필요 |

X1이 실패하면 A가 직접 제시한 대안대로 주장을 평가 플랫폼과 설정으로 한정하고, 대신 hit rate 동일성과 Fig 16 구간값, 압축과 block size 사실로 남은 변수가 transport 하나뿐임을 보인다.

## 사용자 결정
1. ~~1000의 단위~~ → 1000 words로 확정 (11-17).
2. ~~새 결과 허용 여부~~ → 안내문에 금지 조항 없음. 논문 수정본만 불가. D가 controlled breakdown을 직접 요청함.
3. **실험 선택과 queue** (미정): X1 탐색, X3, X6 추천. 같은 계정에 다른 프로젝트 job이 대기 중이라 우선순위 결정 필요.
4. **E2E를 compressed로 잘못 쓴 문장을 답변에서 인정하는 것** (미정). 인정하지 않으면 F1을 쓸 수 없다.
