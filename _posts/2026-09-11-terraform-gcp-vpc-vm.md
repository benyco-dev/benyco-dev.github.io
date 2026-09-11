---
title: Terraform으로 GCP에 VPC + VM 만들고 지워보기
date: 2026-09-11
categories:
  - Google Cloud
  - Terraform
tags:
  - GCP
  - Terraform
  - Compute Engine
  - VPC
  - 트러블슈팅
excerpt: 코드 한 파일로 VPC, 서브넷, VM, 방화벽을 만들고 콘솔에서 확인한 뒤 destroy 한 줄로 전부 지웠다. 새 프로젝트라 Compute Engine API에서 한 번 걸렸다.
---

[Compute Engine에 Docker로 Nginx를 띄울 때](/gcp-docker-nginx/)도, [Cloud Storage로 정적 사이트를 호스팅할 때](/gcs-static-hosting/)도 리소스는 전부 콘솔 클릭이나 `gcloud` 명령으로 만들었다. 그러다 보니 한계가 느껴졌다. 뭘 만들었는지 기록이 남지 않고, 지울 때도 하나씩 찾아서 지워야 했다.

그래서 인프라를 코드로 관리하는 **Terraform**을 직접 써보기로 했다. 이번 실습의 목표는 간단하다.

> 코드 한 파일로 **VPC → 서브넷 → VM → 방화벽**을 만들고, 확인하고, 한 번에 지운다.

## 실습 환경

- Terraform v1.16.1
- Google Cloud SDK 583.0.0
- google provider `~> 5.0`
- 리전: 서울(`asia-northeast3`)

## 작성한 코드

`main.tf` 하나에 전부 넣었다. `my-project` 는 본인 프로젝트 ID로 바꿔 쓰면 된다.

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = "my-project"
  region  = "asia-northeast3"
  zone    = "asia-northeast3-a"
}

# 1. 커스텀 VPC
resource "google_compute_network" "custom_vpc" {
  name                    = "tf-custom-vpc"
  auto_create_subnetworks = false
}

# 2. 서브넷
resource "google_compute_subnetwork" "custom_subnet" {
  name          = "tf-subnet-seoul"
  ip_cidr_range = "10.0.1.0/24"
  region        = "asia-northeast3"
  network       = google_compute_network.custom_vpc.id
}

# 3. VM
resource "google_compute_instance" "vm_instance" {
  name         = "tf-demo-vm"
  machine_type = "e2-micro"
  zone         = "asia-northeast3-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-12"
    }
  }

  network_interface {
    subnetwork = google_compute_subnetwork.custom_subnet.id
    access_config {} # 외부 공인 IP 할당
  }

  tags = ["web-server"]
}

# 4. 방화벽 (SSH 허용)
resource "google_compute_firewall" "allow_ssh" {
  name    = "tf-allow-ssh"
  network = google_compute_network.custom_vpc.id

  allow {
    protocol = "tcp"
    ports    = ["22"]
  }

  source_ranges = ["0.0.0.0/0"]
  target_tags   = ["web-server"]
}
```

코드를 쓰면서 눈에 띈 점이 두 가지 있었다.

**리소스끼리 참조하면 순서가 자동으로 정해진다.** 서브넷의 `network = google_compute_network.custom_vpc.id` 처럼 다른 리소스를 가리키기만 하면, Terraform이 VPC를 먼저 만들어야 한다는 걸 알아서 판단한다. `depends_on` 을 따로 쓸 필요가 없었다.

**방화벽과 VM은 태그로 연결된다.** 방화벽의 `target_tags` 와 VM의 `tags` 가 `web-server` 로 같으면 규칙이 적용된다. 코드상 참조 관계가 없어서 Terraform은 둘을 병렬로 처리한다.

```
tf-custom-vpc
 ├── tf-subnet-seoul
 │    └── tf-demo-vm
 └── tf-allow-ssh ···(태그 web-server로 매칭)··· tf-demo-vm
```

## 1. 인증

Terraform이 내 GCP 계정으로 API를 호출하려면 자격증명이 필요하다.

```bash
$ gcloud auth application-default login
```

브라우저로 로그인하면 로컬에 **ADC(Application Default Credentials)** 가 저장된다. Terraform google provider는 이 파일을 자동으로 읽는다. 서비스 계정 키 파일을 코드 옆에 둘 필요가 없어서 편했다.

로그인 후 `Quota project "..." was added to ADC` 라는 메시지가 떴다. gcloud 기본 프로젝트가 quota 프로젝트로 잡혔다는 뜻이다. 실습 프로젝트로 맞추고 싶다면 아래 명령을 쓰면 된다.

```bash
$ gcloud auth application-default set-quota-project my-project
```

## 2. init과 plan

```bash
$ terraform init
$ terraform plan
```

`init` 은 google provider 플러그인을 내려받는 단계다. `plan` 은 실제로 만들기 전에 무엇이 바뀌는지 미리 보여준다.

```
Plan: 4 to add, 0 to change, 0 to destroy.
```

예상대로 리소스 4개가 추가된다고 나왔다. `id` 나 `self_link` 같은 값은 `(known after apply)` 로 표시됐다. GCP가 리소스를 실제로 만들어야 정해지는 값들이다.

## 3. apply — 첫 번째로 걸린 곳

```bash
$ terraform apply
```

`yes` 를 입력하자 VPC 생성에서 바로 실패했다.

```
Error: Error creating Network: googleapi: Error 403: Compute Engine API has not
been used in project my-project before or it is disabled.
reason: SERVICE_DISABLED
```

새로 만든 프로젝트라 **Compute Engine API가 꺼져 있었던 것**이다. 콘솔에서 VM을 처음 만들 때는 API를 켜라는 안내가 뜨지만, Terraform은 API를 대신 켜주지 않는다.

```bash
$ gcloud services enable compute.googleapis.com --project=my-project
Operation "operations/..." finished successfully.
```

API를 켠 직후에는 반영까지 몇 분 걸릴 수 있다고 해서 잠깐 기다렸다가 다시 apply했다.

## 4. 다시 apply해서 완료

```
google_compute_instance.vm_instance: Creating...
google_compute_instance.vm_instance: Still creating... [00m10s elapsed]
google_compute_instance.vm_instance: Creation complete after 13s

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

