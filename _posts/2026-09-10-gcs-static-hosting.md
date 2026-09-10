---
title: Google Cloud Storage로 정적 웹사이트 호스팅하기 (+ 로또 예상번호 사이트)
date: 2026-09-10
categories:
  - Google Cloud
tags:
  - GCP
  - Cloud Storage
  - GitHub Actions
  - 정적 호스팅
  - 트러블슈팅
excerpt: 서버도 컨테이너도 없이 버킷 하나로 웹사이트를 띄우고, GitHub Actions가 키 파일 없이 배포하게 만들었다. 권한과 캐시에서 한 번씩 걸렸다.
---

[Compute Engine에 Docker로 Nginx를 띄워보고](/gcp-docker-nginx/), [Cloud Run에 이미지를 올려보고](/cloud-run-docker-gui/) 나니 자연스럽게 다음 질문이 생겼다. **애초에 서버가 필요 없는 사이트라면 뭘 써야 하나?**

HTML, CSS, JS, JSON 몇 개를 그대로 내려주기만 하면 되는 사이트에 VM은 과하고, Cloud Run도 컨테이너 빌드와 레지스트리가 따라붙는다. 그래서 이번엔 **Cloud Storage 버킷 하나로** 사이트를 띄워봤다.

만든 건 로또 6/45 예상번호 사이트다. 주제는 가벼운데 구성은 실전에 가깝다 — 매주 데이터가 갱신돼야 하고, 그 갱신이 자동으로 배포까지 이어져야 한다.

