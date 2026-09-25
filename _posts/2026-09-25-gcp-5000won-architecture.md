---
title: 월 5천원으로 GCP에 커뮤니티 사이트 운영하기 (Cloud SQL·로드밸런서 없이)
date: 2026-09-25
categories:
  - Google Cloud
  - Terraform
tags:
  - GCP
  - Cloud Run
  - Compute Engine
  - Cloud Storage
  - 비용 절감
  - 무료 티어
excerpt: 구글 로그인, 게시판, 이미지 업로드, 커스텀 도메인까지 있는 사이트를 월 5천원에 돌린다. Cloud SQL, 로드밸런서, VPC 커넥터를 빼고 무엇으로 대신했는지, 무료 티어에서 어디서 걸렸는지 정리했다.
---

프로젝트를 올리고 서로 구경하는 사이트 [Pro-v](https://pro-v.co.kr)를 만들었다. 구글 로그인, 게시물·댓글·좋아요, 썸네일 업로드, 커스텀 도메인까지 있는 평범한 웹 서비스다.

평범한 서비스를 GCP 교과서대로 올리면 이렇게 된다. **Cloud SQL**에 DB를 두고, **외부 HTTPS 로드밸런서**로 도메인을 붙이고, **Serverless VPC 커넥터**로 둘을 잇는다. 트래픽이 거의 없는 개인 프로젝트인데도 이것만으로 월 5만원 이상이 나간다.

원칙은 하나였다. **"저렴할수록 좋다."** 셋을 전부 빼고 무료 티어 안에서 대신할 방법을 찾았다. 지금 청구서는 한 달 5천원대다.

최종 구성은 이렇다.

```
사용자 ──HTTPS──> Cloud Run (prov-web, 인스턴스 0~3)
                   │   └─ 커스텀 도메인: Cloud Run 도메인 매핑 (로드밸런서 없음)
                   │
                   └─ Direct VPC egress (커넥터 없음), 내부 IP:5432
                        │
                        v
                  e2-micro VM (무료 티어) ── PostgreSQL 15
                        ├─ 매일 pg_dump ──> Cloud Storage (us-central1 단일 리전)
                        └─ 매일 디스크 스냅샷 (7일)
```

인프라는 전부 Terraform으로 만들었다. 아래 코드 조각은 실제 구성에서 필요한 부분만 뗀 것이다.

## 1. 비용부터: 흔한 구성과 비교

| 역할 | 흔한 선택 | 월 비용(대략) | 이번 선택 | 월 비용 |
| --- | --- | --- | --- | --- |
| DB | Cloud SQL (가장 작은 인스턴스) | 약 15,000원 | e2-micro VM + Postgres | 0원 (무료 티어) |
| 커스텀 도메인·HTTPS | 외부 HTTPS 로드밸런서 | 약 30,000원 | Cloud Run 도메인 매핑 | 0원 |
| Cloud Run → DB 연결 | Serverless VPC 커넥터 | 약 10,000원 | Direct VPC egress | 0원 |
| 합계 | | **약 55,000원** | | **0원** |

남는 고정비는 두 개뿐이다.

| 항목 | 월 |
| --- | --- |
| VM의 외부 IPv4 1개 (패키지 업데이트용) | 약 5,000원 |
| Cloud DNS 존 1개 | 약 300원 |
| Cloud Run, Cloud Storage, Artifact Registry, Logging, Pub/Sub | 0원 (무료 한도 안) |

도메인 등록비(연 2만원 내외)는 별도다.

## 2. Cloud SQL 대신 e2-micro VM에 Postgres

Compute Engine 무료 티어에는 **e2-micro 1대 + 표준 디스크 30GB**가 들어 있다. 조건이 두 개 있다.

- 리전이 `us-central1`, `us-west1`, `us-east1` 중 하나여야 한다.
- **결제 계정당 1대**다. 다른 프로젝트에 e2-micro를 또 만들면 그 VM은 과금된다.

그래서 VM을 `us-central1`에 두고, Cloud Run·버킷·Artifact Registry도 전부 같은 리전에 맞췄다. 리전이 다르면 내부 통신에도 리전 간 전송 요금이 붙는다.

```hcl
resource "google_compute_instance" "db" {
  name         = "prov-db"
  machine_type = "e2-micro"
  zone         = var.zone # us-central1-a

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
      size  = 30
      type  = "pd-standard" # 무료 티어는 표준 디스크 30GB까지
    }
  }

  network_interface {
    network    = data.google_compute_network.default.id
    subnetwork = data.google_compute_subnetwork.default.id
    access_config {} # 패키지·보안 업데이트용 외부 IP (월 약 5천원)
  }

  metadata_startup_script = templatefile("${path.module}/scripts/vm-startup.sh", { ... })

  lifecycle {
    # 부팅 스크립트가 바뀌어도 VM을 다시 만들지 않는다.
    # 다시 만들면 부팅 디스크와 함께 DB가 통째로 날아간다.
    ignore_changes = [metadata_startup_script]
  }
}
```

Postgres 설치와 계정 생성은 부팅 스크립트가 한다. DB 비밀번호는 Secret Manager에서 읽는다.

### 대가: 단일 장애점

Cloud SQL이 해 주던 걸 이제 직접 챙겨야 한다. 가장 큰 건 **VM이 죽으면 사이트도 멈춘다**는 점이다. 이중화는 이 예산으로는 불가능하다. Cloud SQL HA만 해도 월 6만원부터다. 그래서 "빨리 알고, 빨리 되살리는" 쪽에 무료 도구를 붙였다.

| 대비 | 방법 | 비용 |
| --- | --- | --- |
| 데이터 백업 | 매일 03:10 `pg_dump` → Cloud Storage, 30일 뒤 자동 삭제 | 0원 (5GB 무료) |
| VM째 복구 | 매일 04:00 디스크 스냅샷, 7일 보관 | 월 수백 원 |
| 다운 감지 | Cloud Monitoring이 5분마다 `/status`(DB까지 확인) 호출, 실패 시 메일 | 0원 |

가동 확인은 [지난 글](/cloud-monitoring-uptime/)에서 만든 방식 그대로다. 주의할 점이 하나 있다. 확인 대상은 메인 페이지 `/`가 아니라 전용 `/status`여야 한다. 쿠키 없는 검사 요청이 매번 새 방문자로 집계돼서 방문자 수가 부풀었다.

### Terraform에서 VM이 다시 만들어질 뻔했다

DB가 VM 디스크에 있으니 **VM 재생성 = 데이터 삭제**다. 두 번 걸렸다.

1. 부팅 스크립트에 스키마 SQL을 넣었더니, 스키마를 고칠 때마다 plan에 VM 교체가 떴다. 위 코드의 `ignore_changes`로 막았다. 스키마 변경은 앱이 부팅할 때 하거나 SQL을 직접 실행한다.
2. 서브넷 데이터 소스에 `depends_on`으로 API 활성화를 걸었더니, API를 하나 추가할 때마다 서브넷 값이 "알 수 없음"이 되어 VM 교체가 떴다. **데이터 소스에는 `depends_on`을 걸지 않는다.**

`terraform apply` 전에 plan에서 `prov-db`에 `-/+`(교체)가 있는지 꼭 보는 습관이 생겼다.

## 3. 로드밸런서 대신 Cloud Run 도메인 매핑

Cloud Run에 커스텀 도메인을 붙이는 공식 추천은 외부 HTTPS 로드밸런서다. 기능은 많지만 트래픽이 0이어도 월 3만원 정도가 나간다.

**Cloud Run 도메인 매핑**을 쓰면 로드밸런서 없이 도메인을 붙이고 인증서도 무료로 자동 발급된다.

```hcl
resource "google_cloud_run_domain_mapping" "web" {
  name     = var.domain # pro-v.co.kr
  location = var.region
  metadata { namespace = var.project_id }
  spec { route_name = google_cloud_run_v2_service.web.name }
}

locals {
  # 매핑이 요구하는 A/AAAA 레코드를 그대로 DNS에 옮긴다.
  mapping_records = try(google_cloud_run_domain_mapping.web.status[0].resource_records, [])
}

resource "google_dns_record_set" "apex_a" {
  name         = "${var.domain}."
  managed_zone = google_dns_managed_zone.web.name
  type         = "A"
  ttl          = 3600
  rrdatas      = [for r in local.mapping_records : r.rrdata if r.type == "A"]
}
```

`www`는 매핑을 하나 더 만들고 CNAME을 `ghs.googlehosted.com.`으로 걸면 된다. `www`와 `*.run.app` 주소로 들어온 요청은 앱이 `pro-v.co.kr`로 301 리디렉션한다. 대표 주소가 하나여야 로그인 콜백 주소와 검색 등록이 꼬이지 않는다.

순서는 두 단계다. 도메인을 사고 소유 확인을 끝내야 매핑이 만들어진다. 그래서 `enable_domain` 변수를 두고 처음엔 `false`로 apply한다. 도메인 구매·소유 확인이 끝나면 `true`로 바꿔 다시 apply한다.

### 도메인 매핑으로 포기한 것

- **Cloud Armor를 못 쓴다.** 로드밸런서에 붙는 기능이라서다. 특정 국가 IP 차단이 필요해서 앱 미들웨어에서 직접 막았다(다음 글에서 다룬다).
- 문서상 도메인 매핑은 **프리뷰** 기능이고, 지원하는 리전이 정해져 있다. `us-central1`은 된다.
- 301 리디렉션 때문에 걸린 곳이 있다. **Cloud Scheduler는 리디렉션을 따라가지 않는다.** 스케줄러가 `*.run.app` 주소를 부르면 매번 301에서 실패했다. 대상 주소를 `https://pro-v.co.kr/...`로 바꿔서 해결했다. 이 실패는 Cloud Run 앱 로그에 아무것도 안 남아서 원인을 찾는 데 오래 걸렸다.

## 4. VPC 커넥터 대신 Direct VPC egress

Cloud Run은 기본적으로 VPC 밖에 있다. VM의 내부 IP로 붙으려면 VPC로 들어가는 길이 필요하다. 예전 방법인 **Serverless VPC 커넥터**는 커넥터용 인스턴스가 최소 2대 상시로 떠 있어서 월 1만원 정도가 든다.

**Direct VPC egress**는 Cloud Run 인스턴스가 서브넷에 직접 붙는 방식이다. 추가 요금이 없다.

```hcl
resource "google_cloud_run_v2_service" "web" {
  name     = "prov-web"
  location = var.region

  template {
    scaling {
      min_instance_count = 0 # 요청 없으면 0원
      max_instance_count = 3 # 트래픽이 튀어도 요금이 튀지 않게
    }

    vpc_access {
      network_interfaces {
        network    = data.google_compute_network.default.name
        subnetwork = data.google_compute_subnetwork.default.name
      }
      egress = "PRIVATE_RANGES_ONLY" # 내부 IP로 가는 트래픽만 VPC로
    }
    ...
  }
}
```

`PRIVATE_RANGES_ONLY`로 두면 DB로 가는 트래픽만 VPC를 타고, 구글 API 호출은 평소처럼 나간다.

VM 쪽 방화벽은 **서브넷 내부에서 오는 5432만** 연다. Postgres 포트는 외부에서 보이지 않는다.

한 가지 더 확인할 게 있다. default VPC에는 `default-allow-ssh`, `default-allow-rdp`(0.0.0.0/0) 규칙이 기본으로 있다. Terraform 밖에서 만들어진 규칙이라 plan에도 안 보인다. 나중에 확인해 보니 VM의 22번 포트가 실제로 열려 있었다. SSH를 안 쓰기로 해서 두 규칙을 지웠다.

## 5. 함정: Cloud Storage 무료 티어는 단일 리전에만

썸네일과 DB 백업을 담을 버킷을 처음엔 `US` **멀티리전**으로 만들었다. "미국이면 무료 티어겠지" 싶었다.

Cloud Storage 무료 5GB는 **`us-central1`, `us-east1`, `us-west1` 단일 리전**에만 적용된다. `US` 멀티리전은 무료 티어가 아니다. 용량이 작아 금액은 크지 않았지만 원칙에 어긋나서 옮겼다.

```hcl
resource "google_storage_bucket" "thumbs" {
  name     = "${var.project_id}-thumbs"
  location = var.region # 무료 티어는 us-central1·us-east1·us-west1 단일 리전만
  ...
}
```

버킷의 위치는 바꿀 수 없어서 **지우고 같은 이름으로 다시 만들어야** 했다. 옮기면서 두 번 더 걸렸다.

1. **IAM 권한이 조용히 사라진다.** 버킷을 교체해도 `google_storage_bucket_iam_member`는 plan에 안 뜬다. 버킷 이름 문자열이 같아서 바뀐 게 없다고 판단한다. 그대로 apply하면 새 버킷에 권한이 없는데 state에는 있는 상태가 된다. 권한 리소스도 같이 교체해야 한다.

   ```bash
   terraform apply \
     -replace=google_storage_bucket.thumbs \
     -replace=google_storage_bucket_iam_member.thumbs_public
   ```

2. **같은 이름을 다른 위치로 다시 만들면 한동안 404가 난다.** 최대 10분 걸렸고, 버킷 생성 자체도 5분 넘게 걸렸다. 사이트 썸네일이 그동안 깨지니 사람이 적은 시간에 옮기자.

버킷 이야기가 나온 김에 하나 더. 공개 이미지 버킷에 `roles/storage.objectViewer`를 `allUsers`로 주면 **파일 목록(objects.list)까지 공개**된다. 누구나 버킷의 파일 이름을 전부 훑어볼 수 있다. 파일 읽기만 필요하면 `roles/storage.legacyObjectReader`를 쓴다.

## 6. 나머지를 무료 한도 안에 묶어 두는 설정

| 서비스 | 설정 | 이유 |
| --- | --- | --- |
| Cloud Run | `min_instance_count = 0`, `max_instance_count = 3` | 유휴 비용 0, 트래픽이 몰려도 요금 상한 |
| Artifact Registry | 이미지 최근 5개만 보관 | 0.5GB 무료 한도 유지 |
| Cloud Storage 백업 | 30일 지나면 자동 삭제 | 5GB 무료 한도 유지 |
| Cloud Logging | 보관 30일 | 기본값 그대로, 개인정보처리방침 기간과 맞춤 |
| 배포 | GitHub Actions + Workload Identity Federation | 서비스 계정 키 없이, Cloud Build 비용 없이 |

그래도 실수로 요금이 튈 수는 있다. 그래서 **월 5만원을 넘으면 사이트를 자동으로 끄는 장치**도 달았다. 이건 따로 글로 쓴다.

## 정리

- 트래픽이 적은 개인 서비스라면 **Cloud SQL, 로드밸런서, VPC 커넥터 없이도** GCP에서 충분히 운영할 수 있다. 셋만 빼도 월 약 5.5만원이 준다.
- DB는 **e2-micro 무료 티어 VM**에 둔다. 리전은 `us-central1`·`us-west1`·`us-east1`, 결제 계정당 1대다. 대가는 단일 장애점이니 백업·스냅샷·가동 확인을 무료로 붙인다.
- 커스텀 도메인은 **Cloud Run 도메인 매핑**으로 붙인다. 대신 Cloud Armor는 못 쓰고, 301 리디렉션을 안 따라가는 호출자(Cloud Scheduler)를 조심한다.
- Cloud Run → VM 연결은 **Direct VPC egress**로 한다. 추가 요금이 없다.
- **Cloud Storage 무료 티어는 단일 리전 버킷만.** `US` 멀티리전은 과금된다. 옮길 때는 IAM 권한을 같이 `-replace` 한다.
- Terraform으로 VM에 DB를 두면 **plan에서 VM 교체(`-/+`)가 있는지** 매번 확인한다.
