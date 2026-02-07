```markdown
# Pterodactyl Egg JSON 만들 때 정리 + CPU 쓰레드/리소스 확인 방법

---

## 1. Egg 개념 & JSON 역할

- Egg는 “어떤 Docker 이미지에, 어떤 커맨드로, 어떤 설정 파일을 어떻게 만져서, 어떤 환경변수를 노출할지”를 정의하는 **템플릿 JSON**입니다.[cite:4]
- **리소스 한도(메모리, CPU, 디스크 등)** 는 Egg JSON 안이 아니라, **서버(Build 설정)** 쪽에서 관리됩니다.[cite:7]
- JSON은 패널에서 **Export** 하면 `egg-xxx.json` 같은 형식으로 나오고, 이걸 수정해서 다시 Import 하는 식으로 작업하는 것이 가장 안전합니다.[cite:11]

---

## 2. Egg JSON 전체 구조 개요

보통 Egg JSON은 대략 이런 구조를 가집니다.[cite:1][cite:5][cite:11]

```json
{
  "meta": {
    "version": "PTDL_v2"
  },
  "exported_at": "2024-01-01T00:00:00+00:00",
  "name": "Example App",
  "author": "you@example.com",
  "description": "Example application egg",
  "features": null,
  "docker_images": {
    "ghcr.io/pterodactyl/yolks:node_20": "Node 20"
  },
  "file_denylist": [],
  "startup": "node {{SERVER_FILE}}",
  "config": {
    "files": {
      "config.yml": {
        "parser": "yaml",
        "find": {
          "server.port": "{{server.build.default.port}}",
          "server.host": "0.0.0.0"
        }
      }
    },
    "startup": "{\r\n  \"done\": \"Server started\"\r\n}",
    "logs": "{}",
    "stop": "stop"
  },
  "scripts": {
    "installation": {
      "script": "#!/bin/bash\ncd /home/container\n# ... 설치 스크립트 ...",
      "container": "ghcr.io/pterodactyl/installers:alpine",
      "entrypoint": "bash"
    }
  },
  "variables": [
    {
      "name": "Server Jar File",
      "description": "JAR 파일 이름",
      "env_variable": "SERVER_JARFILE",
      "default_value": "server.jar",
      "user_viewable": true,
      "user_editable": true,
      "rules": "required|string|max:128",
      "field_type": "text"
    }
  ]
}
```

---

## 3. 필드별 상세 & 주의점

### 3.1 `meta`, `exported_at`

- `meta.version` 은 Egg 포맷 버전입니다. 최신 패널 기준 `PTDL_v2` 계열 사용.
- 다른 사람이 만든 Egg를 가져와서 쓸 때, **패널 버전과 meta 버전이 맞는지** 체크하면 호환성 이슈 줄어듭니다.[cite:11]

### 3.2 기본 정보: `name`, `description`, `docker_images`, `startup`

- `name`, `description`  
  - 패널 UI에 표시되는 이름/설명입니다.  
  - 여러 Egg를 운용할 때, 버전/런타임(JDK 17, Node 20 등)을 이름에 넣으면 관리가 쉽습니다.

- `docker_images`[cite:6][cite:15]  
  - `"이미지명": "라벨"` 형식의 map입니다.
  - 반드시 **신뢰할 수 있는 공식/검증된 이미지**를 사용해야 합니다. 임의 Docker Hub 이미지 사용은 보안 리스크가 큽니다.[cite:6]
  - Pterodactyl 공식 yolk 예:
    - `ghcr.io/pterodactyl/yolks:java_17`
    - `ghcr.io/pterodactyl/yolks:node_20` 등[cite:15]

- `startup` (시작 커맨드)[cite:2]  
  - 예: `java -Xms128M -Xmx{{SERVER_MEMORY}}M -jar {{SERVER_JARFILE}}`  
  - 여기서 `{{SERVER_MEMORY}}`, `{{SERVER_JARFILE}}`, `{{SERVER_PORT}}` 같은 값은:
    - 패널에서 서버 생성 시 설정한 값
    - 또는 `variables` 에서 정의한 env 변수로부터 치환됩니다.[cite:2][cite:8]
  - Egg 내에서 사용 가능한 기본 변수 예시:[cite:2]
    - `SERVER_MEMORY` – 서버 메모리(MB)
    - `SERVER_IP` – 서버 IP
    - `SERVER_PORT` – 기본 포트
    - `P_SERVER_UUID` – 서버 UUID  
  - 주의:
    - **대소문자, 철자 정확히** 써야 합니다. 플레이스홀더는 **case-sensitive** 입니다.[cite:2]

### 3.3 `config` 블록: 파일 파서, 로그, stop

`config` 는 Wings가 서버 시작 전에 설정 파일을 자동으로 수정해주는 블록입니다.[cite:2]

```json
"config": {
  "files": {
    "server.properties": {
      "parser": "properties",
      "find": {
        "server-ip": "0.0.0.0",
        "enable-query": "true",
        "server-port": "{{server.build.default.port}}",
        "query.port": "{{server.build.default.port}}"
      }
    },
    "config.yml": {
      "parser": "yaml",
      "find": {
        "listeners.host": "0.0.0.0:{{server.build.default.port}}",
        "servers.*.address": {
          "127.0.0.1": "{{config.docker.interface}}",
          "localhost": "{{config.docker.interface}}"
        }
      }
    }
  },
  "startup": "{\r\n  \"done\": \"Done (.*)! For help, type \"\r\n}",
  "logs": "{}",
  "stop": "^C"
}
```

- `files`[cite:2]
  - `parser` 타입:
    - `file` – 단순 라인 매칭 (가능하면 지양)
    - `yaml`, `json` – 계층 구조 파싱 및 `*` 와일드카드 지원
    - `ini`, `properties`, `xml` 등 지원[cite:2]
  - `find` 키 안에서:
    - `server.build.default.port` 같은 서버 메타데이터,  
      `config.docker.interface`, `env.VAR_NAME` 등의 **템플릿 변수** 사용 가능[cite:2].
  - 주의점:
    - YAML/JSON은 구조, 인덴트 깨지면 앱이 아예 안 뜨니, 파서가 건드리는 path를 최소화하는 것이 좋습니다.
    - `servers.*.address` 같이 `*` 와일드카드를 사용할 때, 실제 파일 구조와 정확히 맞는지 반드시 테스트 필요.[cite:2]

- `startup` (로그에서 “서버가 떴다”를 감지하는 패턴)[cite:2]
  - Wings가 로그를 보고 “부팅 완료” 판단에 사용하는 정규식/텍스트 패턴 JSON입니다.
  - 일부 앱은 부팅 완료 로그가 없어서, 너무 넓은 패턴을 쓰면 조기완료로 인식하거나 영원히 pending 에 빠질 수 있습니다.
  
- `logs`
  - 일반적으로 `{}` 로 두는 경우가 많습니다. 커스텀 로그 위치를 잡고 싶을 때 사용.

- `stop`
  - 예: `"^C"` – `SIGINT` (Ctrl+C) 보내기.[cite:2]
  - 앱에 따라 `stop`, `quit` 같은 텍스트 명령을 stdin으로 보내게도 설정 가능.

### 3.4 `scripts.installation`: 설치 스크립트

```json
"scripts": {
  "installation": {
    "script": "#!/bin/bash\ncd /home/container\n# curl 또는 wget으로 파일 다운로드 등...",
    "container": "ghcr.io/pterodactyl/installers:alpine",
    "entrypoint": "bash"
  }
}
```

- `script`
  - 기본적으로 `/home/container` 를 작업 디렉터리로 잡고, 필요한 바이너리/서버 파일을 설치합니다.
  - `LF` 줄바꿈을 유지해야 하고, Windows에서 작성 후 복사하면 CRLF 이슈가 날 수 있으니 주의.
  - 설치가 길면 `set -e` 또는 꼼꼼한 에러 체크로 중간 실패 시 바로 에러 반환하도록 하는 게 좋습니다.

- `container`
  - 설치 시에만 사용하는 전용 컨테이너 이미지입니다.
  - `installer_limits` 설정에 따라 이 컨테이너에도 CPU/메모리 제한이 걸립니다.[cite:10]

- `entrypoint`
  - 보통 `bash` 또는 `sh` 사용. 사용 중인 이미지를 확인해서 어떤 쉘이 있는지 확인해야 함.

### 3.5 `variables`: 환경 변수 정의 & 검증

`variables` 는 패널에서 유저가 수정할 수 있는 설정들을 정의합니다.[cite:2][cite:8]

```json
{
  "name": "Server Jar File",
  "description": "The name of the server jarfile to run",
  "env_variable": "SERVER_JARFILE",
  "default_value": "server.jar",
  "user_viewable": true,
  "user_editable": true,
  "rules": "required|string|max:20",
  "field_type": "text"
}
```

- `env_variable`
  - 반드시 **대문자 + 숫자 + 언더스코어** 정도의 안전한 문자로 구성 (예: `MY_APP_MODE`).[cite:2]
  - Startup 커맨드에서 `{{MY_APP_MODE}}`, config 파서에서는 `{{env.MY_APP_MODE}}`, 쉘 스크립트에서는 `$MY_APP_MODE` 로 사용.[cite:2]

- `rules` – Laravel Validation Rules[cite:2][cite:8]
  - 예:
    - `required|string|between:1,10`
    - `nullable|numeric`
    - `required|regex:/^([\\w\\d._-]+)(\\.jar)$/`
  - 복잡한 포맷 제약이 필요하면 `regex:` 를 활용.

- `user_viewable` / `user_editable`
  - true/false 조합으로:
    - 유저가 볼 수만 있게 할지,
    - 아예 안 보이게 숨길지,
    - 수정까지 가능하게 할지 제어.

- `field_type`
  - 보통 `text`를 사용하지만, 패널 버전에 따라 select형 등도 존재.

---

## 4. CPU/메모리/디스크 리소스와 Egg의 관계

Egg JSON에는 **리소스 한도 정보가 직접 들어가지 않습니다.**  
실제 리소스 한도는 서버의 `build.limits` 객체로 관리됩니다.[cite:7]

`limits` 예시 (Application API 기준):[cite:7]

```json
"limits": {
  "memory": 2048,
  "swap": 0,
  "disk": 10240,
  "io": 500,
  "cpu": 200,
  "threads": "0-3"
}
```

- `memory` – MB 단위 메모리 제한
- `disk` – MB 단위 디스크 제한
- `cpu` – **CPU 사용률 한도(%)**[cite:7]
  - 예: `200`이면 “200%” = 보통 **코어 2개 상당**으로 간주됩니다.
- `threads` – CPU 쓰레드(코어) 핀닝 정보 (옵션)[cite:7]
  - 예: `"0-3"` → 0,1,2,3 번 쓰레드에만 스케줄링
  - 예: `"0,2"` → 0,2 번 쓰레드에만 스케줄링  
- `io` – 블록 IO weight

호스팅 업체 문서 기준, **“CPU 100% = 쓰레드 1개”**로 보는 경우가 많습니다.[cite:13]

---

## 5. “CPU 쓰레드 할당된 거 얼마인지” 확인 방법

### 5.1 패널 UI에서 확인

1. **관리자(Admin) 패널** 접속
2. `Servers` → 해당 서버 선택
3. 상단 탭에서 **Build Configuration** 또는 비슷한 메뉴 선택
4. 여기서:
   - `CPU Limit` (예: 200%)  
   - `CPU Threads` 또는 `CPU Pinning` (예: 빈 값, 또는 `0-3`, `0,1`)  
   를 확인할 수 있습니다.

일반 유저 패널에서는 보통 “리소스 한도” 같은 곳에 메모리/디스크/CPU % 정도가 표시됩니다.  
쓰레드 핀닝(`threads`)은 관리자 쪽 Build 설정에서 주로 보입니다.

---

### 5.2 API로 확인 (정확/자동화용)

Pterodactyl Application API 문서 기준, 서버 정보 조회 시 `limits` 객체에 `cpu`와 `threads`가 포함됩니다.[cite:7]

- 엔드포인트 예시 (v0.7 문서 기준):[cite:7]

```bash
curl -X GET "https://your-panel.com/api/application/servers/{server_id}" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Accept: application/json"
```

- 응답 중 일부:

```json
"attributes": {
  "id": 1,
  "uuid": "xxxx-xxxx",
  "name": "Example Server",
  "limits": {
    "memory": 2048,
    "swap": 0,
    "disk": 10240,
    "io": 500,
    "cpu": 200,
    "threads": "0-3"
  },
  "feature_limits": {
    "databases": 2,
    "allocations": 1,
    "backups": 1
  }
}
```

여기서:

- `cpu: 200` → CPU 200%  
- `threads: "0-3"` → CPU 쓰레드 4개(0,1,2,3) 핀닝

을 그대로 읽으면 됩니다.

---

### 5.3 컨테이너/노드에서 직접 확인 (깊게 파는 방법)

**전제로** Wings는 Docker의 CPU quota / cpuset 기능을 사용해 제한을 겁니다.

1. **컨테이너 내부에서 cgroup 확인 (cgroup v2 예시)**  
   - `/sys/fs/cgroup/cpu.max` 파일 내용:
     - 예: `200000 100000` → `quota = 200000`, `period = 100000`
     - 쓰레드(논리코어) 수 ≈ `quota / period = 2`
2. **cgroup v1인 경우**
   - `/sys/fs/cgroup/cpu/cpu.cfs_quota_us`
   - `/sys/fs/cgroup/cpu/cpu.cfs_period_us`
   - 마찬가지로 `quota / period` 계산.
3. **Docker Host에서 `docker inspect`**
   - `HostConfig.CpuQuota`, `CpuPeriod`, `CpusetCpus` 를 보면:
     - `CpuQuota / CpuPeriod` → 논리 코어 상대 할당량
     - `CpusetCpus` → 실제 핀닝된 쓰레드 리스트 (예: `"0-3"`)

이 방법은 Pterodactyl 레벨이 아니라 리눅스/도커 레벨에서 실제로 어떻게 제한되어 있는지 보는 방식이라, 호스팅 환경을 디버깅할 때 유용합니다.

---

### 5.4 “CPU % → 쓰레드 수” 대략적인 해석

호스팅사 일반 가이드 기준으로:[cite:13]

- `CPU 100` → 쓰레드 1개
- `CPU 200` → 쓰레드 2개
- `CPU 50` → 전체 코어의 절반 정도

즉, 특별히 쓰레드 핀닝(`threads`)을 쓰지 않는다면:

- **쓰레드 수 ≈ `cpu / 100`** 라고 볼 수 있습니다.  
  (물론 실제 물리 코어/논리 코어 개수와 노드 전체 상황에 따라 다소 차이는 있을 수 있음)

---

## 6. Wings 설정값과 CPU 관련 추가 주의점

Wings `config.yml` 에도 CPU/리소스 관련 옵션이 있습니다.[cite:10]

### 6.1 `installer_limits`

```yaml
installer_limits:
  memory: 1024
  cpu: 100