- 사이트: [storage.googleapis.com/benycolottery/index.html](https://storage.googleapis.com/benycolottery/index.html)
- 코드: [github.com/benyco-dev/lottery](https://github.com/benyco-dev/lottery)

![로또 예상번호 사이트 메인 화면](/images/uploads/2026-09-10-gcs-static-hosting/01-site.webp)

*버킷에서 그대로 내려온 정적 페이지. 서버 프로세스는 0개다.*

최종 구성은 이렇다.

```
GitHub Actions ──(Workload Identity Federation, 키 파일 없음)──> GCS 버킷 ──> 공개 URL
      │
      └─ 데이터 수집 → 검증 → 갱신 커밋 → rsync 배포
```

## 1. 버킷 만들기

프로젝트를 새로 파고 결제를 연결한 다음 버킷을 만들었다. 서울 리전(`asia-northeast3`)으로 잡았다.

```bash
$ gcloud storage buckets create gs://my-project \
    --project=my-project --location=asia-northeast3 --uniform-bucket-level-access
Creating gs://my-project/...
```

`--uniform-bucket-level-access` 를 켜면 객체마다 ACL을 따로 두지 않고 **버킷 IAM 하나로만** 접근을 통제한다. 정적 사이트처럼 "전부 공개"인 경우엔 이게 훨씬 단순하고, 파일을 새로 올릴 때마다 권한이 제각각이 되는 사고를 막아준다.

## 2. 공개 읽기 권한

버킷을 만들었다고 바로 보이는 건 아니다. `allUsers` 에게 읽기 권한을 줘야 한다.

```bash
$ gcloud storage buckets add-iam-policy-binding gs://my-project \
    --member=allUsers --role=roles/storage.objectViewer --project=my-project
  - allUsers
  role: roles/storage.objectViewer
```

조직에 `constraints/storage.publicAccessPrevention` 정책이 걸려 있으면 이 명령이 거부된다. 그럴 땐 조직 정책에서 해당 프로젝트를 예외로 두거나, 로드밸런서를 앞에 세우는 구성으로 가야 한다.

## 3. 업로드 — 첫 번째로 걸린 곳

`gcloud storage rsync` 로 폴더를 통째로 올린다.

```bash
$ gcloud storage rsync site gs://my-project --recursive \
    --delete-unmatched-destination-objects \
    --cache-control="public, max-age=300" \
    --project=my-project
```

두 옵션이 중요하다.

`--delete-unmatched-destination-objects` 는 로컬에서 지운 파일을 버킷에서도 지운다. 이걸 빼면 예전 파일이 계속 남아 서빙된다. 실제로 중간에 쓰다 만 `independence.json` 을 지웠는데, 이 옵션 덕분에 버킷에서도 같이 사라져서 404가 떴다.

```bash
$ curl -s -o /dev/null -w "%{http_code}\n" \
    https://storage.googleapis.com/my-project/data/independence.json
404
```

`--cache-control` 이 두 번째다. **이걸 안 주면 GCS가 기본값 `max-age=3600` 을 붙인다.** 파일을 새로 올려도 브라우저와 중간 캐시가 한 시간 동안 옛날 걸 보여준다. 배포했는데 왜 안 바뀌지 하고 한참 헤맬 수 있는 지점이다. 이 사이트는 주 1회 갱신이라 5분으로 뒀다.

```bash
$ curl -sI https://storage.googleapis.com/my-project/index.html | grep -i cache
cache-control: public, max-age=300
```

Content-Type은 확장자로 자동 판별된다. ES 모듈로 쓴 `.js` 파일도 `text/javascript` 로 정상적으로 내려왔다.

```bash
$ curl -s -o /dev/null -w '%{http_code} %{content_type}\n' \
    https://storage.googleapis.com/my-project/app.js
200 text/javascript
```

## 4. GitHub Actions에서 키 없이 배포하기

여기가 이번 실습에서 제일 배운 게 많은 부분이다.

보통은 서비스 계정 JSON 키를 만들어 저장소 시크릿에 넣는다. 그런데 이 키는 **만료가 없다.** 한 번 새면 계속 쓸 수 있다. 그래서 **Workload Identity Federation**(WIF)을 썼다. GitHub Actions가 발급한 OIDC 토큰을 GCP가 직접 검증하는 방식이라, 저장할 키 파일 자체가 없다.

먼저 배포 전용 서비스 계정을 만들고 버킷 권한을 준다. 아래 `<서비스계정>` 은 원하는 이름으로 바꿔 쓰면 된다.

```bash
$ gcloud iam service-accounts create <서비스계정> \
    --project=my-project --display-name="GitHub Actions deploy"
Service account email: <서비스계정>@my-project.iam.gserviceaccount.com

$ gcloud storage buckets add-iam-policy-binding gs://my-project \
    --member="serviceAccount:<서비스계정>@my-project.iam.gserviceaccount.com" \
    --role=roles/storage.objectAdmin --project=my-project
```

그다음 워크로드 아이덴티티 풀과 프로바이더를 만든다.

```bash
$ gcloud iam workload-identity-pools create github \
    --location=global --project=my-project
Created workload identity pool [github].

$ gcloud iam workload-identity-pools providers create-oidc github-actions \
    --location=global --workload-identity-pool=github --project=my-project \
    --issuer-uri="https://token.actions.githubusercontent.com" \
    --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
    --attribute-condition="assertion.repository=='benyco-dev/lottery'"
Created workload identity pool provider [github-actions].
```

`--attribute-condition` 은 반드시 넣는다. 이걸 빼면 GitHub의 **어떤 저장소든** 이 서비스 계정을 빌려 쓸 수 있게 된다.

마지막으로 이 저장소에서만 서비스 계정을 위임받을 수 있도록 묶어준다.

```bash
$ PN=$(gcloud projects describe my-project --format='value(projectNumber)')
$ gcloud iam service-accounts add-iam-policy-binding \
    <서비스계정>@my-project.iam.gserviceaccount.com --project=my-project \
    --role=roles/iam.workloadIdentityUser \
    --member="principalSet://iam.googleapis.com/projects/$PN/locations/global/workloadIdentityPools/github/attribute.repository/benyco-dev/lottery"
```

워크플로에서는 `id-token: write` 권한이 있어야 OIDC 토큰이 발급된다. 이게 없으면 인증 단계에서 그냥 실패한다.

```yaml
permissions:
  contents: write   # 갱신된 데이터를 되커밋
  id-token: write   # WIF용 OIDC 토큰. 없으면 인증 실패

steps:
  - uses: google-github-actions/auth@v2
    with:
      workload_identity_provider: ${{ secrets.GCP_WIF_PROVIDER }}
      service_account: ${{ secrets.GCP_SA_EMAIL }}
  - uses: google-github-actions/setup-gcloud@v2
  - run: |
      gcloud storage rsync site "gs://${{ secrets.GCS_BUCKET }}" \
        --recursive --delete-unmatched-destination-objects \
        --cache-control="public, max-age=300"
```

시크릿은 세 개만 등록했다. 프로바이더 경로, 서비스 계정 이메일, 버킷 이름. **키 파일은 없다.**

## 5. 두 번째로 걸린 곳 — objectAdmin만으론 rsync가 안 된다

권한을 `roles/storage.objectAdmin` 만 주고 돌렸더니 배포가 실패했다. 객체를 만들고 지울 권한은 있는데, **버킷 자체의 메타데이터를 읽을 권한이 없어서**다. `rsync` 는 동기화 전에 버킷 상태를 조회한다.

`roles/storage.legacyBucketReader` 를 같이 줘야 통과한다.

```bash
$ gcloud storage buckets add-iam-policy-binding gs://my-project \
    --member="serviceAccount:<서비스계정>@my-project.iam.gserviceaccount.com" \
    --role=roles/storage.legacyBucketReader --project=my-project
```

이름에 `legacy` 가 붙어 있어서 쓰면 안 되는 것처럼 보이는데, uniform bucket-level access 환경에서도 버킷 조회 권한을 주는 정식 역할이다.

최종 권한은 이렇게 정리됐다.

```bash
$ gcloud storage buckets get-iam-policy gs://my-project --format=json
  roles/storage.objectViewer        -> allUsers
  roles/storage.objectAdmin         -> serviceAccount:<서비스계정>@...
  roles/storage.legacyBucketReader  -> serviceAccount:<서비스계정>@...
```

## 6. 세 번째로 걸린 곳 — CI에서 수집이 타임아웃

첫 워크플로 실행은 배포까지 가지도 못하고 데이터 수집 단계에서 죽었다.

```
urllib.error.URLError: <urlopen error timed out>
##[error]Process completed with exit code 1.
```

로컬에서는 잘 되던 게 러너에서만 실패했다. 원인은 단순했다. 매 실행마다 1회차부터 1,240회차까지 **124페이지를 연속으로 요청**하고 있었다. 로컬에서는 통과하지만 러너에서는 중간에 끊긴다.

고치는 방향은 재시도가 아니라 **애초에 덜 요청하는 것**이었다. 이미 받아둔 회차는 다시 긁지 않고 새 회차만 가져오게 바꿨다. 주 1회 실행이니 매번 1페이지면 끝난다.

```python
draws = load_existing()
have = max(draws, default=0)
if have >= latest:
    print(f"이미 최신 (1~{latest}회). 받을 회차 없음.")
    return
```

여기에 함정이 하나 더 있다. 기존 파일에 **구멍이 있는데도 그대로 재사용하면 누락된 회차가 영구히 남는다.** 그래서 1회차부터 연속으로 이어질 때만 재사용하고, 아니면 전량 다시 받도록 했다.

```python
def load_existing():
    """1회부터 연속으로 이어지지 않으면 증분을 포기하고 전량 수집한다."""
    ...
    if [d["e"] for d in draws] != list(range(1, len(draws) + 1)):
        return {}
    return {d["e"]: d for d in draws}
```

이 조건은 눈으로 확인하기 어려워서 테스트로 박아뒀다. 중간이 빈 파일, 1회차부터 시작하지 않는 파일, 형식이 깨진 파일을 넣고 전부 `{}` 가 나오는지 본다.

고치고 다시 돌리니 수집부터 배포까지 전부 통과했다.

![GitHub Actions 워크플로 성공 화면](/images/uploads/2026-09-10-gcs-static-hosting/03-actions.webp)

*수집 → 검증 → 채점 → 커밋 → WIF 인증 → GCS 배포까지 38초.*

## 7. 비용

| 항목 | 규모 | 월 비용 |
| --- | --- | --- |
| 스토리지 | 약 200KB | 사실상 0 |
| 클래스 A 작업(업로드) | 주 10여 건 | 사실상 0 |
| 네트워크 이그레스 | 방문자 수에 비례 | 1GB당 약 $0.12 |

Cloud Run도 요청이 없으면 0으로 스케일되지만, 컨테이너 빌드와 레지스트리 보관, 콜드스타트가 따라온다. **내려줄 게 정적 파일뿐이라면 버킷이 더 단순하고 싸다.**

## 그래서 사이트는 뭘 하나

로또 6/45 1회차부터 1,240회차까지 1등 당첨번호를 모아, 번호들 사이의 연관성 네 가지를 뽑아서 다음 회차 예상번호 6자리 5세트를 만든다.

![모델 구성과 번호쌍 연관성 화면](/images/uploads/2026-09-10-gcs-static-hosting/02-model.webp)

*동반출현과 전이 리프트. 리프트 1.0이 기준선이고, ×1.81이면 기대보다 1.81배 자주 같이 나왔다는 뜻이다.*

| 요소 | 내용 | 가중 |
| --- | --- | --- |
| 동반출현 | 이미 고른 번호와 같은 회차에 나온 빈도 | 1.4 |
| 최근 빈도 | 반감기 300회로 가중한 출현 | 1.0 |
| 직전 회차 전이 | 직전 회차 번호 다음에 나온 번호 | 0.8 |
| 미출현 기간 | 마지막 출현 이후 회차 ÷ 기대 간격 7.5 | 0.6 |

네 점수를 z점수로 맞춰 더한 뒤 가중 추출한다. 재미있는 부분은 **번호를 하나씩 여섯 번 뽑으면서 매번 이미 고른 번호와의 동반출현을 다시 계산**한다는 점이다. 그래서 한 세트 안의 여섯 개가 서로 엮인 조합이 된다.

예상번호는 추첨 **전에** 저장소에 커밋되고, 추첨이 끝나면 다음 실행에서 자동 채점된다. 시드가 회차번호에 묶여 있어서 같은 코드로 누구나 같은 결과를 재현할 수 있고, 나중에 성적을 고쳐 쓸 수도 없다.

물론 **로또는 매 회차 독립시행이라 어떤 조합이든 1등 확률은 8,145,060분의 1로 같다.** 백테스트로 4만 세트를 돌려봤는데 5등 적중률이 이론값과 구별되지 않았다. 예상번호는 재미로 보는 것이고, 이 프로젝트의 실제 목적은 그 아래 깔린 호스팅 구성이다.

## 정리

- 정적 파일만 내려주는 사이트에 VM이나 컨테이너는 과하다. **버킷 하나로 충분하다.**
- **`--cache-control` 을 직접 주자.** 안 주면 기본 1시간이 붙어서 배포가 반영 안 된 것처럼 보인다.
- `rsync` 를 CI에서 돌리려면 **`objectAdmin` 에 `legacyBucketReader` 를 더해야** 한다.
- 서비스 계정 키 파일 대신 **WIF**를 쓰면 저장할 비밀이 없어진다. `attribute-condition` 으로 저장소를 반드시 못 박자.
- CI에서 외부 API를 긁는다면 **매번 전부 받지 말고 증분으로.** 재시도보다 요청을 줄이는 쪽이 근본 해결이었다.

다음엔 커스텀 도메인을 붙여보거나, 이 구성을 Terraform으로 옮겨볼 생각이다.
