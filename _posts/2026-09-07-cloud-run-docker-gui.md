---
title: Cloud Run에 Docker 이미지 배포하기 (콘솔 GUI 편)
date: 2026-09-07
tags:
  - GCP
  - Docker
  - Cloud Run
  - Artifact Registry
excerpt: 직접 만든 Docker 이미지를 Artifact Registry에 올리고, Google Cloud 콘솔 GUI만으로 Cloud Run 서비스를 배포해봤다. 클릭 몇 번으로 끝나지만 리전과 서비스 이름에서 한 번씩 걸린다.
---

Docker 이미지를 만들어 **Cloud Run**에 올리는 가장 기본적인 흐름을 처음부터 끝까지 해봤다. `gcloud run deploy` 한 줄로도 끝나는 작업이지만, 이번에는 **콘솔 GUI에서 클릭으로** 배포하는 과정을 정리했다. 각 설정이 화면 어디에 있는지 눈으로 익혀두면 나중에 CLI 옵션을 볼 때도 훨씬 빨리 이해된다.

전체 흐름은 이렇다.

1. 앱과 Dockerfile 작성 → 이미지 빌드 → 로컬에서 동작 확인
2. Artifact Registry 저장소를 만들고 이미지 푸시
3. Cloud Run 콘솔에서 그 이미지를 골라 서비스 생성
4. 발급된 URL로 접속 확인

> 아래 스크린샷의 프로젝트 ID는 `my-project`, 프로젝트 번호는 `XXXXXXXXXXXX`로 가렸다.

## 1. 컨테이너에 올릴 앱 준비

Cloud Run은 **컨테이너가 `$PORT`(기본 8080)로 들어오는 HTTP 요청을 받아주기만 하면** 언어나 프레임워크를 가리지 않는다. 그래서 의존성 없는 Node.js 기본 http 서버로 최소 구성을 만들었다.

```javascript
// app.js
const http = require('http');
const port = process.env.PORT || 8080;

http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello from Docker on Cloud Run!\n');
}).listen(port, () => console.log(`Server listening on port ${port}`));
```

포트를 하드코딩하지 않고 `process.env.PORT`를 먼저 읽는 게 핵심이다. Cloud Run은 컨테이너를 띄울 때 `PORT` 환경변수를 주입하고, 그 포트로 헬스 체크를 한다. 여기서 어긋나면 배포는 성공한 것처럼 보이다가 리비전이 준비되지 않아 실패한다.

```dockerfile
# Dockerfile
FROM node:18-slim
WORKDIR /app
COPY package.json .
COPY app.js .
EXPOSE 8080
CMD ["node", "app.js"]
```

`EXPOSE`는 문서화 목적이라 Cloud Run 동작에 직접 영향을 주진 않는다. 실제 포트 매핑은 Cloud Run이 컨테이너 설정의 **Container port** 값으로 처리한다.

## 2. 이미지 빌드 후 로컬에서 먼저 확인

Cloud Shell에서 바로 빌드했다. 로컬 Docker가 없어도 브라우저만으로 되니 실습에는 이 편이 편하다.

```bash
$ docker build -t docker-cloudrun-demo .
[+] Building 26.3s (9/9) FINISHED
 => [1/4] FROM docker.io/library/node:18-slim
 => [2/4] WORKDIR /app
 => [3/4] COPY package.json .
 => [4/4] COPY app.js .
 => exporting to image
```

바로 배포하지 않고 **로컬에서 한 번 띄워보는 습관**이 시간을 아껴준다. 컨테이너가 안 뜨는 문제를 클라우드 로그로 디버깅하는 것보다 훨씬 빠르다.

```bash
$ docker run -d -p 8080:8080 --name demo docker-cloudrun-demo
$ curl -s localhost:8080
Hello from Docker on Cloud Run!

$ docker ps
CONTAINER ID   IMAGE                  STATUS         PORTS
3688356b6355   docker-cloudrun-demo   Up 2 seconds   0.0.0.0:8080->8080/tcp
```

응답이 확인되면 컨테이너를 정리한다.

```bash
$ docker stop demo && docker rm demo
```

## 3. Artifact Registry에 이미지 푸시

Cloud Run은 **레지스트리에 올라간 이미지**만 배포할 수 있다. 로컬에만 있는 이미지는 고를 수 없으니 먼저 저장소를 만들고 푸시한다.

