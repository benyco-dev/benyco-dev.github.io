---
title: "Claude Code와 토스증권 API로 자동매매 시작하기 (환경 구성)"
date: 2026-09-10
categories:
  - Auto Trading
tags:
  - 자동매매
  - Claude Code
  - 토스증권 API
  - GCP
  - Docker
excerpt: Claude Code로 바이브 코딩하고, 토스증권 Open API와 GCP로 무인 운영 환경을 만들었다. 강의를 참고해 시작한 뒤 전략은 내 목표에 맞게 바꿨다.
---

유튜브 자동매매 강의 시리즈([1강](https://www.youtube.com/watch?v=7VG7ugLu6qs))를 참고해서 미국 주식 자동매매를 만들어봤다. 코드는 전부 공개해뒀다 — [`project-auto-trading-scanner`](https://github.com/benyco-dev/project-auto-trading-scanner).

3부작으로 나눠 정리한다. 1편은 환경 구성과 전략 결정, 2편은 수수료, 3편은 백테스트를 어디까지 믿을 것인가.

먼저 어떻게 구성했는지부터 훑고, 후반부에 전략을 왜 바꿨는지로 넘어간다.

## 구성 요약

| 영역 | 사용한 것 |
|---|---|
| 코드 작성 | Claude Code (바이브 코딩) |
| 시세 데이터 | `yfinance` (무료, 지연 시세) |
| 주문 | 토스증권 Open API |
| 서버 | GCP Compute Engine (서울, `e2-small`) |
| 배포 | GitHub Actions → Docker Hub → VM `docker pull` |
| 스케줄링 | VM cron 2개 (마감 후 스캔 / 개장 후 매매) |

하루 흐름은 이렇다.

```
06:30 KST (미국장 마감 후)
  cron → docker pull → S&P 500 스캔 → signals_history.csv 누적

22:45 KST (미국장 개장 후)  청산 규칙 검사 → 매도
22:50 KST                  확보된 현금으로 매수
```

스캔은 그날 종가가 확정된 뒤에, 주문은 장이 열린 뒤에 나간다.

## Claude Code로 바이브 코딩하기

코드는 처음부터 끝까지 Claude Code로 작성했다. 터미널에서 자연어로 요구사항을 말하면 파일을 직접 읽고 고치는 CLI 도구다.

```bash
npm install -g @anthropic-ai/claude-code
cd auto-trading-scanner
claude
```

실제로 쓴 흐름은 이렇다.

**1. 프로젝트 규칙을 `CLAUDE.md`에 적어둔다**

세션마다 자동으로 읽히는 파일이다. 매번 설명하기 싫은 것들을 여기 넣는다.

```markdown
- 로컬에서 docker build 하지 않는다. GitHub Actions에서 빌드/푸시.
- 실주문 경로를 건드리는 변경은 항상 dry-run 기본값을 유지할 것.
- 백테스트 결과는 커밋 메시지에 숫자로 남긴다.
```

**2. 한 번에 한 덩어리씩 요청한다**

"자동매매 만들어줘"가 아니라 이 정도 크기로 끊는다.

```
RSI(2)가 5 이하이고 종가가 200일선 위일 때 진입하는 전략을
strategies.py에 추가해줘. 청산은 10일선 회복 / ATR 2.5배 손절 /
20거래일 시간청산.
```

**3. 돌려보고, 결과를 그대로 다시 던진다**

```
백테스트 돌렸더니 트레이드당 평균이 +0.37%인데 왕복 수수료가 0.2%야.
이거 수수료 반영한 포트폴리오 시뮬레이션도 만들어줘.
```

2편의 발견이 실제로 이 대화에서 나왔다. 바이브 코딩이 잘 굴러가는 지점은 "코드를 대신 써주는 것"보다 **결과를 붙여넣고 다음 질문으로 넘어가는 속도**다.

**4. 검증은 코드로 남긴다**

돌려보고 "잘 되네" 하고 넘어가면 다음 수정에서 조용히 깨진다. 최소한의 assert 파일 하나를 만들게 해서 계속 갱신했다.

```bash
python test_checks.py   # → all checks passed
```

**5. 커밋 메시지에 실험 결과를 적게 한다**

```
Hold longer (SMA10 / 20 days) so commission stops eating the edge

Round-trip commission is ~0.2% and fixed per trade, while the SMA5 exit
averaged only +0.41% per trade over 3.5 days — roughly half the gross
edge went to fees. Exiting on SMA10 with a 20-day cap averages +0.57%
over 5.9 days.
```

2주 뒤에 "이거 왜 SMA10이었지?" 하고 다시 안 돌려봐도 된다.

**6. 위험한 건 권한으로 막는다**

실주문이 나가는 스크립트는 자동 실행 대상에서 빼두고, 수정 제안만 받았다.

## 토스증권 Open API 설정

주문은 [토스증권 Open API](https://corp.tossinvest.com/ko/open-api)로 나간다. 토스증권 계좌가 있어야 하고, WTS(웹 트레이딩 시스템) → 설정 → Open API에서 발급한다.

발급하면서 등록할 것이 두 가지다.

1. **`client_id` / `client_secret`** — `client_secret`은 발급 시 한 번만 보여준다.
2. **허용 IP 목록** — 여기 등록된 IP에서만 호출된다. 등록 안 된 IP는 403. 그래서 서버는 고정 IP를 잡아뒀다.

인증은 OAuth2 client credentials다.

```bash
curl -X POST 'https://openapi.tossinvest.com/oauth2/token' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'grant_type=client_credentials' \
  -d 'client_id=...' -d 'client_secret=...'
```

`access_token`은 3600초 만료라 클라이언트에서 자동 재발급하게 했다. 계좌·자산·주문 API는 `X-Tossinvest-Account` 헤더에 `accountSeq`가 필요하고, 이 값은 계좌 목록 조회로 확인한다.

자격증명은 GCP Secret Manager에 넣고 VM에서 꺼내 쓴다.

```bash
echo -n "발급받은_client_secret" | gcloud secrets create toss-client-secret --data-file=-
```

## GCP 구성

서버는 GCP Compute Engine에 만들었다.

| 항목 | 값 |
|---|---|
| 리전 / 존 | `asia-northeast3-a` (서울) |
| 머신 타입 | `e2-small` |
| OS | Debian 12 |
| 디스크 | 10GB |
| 외부 IP | 고정 IP 예약 (토스 API 허용목록 등록용) |

VM 생성은 [`cloud/deploy.sh`](https://github.com/benyco-dev/project-auto-trading-scanner/blob/main/cloud/deploy.sh) 한 번으로 끝난다.

```bash
./cloud/deploy.sh
```

보안 설정은 [`cloud/security-hardening.sh`](https://github.com/benyco-dev/project-auto-trading-scanner/blob/main/cloud/security-hardening.sh)로 따로 적용했다. 주요 항목 세 가지.

| 항목 | 기본 상태 | 적용한 것 |
|---|---|---|
| **IAP 전용 SSH** | `0.0.0.0/0`에서 22번 포트 허용 | 공개 방화벽 규칙 삭제, IAP 터널로만 접속 |
| **최소 권한 서비스 계정** | 프로젝트 Editor 권한의 기본 계정 | 전용 계정 생성 후 교체 |
| **Shielded VM** | 일반 부팅 | Secure Boot 활성화 |

접속은 이렇게 한다.

```bash
gcloud compute ssh auto-trading-scanner --zone=asia-northeast3-a --tunnel-through-iap
```

## 배포

```
git push
  → GitHub Actions가 Dockerfile로 이미지 빌드
  → Docker Hub push (:latest 와 :커밋SHA 두 태그)
  → VM이 다음 cron 때 docker pull
```

CI 설정은 [`.github/workflows/docker-build.yml`](https://github.com/benyco-dev/project-auto-trading-scanner/blob/main/.github/workflows/docker-build.yml)이고, Docker Hub 자격증명은 저장소 Secrets에 넣어뒀다. 로컬에서는 `docker build`를 하지 않는다.

이미지는 non-root 사용자로 돌고, 상태 파일(`signals_history.csv` 등)은 `/data` 볼륨 마운트로 VM에 남긴다.

```dockerfile
RUN useradd --create-home --uid 1000 scanner \
    && mkdir -p /data && chown -R scanner:scanner /app /data
USER scanner
WORKDIR /data
```

VM cron은 [`cloud/install-trading-cron.sh`](https://github.com/benyco-dev/project-auto-trading-scanner/blob/main/cloud/install-trading-cron.sh)로 등록한다.

여기까지가 환경이다. 아래부터는 그 위에 무엇을 올릴지 정하는 이야기다.

## 여기까지가 강의를 그대로 따라간 부분

강의는 매매 로직을 정의하고 스캐너를 만드는 것까지를 다룬다. 시작하기에 딱 좋은 범위였고, 1강 로직을 그대로 옮겨서 돌려보는 것부터 했다.

- **매수**: MACD(5,25,9) 골든크로스 + 15일 이격도 85% 이하

여기서 갈라지기 시작했다. 강의의 목표는 **스캐너**를 만드는 것이고, 내 목표는 **사람이 안 보는 동안 도는 시스템**이었다. 스캐너는 "지금 살 만한 종목"만 알려주면 되지만, 자동으로 돌아가는 시스템은 **언제 팔지**도 규칙으로 갖고 있어야 한다. 강의 범위 밖의 부분이라, 내가 채워야 했다.

사람이 지켜보는 매매라면 "적당히 오르면 판다"가 명시적 규칙이 아니어도 굴러간다. cron이 매일 주문을 넣는 구조에서는 그 "적당히"를 아무도 판단하지 않는다. 손절가가 정해지지 않은 포지션은 그냥 열려 있을 뿐이다.

그래서 첫 결정을 이렇게 잡았다. **진입 규칙보다 청산 규칙을 먼저 고정한다.**

## 내 방식으로 고른 전략: RSI(2) + ATR

전략을 새로 발명하지는 않았다. 개인이 백테스트 몇 번 돌려 찾은 규칙보다, 오래 검증되고 문서화된 규칙을 조합하는 쪽이 낫다고 봤다.

- **추세 필터**: 종가가 200일 이동평균 위일 때만 매수 고려
- **진입**: RSI(2)가 5 이하 — Larry Connors & Cesar Alvarez의 RSI(2) 평균회귀
- **청산**: 10일 이동평균 회복 시 익절 / 진입가 대비 ATR(14) × 2.5 하락 시 손절 / 20거래일 시간 청산
- **사이징**: `수량 = (계좌자산 × 리스크비율) / (진입가 − 손절가)` — Turtle Trading 방식

핵심은 마지막 두 줄이다. **손절가가 먼저 정해지고, 그 손절가로부터 수량이 역산된다.** 손절폭이 넓은 종목은 자동으로 적게 사고, 좁은 종목은 많이 산다. 결과적으로 어떤 트레이드든 실패했을 때 잃는 금액이 계좌의 1%로 같아진다.

"얼마 살까"를 사람이 판단하지 않아도 되게 만드는 부분이라, 자동화에서는 진입 신호보다 중요했다.

### RSI 기준값: 5냐 10이냐

Connors 원본은 5다. 10으로 완화하면 트레이드가 두 배로 늘어난다. S&P 500 무작위 100종목 × 3년으로 확인했다.

| RSI 기준 | 트레이드 수 | Profit Factor | 최악의 트레이드 |
|---|---|---|---|
| 5 (원본) | 기준 | **1.47** | −17.6% |
| 10 (완화) | 약 2배 | 1.42 | −19.1% |

트레이드 수는 두 배인데 트레이드 품질은 떨어지고 꼬리 위험은 더 깊어진다. 원본값을 유지했다. **"기회를 더 많이 잡는다"가 공짜가 아니라는 것** — 2편에서 이 얘기가 훨씬 더 아프게 돌아온다.

## 두 전략을 같이 돌려서 비교했다

강의 로직을 지운 게 아니라 `legacy`라는 이름으로 코드에 남겨뒀다. `--strategy legacy`로 언제든 비교할 수 있다. S&P 500 무작위 100종목 × 5년:

| 전략 | 트레이드 | 승률 | 평균 수익률 | Profit Factor | 평균 보유일 | 최악 |
|---|---|---|---|---|---|---|
| legacy (MACD+이격도) | 338 | 36.1% | +1.54% | **1.59** | 11.6일 | −21.0% |
| rsi2_trend | 2799 | **67.8%** | +0.37% | 1.37 | 3.5일 | −15.9% |

**Profit Factor만 보면 강의 로직이 더 좋다.** 가끔 크게 먹는 트레이드가 있어서 그렇다. 트레이드당 평균 수익률도 4배 이상 높다.

그런데도 `rsi2_trend`를 기본값으로 삼았다. 이유는 두 가지다.

1. 저 1.59는 손절이 없는 상태로 측정된 숫자다. 이 백테스트는 종가 기준이라 갭 하락 같은 시나리오를 제대로 잡지 못한다. (비교를 위해 "MACD 데드크로스 또는 30거래일 시간청산"을 내가 임의로 붙여서 돌린 결과다 — 강의가 제시한 규칙이 아니다.)
2. 승률이 높고 보유 기간이 짧으면 자본 회전이 빠르다. 자본이 작을수록 이게 중요하다.

숫자가 지는 쪽을 골랐다는 게 이상하게 들리는데, 여기서 실제로 고른 건 수익률이 아니라 **"모든 트레이드에 리스크가 정의되어 있다"는 속성**이었다. 목표가 자동화라서 그렇다.

## 강의와 달라진 지점 정리

| 강의 | 강의에서 다루는 것 | 내가 대신 택한 것 | 왜 |
|---|---|---|---|
| 1강 | MACD+이격도 로직 | `legacy`로 보존, 기본값은 `rsi2_trend` | 무인 운영이라 청산 규칙이 필요했다 |
| 2강 | GitHub Actions + 텔레그램 알림 | GCE VM cron + CSV 누적 로그 | 토스 API의 고정 IP 요구 |
| 3강 | Cloudflare Worker로 스케줄링 | 이미 cron이 담당 | 중복이라 불필요 |
| 이후 | 증권사 API 자동 주문 | 구현하되 **cron에 안 붙임** | 아래 |

마지막 줄이 제일 의도적인 차이다. 시그널을 찾는 것과 실제로 돈을 움직이는 것은 분리된 결정이어야 한다고 봐서, 주문 도구는 사람이 직접 실행하는 수동 도구로 남겼다. 실주문은 `--live` 플래그와 `TOSS_LIVE_TRADING=CONFIRM` 환경변수를 **둘 다** 줘야 나간다.

```bash
# 미리보기 (자격증명 없어도 동작)
python -m toss.place_orders --account-seq 1

# 실제 주문
TOSS_LIVE_TRADING=CONFIRM python -m toss.place_orders --account-seq 1 --live
```

강의 시리즈가 없었으면 시작 자체를 안 했을 거다. 방향을 잡아준 지점과, 내 목표가 달라서 갈라진 지점이 명확해서 오히려 왜 그렇게 만들었는지를 계속 설명할 수 있게 됐다.

---

다음 편: [수수료의 함정](/auto-trading-2-commission/) — 백테스트는 22% 수익이라고 했는데, 수수료를 넣으니 4%가 됐다.

> 투자 조언이 아니다. 백테스트 결과는 참고용 통계일 뿐이고 실거래 결과를 보장하지 않는다.