```

- 설치 컨테이너에 대한 한도입니다.
- “서버 리소스 한도”와 비교해 **더 큰 값이 우선** 적용됩니다.[cite:10]
- 설치 과정에서 CPU 100% / 메모리 1024MB 이상을 쓰지 못하게 막아주는 안전장치.

### 6.2 Throttles 및 기타

- `throttles` 설정으로 과도한 로그 출력 시 서버를 강제 중지시키는 기능이 있습니다.[cite:10]
- `container_pid_limit` 으로 컨테이너 내 **프로세스/쓰레드 수** 상한을 지정할 수 있습니다.[cite:10]
  - 기본값 512, 0이면 제한 없음 (비추천).

### 6.3 CPU 0% (무제한) 설정 주의

- `limits.cpu: 0` → **CPU 제한 없음**.[cite:10]
- 문서에서도 이 설정은 악용 시 노드 전체를 먹어버릴 수 있으므로 추천하지 않는다고 명시합니다.[cite:10]
- 퍼블릭 호스팅/여러 유저가 쓰는 노드에서는 반드시 최소한의 CPU 제한을 두는 것이 좋습니다.

---

## 7. Egg JSON 작성 시 실무적 주의사항 정리

### 7.1 기본 원칙

1. **기본 제공 Egg는 수정하지 말 것**  
   - 패널 업그레이드 때 덮어씌워질 수 있습니다.[cite:2]
   - 새 Nest & Egg를 만들어 **복사본**을 기반으로 수정하는 게 안전.

2. **패널에서 먼저 만들고 → Export → JSON 편집**  
   - JSON을 맨바닥에서 쓰기보다는, 패널에서 한 번 UI로 만들고 Export 후 수정하는 방식이 에러가 훨씬 적습니다.

3. **GitHub 등 검증된 Egg 레포 참고**[cite:1][cite:3][cite:5][cite:11]
   - 공식/커뮤니티 Egg들을 보면:
     - variables 설계 패턴
     - 설치 스크립트 구조
     - config 파서 사용 예시  
     등을 그대로 가져와 재활용할 수 있습니다.

### 7.2 JSON/스크립트 포맷 관련

- JSON은 **코멘트 불가**, 마지막 쉼표 금지, 문자열은 반드시 `"` 로 감싸야 합니다.
- `scripts.installation.script` 안의 줄바꿈/따옴표는 **JSON Escape** 가 정확해야 합니다.
  - 큰 스크립트는 `.sh` 파일로 따로 관리 후, 나중에 JSON에 넣을 때 자동 변환 스크립트를 쓰는 것도 방법입니다.