```bash
$ export REGION=asia-northeast3
$ export PROJECT_ID=$(gcloud config get-value project)

$ gcloud artifacts repositories create docker-repo \
    --repository-format=docker \
    --location=$REGION \
    --description='docker basics repo'
Created repository [docker-repo].

$ gcloud auth configure-docker $REGION-docker.pkg.dev
Adding credentials for: asia-northeast3-docker.pkg.dev

$ docker tag docker-cloudrun-demo \
    $REGION-docker.pkg.dev/$PROJECT_ID/docker-repo/docker-cloudrun-demo:v1
$ docker push \
    $REGION-docker.pkg.dev/$PROJECT_ID/docker-repo/docker-cloudrun-demo:v1
v1: digest: sha256:1002cc002b8199a84ef27698baf1bb19d6c0efa474e0c812691e6e337689248c
```

`gcloud auth configure-docker`를 빼먹으면 푸시할 때 인증 오류가 난다. Docker에게 "이 호스트로 갈 땐 gcloud 자격 증명을 써라"라고 알려주는 단계다. 저장소 리전(`asia-northeast3`)은 이미지 주소의 접두사가 되므로 나중에 Cloud Run 리전과 맞춰두는 편이 지연 시간 면에서 유리하다.

## 4. 콘솔에서 Cloud Run 서비스 만들기

여기서부터가 GUI 작업이다. 콘솔에서 **Cloud Run** 으로 이동하면 개요 화면이 나온다. 하단의 **Deploy container** 카드로 시작한다.

![Cloud Run 개요 화면과 Deploy container 카드](/images/uploads/2026-09-07-cloud-run-docker/01-cloud-run-overview.webp)

*Cloud Run 개요. 기존 서비스 목록과 함께 배포 진입점이 카드로 제공된다.*

서비스 생성 화면은 배포 소스를 먼저 고르게 되어 있다. 선택지는 세 가지다.

- **기존 컨테이너 이미지에서 리비전 1개 배포** — 이미 만들어 둔 이미지를 그대로 올린다 (이번 실습)
- **저장소에서 지속적 배포** — GitHub 등을 연결해 푸시할 때마다 Cloud Build로 빌드
- **인라인 편집기로 함수 작성** — 소스만 붙여넣어 배포

![Create service 화면의 배포 소스 선택 영역](/images/uploads/2026-09-07-cloud-run-docker/02-create-service-form.webp)

*Create service 초기 화면. Container image URL은 비어 있고, 리전 기본값은 `europe-west1`이다.*

**Select**를 누르면 Artifact Registry 브라우저가 열린다. 아까 만든 저장소가 트리로 보인다.

![Artifact Registry에서 이미지를 선택하는 패널](/images/uploads/2026-09-07-cloud-run-docker/03-select-image-panel.webp)

*프로젝트에 있는 저장소 목록. 앞서 만든 `docker-repo`가 보인다.*

저장소를 펼치면 이미지 이름이, 그 아래에 다이제스트와 태그가 나온다. `v1` 태그가 붙은 항목을 고르고 **Select**를 누른다.

![docker-cloudrun-demo 이미지의 v1 태그를 선택한 화면](/images/uploads/2026-09-07-cloud-run-docker/04-select-image-tag.webp)

*태그(`v1`)와 다이제스트가 함께 표시된다. 푸시한 지 9분 된 이미지가 정상적으로 올라와 있다.*

여기서 **처음 걸렸던 부분**이 두 가지 있다.

**첫째, 서비스 이름이 자동으로 채워진다.** 이미지를 선택하면 이미지 이름(`docker-cloudrun-demo`)이 Service name에 자동 입력된다. 모르고 그냥 타이핑하면 기존 값 뒤에 이어 붙어 `docker-cloudrun-demodocker-gui-demo` 같은 이름이 된다. 필드를 전체 선택 후 덮어써야 한다.

**둘째, 리전 기본값이 `europe-west1`(벨기에)이다.** 서울에서 쓸 서비스라면 반드시 `asia-northeast3`로 바꿔야 한다. 서비스 이름과 리전은 **생성 후 변경할 수 없다.**

