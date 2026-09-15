---
title: Cloud Monitoring으로 내 사이트 감시하기 (가동시간 체크 + 이메일 알림)
date: 2026-09-15
categories:
  - Google Cloud
tags:
  - GCP
  - Cloud Monitoring
  - Cloud Logging
  - BigQuery
  - 알림
excerpt: 운영 중인 정적 사이트가 죽거나 BigQuery 예약 쿼리가 실패하면 메일이 오게 만들었다. 서버 0개, 비용 0원. 검사 주기 제한과 로그 알림 테스트에서 한 번씩 걸렸다.
---

실습하면서 만든 사이트가 어느새 네 개가 됐다. [로또 예상번호](/gcs-static-hosting/), [점심 메뉴](/lunch-box/), BigQuery로 모으는 트렌드 아카이브, 그리고 이 블로그. 문제는 **하나가 죽어도 내가 들어가 보기 전까지는 모른다**는 거였다. 트렌드 아카이브는 더 조용하다. 예약 쿼리가 실패해도 사이트는 멀쩡히 뜨고, 데이터만 그날부터 멈춘다.

그래서 Cloud Monitoring으로 감시를 붙였다. 새로 띄운 서버, 컨테이너, 함수는 하나도 없다. Monitoring 설정만으로 끝난다.

최종 구성은 이렇다.

```
[가동시간 체크]  15분마다
  Google 체크 서버 (asia-pacific / usa-oregon / europe)
     ├─ HTTPS GET ──> 사이트 4곳
     └─ 결과 기록 ──> 지표 check_passed ──> 알림 정책 ──> 이메일

[로그 기반 알림]
  BigQuery 예약 쿼리 ──> Cloud Logging (ERROR) ──> 알림 정책 ──> 이메일
```

## 1. 비용부터 확인했다

