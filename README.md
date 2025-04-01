
# Elastic stack (ELK) on Docker
도커(Docker)와 도커 컴포즈(Docker Compose)를 사용하여 최신 버전의 Elastic 스택을 실행해보세요.

[![Elastic Stack version](https://img.shields.io/badge/Elastic%20Stack-8.15.2-00bfb3?style=flat&logo=elastic-stack)](https://www.elastic.co/blog/category/releases)  
[![Build Status](https://github.com/deviantony/docker-elk/workflows/CI/badge.svg?branch=main)](https://github.com/deviantony/docker-elk/actions?query=workflow%3ACI+branch%3Amain)  
[![Join the chat](https://badges.gitter.im/Join%20Chat.svg)](https://app.gitter.im/#/room/#deviantony_docker-elk:gitter.im)

이 프로젝트는 Elasticsearch의 검색/집계 기능과 Kibana의 시각화 기능을 활용하여 어떤 데이터셋이든 분석할 수 있는 환경을 제공합니다.

Elastic에서 제공하는 [공식 Docker 이미지][elastic-docker]를 기반으로 구성되어 있습니다:

* [Elasticsearch](https://github.com/elastic/elasticsearch/tree/main/distribution/docker)
* [Logstash](https://github.com/elastic/logstash/tree/main/docker)
* [Kibana](https://github.com/elastic/kibana/tree/main/src/dev/build/tasks/os_packages/docker_generator)

다른 스택 변형도 사용 가능합니다:

* [`tls`](https://github.com/deviantony/docker-elk/tree/tls): Elasticsearch, Kibana(옵션), Fleet에 TLS 암호화 활성화
* [`searchguard`](https://github.com/deviantony/docker-elk/tree/searchguard): Search Guard 지원

> [!IMPORTANT]  
> [플래티넘][subscriptions] 기능은 기본적으로 **30일**의 [체험판][license-mngmt] 기간 동안 활성화됩니다.  
> 이 평가 기간이 종료되면, 수동 조치 없이 데이터 손실 없이 Open Basic 라이선스에 포함된 모든 무료 기능에 자동으로 전환됩니다.  
> 이 동작을 원하지 않는 경우, [유료 기능 비활성화 방법](#how-to-disable-paid-features) 섹션을 참고하세요.

---

## tl;dr

```sh
docker compose up setup
```

```sh
docker compose up
```

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://github.com/user-attachments/assets/6f67cbc0-ddee-44bf-8f4d-7fd2d70f5217">
  <img alt="Animated demo" src="https://github.com/user-attachments/assets/501a340a-e6df-4934-90a2-6152b462c14a">
</picture>

## Philosophy

이 프로젝트의 목표는 Elastic 스택이라는 강력한 기술 조합을 실험해보고 싶은 사람들에게  
**가장 간단한 진입점**을 제공하는 것입니다.  

기본 설정은 **의도적으로 최소화**되어 있으며, 특정한 방식을 강요하지 않습니다.  
외부 의존성 없이 작동하며, 필요한 최소한의 자동화만 사용하여 작동합니다.

우리는 복잡한 자동화보다는 **좋은 문서화**를 통해 여러분이 이 레포지토리를 템플릿으로 활용하고,  
자신의 환경에 맞게 조정하고, _자신만의_ 설정을 만들 수 있도록 돕고자 합니다.  
[sherifabdlnaby/elastdocker][elastdocker]는 이러한 철학 위에 구축된 여러 프로젝트 중 하나입니다.

---

## Contents

1. [Requirements](#requirements)  
   * [Host setup](#host-setup)  
   * [Docker Desktop](#docker-desktop)  
     * [Windows](#windows)  
     * [macOS](#macos)  
1. [Usage](#usage)  
   * [Bringing up the stack](#bringing-up-the-stack)  
   * [Initial setup](#initial-setup)  
     * [Setting up user authentication](#setting-up-user-authentication)  
     * [Injecting data](#injecting-data)  
   * [Cleanup](#cleanup)  
   * [Version selection](#version-selection)  
1. [Configuration](#configuration)  
   * [How to configure Elasticsearch](#how-to-configure-elasticsearch)  
   * [How to configure Kibana](#how-to-configure-kibana)  
   * [How to configure Logstash](#how-to-configure-logstash)  
   * [How to disable paid features](#how-to-disable-paid-features)  
   * [How to scale out the Elasticsearch cluster](#how-to-scale-out-the-elasticsearch-cluster)  
   * [How to re-execute the setup](#how-to-re-execute-the-setup)  
   * [How to reset a password programmatically](#how-to-reset-a-password-programmatically)  
1. [Extensibility](#extensibility)  
   * [How to add plugins](#how-to-add-plugins)  
   * [How to enable the provided extensions](#how-to-enable-the-provided-extensions)  
1. [JVM tuning](#jvm-tuning)  
   * [How to specify the amount of memory used by a service](#how-to-specify-the-amount-of-memory-used-by-a-service)  
   * [How to enable a remote JMX connection to a service](#how-to-enable-a-remote-jmx-connection-to-a-service)  
1. [Going further](#going-further)  
   * [Plugins and integrations](#plugins-and-integrations)

## Requirements  
## 요구사항

### Host setup  
### 호스트 설정

* [Docker Engine][docker-install] 버전 **18.06.0** 이상  
* [Docker Compose][compose-install] 버전 **2.0.0** 이상  
* 최소 1.5GB의 RAM

> [!NOTE]  
> 특히 **리눅스** 환경에서는 사용자가 Docker 데몬과 상호작용할 수 있는 [필수 권한][linux-postinstall]이 설정되어 있는지 확인하세요.

기본적으로 스택은 아래 포트를 노출합니다:

* 5044: Logstash Beats 입력 포트  
* 50000: Logstash TCP 입력 포트  
* 9600: Logstash 모니터링 API  
* 9200: Elasticsearch HTTP  
* 9300: Elasticsearch TCP 전송 포트  
* 5601: Kibana

> [!WARNING]  
> 개발 환경에서 Elastic 스택 구성을 쉽게 하기 위해 Elasticsearch의 [bootstrap 검사][bootstrap-checks]는 의도적으로 비활성화되어 있습니다.  
> 운영 환경에서는 [Elasticsearch 공식 문서의 시스템 구성 가이드][es-sys-config]를 참고하여 호스트를 설정하는 것을 권장합니다.

---

### Docker Desktop

#### Windows

_Docker Desktop for Windows_의 이전 Hyper-V 모드를 사용하는 경우, `C:` 드라이브에 대해 [파일 공유][win-filesharing]가 활성화되어 있는지 확인하세요.

#### macOS

_Docker Desktop for Mac_의 기본 설정은 다음 경로의 파일만 마운트할 수 있습니다:  
`/Users/`, `/Volume/`, `/private/`, `/tmp`, `/var/folders`  
레포지토리가 이 경로들 중 하나에 클론되어 있는지 확인하거나, [공식 문서][mac-filesharing]를 참고하여 추가 경로를 설정하세요.

## Usage  
## 사용법

> [!WARNING]  
> 브랜치를 변경하거나 기존 스택의 [버전](#version-selection)을 업데이트할 경우,  
> `docker compose build` 명령어로 스택 이미지를 반드시 다시 빌드해야 합니다.

---

### Bringing up the stack  
### 스택 실행하기

아래 명령어로 이 레포지토리를 Docker 호스트에 클론하세요:

```sh
git clone https://github.com/deviantony/docker-elk.git
```

그런 다음, docker-elk에서 필요한 Elasticsearch 사용자 및 그룹을 초기화합니다:

```sh
docker compose up setup
```

모든 설정이 문제 없이 완료되었다면, 나머지 스택 구성 요소들을 시작합니다:

```sh
docker compose up
```

> [!NOTE]  
> `-d` 플래그를 추가하면 모든 서비스를 백그라운드(디태치드 모드)로 실행할 수 있습니다.

Kibana가 초기화되는 데 약 1분 정도 걸립니다.  
이후 웹 브라우저에서 <http://localhost:5601>에 접속한 뒤, 아래 기본 자격증명으로 로그인하세요:

* 사용자명: *elastic*  
* 비밀번호: *changeme*

> [!NOTE]  
> 초기 실행 시, `elastic`, `logstash_internal`, `kibana_system` Elasticsearch 사용자가  
> [`.env`](.env) 파일에 정의된 비밀번호 값(기본값은 "changeme")으로 초기화됩니다.  
> `elastic`은 [내장된 슈퍼유저][builtin-users]이며, 나머지 두 사용자는 각각 Kibana와 Logstash가  
> Elasticsearch와 통신할 때 사용됩니다.  
> 이 작업은 **최초 실행 시에만** 수행됩니다.  
> 초기화 이후 사용자 비밀번호를 변경하려면 다음 섹션의 안내를 참고하세요.

---

### Initial setup  
### 초기 설정

#### Setting up user authentication  
#### 사용자 인증 설정하기

> [!NOTE]  
> 인증 기능을 비활성화하려면 [Elasticsearch 보안 설정][es-security]을 참고하세요.

> [!WARNING]  
> Elastic v8.0.0부터는 Kibana를 `elastic` 부트스트랩 슈퍼유저로 실행할 수 없습니다.

기본 설정된 `"changeme"` 비밀번호는 **보안상 매우 취약**합니다.  
보안을 강화하기 위해 위에서 언급한 Elasticsearch 사용자들의 비밀번호를 무작위로 재설정하겠습니다.

---

**1. 기본 사용자 비밀번호 재설정**

아래 명령어를 통해 `elastic`, `logstash_internal`, `kibana_system` 사용자의 비밀번호를 각각 재설정하세요.  
생성된 비밀번호를 따로 기록해두세요.

```sh
docker compose exec elasticsearch bin/elasticsearch-reset-password --batch --user elastic
```

```sh
docker compose exec elasticsearch bin/elasticsearch-reset-password --batch --user logstash_internal
```

```sh
docker compose exec elasticsearch bin/elasticsearch-reset-password --batch --user kibana_system
```

다른 [내장 사용자][builtin-users]의 비밀번호도 필요에 따라 위 방식으로 재설정할 수 있습니다.  
예: Beats나 기타 구성요소를 통해 [모니터링 정보 수집][ls-monitoring]을 하고자 할 때 등

---

**2. 구성 파일 내 사용자명 및 비밀번호 교체**

`.env` 파일 내 `elastic` 사용자의 비밀번호를 앞서 생성한 값으로 바꾸세요.  
이 비밀번호는 핵심 구성요소에서는 사용되지 않지만, [확장 기능](#how-to-enable-the-provided-extensions)에서 Elasticsearch에 연결할 때 사용됩니다.

> [!NOTE]  
> 만약 확장 기능을 사용하지 않을 예정이거나, 직접 사용자 및 역할을 정의해서 인증하고 싶다면  
> 스택 초기화 이후 `.env` 파일의 `ELASTIC_PASSWORD` 항목은 삭제해도 무방합니다.

`.env` 파일의 `logstash_internal` 비밀번호를 교체하세요.  
이 값은 Logstash 파이프라인 설정 파일(`logstash/pipeline/logstash.conf`)에서 참조됩니다.

`.env` 파일의 `kibana_system` 비밀번호를 교체하세요.  
이 값은 Kibana 설정 파일(`kibana/config/kibana.yml`)에서 참조됩니다.

자세한 내용은 아래 [Configuration](#configuration) 섹션을 참고하세요.

---

**3. Logstash와 Kibana 재시작**

```sh
docker compose up -d logstash kibana
```

> [!NOTE]  
> Elastic 스택 보안에 대한 자세한 내용은 [Secure the Elastic Stack][sec-cluster]를 참고하세요.


#### Injecting data  
#### 데이터 주입

웹 브라우저에서 <http://localhost:5601>에 접속하여 Kibana UI를 실행하세요.  
로그인 시 아래 자격 증명을 사용합니다:

* 사용자명: *elastic*  
* 비밀번호: *\<앞서 생성한 elastic 비밀번호>*

이제 스택 구성이 완료되었으므로, 로그 데이터를 주입해볼 수 있습니다.

기본으로 제공되는 Logstash 설정은 TCP 포트 **50000**을 통해 데이터를 수신하도록 되어 있습니다.  
예를 들어, `nc`(Netcat)의 설치 버전에 따라 아래 명령어 중 하나를 사용하여  
`/path/to/logfile.log` 파일의 내용을 Logstash를 통해 Elasticsearch로 전송할 수 있습니다:

```sh
# `nc -h` 명령어로 설치된 Netcat 버전을 확인하세요

cat /path/to/logfile.log | nc -q0 localhost 50000          # BSD
cat /path/to/logfile.log | nc -c localhost 50000           # GNU
cat /path/to/logfile.log | nc --send-only localhost 50000  # nmap
```

또는 Kibana에서 제공하는 **샘플 데이터**를 불러올 수도 있습니다.

---

### Cleanup  
### 정리 (클린업)

Elasticsearch 데이터는 기본적으로 **볼륨(volume)** 에 저장됩니다.

스택을 완전히 종료하고 저장된 모든 데이터를 삭제하려면, 아래 Docker Compose 명령어를 실행하세요:

```sh
docker compose down -v
```

---

### Version selection  
### 버전 선택

이 레포지토리는 Elastic 스택의 최신 버전과 항상 일치하도록 유지됩니다.  
`main` 브랜치는 현재의 주요 버전(8.x)을 추적합니다.

다른 버전의 Elastic 구성 요소를 사용하려면, [.env](.env) 파일 내 버전 번호를 변경하면 됩니다.  
기존 스택을 업그레이드하는 경우, `docker compose build` 명령어로 반드시 모든 이미지를 다시 빌드해야 합니다.

> [!IMPORTANT]  
> 스택 업그레이드 전에 반드시 각 구성 요소의 [공식 업그레이드 가이드][upgrade]를 참고하세요.

이전 주요 버전도 다음과 같이 별도 브랜치로 지원됩니다:

* [`release-7.x`](https://github.com/deviantony/docker-elk/tree/release-7.x): 7.x 시리즈  
* [`release-6.x`](https://github.com/deviantony/docker-elk/tree/release-6.x): 6.x 시리즈 (지원 종료됨)  
* [`release-5.x`](https://github.com/deviantony/docker-elk/tree/release-5.x): 5.x 시리즈 (지원 종료됨)

## Configuration  
## 설정

> [!IMPORTANT]  
> 설정은 **동적으로 반영되지 않으며**, 구성 변경 후에는 해당 구성 요소를 **재시작해야** 합니다.

---

### How to configure Elasticsearch  
### Elasticsearch 설정 방법

Elasticsearch 설정 파일은 [`elasticsearch/config/elasticsearch.yml`][config-es]에 있습니다.

또한, `docker-compose.yml` 파일 내 환경 변수로 덮어쓰기할 옵션을 지정할 수 있습니다:

```yml
elasticsearch:

  environment:
    network.host: _non_loopback_
    cluster.name: my-cluster
```

Elasticsearch를 Docker에서 구성하는 방법에 대한 자세한 내용은  
[Elasticsearch Docker 설치 가이드][es-docker]를 참고하세요.

---

### How to configure Kibana  
### Kibana 설정 방법

Kibana의 기본 설정 파일은 [`kibana/config/kibana.yml`][config-kbn]에 위치합니다.

마찬가지로 `docker-compose.yml` 파일 내 환경 변수로 설정을 덮어쓸 수 있습니다:

```yml
kibana:

  environment:
    SERVER_NAME: kibana.example.org
```

Kibana를 Docker에서 구성하는 방법에 대한 자세한 내용은  
[Kibana Docker 설치 가이드][kbn-docker]를 참고하세요.

---

### How to configure Logstash  
### Logstash 설정 방법

Logstash 설정 파일은 [`logstash/config/logstash.yml`][config-ls]에 위치합니다.

`docker-compose.yml` 파일에서 환경 변수로 설정 값을 덮어쓸 수 있습니다:

```yml
logstash:

  environment:
    LOG_LEVEL: debug
```

Logstash를 Docker에서 구성하는 방법에 대한 자세한 내용은  
[Logstash Docker 설정 가이드][ls-docker]를 참고하세요.

---

### How to disable paid features  
### 유료 기능 비활성화 방법

라이선스가 만료되기 전 **체험판(trial)** 을 중단하고 **기본 라이선스(basic)** 로 전환하려면,  
Kibana의 [라이선스 관리][license-mngmt] 화면 또는 Elasticsearch의 `start_basic` [라이선스 API][license-apis]를 사용할 수 있습니다.

후자의 방법은 체험판이 만료된 후 Kibana에 접근이 불가능한 경우 유일한 복구 방법입니다.

Elasticsearch의 `xpack.license.self_generated.type` 설정 값을 `trial`에서 `basic`으로 변경해도 되지만,  
이 설정은 **초기 실행 전에만 유효**합니다.  
이미 체험판이 시작된 이후에는, 기능 축소에 대해 **명시적으로 동의**해야 하며, 위의 두 가지 방법 중 하나를 사용해야 합니다.

---

### How to scale out the Elasticsearch cluster  
### Elasticsearch 클러스터 확장 방법

위키에서 자세한 안내를 참고하세요:  
[Scaling out Elasticsearch](https://github.com/deviantony/docker-elk/wiki/Elasticsearch-cluster)

---

### How to re-execute the setup  
### 설정 재실행 방법

`.env` 파일에 비밀번호가 정의된 모든 사용자 계정을 **재초기화**하려면,  
다음 명령어로 `setup` 서비스를 다시 실행하면 됩니다:

```sh
docker compose up setup
 ⠿ Container docker-elk-elasticsearch-1  Running
 ⠿ Container docker-elk-setup-1          Created
Attaching to docker-elk-setup-1
...
docker-elk-setup-1  | [+] User 'monitoring_internal'
docker-elk-setup-1  |    ⠿ User does not exist, creating
docker-elk-setup-1  | [+] User 'beats_system'
docker-elk-setup-1  |    ⠿ User exists, setting password
docker-elk-setup-1 exited with code 0
```

---

### How to reset a password programmatically  
### 비밀번호를 프로그래밍 방식으로 재설정하는 방법

Kibana를 통해 사용자 비밀번호를 변경할 수 없는 상황(예: Kibana 접근 불가)에서는  
Elasticsearch API를 이용해 동일한 작업을 수행할 수 있습니다.

예를 들어, `elastic` 사용자의 비밀번호를 변경하려면 다음과 같이 실행합니다  
(URL에 `/user/elastic`이 포함되어 있음에 주목):

```sh
curl -XPOST -D- 'http://localhost:9200/_security/user/elastic/_password' \
    -H 'Content-Type: application/json' \
    -u elastic:<현재 비밀번호> \
    -d '{"password" : "<새 비밀번호>"}'
```

## Extensibility  
## 확장성

---

### How to add plugins  
### 플러그인 추가 방법

ELK 구성요소 중 어떤 것에든 플러그인을 추가하려면 아래 단계를 따르면 됩니다:

1. 해당 구성 요소의 `Dockerfile`에 `RUN` 명령을 추가하세요  
   (예: `RUN logstash-plugin install logstash-filter-json`)

2. 관련 플러그인 설정 코드를 서비스 설정 파일에 추가하세요  
   (예: Logstash input/output 설정)

3. 아래 명령어로 이미지들을 다시 빌드하세요:

```sh
docker compose build
```

---

### How to enable the provided extensions  
### 제공되는 확장 기능 활성화 방법

[`extensions`](extensions) 디렉토리 내부에는 몇 가지 확장 기능이 포함되어 있습니다.  
이 확장 기능들은 기본 Elastic 스택에는 포함되지 않지만,  
추가적인 통합 기능을 통해 스택을 강화할 수 있습니다.

각 확장 기능에 대한 문서는 해당 서브디렉토리 내부에 포함되어 있으며,  
일부 확장 기능은 기본 ELK 설정을 수동으로 수정해야 할 수도 있습니다.

## JVM tuning  
## JVM 튜닝

---

### How to specify the amount of memory used by a service  
### 서비스별 메모리 사용량 설정 방법

Elasticsearch와 Logstash는 환경 변수로 전달된 JVM 옵션을 읽어들여  
사용할 수 있는 메모리 양을 설정할 수 있습니다.

| 서비스        | 환경 변수           |
|---------------|---------------------|
| Elasticsearch | `ES_JAVA_OPTS`      |
| Logstash      | `LS_JAVA_OPTS`      |

Docker Desktop for Mac 같은 환경에서는 기본적으로 사용 가능한 메모리가 2GB로 제한되어 있기 때문에,  
`docker-compose.yml` 파일에서는 기본적으로 다음과 같이 JVM 힙 사이즈를 제한하고 있습니다:

- Elasticsearch: 512MB  
- Logstash: 256MB

이 값을 변경하려면 `docker-compose.yml`에서 해당 환경 변수를 수정하세요.  
예를 들어, Logstash의 JVM 최대 힙 사이즈를 1GB로 늘리고 싶다면:

```yml
logstash:

  environment:
    LS_JAVA_OPTS: -Xms1g -Xmx1g
```

**환경 변수를 설정하지 않으면 다음과 같은 기본값이 적용됩니다:**

- Elasticsearch는 [자동으로 힙 사이즈를 결정][es-heap]합니다.  
- Logstash는 기본적으로 1GB 힙 사이즈를 사용합니다.

---

### How to enable a remote JMX connection to a service  
### 서비스에 원격 JMX 연결을 활성화하는 방법

앞서 설명한 힙 메모리 설정과 마찬가지로,  
JVM 옵션을 통해 **JMX를 활성화하고** 해당 포트를 Docker 호스트에 매핑할 수 있습니다.

아래는 Logstash에 대해 예시를 든 설정입니다.  
JMX 서비스를 포트 `18080`에 바인딩하고 있으며, `DOCKER_HOST_IP`는 호스트의 실제 IP로 바꿔야 합니다:

```yml
logstash:

  environment:
    LS_JAVA_OPTS: >-
      -Dcom.sun.management.jmxremote
      -Dcom.sun.management.jmxremote.ssl=false
      -Dcom.sun.management.jmxremote.authenticate=false
      -Dcom.sun.management.jmxremote.port=18080
      -Dcom.sun.management.jmxremote.rmi.port=18080
      -Djava.rmi.server.hostname=DOCKER_HOST_IP
      -Dcom.sun.management.jmxremote.local.only=false
```


## Going further  
## 더 나아가기

---

### Plugins and integrations  
### 플러그인 및 통합

아래 위키 페이지들을 참고하세요:

* [외부 애플리케이션](https://github.com/deviantony/docker-elk/wiki/External-applications)  
* [인기 있는 통합](https://github.com/deviantony/docker-elk/wiki/Popular-integrations)