인증(Authentication)에서는 **Allow public access**를 선택했다. 로그인 없이 URL로 바로 접근하게 하는 옵션으로, 내부적으로는 `allUsers`에게 `run.invoker` 권한을 주는 것과 같다. 실습이 아니라면 기본값인 **Require authentication**을 유지하는 편이 안전하다.

![인증, 결제, 스케일링 설정 영역](/images/uploads/2026-09-07-cloud-run-docker/05-configure-service.webp)

*Billing은 Request-based(요청 처리 중에만 과금), 최소 인스턴스는 0이 기본값이다.*

최소 인스턴스가 0이면 트래픽이 없을 때 인스턴스가 완전히 내려가 비용이 들지 않는 대신, 다음 요청에서 **콜드 스타트**가 생긴다. 실습·토이 프로젝트라면 0이 맞다.

배포 전 설정을 다시 확인한다. 이미지 주소, 서비스 이름, 리전, 공개 액세스까지 네 가지만 보면 된다.

![배포 직전 최종 설정 화면](/images/uploads/2026-09-07-cloud-run-docker/06-final-settings-before-create.webp)

*이미지는 Artifact Registry 경로, 리전은 `asia-northeast3 (Seoul)`로 수정된 상태.*

## 5. 배포 결과 확인

**Create**를 누르면 서비스 생성 → 리비전 생성 → 트래픽 라우팅 순으로 진행된다. 세 단계가 모두 Completed가 되면 끝이다. 이미지가 이미 빌드되어 있어 20초 정도밖에 걸리지 않았다.

![배포 완료된 서비스 상세 화면](/images/uploads/2026-09-07-cloud-run-docker/07-deployment-success.webp)

*리비전 `docker-gui-demo-00001`이 트래픽 100%를 받고 있다. Port 8080, CPU 1, 메모리 512MiB가 기본값이다.*

상세 화면에서 눈여겨볼 값들이다.

| 항목 | 값 | 의미 |
| --- | --- | --- |
| Port | 8080 | 컨테이너가 리스닝해야 하는 포트 |
| Concurrency | 80 | 인스턴스 1개가 동시에 처리하는 요청 수 |
| Request timeout | 300초 | 이 시간을 넘기면 요청이 끊긴다 |
| Memory / CPU | 512MiB / 1 | 리비전 단위로 조정 가능 |

Concurrency 80은 다른 서버리스 제품과 구분되는 지점이다. 요청마다 인스턴스를 하나씩 쓰는 게 아니라, **인스턴스 하나가 최대 80개 요청을 동시에 처리**하기 때문에 I/O 위주 앱에서는 비용이 크게 절약된다.

발급된 URL로 접속하면 컨테이너 안의 앱이 그대로 응답한다.

![배포된 서비스 URL 접속 결과](/images/uploads/2026-09-07-cloud-run-docker/08-running-service-browser.webp)

*`https://docker-gui-demo-XXXXXXXXXXXX.asia-northeast3.run.app` 접속 결과.*

## 정리

GUI로 해보니 CLI 명령어의 옵션들이 화면의 어떤 항목에 대응하는지 명확해졌다. 결국 아래 한 줄과 같은 일을 한 셈이다.

```bash
$ gcloud run deploy docker-gui-demo \
    --image=asia-northeast3-docker.pkg.dev/my-project/docker-repo/docker-cloudrun-demo:v1 \
    --region=asia-northeast3 \
    --allow-unauthenticated
```

- `--image` → Container image URL (Select 버튼)
- `--region` → Region 드롭다운 (**기본값이 유럽이므로 반드시 확인**)
- `--allow-unauthenticated` → Authentication의 Allow public access

이번에 얻은 것을 정리하면,

- 컨테이너는 반드시 `$PORT`(기본 8080)를 읽어서 리스닝해야 한다
- Cloud Run은 레지스트리의 이미지만 배포하므로 Artifact Registry 푸시가 선행되어야 한다
- 서비스 이름과 리전은 생성 후 변경할 수 없으니 Create 전에 확인한다
- 이미지를 선택하면 서비스 이름이 자동으로 채워지므로 덮어쓸 때 주의한다

반복 작업이라면 결국 CLI가 빠르지만, 처음 한 번은 GUI로 훑어보는 게 각 설정의 의미를 이해하는 데 도움이 됐다.