"무료로" 하는 게 조건이라 [가격표](https://cloud.google.com/stackdriver/pricing)부터 봤다.

| 항목 | 무료 한도 / 가격 |
| --- | --- |
| 가동시간 체크 실행 | **프로젝트당 월 100만 회** 무료, 넘으면 1,000회당 $0.30 |
| 알림 정책 | **2027년 9월 1일 이전에는 과금 없음** |
| 가동시간 지표로 거는 알림 | 과금이 시작돼도 무료 |
| 로그 기반 알림 | 과금 시작 뒤 조건당 월 $0.35 |

가동시간 체크는 **리전별로 한 번씩** 센다. 3개 리전에서 검사하면 1회 검사가 3회 실행이다. 사이트 4곳을 15분마다 검사하면 이렇게 된다.

```
4곳 × 3개 리전 × 하루 96회 × 30일 = 월 34,560회
```

한도의 3.5%라 한참 남는다.

## 2. 가동시간 체크 — 30분 주기는 없다

처음엔 30분마다 검사하려고 했다. 그런데 **가동시간 체크 주기는 1분, 5분, 10분, 15분 넷 중 하나만** 고를 수 있다. 30분이나 1시간은 API가 받지 않는다. 가장 긴 15분으로 정했다.

체크 리전도 제약이 있다. 리전을 직접 고르려면 **최소 3개**를 골라야 한다. 한 리전의 일시적인 네트워크 문제로 오탐이 나는 걸 막으려는 설계로 보인다.

REST API로 보낸 체크 설정은 이렇다.

```python
{
    "displayName": "site-watch lottery",
    "monitoredResource": {"type": "uptime_url",
                          "labels": {"project_id": project, "host": "storage.googleapis.com"}},
    "httpCheck": {"path": "/<bucket>/index.html", "port": 443, "useSsl": True, "validateSsl": True,
                  "acceptedResponseStatusCodes": [{"statusClass": "STATUS_CLASS_2XX"}]},
    "period": "900s",
    "timeout": "10s",
    "selectedRegions": ["ASIA_PACIFIC", "USA_OREGON", "EUROPE"],
}
```

GCS 정적 사이트는 호스트가 `storage.googleapis.com` 이고 버킷 이름이 경로에 들어간다. `host` 와 `path` 를 나눠 넣어야 한다.

가동시간 체크는 **리전이 없는 프로젝트 단위 리소스**다. VPC도 서비스 계정도 필요 없다. 대상이 공개 URL이라 구글 체크 서버가 인터넷으로 접속하면 된다. 그래서 사이트가 어느 프로젝트에 있든 상관없이 감시용 프로젝트 하나에 몰아서 만들었다.

## 3. 알림 조건 — "2개 이상 리전에서 실패"

체크 결과는 `monitoring.googleapis.com/uptime_check/check_passed` 지표에 **리전별 true/false**로 쌓인다. 알림 정책은 이 지표를 이렇게 집계한다.

```python
"conditionThreshold": {
    "filter": 'metric.type="monitoring.googleapis.com/uptime_check/check_passed" AND resource.type="uptime_url"',
    "aggregations": [{
        "alignmentPeriod": "1800s",
        "perSeriesAligner": "ALIGN_NEXT_OLDER",      # 리전별로 가장 최근 값 하나
        "crossSeriesReducer": "REDUCE_COUNT_FALSE",  # 그중 false 인 리전 수
        "groupByFields": ["metric.label.check_id"],  # 사이트별로 따로
    }],
    "comparison": "COMPARISON_GT", "thresholdValue": 1,  # 1보다 크면 = 2개 이상
    "duration": "60s",
}
```

`groupByFields` 를 체크 ID로 두면 **정책 하나로 모든 사이트를 감시**한다. 나중에 체크를 추가해도 정책은 손댈 필요가 없다.

같은 방식으로 `time_until_ssl_cert_expires` 지표에 "7일 미만" 조건을 걸어서 **인증서 만료 임박** 알림도 만들었다. 체크에 `validateSsl` 을 켜야 이 지표가 쌓인다.

## 4. 로그 기반 알림 — 예약 쿼리 실패 잡기

BigQuery 예약 쿼리는 실패해도 사이트는 멀쩡하다. 가동시간 체크로는 못 잡는다. 대신 실행할 때마다 Cloud Logging에 로그가 남는다.

```bash
$ gcloud logging read 'resource.type="bigquery_dts_config"' --project=<project> --limit=3 \
    --format='value(severity,textPayload)'
INFO    Summary: succeeded 1 jobs, failed 0 jobs.
INFO    Job scheduled_query_... completed successfully.
INFO    Job scheduled_query_... started.
```

그래서 로그 필터 `resource.type="bigquery_dts_config" AND severity>=ERROR` 에 걸리는 로그가 생기면 알림을 보내게 했다.

여기서 두 가지를 알게 됐다.

**로그 기반 알림은 `notificationRateLimit` 이 필수다.** 에러 로그가 수백 줄 쏟아지면 메일도 수백 통 올 수 있어서다. 1시간에 최대 1통으로 잡았다.

```python
"conditions": [{"displayName": name, "conditionMatchedLog": {"filter": log_filter}}],
"alertStrategy": {"notificationRateLimit": {"period": "3600s"}, "autoClose": "1800s"},
```

**알림 채널은 프로젝트에 속한다.** 로그 기반 알림은 로그가 쌓이는 프로젝트에 만들어야 하고, 그 정책이 쓰는 이메일 채널도 같은 프로젝트에 있어야 한다. 감시용 프로젝트에 만든 채널을 가져다 쓸 수 없어서 채널을 하나 더 만들었다.

## 5. 한 번에 만드는 스크립트

콘솔에서 클릭으로 만들면 다음에 또 해야 할 때 기억에 의존하게 된다. 그래서 설정 파일 하나를 읽어서 전부 만드는 파이썬 스크립트로 짰다. 표준 라이브러리만 쓰고, 인증은 `gcloud auth print-access-token` 으로 받는다.

```json
{
  "project": "",
  "email": "",
  "sites": {
    "lottery": "https://storage.googleapis.com/<bucket>/index.html",
    "blog": "https://<github-user>.github.io/"
  },
  "log_alerts": {
    "bigquery-scheduled-query-failed": {
      "project": "<project-with-scheduled-queries>",
      "filter": "resource.type=\"bigquery_dts_config\" AND severity>=ERROR"
    }
  }
}
```

`project` 와 `email` 을 비워두면 gcloud의 현재 프로젝트와 로그인 계정을 쓴다. 실제 URL과 주소가 들어간 설정 파일은 `.gitignore` 에 넣었다.

핵심은 **이름으로 찾고, 없을 때만 만드는** 함수 하나다.

```python
def ensure(project, kind, body):
    """kind = notificationChannels | uptimeCheckConfigs | alertPolicies"""
    for item in call("GET", f"projects/{project}/{kind}?pageSize=1000").get(kind, []):
        if item["displayName"] == body["displayName"]:
            print(f"있음  {kind:20} {body['displayName']}")
            return item["name"]
    print(f"생성  {kind:20} {body['displayName']}")
    return call("POST", f"projects/{project}/{kind}", body)["name"]
```

세 가지 리소스 모두 목록 조회 응답의 키가 URL 경로와 같아서 함수 하나로 처리된다.

```bash
$ python3 watch.py
생성  uptimeCheckConfigs   site-watch lottery
생성  uptimeCheckConfigs   site-watch lunch-box
생성  uptimeCheckConfigs   site-watch trends-kr
생성  uptimeCheckConfigs   site-watch blog
생성  notificationChannels site-watch email
생성  alertPolicies        site-watch: 사이트 다운
생성  alertPolicies        site-watch: SSL 인증서 만료 임박
생성  notificationChannels site-watch email
생성  alertPolicies        site-watch: bigquery-scheduled-query-failed

$ python3 watch.py        # 다시 돌려도 중복 생성 없음
있음  uptimeCheckConfigs   site-watch lottery
...
```

한계도 있다. 이름이 같으면 내용이 바뀌어도 건너뛴다. 설정을 고치려면 콘솔에서 지우고 다시 돌린다. 리소스가 몇 개 안 돼서 갱신 로직은 넣지 않았다.

## 6. 알림이 진짜 오는지 테스트

만들기만 하고 알림이 안 오면 의미가 없다. 사이트를 일부러 죽일 수는 없으니 **실패하는 체크를 하나 더** 만들었다. 존재하지 않는 경로를 1분 주기로 검사한다.

```bash
$ gcloud monitoring uptime create "site-watch test-404" \
    --resource-type=uptime-url \
    --resource-labels=host=storage.googleapis.com,project_id=<project> \
    --path=/<bucket>/no-such-page.html --protocol=https \
    --period=1 --regions=asia-pacific,usa-oregon,europe
```

2분 뒤 3개 리전 모두 실패로 기록됐고, **만든 지 5분 만에 인시던트가 열리고 메일이 왔다.** 확인 후 테스트 체크는 지웠다.

로그 기반 알림은 가짜 ERROR 로그를 한 줄 써서 테스트했다. `gcloud logging write` 는 기본 리소스 타입이 `global` 이라서, 필터에 걸리도록 `bigquery_dts_config` 타입을 직접 지정해야 한다.

```bash
$ gcloud logging write site-watch-test "알림 시험용 가짜 에러" --project=<project> \
    --severity=ERROR --monitored-resource-type=bigquery_dts_config \
    --monitored-resource-labels=project_id=<project>,location=us,config_id=test
```

**첫 시도는 알림이 안 왔다.** 정책을 만들고 1분쯤 뒤에 로그를 썼는데, 로그는 분명히 기록됐는데도 인시던트가 열리지 않았다. 정책이 만들어진 직후라 아직 적용 전이었던 것으로 보인다. 6분 뒤 같은 로그를 다시 쓰니 **2분 만에 인시던트가 열렸다.** 로그 기반 알림을 테스트할 땐 정책을 만들고 몇 분 기다렸다가 로그를 쓰는 게 좋다.

테스트로 열린 인시던트는 조건이 사라지면 `autoClose` 로 30분 안에 닫히고, 그때 "해결" 메일이 한 통 더 온다.

## 정리

- 정적 사이트 감시는 **Cloud Monitoring 가동시간 체크 + 알림 정책**이면 된다. 서버 0개, 개인 규모면 비용 0원이다.
- 검사 주기는 **최대 15분**이다. 30분, 1시간은 없다. 리전은 최소 3개를 골라야 한다.
- 알림 조건은 `REDUCE_COUNT_FALSE` 로 **실패한 리전 수**를 세서 2개 이상일 때만 울리게 하면 오탐이 준다.
- 사이트는 멀쩡한데 데이터가 멈추는 문제는 가동시간 체크로 못 잡는다. **실패 로그에 로그 기반 알림**을 건다. `notificationRateLimit` 은 필수다.
- 알림 채널은 프로젝트 소속이다. 로그 알림을 다른 프로젝트에 걸면 채널도 거기에 하나 더 필요하다.
- 알림은 **일부러 실패시켜서** 확인하자. 로그 알림은 정책을 만든 직후에는 안 잡힐 수 있다.
