# Ubuntu 24.04 설치 이미지 아키텍처 분석

> 2026-09-05. 계기: VirtualBox가 이 노트북에서 안 되니 우분투 ISO(6.2GB, `E:\ISO\`)를
> 대신 직접 마운트해서 내부 구조를 뜯어보고, 설치 로직을 실제로 구동하는 GitHub 오픈소스
> (`canonical/subiquity` — 설치 백엔드, `canonical/ubuntu-desktop-installer` — Flutter
> 프론트엔드)까지 확인함. "오래된 리눅스 배포판 설치 프로그램이라 별거 없을 것"이란
> 선입견 없이 봤고, 실제로 세 가지가 눈에 띄었다.
>
> **근거 자료**: ISO를 Windows `Mount-DiskImage`로 직접 마운트해 구조 확인 +
> 공식 GitHub `canonical/subiquity` 소스 fetch(`source.py`, `server.py`, `api/defs.py`).

---

## 1. 레이어드 SquashFS + 매니페스트 기반 설치 프로파일 선택

### 설계 원리
ISO 안(`casper/`)에는 완성된 OS 이미지 하나가 아니라, 독립적인 SquashFS **레이어들**이
따로 들어있다: `minimal.squashfs`(1.8GB, 최소 설치) → `minimal.standard.squashfs`
(562MB, 풀 설치 추가분) → `minimal.standard.live.squashfs`(991MB, 라이브 세션 전용
추가분). 여기에 언어별 팩(`minimal.en.squashfs` 등, 각 16MB 안팎)과, **시큐어부트
강화 커널/모듈로 다시 빌드한 `enhanced-secureboot` 버전 세트가 모든 레이어마다
한 벌씩 통째로 더** 들어있다. 어떤 조합을 쓸지는 `casper/install-sources.yaml`
매니페스트 하나가 결정한다:

```yaml
- id: ubuntu-desktop-minimal
  type: fsimage-layered
  variations:
    minimal: {path: minimal.squashfs, size: 4676382720}
    minimal-enhanced-secureboot: {path: minimal.enhanced-secureboot.squashfs, ...}
- id: ubuntu-desktop
  type: fsimage-layered
  variations:
    standard: {path: minimal.standard.squashfs, ...}
    enhanced-secureboot: {path: minimal.standard.enhanced-secureboot.squashfs, ...}
```

### 실제 코드 근거
`canonical/subiquity`의 `subiquity/server/controllers/source.py`:
```python
SOURCES_DEFAULT_PATH = "/cdrom/casper/install-sources.yaml"
...
if os.path.exists(path):
    with open(path) as fp:
        self.model.load_from_file(fp)
self._update_variant(self.model.current.variant)
```
이 매니페스트를 파싱해서 설치 화면에 "최소 설치/풀 설치" 선택지로 띄우고,
`_update_variant()`가 **호스트가 시큐어부트인지 런타임에 감지해서** 맞는 레이어
조합으로 자동 전환한다. 실제 압축 해제/적용은 별도 프로젝트 `curtin`에 위임 —
설치 백엔드(subiquity)와 저수준 디스크 조작(curtin)도 서로 분리돼 있다.

### 왜 강력한가
"설치 타입 × 시큐어부트 여부 × 언어" 조합마다 별도 ISO를 만드는 대신, **조합 가능한
독립 레이어 + 그걸 고르는 매니페스트 하나**로 구성 공간 전체를 커버한다. 새 조합이
필요해지면 레이어 하나만 추가하고 매니페스트에 한 줄 더 적으면 된다 — Docker/OCI
이미지 레이어와 개념적으로 동일한 것을 OS 설치 프로그램에 적용한 사례.

### 응용 착안점
미림 프로젝트에서 "감지된 하드웨어 능력(VRAM 용량, GPU 세대)에 따라 다른 모델/에셋
번들을 자동 선택"하는 기능이 필요해지면, 이 패턴을 그대로 가져올 수 있다 — 조합마다
전체를 다시 패키징하지 않고, 독립적인 콘텐츠 레이어들 + "어떤 조합을 어떤 조건에서
쓸지" 서술하는 매니페스트 하나로 구성. 실행 시점에 감지한 환경(GPU 스펙 등)에 맞는
레이어를 골라 조합하면 된다.

---

## 2. REST API + Unix 소켓 — 백엔드와 프론트엔드를 언어 무관하게 분리

### 설계 원리
설치 백엔드(subiquity)는 **REST API를 Unix 도메인 소켓 위에 얹어서** 서비스한다.
이 덕분에 원래 파이썬 curses(텍스트) UI였던 프론트엔드를, 이번 24.04에서 **완전히
다른 언어/프레임워크(Dart/Flutter, `ubuntu-desktop-installer`)로 통째로 교체**했는데도
백엔드는 손댈 필요가 없었다.

### 실제 코드 근거
`subiquity/server/server.py`:
```python
site = web.UnixSite(runner, self.opts.socket)
```
`aiohttp`의 `UnixSite`로 TCP 포트가 아니라 로컬 소켓 파일에 HTTP 서버를 띄운다.
어떤 언어든 그 소켓에 HTTP 요청만 보낼 수 있으면 프론트엔드가 될 수 있다.

### 왜 강력한가
VirtualBox의 COM 기반 Main API([[02번 문서]])와 목적은 같다(제어 UI와 실행 로직
분리) — 근데 COM은 Windows/특정 언어 바인딩에 묶이는 반면, **REST-over-Unix-socket은
플랫폼·언어에 완전히 무관**하다. 그래서 Canonical은 UI 프레임워크를 파이썬 curses에서
Dart/Flutter로 완전히 갈아엎으면서도 설치 로직(subiquity 서버)은 한 줄도 안 바꿨다.

### 응용 착안점
미림 프로젝트의 추론/생성 백엔드를 만들 때, 프론트엔드(웹, 데스크톱 툴, 나중에 만들
UI)를 나중에 자유롭게 바꿀 수 있게 하려면, 처음부터 **로컬 REST API(소켓 또는
localhost 포트)로 백엔드를 노출**하고 프론트엔드는 그 API의 클라이언트일 뿐이도록
설계하는 게 좋다. [[02번 문서]]의 "컨트롤/실행 프로세스 분리" 착안점을 실제로 구현할
때, 그 사이의 통신 방식으로 이 패턴(REST-over-소켓)을 채택하면 언어 선택의 자유가
생긴다.

---

## 3. 클래스 계층 자체가 라우팅 테이블이다 (선언적 API 정의)

### 설계 원리
subiquity의 API는 URL을 문자열로 일일이 등록하는 대신, **중첩된 파이썬 클래스
구조 자체가 URL 경로를 결정**하는 방식으로 정의된다.

### 실제 코드 근거
`subiquity/common/api/defs.py`의 `api()` 함수:
```python
for k, v in cls.__dict__.items():
    if isinstance(v, type):
        v.__shortname__ = k
        v.__name__ = cls.__name__ + "." + k
        path_part = k
        if getattr(v, "__parameter__", False):
            path_part = "{" + path_part + "}"
        ...
```
`API.source.next`처럼 클래스를 중첩해서 선언하면 `/source/next`라는 경로가 **자동으로**
생긴다. 경로 문자열을 따로 어딘가에 다시 적을 필요가 없다 — 타입 계층이 곧 라우팅
테이블이자 요청/응답 타입 스키마다.

### 왜 강력한가
"엔드포인트 목록"과 "그 엔드포인트의 요청/응답 타입"이 서로 다른 곳에 따로 존재하면
반드시 어긋나는 순간이 온다(경로는 바꿨는데 문서/클라이언트는 그대로인 경우 등).
여기서는 **구조 하나를 선언하면 라우팅표·타입 스키마·(서버/클라이언트 양쪽의) 호출
시그니처가 전부 그 구조에서 파생**된다 — Blender RNA([[01번 문서]] 1번 항목)의
"속성 하나 선언하면 UI·API·직렬화가 전부 따라온다"는 원칙과 정확히 같은 결이다.

### 응용 착안점
미림 백엔드의 REST API(위 2번 착안점)를 실제로 만들 때, 엔드포인트 목록을 라우터
설정 파일과 타입 정의 파일에 각각 따로 적지 말고, 이런 식으로 **하나의 선언(중첩
클래스/타입 구조)에서 경로·요청타입·응답타입이 전부 자동으로 나오는 프레임워크**를
쓰거나 직접 그렇게 설계하면, API가 커져도 경로와 타입이 어긋날 일이 없다.

---

## 응용 착안 리스트 (요약)

1. **레이어드 콘텐츠 + 선택 매니페스트**: 실행 환경(하드웨어 능력 등)에 따라 다른
   에셋/모델 조합을 골라 쓰는 기능은, 조합마다 통째로 패키징하지 말고 독립 레이어 +
   조건-매핑 매니페스트로 구성.
2. **REST-over-소켓으로 프론트/백엔드 언어 무관 분리**: [[02번 문서]]의 컨트롤/실행
   분리를 실제 구현할 때 통신 방식으로 채택 — 나중에 UI 프레임워크를 자유롭게 교체
   가능해짐.
3. **선언 하나 = 라우팅표 + 타입 스키마**: API가 커질수록 이 원칙(01번 문서의 RNA
   원칙과 동일한 결)을 지키면 경로/타입 불일치 버그를 원천 차단.

---

## 참고 자료 (Sources)

- 로컬: `E:\ISO\ubuntu-24.04.4-desktop-amd64.iso`(마운트해서 `casper/install-sources.yaml`
  등 직접 확인)
- [canonical/subiquity — GitHub](https://github.com/canonical/subiquity)
  (`subiquity/server/controllers/source.py`, `subiquity/server/server.py`,
  `subiquity/common/api/defs.py`)
- [canonical/ubuntu-desktop-installer — GitHub](https://github.com/canonical/ubuntu-desktop-installer)
