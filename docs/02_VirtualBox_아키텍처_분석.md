# Oracle VirtualBox 아키텍처 심층 분석

> 2026-09-05. 계기: 이 노트북(LG 15UD490)에서 VirtualBox VM을 직접 실행하려다
> `VERR_SVM_DISABLED`(BIOS 레벨 가상화 잠금, LG가 하드웨어단에서 막아둠)로 실패함.
> VM 자체는 못 띄웠지만, "이왕 막힌 김에 왜 막혔는지, 그리고 이 오래된(20년+) 오픈소스
> 하이퍼바이저가 어떻게 설계돼 있는지"를 선입견 없이 뜯어봄 — 상용 제품(Hyper-V,
> VMware)이라 특별할 거란 가정도, 오픈소스라 허술할 거란 가정도 없이 실제 코드로 확인.
>
> **근거 자료**: 공식 GitHub 미러 `github.com/VirtualBox/virtualbox`(1.4GB, 실제 소스
> 직접 fetch — `SUPDrv.cpp`, `HMR3-x86.cpp`, `HMR0-x86.cpp`, `VBoxManage.cpp`,
> `VBoxManageMisc.cpp`, `DevAHCI.cpp`, `VD.cpp`). 이 노트북에서 VM을 못 띄우다 보니
> "실행해보며 관찰"이 아니라 순수하게 소스 리딩으로만 진행.

---

## 1. 제어(Control)와 실행(Execution)을 완전히 분리된 프로세스로 쪼갠다

### 설계 원리
`VBoxManage`(CLI)나 GUI는 VM을 직접 실행하지 않는다. COM 기반 **Main API**를 통해
"이 VM을 켜줘"라는 요청만 보내고, 실제 VM 실행은 **완전히 별도의 OS 프로세스**(헤드리스
러너 또는 GUI 러너)가 새로 스폰되어 담당한다. 두 프로세스 사이는 `IProgress`라는
비동기 진행상황 객체로 연결된다 — 컨트롤 프로세스는 그 객체를 폴링/대기만 하고, 실제
하이퍼바이저 초기화·CPU 가상화 활성화·에러 발생은 전부 실행 프로세스 안에서 일어난다.

### 실제 코드 근거
`VBoxManageMisc.cpp`의 `handleStartVM`:
```c
CHECK_ERROR(machine, LaunchVMProcess(a->session, sessionType.raw(),
                                     ComSafeArrayAsInParam(aBstrEnv), progress.asOutParam()));
RTPrintf("Waiting for VM \"%s\" to power on...\n", pszVM);
CHECK_ERROR(progress, WaitForCompletion(-1));
```
이 노트북에서 `VBoxManage startvm`을 실행했을 때 실제로 터미널에 찍혔던
`"Waiting for VM ... to power on..."`이 정확히 이 줄이다. 그리고 그 뒤에 받은
`VERR_SVM_DISABLED` 에러는 `VBoxManage.exe` 프로세스가 낸 게 아니라, **스폰된 별도
프로세스 안에서 VMM이 초기화 실패한 결과가 `IProgress`를 통해 컨트롤 프로세스로
전달되어 출력된 것**이다 — 즉 에러의 발생 위치와 출력 위치가 프로세스 경계를
하나 건너 뛴다.

### 왜 강력한가
- 실행 프로세스가 죽어도(크래시/행) 컨트롤 프로세스(CLI/GUI)는 살아있다 — 장애 격리.
- 무거운 초기화(가상화 활성화, 디바이스 모델 로딩)를 컨트롤 프로세스의 응답성과
  완전히 분리해서, CLI/GUI는 항상 가볍고 즉각 반응한다.
- 여러 VM을 동시에 켜도 서로의 실행 프로세스가 독립적이라 한쪽이 죽어도 나머지에
  영향이 없다.

### 응용 착안점
미림 프로젝트에서 "실시간 캐릭터 추론/렌더링" 같은 무거운 작업을 돌릴 때, 편집
도구(캐릭터 설정 UI, 프롬프트 관리 툴)와 실제 추론 실행체를 **처음부터 별도 프로세스로
분리**하고 그 사이를 진행상황 객체(진행률/완료/에러 코드)로만 연결하는 설계를 고려할
만하다. 언리얼 에디터의 "플레이 버튼 → 별도 게임 프로세스" 구조나, 크롬의 탭-프로세스
분리와 같은 결 — "무거운 실행체가 죽어도 컨트롤러는 살아있어야 한다"는 원칙을 초반
아키텍처 설계 단계에서부터 반영하면, 나중에 추론 엔진 하나가 죽었다고 전체 툴이
멎는 상황을 피할 수 있다.

---

## 2. 디바이스 에뮬레이션의 "컴포넌트별 저장 상태 버전 관리"

### 설계 원리
VM을 일시정지/스냅샷할 때는 실제 물리 하드웨어가 아니라 **에뮬레이션된 가짜 하드웨어의
내부 레지스터 상태**까지 통째로 직렬화해야 한다. VirtualBox는 이걸 디바이스 모델
하나하나가 **자기 저장 포맷에 명시적 버전 번호를 스스로 관리**하는 방식으로 해결한다.