마지막 apply에서는 VM 하나만 추가됐다. 나머지 리소스는 앞선 시도에서 이미 만들어져 **state 파일에 기록**돼 있었기 때문이다.

여기서 Terraform의 장점을 확실히 느꼈다. apply가 중간에 실패해도 성공한 리소스는 state에 남는다. 원인을 고치고 다시 실행하면 **빠진 것만 채운다.** 처음부터 다시 만들거나 중복으로 생기지 않는다.

## 5. 콘솔에서 확인

Compute Engine → VM instances에 들어가 보니 `tf-demo-vm` 이 서울 존에서 실행 중이었다.

- **내부 IP는 `10.0.1.2` 였다.** `/24` 서브넷에서 `.0` 은 네트워크 주소, `.1` 은 게이트웨이로 GCP가 예약하기 때문에 첫 VM이 `.2` 를 받는다.
- **외부 IP도 붙어 있었다.** `access_config {}` 를 비워두기만 해도 임시(ephemeral) 공인 IP가 할당된다.

코드 몇십 줄로 네트워크부터 VM까지 한 번에 올라온 걸 보니 신기했다.

## 6. destroy로 전부 정리

실습이 끝났으니 요금이 나가지 않게 바로 지웠다.

```bash
$ terraform destroy
```

```
Plan: 0 to add, 0 to change, 4 to destroy.
```

삭제 순서가 흥미로웠다. **만든 순서의 반대로** 지워졌다.

| 순서 | 리소스 | 소요 시간 |
| --- | --- | --- |
| 1 | 방화벽 `tf-allow-ssh` (병렬) | 12초 |
| 1 | VM `tf-demo-vm` (병렬) | 21초 |
| 2 | 서브넷 `tf-subnet-seoul` | 21초 |
| 3 | VPC `tf-custom-vpc` | 22초 |

```
Destroy complete! Resources: 4 destroyed.
```

VM이 서브넷을 쓰고 있고, 서브넷과 방화벽은 VPC에 속해 있다. 그래서 바깥쪽부터 지워야 하는데, 이 순서도 Terraform이 알아서 계산했다. 콘솔에서 손으로 지웠다면 "사용 중인 리소스라 삭제할 수 없다"는 에러를 몇 번은 만났을 것이다.

## 사용한 GCP 서비스

| 서비스 | 용도 |
| --- | --- |
| Compute Engine | VM 인스턴스 실행 |
| VPC Network | 커스텀 VPC와 서브넷으로 사설 네트워크 구성 |
| VPC 방화벽 규칙 | 태그 기반으로 SSH 인바운드 허용 |
| Service Usage API | Compute Engine API 활성화 |
| IAM / ADC | Terraform이 사용할 자격증명 발급 |

## 정리

- **선언형이라 편하다.** "무엇이 있어야 하는가"만 쓰면 생성·삭제 순서는 Terraform이 참조 관계로 계산한다.
- **state가 기준이다.** Terraform은 state 파일과 실제 클라우드를 비교해서 plan을 만든다. state에는 프로젝트 ID나 IP 같은 정보가 들어가므로 **`.gitignore` 에 꼭 넣자.**
- **실패해도 다시 돌리면 된다.** 부분 실패 후 재실행하면 남은 것만 만든다.
- **새 프로젝트에선 API부터 켜자.** Terraform은 꺼진 API를 대신 켜주지 않는다.
- **정리까지가 실습이다.** `destroy` 한 줄로 흔적 없이 지울 수 있다. 콘솔에서 수동으로 지우면 state와 실제 상태가 어긋난다.

다음엔 이번 코드를 조금 더 실전에 가깝게 다듬어볼 생각이다.

- 프로젝트 ID를 `variable` 로 빼고 `terraform.tfvars` 로 주입하기
- SSH 허용 범위를 `0.0.0.0/0` 에서 IAP 대역(`35.235.240.0/20`)으로 좁히고 `gcloud compute ssh --tunnel-through-iap` 로 접속하기
- API 활성화도 `google_project_service` 리소스로 코드화하기
- state를 GCS 버킷 backend에 저장하기