- Windows에서 복사해오는 경우 CRLF → LF 변환을 꼭 확인.

### 7.3 Docker 이미지/권한 관련

- 대부분의 yolk 이미지는 **유저 UID 1000, /home/container** 를 기본으로 사용합니다.
- 설치 스크립트/앱이 `/root` 를 전제로 하지 않도록 주의.
- 이미지 태그를 `latest` 로 두면 나중에 런타임 버전 변동으로 서버가 갑자기 안 뜨는 일이 생기니,
  - 가급적 `java_17`, `node_20` 같이 **고정 태그** 사용을 추천합니다.[cite:15]

### 7.4 Variables/Validation 설계 팁

- 자주 바꾸는 값:
  - 포트, JAR 이름, 모드/환경(DEV/PROD), 추가 인자 등은 `variables` 로 빼는 게 좋습니다.
- 유저 입력을 제한하고 싶을 때:
  - `rules` 에 `regex:` 를 적극적으로 사용하면, 잘못된 값으로 서버가 안 뜨는 상황을 많이 줄일 수 있습니다.[cite:2][cite:8]

---

## 8. 간단 예제: Node.js 앱용 Egg JSON (축약본)

실제로 쓸 수 있는 최소한의 구조 예시입니다.

```json
{
  "meta": {
    "version": "PTDL_v2"
  },
  "exported_at": "2026-02-07T00:00:00+09:00",
  "name": "Custom Node App",
  "author": "you@example.com",
  "description": "Custom Node.js application",
  "features": null,
  "docker_images": {
    "ghcr.io/pterodactyl/yolks:node_20": "Node 20"
  },
  "file_denylist": [],
  "startup": "node {{SERVER_FILE}}",
  "config": {
    "files": {},
    "startup": "{\r\n  \"done\": \"Listening on port\"\r\n}",
    "logs": "{}",
    "stop": "^C"
  },
  "scripts": {
    "installation": {
      "script": "#!/bin/bash\ncd /home/container\n\nif [ ! -f package.json ]; then\n  echo \"{}\" > package.json\nfi\n\necho \"Install dependencies...\"\nnpm install\n",
      "container": "ghcr.io/pterodactyl/installers:alpine",
      "entrypoint": "bash"
    }
  },
  "variables": [
    {
      "name": "Server File",
      "description": "Node entry file",
      "env_variable": "SERVER_FILE",
      "default_value": "index.js",
      "user_viewable": true,
      "user_editable": true,
      "rules": "required|string|max:128",
      "field_type": "text"
    }
  ]
}
```

이 예제는:

- Docker yolk: Node 20 사용
- `SERVER_FILE` 변수로 엔트리 파일 변경 가능
- 설치 시 `npm install` 수행
- 로그에서 `"Listening on port"` 포함 라인을 부팅 완료로 간주

정도로 최소 기능만 구현한 형태입니다.

---

## 9. 요약

- Egg JSON은 **앱 설치/실행 방법과 설정 파일 자동 수정, 환경 변수 정의**를 담당하고,
- **CPU/메모리/디스크/쓰레드 제한은 서버 Build 설정(및 API의 `limits`)** 에서 관리합니다.[cite:7][cite:10]
- CPU 쓰레드/할당량을 알고 싶다면:
  - 패널의 Build Configuration
  - Application API (`limits.cpu`, `limits.threads`)
  - 컨테이너 내부 cgroup / Docker inspect  
  순서대로 점점 더 로우 레벨 방식으로 확인할 수 있습니다.
- Egg JSON을 직접 작성할 때는:
  - 기존 Egg Export → 수정
  - Docker 이미지/스크립트 포맷/변수 검증/Config 파서 설정  
  네 가지를 특히 신경 쓰면 안정성이 크게 올라갑니다.
```