### 실제 코드 근거
`DevAHCI.cpp`(SATA 디스크 컨트롤러 에뮬레이션) 상단:
```c
#define AHCI_SAVED_STATE_VERSION                        9
#define AHCI_SAVED_STATE_VERSION_PRE_ATAPI_REMOVE       8
#define AHCI_SAVED_STATE_VERSION_PRE_PORT_RESET_CHANGES 7
#define AHCI_SAVED_STATE_VERSION_PRE_HOTPLUG_FLAG       6
#define AHCI_SAVED_STATE_VERSION_IDE_EMULATION          5
#define AHCI_SAVED_STATE_VERSION_PRE_ATAPI              3
#define AHCI_SAVED_STATE_VERSION_VBOX_30                2
```
디바이스 하나(SATA 컨트롤러)만 놓고 봐도 최소 7단계의 이전 포맷 버전을 이름 붙여
기억하고 있다. 로드 시점에 저장된 버전 번호를 읽어서 "너는 몇 번 포맷이었지"를
분기 처리해 옛 스냅샷도 새 버전 VirtualBox에서 깨지지 않고 읽힌다. 같은 파일 상단
설계 주석에는 "레지스터 인터페이스(게스트가 보는 부분)와 실제 데이터 전송(워커
스레드)을 분리한다"는 원칙도 명시돼 있어, **저장 대상(레지스터 상태)과 실행 로직
(스레드/큐)이 처음부터 분리 설계**돼 있다는 것도 확인된다.

### 왜 강력한가
버전 관리를 파일 전체 단위("이 스냅샷 파일은 v9")가 아니라 **컴포넌트(디바이스) 단위**로
쪼개서 각자 독립적으로 진화시킨다. SATA 컨트롤러 포맷이 바뀌어도 네트워크 카드나
사운드 디바이스의 저장 포맷 버전에는 전혀 영향이 없다 — 전체를 한 번에 마이그레이션할
필요 없이 컴포넌트별로 국소적으로 버전을 올릴 수 있다.

### 응용 착안점
미림/게임엔진 캐릭터 저장 포맷(예: 리그 상태, 표정 파라미터, 애니메이션 컨트롤러 상태)을
설계할 때, 파일 전체에 버전 하나를 붙이는 대신 **컴포넌트별(뼈대/표정/의상/물리 등)로
독립적인 버전 번호를 부여**하는 구조를 쓰면, 나중에 "표정 시스템만 리팩터링했는데 저장된
캐릭터 전체가 깨지는" 문제를 피할 수 있다. 로드 시점에 컴포넌트별 버전을 보고 각자
맞는 마이그레이션 함수를 타게 하면 되고, 이건 [[01번 문서]]의 "자기 기술적 저장 포맷"
착안점과도 직접 연결된다 — 다만 거기서는 파일 전체가 스키마를 동봉하는 방식이었다면,
여기서는 그 스키마 버전 관리를 **컴포넌트 단위로 더 잘게 쪼갠 버전**이라는 차이가 있다.

---

## 3. (참고) 왜 이 노트북에서는 애초에 못 켰는가 — 하드웨어 잠금의 실체

`SUPDrv.cpp`의 `SUPR0GetSvmUsability()`:
```c
fVmCr = ASMRdMsr(MSR_K8_VM_CR);              // AMD 전용 MSR 0xC0010114 직접 읽기
if (!(fVmCr & MSR_K8_VM_CR_SVM_DISABLE))
    rc = VINF_SUCCESS;
else
    rc = VERR_SVM_DISABLED;                  // 우리가 본 그 에러
```
커널 드라이버 권한을 가진 VirtualBox조차 이 MSR 비트를 **읽기만 하지 쓸 수 없다** —
AMD 스펙상 BIOS가 부팅 시 한 번 잠그면 이후 어떤 소프트웨어도 못 푸는 구조. 응용 착안점
성격의 내용은 아니고, "소프트웨어로 풀 수 없다"는 판단이 추측이 아니라 실제 하이퍼바이저
자신의 커널 드라이버 소스로 확정됐다는 근거 기록 차원에서 남겨둔다.

---

## 응용 착안 리스트 (요약)

1. **컨트롤/실행 프로세스 분리 + 진행상황 객체**: 무거운 실시간 추론/렌더링 실행체와
   편집·관리 툴을 처음부터 별도 프로세스로 분리하고, 그 사이를 진행률/완료/에러코드
   객체로만 연결 — 실행체 크래시가 툴 전체를 죽이지 않도록.
2. **컴포넌트 단위 저장 상태 버전 관리**: 캐릭터 저장 포맷을 파일 전체 버전 하나가
   아니라 컴포넌트(뼈대/표정/의상 등)별 독립 버전으로 설계 — 일부 시스템만 리팩터링해도
   나머지 저장 데이터가 깨지지 않게. [[01_아키텍처_분석.md]]의 "자기 기술적 저장 포맷"
   착안점을 컴포넌트 단위로 세분화한 버전.

---

## 참고 자료 (Sources)

- [VirtualBox/virtualbox — 공식 GitHub 미러](https://github.com/VirtualBox/virtualbox)
- 로컬 fetch 소스: `src/VBox/HostDrivers/Support/SUPDrv.cpp`,
  `src/VBox/VMM/VMMR3/target-x86/HMR3-x86.cpp`,
  `src/VBox/VMM/VMMR0/target-x86/HMR0-x86.cpp`,
  `src/VBox/Frontends/VBoxManage/VBoxManage.cpp`,
  `src/VBox/Frontends/VBoxManage/VBoxManageMisc.cpp`,
  `src/VBox/Devices/Storage/DevAHCI.cpp`,
  `src/VBox/Storage/VD.cpp`
