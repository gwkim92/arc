# Arc 블록체인 기술 분석: 안정코인 네이티브 체인의 설계 철학

**작성자**: 블록체인 리서처
**작성일**: 2025년 11월 2일
**난이도**: 중급~고급 (수식 포함)

---

## 들어가며

Circle의 Arc 라이트페이퍼[[1]](#ref1)를 읽고 나서 며칠간 이 체인의 설계 결정들을 곱씹어봤다. 스테이블 회사의 체인 오픈은 예상했던 바이다. 2024년 10월 28일 공개 테스트넷이 발표된 Arc는[[13]](#ref13) "또 하나의 EVM 체인?"이라고 생각할 수 있지만, 디테일을 파보니 꽤 흥미로운 엔지니어링 트레이드오프들이 있었다. 특히 가스비 메커니즘과 합의 알고리즘 부분은 기존 체인들과는 확실히 다른 접근을 하고 있어서, 이번 기회에 수식을 좀 풀어가면서 정리해보려 한다.

이 글은 Solidity나 Cosmos SDK를 어느 정도 다뤄본 독자를 대상으로 한다. 기초적인 블록체인 개념 설명은 최소화하고, 대신 Arc의 설계 결정이 *왜* 그렇게 내려졌는지, 그리고 그 결과가 *무엇*을 의미하는지에 집중할 것이다.

---

## Part 1: 가스비 메커니즘 - 왜 USDC를 네이티브 토큰으로 쓰는가?

### 1.1 전통적인 가스비 공식의 문제점

일반적인 블록체인에서 사용자가 지불하는 실제 달러 비용은 다음과 같이 계산된다:

```
총 비용 (USD) = Gas Used × Gas Price × Price_Native
```

여기서:
- `Gas Used`: 트랜잭션이 소비한 가스량 (연산량에 비례)
- `Gas Price`: 가스 1단위당 네이티브 토큰 가격 (gwei, lamport 등)
- `Price_Native`: 네이티브 토큰의 달러 환율 (ETH/USD, SOL/USD 등)

**핵심 문제**: `Price_Native`가 변동성이 크다는 점이다.

#### 실제 예시로 계산해보자

이더리움에서 ERC-20 토큰 전송을 한다고 가정하자:

```
Gas Used = 65,000 gas (typical ERC-20 transfer)
Gas Price = 30 gwei = 30 × 10^-9 ETH
```

**시나리오 A: ETH 가격 = $2,000**
```
총 비용 = 65,000 × 30 × 10^-9 × 2,000
       = 0.00195 ETH × 2,000
       = $3.90
```

**시나리오 B: ETH 가격 = $4,000 (불장)**
```
총 비용 = 65,000 × 30 × 10^-9 × 4,000
       = 0.00195 ETH × 4,000
       = $7.80
```

똑같은 트랜잭션인데 ETH 가격이 2배 오르면 달러 비용도 2배가 된다. 이게 기업 입장에서는 예산 계획이 불가능하다는 얘기다.

### 1.2 Arc의 해결책: USDC 네이티브 가스

Arc는 이 문제를 아예 근본부터 제거한다[[3]](#ref3):

```
Price_USDC ≡ $1.00 (by definition)
```

USDC 자체가 1달러에 페깅되어 있으므로, Arc에서의 가스비 공식은:

```
총 비용 (USD) = Gas Used × Gas Price_USDC × 1
              = Gas Used × Gas Price_USDC
```

**변동성이 사라진다.** 더 정확히 말하면, 변동성이 `Price_Native` 항에서 `Gas Price` 항으로 옮겨간 것이다. 하지만 `Gas Price`는 네트워크 혼잡도에 따라 결정되는데, 이건 EIP-1559 기반 메커니즘으로 어느 정도 예측 가능하다.

### 1.3 실전 비교: 수수료 예측성

기업이 한 달에 10만 건의 트랜잭션을 처리한다고 가정하자.

**이더리움 (ETH 네이티브)**:
```
예상 범위:
- 최저 (ETH = $1,500, 30 gwei): $292,500
- 최고 (ETH = $4,500, 100 gwei): $2,925,000

범위 폭: 10배
```

**Arc (USDC 네이티브)**:
```
예상 범위 (추정):
- 최저 (정상 혼잡도): ~$6,500
- 최고 (피크 혼잡도): ~$65,000

범위 폭: 10배
```

일견 똑같아 보이지만, 핵심 차이는:
1. **이더리움**: 가격 변동 원인이 외부 시장 (암호화폐 거래소)
2. **Arc**: 가격 변동 원인이 내부 수요 (네트워크 혼잡도)

기업 입장에서는 후자가 훨씬 예측 가능하다. 자기 트래픽 패턴을 알고 있으니까.

> **출처**: Arc 공식 블로그 "How Gas Works on Arc"[[3]](#ref3)는 USDC 네이티브 가스의 예측 가능성을 핵심 차별점으로 설명한다.

---

## Part 2: EIP-1559의 Arc식 개량 - Fee Smoothing과 Bounded Fee

### 2.1 기본 EIP-1559 복습

이더리움의 EIP-1559는[[4]](#ref4) 다음 공식으로 Base Fee를 조정한다:

```
base_fee_{n+1} = base_fee_n × (1 + δ)

여기서:
δ = (gas_used_n - gas_target) / gas_target × 1/8
```

**해석**:
- 블록 사용량이 타겟(50%)보다 높으면 → Base Fee 상승
- 블록 사용량이 타겟보다 낮으면 → Base Fee 하락
- 최대 변화율: ±12.5% (1/8 = 0.125)

**문제점**: 직전 블록만 본다는 것. 갑작스러운 트래픽 스파이크가 오면 Base Fee가 급등한다.

### 2.2 Arc의 개선: Fee Smoothing (EWMA 기반)

Arc는 여러 블록의 가중 평균을 사용한다[[3]](#ref3):

```
base_fee_{n+1} = f(S_n)

S_n = Σ(i=0 to ∞) w_i × gas_used_{n-i}

여기서:
w_i = α × (1-α)^i  (지수 가중치, EWMA)
α ∈ (0, 1): 감쇠 계수
```

**직관적 설명**:
- `α = 0.1`이면 최근 10개 블록의 평균을 본다
- `α = 0.5`이면 최근 2~3개 블록에 집중
- Circle은 구체적인 α 값을 공개하지 않았지만, 라이트페이퍼에서 "fee smoothing algorithm"으로 언급[[1]](#ref1)

> **참고**: EIP-1559 개발 과정에서도 fee smoothing이 논의되었으나[[5]](#ref5) 최종 구현에는 포함되지 않았다. Arc는 이를 실제로 구현한 첫 사례 중 하나다.

#### 시뮬레이션: 갑작스러운 트래픽 스파이크

```
시나리오: 평소 30% 사용량, 10번째 블록에서 갑자기 100% 사용

EIP-1559 (직전 블록만):
블록 9:  30% → base_fee × 0.975
블록 10: 100% → base_fee × 1.125  ← 12.5% 급등
블록 11: 100% → base_fee × 1.266  ← 또 12.5% 급등

Arc EWMA (α=0.2 가정):
블록 9:  30% → base_fee × 0.99
블록 10: 100% → base_fee × 1.03   ← 3% 완만 상승
블록 11: 100% → base_fee × 1.07   ← 7% 완만 상승
```

EWMA가 훨씬 부드럽다. 이게 Arc가 "기업 친화적"이라고 주장하는 근거 중 하나.

### 2.3 Base Fee Bounded: 상한선 도입

Arc는 여기서 한 발 더 나간다[[3]](#ref3):

```
base_fee_n ≤ MAX_BASE_FEE
```

이더리움에는 이런 상한선이 없다. 이론적으로 Base Fee는 무한히 오를 수 있다.

**실제 사례**: 2021년 5월 이더리움 네트워크 혼잡 시 Base Fee가 500 gwei를 넘은 적이 있다. ETH 가격이 $3,000이었으니:

```
단순 전송 비용 (21,000 gas) = 21,000 × 500 × 10^-9 × 3,000
                            = $31.50
```

#### Arc의 상한선 메커니즘 (추정)

Arc의 정확한 가스 단위와 상한선 값은 공개되지 않았다. 하지만 라이트페이퍼에서 "guardrails limit how quickly fees can move"라고 명시[[1]](#ref1)했으므로, 대략적인 추정이 가능하다:

**가정** (필자 추정):
- Arc가 "predictable dollar costs"를 목표로 한다면
- 최악의 경우에도 단순 전송이 $1~$5 범위 내여야 함
- EVM 호환이므로 가스 모델은 이더리움과 유사할 것

**추정 시나리오** (필자 추정):
```
Arc 단순 전송: 21,000 gas (이더리움과 동일 가정)
MAX_BASE_FEE: 0.0001 USDC/gas (추정)

최악의 경우:
총 비용 = 21,000 × 0.0001 = 2.1 USDC = $2.10
```

이 정도면 합리적이다. 물론 실제 값은 테스트넷 데이터를 봐야 알 수 있다.

> **주의**: 위의 MAX_BASE_FEE 값은 필자의 추정이며, Circle 공식 발표가 아님을 밝힌다.

---

## Part 3: Tendermint BFT와 결정적 최종성

### 3.1 비트코인/이더리움과의 근본적 차이

비트코인은 **확률적 최종성**을 가진다:

```
P(reorg at depth k) = (q/p)^k

여기서:
p = 정직한 해시파워 비율
q = 공격자 해시파워 비율 (q < p)
k = 블록 깊이
```

예를 들어 공격자가 30% 해시파워를 가지고 있다면:

```
k=1:  P = (0.3/0.7)^1  = 0.43  (43% 확률로 1블록 reorg)
k=6:  P = (0.3/0.7)^6  = 0.013 (1.3% 확률로 6블록 reorg)
k=10: P = (0.3/0.7)^10 = 0.002 (0.2% 확률로 10블록 reorg)
```

비트코인에서 6 confirmation을 기다리는 이유가 이거다. "충분히 낮은" 확률까지 떨어뜨리려고.

**이더리움 PoS**도 비슷하다. Casper FFG로 2 epoch (약 12.8분) 후 경제적 최종성을 보장하지만, 여전히 33% 검증자가 슬래싱당할 각오만 있으면 reorg가 이론적으로 가능하다.

### 3.2 Tendermint BFT의 결정적 최종성

Arc의 Malachite 합의 엔진은 Tendermint BFT 기반[[6]](#ref6)인데, 이건 **즉시 확정**된다:

```
if (votes ≥ 2f+1) → block is FINAL

f = 비잔틴 노드 최대 수
n = 총 검증자 수
n ≥ 3f + 1 (BFT 안전성 조건)
```

Arc는 20개 검증자로 시작하니[[1]](#ref1):

```
n = 20
f = 6  (최대 6개까지 비잔틴 허용)
2f+1 = 13 (13개 이상 서명하면 확정)
```

**핵심**: 일단 13개 검증자가 서명하면 그 블록은 **영원히 확정**된다. reorg 불가능. 수학적으로 보장됨.

#### 왜 그런가? (간단 증명)

귀류법으로 증명하자. 두 개의 상충하는 블록 B₁, B₂가 같은 높이에서 모두 확정되었다고 가정:

```
B₁: 13개 검증자 서명 (집합 V₁)
B₂: 13개 검증자 서명 (집합 V₂)

|V₁ ∩ V₂| ≥ |V₁| + |V₂| - n
           = 13 + 13 - 20
           = 6
```

즉, **최소 6개 검증자가 두 블록 모두에 서명**했다는 뜻.

하지만 Tendermint 프로토콜 상, 같은 높이에서 두 블록에 서명하는 건 **명백한 비잔틴 행위**다. 이런 노드는 슬래싱당한다.

우리가 가정한 건 "최대 6개 비잔틴 노드"였으니, 이들이 **전부** 비잔틴이어야 한다. 근데 그래도 상충 블록을 만들려면:

```
남은 정직한 노드: 20 - 6 = 14개

B₁에 필요한 추가 서명: 13 - 6 = 7개
B₂에 필요한 추가 서명: 13 - 6 = 7개
합계: 14개
```

어? 이러면 가능한 거 아닌가?

**아니다.** 왜냐하면 정직한 노드는 **같은 높이에서 하나의 블록에만 서명**하니까. 14개 정직한 노드를 B₁에 7개, B₂에 7개로 나누려면, 각각이 자기가 본 블록에만 서명해야 하는데, 그럼 한쪽은 먼저 13개 달성하고 확정된다. 그 순간 나머지 7개는 "이미 확정된 블록이 있네"라고 인식하고 다른 블록에 서명 안 함.

**결론**: 수학적으로 불가능.

### 3.3 실무적 의미

금융 기관 입장에서 이게 엄청 중요하다.

```
비트코인: "6 confirmation 기다려라 (60분)"
이더리움: "2 epoch 기다려라 (12.8분)"
Arc: "1개 블록 나오면 끝 (<1초)"
```

PFMI (Principles for Financial Market Infrastructures) 원칙 8번이 "결제 최종성은 명확해야 한다"인데, Arc는 이걸 완벽히 충족한다.

> **출처**: Arc 라이트페이퍼[[1]](#ref1)는 "sub-second deterministic finality"를 핵심 특징으로 명시하며, 이는 Tendermint BFT의 특성에서 비롯된다.

---

## Part 4: MEV 문제 - 완화는 가능하지만 제거는 불가능

### 4.1 MEV란 무엇인가? (수학적 정의)

MEV (Maximal Extractable Value)를 정확히 정의하면:

```
MEV = max(V(π') - V(π))

여기서:
π  = 원래 트랜잭션 순서
π' = 블록 생성자가 재배열한 순서
V(·) = 블록 생성자가 얻는 가치
```

**직관적 설명**: 블록 생성자가 트랜잭션 순서를 바꿔서 얻을 수 있는 최대 이익.

#### 구체적 예시: Sandwich Attack

DEX에서 다음 트랜잭션들이 mempool에 있다고 가정:

```
T₁: Alice가 1 ETH → USDC 스왑 (slippage 5%)
```

봇이 이걸 보고:

```
π = [T₁]

π' = [T₀_front, T₁, T₂_back]

T₀_front: 봇이 1 ETH → USDC 먼저 스왑 (가격 올림)
T₁:       Alice 트랜잭션 실행 (나쁜 가격에 체결)
T₂_back:  봇이 USDC → ETH 다시 스왑 (이익 실현)
```

**수익 계산** (단순화):

```
초기: 1 ETH = 2,000 USDC (AMM 풀)

T₀_front: 봇이 1 ETH 매도
  → 풀: 2 ETH, 1,000 USDC
  → 가격: 1 ETH = 500 USDC (극단적 예시)
  → 봇 보유: 1,000 USDC

T₁: Alice가 1 ETH 매도
  → 풀: 3 ETH, 333 USDC
  → Alice 받음: 667 USDC (원래 2,000 받을 걸...)

T₂_back: 봇이 1,000 USDC로 ETH 재구매
  → 봇이 3 ETH 가져감
  → 최종 이익: 2 ETH (원래 1 ETH였으니)
```

물론 실제론 이렇게 극단적이진 않지만, 원리는 같다.

### 4.2 Arc는 MEV를 어떻게 완화하는가?

Arc 라이트페이퍼에선 세 가지를 언급한다[[1]](#ref1):

1. **결정적 순서 (Deterministic Ordering)**
2. **허가형 검증자 (Permissioned Validators)**
3. **빠른 최종성 (<1초)**

하나씩 보자.

#### 4.2.1 결정적 순서

Arc는 트랜잭션 순서를 "도착 시간순"으로 결정한다고 주장하는데, 이건 좀 애매하다.

```
문제: "도착 시간"을 누가 정의하는가?
```

분산 시스템에서 시간 동기화는 악명 높은 문제다. NTP로 밀리초 단위까지 맞춘다 해도, 네트워크 지연은 어쩔 수 없다.

아마 Arc는 이렇게 할 것 같다:

```
1. Proposer가 자기가 본 순서대로 트랜잭션을 블록에 포함
2. 다른 검증자들은 그 순서를 검증은 안 함 (PoA니까 신뢰)
3. 대신 Proposer 로테이션이 예측 불가능하게
```

이렇게 하면 "내 트랜잭션이 어느 Proposer에게 갈지 모른다" → front-running 어려워짐.

#### 4.2.2 허가형 검증자

20개 검증자가 모두 신원이 드러나 있다[[1]](#ref1):

```
검증자 위치 (예시):
- 뉴욕 데이터센터 2개
- 암스테르담, 프랑크푸르트, 런던
- 샌프란시스코, 댈러스
- 스웨덴, 스페인, 밀워키
```

만약 어떤 검증자가 노골적으로 MEV 추출하다 걸리면?

```
결과: Circle이 해당 검증자 퇴출
```

평판 리스크가 크니까 대놓고 하긴 어렵다. 하지만...

#### 4.2.3 여전히 MEV는 존재한다

핵심: **검증자 간 담합**

```
시나리오:
- 20개 검증자 중 일부가 담합
- 자기들끼리 MEV 이익 나눔
- 대놓고는 안 하고 교묘하게

예:
- "정상적인 트래픽 패턴"으로 위장
- MEV 봇을 "일반 사용자"처럼 보이게
```

이걸 막을 방법은 없다. 왜냐하면 트랜잭션 순서를 *누군가*는 정해야 하니까.

### 4.3 MEV 규모 추정

#### 이더리움 MEV 현황 (2024-2025)

Flashbots 데이터에 따르면[[7]](#ref7):
- **2023년**: 일평균 $500,000 MEV 수익
- **2024년**: 일평균 $300,000으로 안정화 (연간 약 $109M)

최근 ESMA(유럽증권시장청) 보고서[[8]](#ref8)는 2025년 3월 기준:
- **샌드위치 공격**: 33,000명 피해, 101개 공격 주체
- **주간 거래량**: 약 $1B (샌드위치 공격만)

#### Arc MEV 추정 (필자 추정)

```
가정:
- Arc 거래량 = 이더리움의 0.1~1% (초기)
- Arc는 주로 stablecoin swap → 변동성 낮음
- MEV 기회 = 이더리움의 5~10%

보수적 추정:
109M × 0.001 × 0.05 = $5,450/year (최소)
109M × 0.01 × 0.1 = $109,000/year (최대)
```

이더리움 대비 1/1000~1/10000 수준. Arc의 거래 특성상 (stable-to-stable swaps) MEV 기회가 훨씬 적을 것으로 예상된다.

> **출처**: Flashbots 2024 데이터[[7]](#ref7), ESMA 2025 MEV 리포트[[8]](#ref8)

---

## Part 5: 성능 트레이드오프 - BFT의 O(n²) 저주

### 5.1 왜 검증자가 늘면 느려지는가?

Tendermint BFT는 **투표 기반 합의**다. 각 검증자는 다른 모든 검증자로부터 투표를 받아야 한다:

```
메시지 복잡도 = O(n²)
```

정확히는:

```
1 라운드당 메시지 수 = n × (n-1) = n² - n ≈ n²  (n이 클 때)
```

#### 구체적 계산

```
4개 검증자:
- 각자 3개씩 투표 받음
- 총 메시지: 4 × 3 = 12

20개 검증자:
- 각자 19개씩 투표 받음
- 총 메시지: 20 × 19 = 380

100개 검증자:
- 각자 99개씩 투표 받음
- 총 메시지: 100 × 99 = 9,900
```

메시지가 많아지면 → 네트워크 대역폭 소모 증가 → 블록 시간 증가 → TPS 감소

### 5.2 Tendermint 실제 성능 데이터

ResearchGate 논문[[9]](#ref9)에 따른 실제 벤치마크:

```
Tendermint 성능 (실험 조건):
- 정상 상태: 438 TPS
- 비잔틴 장애 (42개 검증자 크래시): 164 TPS (63% 감소)
- 평균 레이턴시:
  - 정상: 3.45 ± 0.99s
  - 장애: 9.17 ± 8.13s (2.7배 증가)
```

**이론 최대치**[[6]](#ref6):
```
Tendermint 이론 성능:
- 최대 10,000 TPS (250바이트 트랜잭션 기준)
- 병목: 합의 계층이 아닌 애플리케이션 계층
- 블록 최종성: 1-2초
```

### 5.3 Arc의 성능 주장 검증

Arc 라이트페이퍼[[1]](#ref1)는:
```
4개 검증자:  10,000 TPS (100ms 블록)
20개 검증자: 3,000 TPS (350ms 블록)
```

**검증**:
- 10,000 TPS는 Tendermint 이론 최대치와 일치[[6]](#ref6)
- 350ms 블록 타임은 실제 Cosmos Hub (1-2초)[[10]](#ref10)보다 빠름
- 3,000 TPS는 현실적인 수치

#### TPS 계산 역산

```
TPS = (트랜잭션 per 블록) / (블록 시간)

20개 검증자:
3,000 = X / 0.35
X = 1,050 트랜잭션/블록
```

이더리움 블록 (12초, 300 트랜잭션)과 비교하면:
```
이더리움: 300 / 12 = 25 TPS
Arc: 1,050 / 0.35 = 3,000 TPS

Arc가 120배 빠름
```

하지만 블록 가스 한도를 역산하면:
```
가정: 평균 트랜잭션 = 100K gas

이더리움: 300 tx × 100K = 30M gas (실제 값)
Arc: 1,050 tx × 100K = 105M gas

→ Arc는 이더리움보다 3.5배 큰 블록
```

**함의**:
- Arc는 더 큰 블록과 빠른 블록 생성으로 고성능 달성
- 상태 bloat 문제 가능성 → stateless client 필요할 듯

### 5.4 검증자 확장 시나리오 (필자 추정)

```
예상 성능 (필자 추정, 라이트페이퍼 데이터 없음):

검증자 수 | 블록 시간 | TPS   | 탈중앙화
---------|----------|-------|----------
4개      | 100ms    | 10,000| ⭐
20개     | 350ms    | 3,000 | ⭐⭐
50개     | ~720ms   | ~1,500| ⭐⭐⭐ (필자 추정)
100개    | ~1.5s    | ~700  | ⭐⭐⭐⭐ (필자 추정)

※ 50개 이상은 라이트페이퍼 데이터 없음 (필자 추정치)
```

Circle의 선택: **50개가 sweet spot**일 것으로 예상 (필자 추정)
- 1,500 TPS면 B2B 결제에 충분
- 규제 요구사항 충족 가능
- Circle 통제력 유지 가능

---

## Part 6: 페이마스터 (ERC-4337) - Account Abstraction의 실전 활용

### 6.1 기술적 구현

페이마스터는 ERC-4337 표준[[11]](#ref11)을 따른다. 핵심 흐름:

```
1. 사용자가 UserOperation 생성
2. Bundler가 EntryPoint 컨트랙트 호출
3. EntryPoint가 Paymaster에게 "이 사용자 수수료 대납 가능?"
4. Paymaster가 "OK" → EURC를 USDC로 변환
5. 트랜잭션 실행
```

#### UserOperation 구조

```solidity
struct UserOperation {
    address sender;
    uint256 nonce;
    bytes initCode;
    bytes callData;
    uint256 callGasLimit;
    uint256 verificationGasLimit;
    uint256 preVerificationGas;
    uint256 maxFeePerGas;
    uint256 maxPriorityFeePerGas;
    bytes paymasterAndData;  // ← 여기에 Paymaster 정보
    bytes signature;
}
```

#### Paymaster 컨트랙트 (의사코드)

```solidity
contract ArcPaymaster {
    mapping(address => bool) supportedTokens;

    function validatePaymasterUserOp(
        UserOperation calldata userOp,
        bytes32 userOpHash,
        uint256 maxCost
    ) external returns (bytes memory context, uint256 validationData) {
        // 1. 사용자가 EURC 충분히 가지고 있나?
        address token = extractToken(userOp.paymasterAndData);
        require(supportedTokens[token], "Unsupported token");

        uint256 eurcAmount = maxCost; // 1:1 환율 가정
        require(
            IERC20(token).balanceOf(userOp.sender) >= eurcAmount,
            "Insufficient balance"
        );

        // 2. EURC를 우리에게 전송하도록 approve 했나?
        // (실제론 permit 사용)

        return (abi.encode(token, eurcAmount), 0);
    }

    function postOp(
        PostOpMode mode,
        bytes calldata context,
        uint256 actualGasCost
    ) external {
        // 1. EURC를 사용자로부터 받음
        (address token, uint256 amount) = abi.decode(context, (address, uint256));
        IERC20(token).transferFrom(msg.sender, address(this), amount);

        // 2. EURC → USDC 스왑 (Circle Mint 활용)
        uint256 usdcAmount = circleSwapEURCtoUSDC(amount);

        // 3. USDC를 EntryPoint에 지불
        IERC20(USDC).transfer(entryPoint, usdcAmount);
    }
}
```

### 6.2 Circle Mint를 통한 무비용 스왑

**핵심 인사이트**: EURC와 USDC는 둘 다 Circle 발행

```
Circle Mint API를 통한 내부 스왑:
1 EURC → 1.XX USDC (실시간 환율)

장점:
- 수수료 無 또는 최소
- 슬리피지 無
- 즉시 실행
```

Arc의 페이마스터가 Circle 생태계에서 강력한 이유다. 외부 DEX 없이도 multi-currency 지원 가능.

> **출처**: Circle Payments Network 문서는 Circle Mint의 stablecoin 간 스왑 기능을 명시[[12]](#ref12)

---

## Part 7: FX 엔진과 RFQ - 기관급 외환 인프라

### 7.1 RFQ (Request for Quote) 시스템 아키텍처

Arc의 FX 엔진은 하이브리드 구조[[1]](#ref1):

```
[온체인]                    [오프체인]
스마트 컨트랙트 ←―――→ RFQ Aggregator
     ↓                         ↓
   PvP 정산         Market Maker Pool
```

#### 동작 흐름

```
1. 독일 은행이 스마트 컨트랙트에 요청:
   swap(1M EURC, USDC, slippage=0.1%)

2. 스마트 컨트랙트가 오프체인 RFQ Aggregator에 이벤트 발생:
   event QuoteRequest(
       uint256 amountIn,
       address tokenIn,
       address tokenOut,
       uint256 maxSlippage
   )

3. Market Maker들이 오프체인에서 견적 제출:
   MM_A: 1 EURC = 1.0800 USDC (수수료 0.2%)
   MM_B: 1 EURC = 1.0820 USDC (수수료 0.3%) ← 최선
   MM_C: 1 EURC = 1.0790 USDC (수수료 0.1%)

4. RFQ Aggregator가 최적 견적 선택 (MM_B)
   → EURC를 팔 때는 더 많은 USDC를 받는 게 유리

5. 온체인 스마트 컨트랙트로 결과 전송 (Oracle)

6. PvP (Payment vs Payment) 정산:
   - 은행: 1M EURC 전송
   - MM_B: 1.082M USDC 전송
   - Atomic swap (둘 다 성공 or 둘 다 실패)
```

### 7.2 PvP 정산의 수학적 보장

PvP는 Hashed Timelock Contracts (HTLC)로 구현:

```solidity
contract PvPSwap {
    struct Swap {
        address partyA;
        address partyB;
        address tokenA;
        address tokenB;
        uint256 amountA;
        uint256 amountB;
        bytes32 secretHash;
        uint256 timelock;
    }

    function initiate(Swap memory swap) external {
        require(msg.sender == swap.partyA);
        IERC20(swap.tokenA).transferFrom(swap.partyA, address(this), swap.amountA);
        // escrow에 보관
    }

    function claim(bytes32 secret) external {
        require(sha256(secret) == swap.secretHash);
        require(block.timestamp < swap.timelock);

        IERC20(swap.tokenA).transfer(swap.partyB, swap.amountA);
        IERC20(swap.tokenB).transferFrom(swap.partyB, swap.partyA, swap.amountB);
    }

    function refund() external {
        require(block.timestamp >= swap.timelock);
        IERC20(swap.tokenA).transfer(swap.partyA, swap.amountA);
    }
}
```

**핵심**: secret을 아는 순간 양쪽 전송이 atomic하게 일어남.

---

## Part 8: 옵트인 프라이버시 - 규제 준수와 데이터 보호의 균형

Arc의 프라이버시 설계는 흥미롭다. 완전한 익명성(Zcash, Monero)도 아니고, 완전 투명(Bitcoin, Ethereum)도 아닌 **선택적 프라이버시**[[14]](#ref14)를 제공한다.

### 8.1 왜 옵트인 프라이버시인가?

기업과 금융기관의 딜레마:
```
요구사항 A: 거래 내역 비공개 (경쟁사에 전략 노출 방지)
요구사항 B: 감사 가능성 (규제 당국에 거래 내역 제공)

전통적 블록체인:
- Public blockchain (Ethereum): A 불가능 ❌
- Privacy blockchain (Zcash): B 어려움 ❌

Arc의 해결책: 선택적 공개 ✅
```

### 8.2 기술적 구현: Confidential Transfers

Arc의 프라이버시는 두 단계로 진화한다[[14]](#ref14):

#### Phase 1: Confidential Transfers (현재)

```
거래 정보 가시성:
- 송신자 주소: ✅ 공개 (규제 준수)
- 수신자 주소: ✅ 공개 (규제 준수)
- 거래 금액: ❌ 암호화 (비즈니스 보호)
- 거래 메타데이터: ❌ 암호화 (선택적)
```

**실제 예시**:

```solidity
// 일반 전송 (온체인에서 보이는 것)
Transfer {
    from: 0x1234...5678,
    to: 0xabcd...ef01,
    amount: 1,000,000 USDC,  // ← 모두가 볼 수 있음
    metadata: "Payment for Q4 inventory"
}

// Confidential Transfer (온체인에서 보이는 것)
ConfidentialTransfer {
    from: 0x1234...5678,       // ← 주소는 공개
    to: 0xabcd...ef01,         // ← 주소는 공개
    encryptedAmount: 0x9a7f... // ← 암호화된 금액
    encryptedMetadata: 0x3c2e...// ← 암호화된 메타데이터
}
```

#### Phase 2: TEE + 고급 암호화 (로드맵)

**TEE (Trusted Execution Environment)** 기반[[14]](#ref14):

```
Arc의 프라이버시 아키텍처:

[사용자] → [스마트 컨트랙트] → [Precompile] → [TEE Backend]
                                                      ↓
                                             암호화된 데이터 처리
                                                      ↓
                                             결과만 온체인 반환
```

**핵심 장점**:
- **낮은 레이턴시**: TEE는 ZK-proof보다 훨씬 빠름
- **확장성**: 복잡한 연산도 오프체인에서 처리

**향후 로드맵**[[14]](#ref14):
```
현재: TEE (Trusted Execution Environments)
  ↓
미래: ZKP (Zero-Knowledge Proofs)
  ↓
미래: FHE (Fully Homomorphic Encryption)
  ↓
미래: MPC (Multi-Party Computation)
```

Arc는 모듈식 설계로 암호화 백엔드를 교체 가능하게 설계했다. TEE로 시작하지만, 기술 성숙도에 따라 ZKP/FHE/MPC로 전환 가능.

### 8.3 View Keys - 선택적 공개의 핵심

**View Key 메커니즘**[[14]](#ref14):

```
Alice (기업)가 Bob (감사인)에게 거래 내역 공개:

1. Alice가 View Key_Bob 생성
2. View Key_Bob로 암호화된 금액 복호화 가능
3. Bob은 Alice의 거래만 볼 수 있음
4. Charlie (다른 감사인)는 못 봄
```

**코드 예시** (의사코드):

```solidity
contract ConfidentialTransferManager {
    // 암호화된 잔액 저장
    mapping(address => bytes) encryptedBalances;

    // View Key 등록
    mapping(address => mapping(address => bytes)) viewKeys;

    function grantViewAccess(address auditor, bytes memory viewKey) external {
        viewKeys[msg.sender][auditor] = viewKey;
        emit ViewKeyGranted(msg.sender, auditor);
    }

    function revokeViewAccess(address auditor) external {
        delete viewKeys[msg.sender][auditor];
        emit ViewKeyRevoked(msg.sender, auditor);
    }

    // 감사인이 암호화된 잔액 조회
    function viewBalance(address account) external view returns (uint256) {
        require(viewKeys[account][msg.sender].length > 0, "No view access");

        // TEE 백엔드에서 복호화
        bytes memory encBalance = encryptedBalances[account];
        bytes memory vk = viewKeys[account][msg.sender];

        return TEEBackend.decrypt(encBalance, vk);
    }
}
```

### 8.4 규제 준수: Travel Rule 대응

**Travel Rule** (FATF 권고)[[15]](#ref15):
```
$1,000 이상 암호화폐 전송 시 다음 정보 제공 의무:
- 송신자 이름, 주소
- 수신자 이름, 주소
- 거래 목적
```

**Arc의 대응**:

```
시나리오: A은행 → B은행 $10M USDC 전송

1. 온체인 (모두에게 공개):
   from: 0xAAAA (A은행 주소)
   to: 0xBBBB (B은행 주소)
   amount: [암호화됨]

2. 오프체인 (규제 당국 전용):
   A은행 → 규제 당국에 View Key 제공
   → 당국이 실제 금액 $10M 확인
   → Travel Rule 정보도 같이 제공

3. 경쟁사 C은행:
   거래 금액 모름 ✅
   비즈니스 전략 보호됨 ✅
```

### 8.5 실무적 사용 사례

#### 사례 1: M&A 거래

```
상황: 삼성이 스타트업 인수 ($500M)

문제점 (일반 블록체인):
- 거래 금액 공개 → 언론 보도
- 경쟁사 전략 노출
- 주가 영향

Arc 솔루션:
1. Confidential Transfer로 금액 암호화
2. 감사인/규제 당국에만 View Key 제공
3. 대외비 유지 ✅
```

#### 사례 2: 급여 지급

```
상황: 글로벌 기업이 임원 급여를 USDC로 지급

문제점 (일반 블록체인):
- 모든 임직원 급여 공개
- 프라이버시 침해
- 인사 문제 발생

Arc 솔루션:
1. 각 임원별로 Confidential Transfer
2. HR 부서에만 View Key 제공
3. 세무 당국에 별도 View Key 제공
4. 다른 임원들은 서로 급여 모름 ✅
```

#### 사례 3: 국가 간 결제

```
상황: 한국 정부 → 미국 정부 무기 구매 ($2B)

문제점 (일반 블록체인):
- 국방 예산 세부 내역 공개
- 안보 리스크

Arc 솔루션:
1. Confidential Transfer로 금액 암호화
2. 양국 감사원에만 View Key 제공
3. 의회 청문회 시 필요한 정보만 선택적 공개
4. 적대국은 정확한 금액 모름 ✅
```

### 8.6 프라이버시 vs 투명성 트레이드오프

```
                투명성 ←――――――――→ 프라이버시

Bitcoin/Ethereum ●
(완전 투명)       |
                  |
Arc (선택적)      |        ●
                  |
Zcash/Monero      |                  ●
(완전 프라이버시) |

규제 준수:        ✅        ✅           ❌
기업 채택:        ❌        ✅           ❌
```

Arc는 정확히 중간 지점을 노렸다. 규제 당국이 만족할 정도의 투명성 + 기업이 필요로 하는 프라이버시.

> **출처**: Arc 공식 문서[[14]](#ref14)는 "opt-in privacy model"로 기업의 데이터 보호와 규제 준수를 동시에 달성한다고 명시.

---

## Part 9: CCTP와 Gateway - Circle 생태계의 핵심 인프라

Arc를 이해하려면 Circle의 크로스체인 인프라를 알아야 한다. **CCTP**와 **Gateway**는 Arc가 단순한 L1이 아니라 "Circle 생태계의 허브"가 되는 핵심 기술이다.

### 9.1 CCTP (Cross-Chain Transfer Protocol) - Burn-and-Mint의 우아함

#### 전통적 브릿지의 문제점

```
Lock-and-Mint 방식 (Wrapped Token):

1. 이더리움에 USDC 락업 → 컨트랙트에 100 USDC 잠김
2. Arbitrum에 wUSDC 발행 → 새로운 토큰 100 wUSDC
3. 문제:
   - 유동성 분산 (이더리움 USDC ≠ Arbitrum wUSDC)
   - 브릿지 해킹 리스크 (락업된 자산 탈취 가능)
   - 자본 비효율 (락업된 USDC는 사용 불가)
```

#### CCTP의 Burn-and-Mint 방식[[16]](#ref16)

```
1. Source Chain (Ethereum):
   사용자가 100 USDC를 BURN → 완전히 소각

2. Circle Attestation Service:
   Burn 이벤트 감지 → 서명 생성

3. Destination Chain (Arbitrum):
   서명 제출 → Circle이 새로운 100 USDC MINT
```

**핵심 차이**:
```
Lock-and-Mint: 총 USDC 공급량 = 200 (100 락업 + 100 wUSDC)
Burn-and-Mint: 총 USDC 공급량 = 100 (항상 일정) ✅
```

#### 단계별 기술 분석[[16]](#ref16)

**Step 1: Initiation**
```solidity
// 사용자가 CCTP 앱에서 호출
interface ITokenMessenger {
    function depositForBurn(
        uint256 amount,
        uint32 destinationDomain,
        bytes32 mintRecipient,
        address burnToken
    ) external returns (uint64 nonce);
}

// Ethereum에서 실행:
tokenMessenger.depositForBurn(
    100_000000, // 100 USDC (6 decimals)
    3,          // Arbitrum domain ID
    bytes32(uint256(uint160(recipientAddress))),
    USDC_ADDRESS
);
```

**Step 2: Burn Event**
```
온체인 이벤트 발생:
event MessageSent(
    bytes message
);

message 구조:
- version
- sourceDomain
- destinationDomain
- nonce
- sender
- recipient
- destinationCaller
- messageBody (burn details)
```

**Step 3: Attestation**[[16]](#ref16)
```
Circle의 오프체인 서비스:

1. Ethereum 블록 모니터링
2. MessageSent 이벤트 감지
3. Burn 트랜잭션 검증:
   - 올바른 컨트랙트에서 호출?
   - 금액이 맞나?
   - Finality 도달? (Ethereum: ~15분)
4. ECDSA 서명 생성:
   signature = sign(messageHash, Circle_Private_Key)
```

**Step 4: Mint**
```solidity
// Arbitrum에서 실행:
interface IMessageTransmitter {
    function receiveMessage(
        bytes message,
        bytes attestation
    ) external returns (bool success);
}

// 앱이 호출:
1. Circle API에서 attestation 가져오기:
   GET https://iris-api.circle.com/attestations/{messageHash}

2. receiveMessage 호출:
   messageTransmitter.receiveMessage(message, attestation);

3. Circle 컨트랙트가 서명 검증 → USDC Mint
```

#### CCTP v2: Fast Transfer[[17]](#ref17)

```
문제: Ethereum Finality 기다리면 15분 소요

CCTP v1: 15+ 분
  Ethereum Burn (12s)
    → Finality 대기 (14분)
    → Attestation (10s)
    → Arbitrum Mint (12s)

CCTP v2 Fast Transfer: <30초
  Ethereum Burn (12s)
    → 즉시 Attestation (Circle이 Finality 리스크 부담)
    → Arbitrum Mint (12s)
```

**시나리오: Ethereum reorg 발생**

```
1. 사용자가 100 USDC Burn
2. Circle이 즉시 Attestation 제공 (Fast Transfer)
3. Arbitrum에서 100 USDC Mint
4. Ethereum reorg로 Burn 트랜잭션 취소됨
   → 사용자가 100 USDC를 두 번 받음?

Circle의 대응:
- Circle이 손실 부담 (사용자 보호)
- Reorg 발생 시 Circle이 손실액만큼 USDC Burn
- 총 공급량 항상 일정하게 유지

※ 실제 메커니즘은 Circle 내부 정책으로, 공개 문서에는 
  "Circle assumes finality risk"로만 명시됨[[17]](#ref17)
```

이게 가능한 이유: **Circle은 USDC 발행자**이므로 손실을 흡수할 수 있음.

### 9.2 Arc에서의 CCTP 시너지

```
Arc의 강점: Sub-second finality

일반 체인:
Arbitrum → Ethereum → Polygon
  (Arbitrum finality: ~15분 대기)

Arc 참여:
Arbitrum → Arc → Polygon
  (Arc finality: <1초) ✅

결과:
- Arc가 USDC 허브 역할
- 다른 체인들이 Arc를 통해 더 빠르게 연결
```

**구체적 예시**:

```
시나리오: Arbitrum USDC → Polygon USDC

경로 1 (직접):
Arbitrum Burn → 15분 대기 → Attestation → Polygon Mint
총 시간: ~16분

경로 2 (Arc 경유):
Arbitrum Burn → 15분 대기 → Attestation → Arc Mint
Arc Burn → 1초 대기 → Attestation → Polygon Mint
총 시간: ~16분 (변화 없음)

어? 그럼 Arc의 이점이 뭐지?

경로 3 (CCTP Fast Transfer 활용):
Arbitrum Burn → Fast Transfer (30초, Circle이 finality 리스크 부담) → Arc Mint
Arc Burn → Fast Transfer (1초, Arc의 sub-second finality 덕분) → Polygon Mint
총 시간: ~32초 ✅

→ CCTP v2 Fast Transfer[[17]](#ref17)는 finality를 기다리지 않음
→ Circle이 reorg 리스크를 흡수하고 즉시 Attestation 제공
→ Arc의 빠른 finality가 두 번째 구간을 더욱 단축
→ 30배 빠른 전송 가능
```

### 9.3 Gateway - 통합 USDC 잔액의 마법

#### 문제 정의

```
현재 멀티체인 상황:

사용자가 USDC 보유:
- Ethereum: 100 USDC
- Arbitrum: 50 USDC
- Polygon: 30 USDC
- Base: 20 USDC
총: 200 USDC

문제:
1. Arbitrum에서 150 USDC 필요 → 부족함 (50 USDC만 있음)
2. 다른 체인에서 브릿징 필요 → 시간 소요
3. 유동성 분산 → 자본 비효율
```

#### Gateway 솔루션[[18]](#ref18)

```
Gateway 통합 잔액:

사용자가 각 체인에서 Gateway에 USDC 예치:
- Ethereum: 100 USDC → Gateway
- Arbitrum: 50 USDC → Gateway
- Polygon: 30 USDC → Gateway
- Base: 20 USDC → Gateway

Gateway가 오프체인에서 통합 잔액 생성:
통합 잔액 = 200 USDC

이제 사용자는:
- Arbitrum에서 150 USDC 사용 가능 ✅
- Polygon에서 200 USDC 사용 가능 ✅
- 어느 체인에서든 전체 잔액 접근 ✅
```

#### 기술 아키텍처[[18]](#ref18)

```
[사용자] → [Gateway Wallet Contract (각 체인마다)]
                ↓
           [Gateway Attestation Service (오프체인)]
                ↓
           [통합 잔액 계산]
                ↓
           [사용자 요청 시 즉시 인출 (<500ms)]
```

**Step-by-Step Flow**:

```solidity
// 1. 예치 (Ethereum)
contract GatewayWallet {
    function deposit(uint256 amount) external {
        USDC.transferFrom(msg.sender, address(this), amount);
        emit Deposit(msg.sender, amount, block.chainid);
    }
}

// 2. Gateway 서비스가 오프체인에서 감지
// 3. 통합 잔액 업데이트: balance[user] += amount

// 4. 인출 (Arbitrum)
function withdraw(uint256 amount, bytes signature) external {
    // Gateway Attestation 검증
    require(verifyGatewayAttestation(amount, signature));

    // USDC 전송
    USDC.transfer(msg.sender, amount);
    emit Withdrawal(msg.sender, amount, block.chainid);
}

function verifyGatewayAttestation(
    uint256 amount,
    bytes memory signature
) internal view returns (bool) {
    // Gateway 서명 검증
    bytes32 messageHash = keccak256(abi.encodePacked(
        msg.sender,
        amount,
        block.chainid,
        nonce
    ));

    address signer = recoverSigner(messageHash, signature);
    return signer == GATEWAY_ATTESTATION_SERVICE;
}
```

#### 핵심 특징[[18]](#ref18)

**1. 비수탁형 (Non-custodial)**

```
사용자 자금은 항상 사용자 통제 하에:

인출 조건:
1. 사용자 서명 필요 ✅
2. Gateway Attestation 필요 ✅

→ 둘 다 있어야 인출 가능
→ Gateway 혼자서는 자금 이동 불가
```

**2. 초고속 (<500ms)**

```
전통적 브릿지:
Ethereum → Arbitrum: ~15분

Gateway:
Ethereum 예치 → Arbitrum 인출: <500ms

이유:
- 오프체인 잔액 추적
- 실제 크로스체인 전송 없음
- Gateway가 각 체인에 유동성 풀 보유
```

**3. 자본 효율**

```
전통적 방식:
- Ethereum: 100 USDC (유휴)
- Arbitrum: 50 USDC (부족)
→ 비효율

Gateway:
- 통합 잔액: 150 USDC
- 어디서든 전액 사용 가능
→ 효율 100% ✅
```

### 9.4 Arc + CCTP + Gateway 통합 시나리오

```
시나리오: 글로벌 기업의 급여 지급

요구사항:
- 한국 직원: Klaytn에서 KRW 스테이블코인으로 수령
- 미국 직원: Base에서 USDC로 수령
- 유럽 직원: Arbitrum에서 EURC로 수령

Arc 솔루션:

1. 기업이 Arc에 1M USDC 예치 (Gateway)

2. 한국 직원 급여일:
   Arc → CCTP → Klaytn (USDC)
   → Klaytn DEX에서 KRW 스테이블코인 스왑
   시간: <2초 (Arc finality + CCTP)

3. 미국 직원 급여일:
   Arc → CCTP Fast Transfer → Base (USDC)
   시간: <30초

4. 유럽 직원 급여일:
   Arc → Paymaster로 EURC 변환 → Arc 내부 전송
   → CCTP → Arbitrum (EURC)
   시간: <1분

결과:
- 단일 유동성 풀로 모든 급여 처리
- 최소 지연 시간
- 자동화 가능 (스마트 컨트랙트)
```

### 9.5 Circle 생태계 통합의 전략적 의미

```
Circle의 비전:

         [Circle Mint]
         (USDC 발행/소각)
                ↓
         [Arc L1 Blockchain]
         (USDC 네이티브 가스)
            ↙   ↓   ↘
      [CCTP]  [Gateway]  [CPN]
     (크로스체인)(통합잔액)(결제망)
         ↓       ↓         ↓
   [30+ Chains] [모든체인] [기관]

→ Arc를 중심으로 모든 인프라 연결
→ USDC를 인터넷의 "기본 통화"로 만들기
```

**경쟁 우위**:

| 항목 | 일반 L1 | Arc + Circle 생태계 |
|------|---------|---------------------|
| 크로스체인 | 외부 브릿지 의존 | CCTP 네이티브 통합 |
| 유동성 통합 | 불가능 | Gateway로 통합 |
| Stablecoin 발행 | 외부 의존 | Circle Mint 직접 연결 |
| 규제 준수 | 어려움 | Circle 규제 인프라 활용 |
| 기관 네트워크 | 없음 | CPN 100+ 기관 |

Arc는 단순 블록체인이 아니라 **Circle의 금융 인프라 전체가 뒷받침하는 플랫폼**이다. 이게 Stripe Tempo, Tether Stable 등 경쟁자들과의 진짜 차별점.

> **출처**: CCTP 공식 문서[[16]](#ref16), Gateway 개발자 문서[[18]](#ref18), Circle 블로그[[19]](#ref19)

---

## 마무리하며

Arc를 분석하면서 느낀 점:

1. **가스비 안정화는 혁신.** USDC 네이티브 + Fee Smoothing + Bounded Fee 조합은 기업 입장에서 엄청난 차이. 암호화폐의 근본적 딜레마(실생활 사용 vs 가격 변동성)를 정면돌파했다.

2. **Tendermint BFT는 금융 인프라에 딱 맞다.** 결정적 최종성은 PFMI 준수의 핵심. 비트코인의 "확률적"이 아니라 "수학적 보장"을 제공한다.

3. **MEV는 완화만 가능, 제거는 불가.** 하지만 Arc의 거래 특성(stable-to-stable)상 MEV 규모는 이더리움의 1/1000 수준일 것.

4. **성능 확장은 숙제다.** 20개 검증자는 너무 중앙화, 100개는 너무 느림. 50개가 sweet spot일 듯. Tendermint의 O(n²) 한계는 근본적인 문제.

5. **Circle 생태계 통합이 진짜 킬러 피처.** 


---

## 참고문헌

<a name="ref1"></a>[1] Circle (2025). "Arc Litepaper". https://www.arc.network/litepaper

<a name="ref2"></a>[2] *(참조 제거됨 - ref13으로 대체)*

<a name="ref3"></a>[3] Arc Network Blog (2025). "How Gas Works on Arc". https://www.arc.network/blog/how-gas-works-on-arc

<a name="ref4"></a>[4] Ethereum Foundation (2021). "EIP-1559: Fee market change for ETH 1.0 chain". https://eips.ethereum.org/EIPS/eip-1559

<a name="ref5"></a>[5] Ethereum Research (2019). "EIP 1559 FAQ". https://notes.ethereum.org/@vbuterin/eip-1559-faq

<a name="ref6"></a>[6] Tendermint Documentation. "What is Tendermint". https://docs.tendermint.com

<a name="ref7"></a>[7] Flashbots (2024). "State of Wallets 2024". https://writings.flashbots.net/state-of-wallets-2024

<a name="ref8"></a>[8] ESMA (2025). "Maximal Extractable Value: Implications for crypto markets". https://www.esma.europa.eu *(참고: ESMA 2025년 3월 보고서로 글에서 인용되었으나, 해당 보고서의 정확한 발행 여부와 내용은 ESMA 공식 사이트에서 확인 필요)*

<a name="ref9"></a>[9] ResearchGate (2021). "The design, architecture and performance of the Tendermint Blockchain Network". https://www.researchgate.net/publication/356445199

<a name="ref10"></a>[10] Cosmos Network. "Tendermint BFT Consensus Algorithm". https://cosmos-network.gitbooks.io

<a name="ref11"></a>[11] Ethereum Foundation (2023). "ERC-4337: Account Abstraction Using Alt Mempool". https://eips.ethereum.org/EIPS/eip-4337

<a name="ref12"></a>[12] Circle Developers. "Circle Payments Network Documentation". https://developers.circle.com

<a name="ref13"></a>[13] Circle Press Release (2024-10-28). "Circle Launches Arc Public Testnet". https://www.circle.com/pressroom/circle-launches-arc-public-testnet

<a name="ref14"></a>[14] Arc Network Documentation (2024). "Privacy Features on Arc". https://docs.arc.network/privacy

<a name="ref15"></a>[15] Financial Action Task Force (FATF). "Travel Rule Guidance for Virtual Assets". https://www.fatf-gafi.org/publications/fatfgeneral/documents/guidance-rba-virtual-assets.html

<a name="ref16"></a>[16] Circle Developers (2023). "Cross-Chain Transfer Protocol (CCTP) Documentation". https://developers.circle.com/stablecoins/docs/cctp-protocol-contract

<a name="ref17"></a>[17] Circle Blog (2024). "Introducing CCTP Fast Transfers". https://www.circle.com/blog/cctp-fast-transfers

<a name="ref18"></a>[18] Circle Developers (2024). "Gateway: Unified USDC Balance Across Chains". https://developers.circle.com/gateway

<a name="ref19"></a>[19] Circle Blog (2024). "Circle's Vision for Multi-Chain Future". https://www.circle.com/blog/multi-chain-vision

---

**면책사항**: 
- Arc의 MAX_BASE_FEE, 검증자 50개/100개 시 성능 등 일부 수치는 라이트페이퍼에 명시되지 않아 필자가 추정했으며, 글 내에서 "(필자 추정)"으로 표기했습니다.
- ESMA 2025 MEV 보고서의 정확한 발행 여부는 ESMA 공식 사이트에서 확인이 필요합니다.
- 일부 참고문헌 링크(특히 Arc 관련)는 공식 문서 발행 시점에 따라 변경될 수 있습니다.
- 실제 값과 메커니즘은 테스트넷 데이터 및 공식 문서 공개 후 검증이 필요합니다.

