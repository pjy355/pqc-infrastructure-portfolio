# PQC Infrastructure Portfolio

Post-Quantum Cryptography 기반 IPsec/IKEv2 터널 구축 및 검증 포트폴리오

[![OpenSSL 3.5](https://img.shields.io/badge/OpenSSL-3.5-green)](https://github.com/openssl/openssl/releases/tag/openssl-3.5.0)
[![strongSwan 6.0.6](https://img.shields.io/badge/strongSwan-6.0.6-blue)](https://www.strongswan.org/)
[![Ubuntu 22.04](https://img.shields.io/badge/Ubuntu-22.04-orange)](https://releases.ubuntu.com/22.04/)

---

## 프로젝트 개요

TRENTO Systems 인턴십 기간(2026년 3월~6월) 동안 Post-Quantum Cryptography를 실무 인프라에 적용하는 전체 과정을 진행했습니다. 양자 컴퓨터의 위협으로부터 통신을 보호하기 위해 NIST가 표준화한 ML-KEM 암호화 알고리즘을 OpenSSL 3.5에서 구현하고, 이를 strongSwan 6.0.6과 통합하여 완전한 PQC 기반 VPN 인프라를 구축했습니다.

단순히 기술을 배우는 것을 넘어, 실제 회사 서버에 적용하는 과정에서 다양한 기술적 도전과제를 해결했으며, 그 과정에서 배운 실무 경험을 정리한 포트폴리오입니다.

---

## 프로젝트 구성

### quickstart/

OpenSSL 3.5 빌드부터 strongSwan 6.0.6 설정까지 전체 프로세스를 단계별로 따라할 수 있는 가이드입니다. 처음부터 끝까지 약 2-3시간이 소요됩니다.

이 가이드는 다음을 포함합니다:
- 사전 요구사항 설치 및 환경 구성
- OpenSSL 3.5 소스 빌드 (ML-KEM 내장 확인)
- strongSwan 6.0.6 빌드 및 RPATH 설정을 통한 라이브러리 통합
- PKI 기반 인증서 체인 구축 (Root CA → Server/Client)
- swanctl.conf 작성 및 설정 로드
- charon 데몬 실행 및 IKEv2 터널 검증
- 예상되는 모든 오류 상황과 해결책

초기화된 Ubuntu 22.04 환경에서 테스트 완료했으며, RPATH 설정 오류, LD_LIBRARY_PATH 문제, 플러그인 로드 실패 등 실무에서 흔히 만나는 모든 문제의 원인 분석과 해결 방법을 포함했습니다.

### openssl-comparison/

OpenSSL 3.5, 3.6, 4.0 세 버전을 직접 구축하고 상세하게 비교 분석한 내용입니다.

주요 비교 항목:
- Post-Quantum Cryptography 지원 현황 (ML-KEM 가용성)
- API 호환성 (SSLv3, ENGINE API 등 레거시 기능)
- 하이브리드 암호화 그룹 구성 가능성
- LMS 알고리즘의 사용 제약사항
- 빌드 난이도 및 의존성

결론적으로 OpenSSL 3.5가 가장 안정적이고 실무적 요구사항을 충족한다는 것을 확인했으며, 이는 이후 회사 서버 구축의 기준이 되었습니다.

### tls-mtls-validation/

NSS, GnuTLS, wolfSSL, BoringSSL, Secure Transport, SChannel 등 6가지 TLS 라이브러리에서 PQC 지원 현황을 검증했습니다.

각 라이브러리별로:
- ML-KEM 알고리즘 지원 여부
- 하이브리드 키 교환 그룹 구현
- 플랫폼별 가용성 (Windows, macOS, Linux)
- 실제 mTLS 핸드셰이크 테스트

중요한 발견: PQC 적용 가능 여부는 운영체제가 아니라 사용하는 TLS 라이브러리에 의해 결정됩니다. 같은 Windows 환경이라도 SChannel을 사용하면 PQC를 적용할 수 없지만, OpenSSL 기반 라이브러리를 사용하면 가능합니다.

### loc-ros2-analysis/

다민 로봇팀과 협업하여 ROS2 기반 딥러닝 위치추정 시스템의 센서 데이터 처리 최적화를 진행했습니다.

구체적으로:
- IMU 센서에서 발생하는 타임스탬프 동기화 문제 분석
- enable_web 파라미터가 센서 데이터 품질에 미치는 영향 평가
- 50Hz 타임스탬프 안정성 측정
- 오프라인 평가 시 정확한 데이터셋 경로 검증

이 분석을 통해 ROS2 시스템에서 센서 데이터의 타임스탬프 일관성이 딥러닝 모델의 성능에 직접적인 영향을 미친다는 것을 파악했습니다.

---

## 기술 스택

| 영역 | 기술 |
|------|------|
| 암호화 | OpenSSL 3.5, ML-KEM (Post-Quantum Cryptography) |
| VPN/IPsec | strongSwan 6.0.6, IKEv2, IPsec (AES-256-SHA256) |
| 운영체제 | Ubuntu 22.04 (ARM64, x86_64) |
| TLS 라이브러리 | NSS, GnuTLS, wolfSSL, BoringSSL, Secure Transport, SChannel |
| 로봇 시스템 | ROS2, UERANSIM, Open5GS |
| 분석 도구 | Wireshark, tcpdump, iperf3 |

---

## 빠른 시작

```bash
# 1. 레포 클론
git clone https://github.com/pjy355/pqc-infrastructure-portfolio.git
cd pqc-infrastructure-portfolio

# 2. QuickStart 문서 확인
cd quickstart
cat README.md
```

최소 요구사항: Ubuntu 22.04(또는 유사 Debian 계열), 디스크 여유공간 5GB 이상, 약 2-3시간의 빌드 시간.

전체 프로세스:
1. OpenSSL 3.5 소스 다운로드 및 빌드
2. strongSwan 6.0.6 빌드 (RPATH 설정 포함)
3. PKI를 이용한 인증서 발급
4. swanctl.conf 작성 및 설정 로드
5. charon 데몬 실행
6. IKEv2 연결 검증

---

## 기술 검증

QuickStart 가이드는 완전히 초기화된 Ubuntu 22.04 환경에서 처음부터 끝까지 테스트되었습니다. 모든 단계가 정상 작동합니다.

검증 항목:
- OpenSSL 3.5 빌드 완료 (ML-KEM-512/768/1024 모두 로드됨)
- strongSwan의 openssl 플러그인이 정확한 라이브러리 버전 사용 (ldd 확인)
- 3개의 인증서(Root CA, Server, Client)가 정상 생성됨
- swanctl.conf가 정상 로드되고 연결 설정이 활성화됨
- charon 데몬이 안정적으로 실행되고 vici 소켓 생성됨
- 모든 인증서와 키가 정상 로드됨

---

## 주요 성과

### OpenSSL 버전 비교를 통한 실무 기술 선택

OpenSSL 3.5, 3.6, 4.0을 모두 구축해본 결과 다음과 같은 구체적인 차이를 발견했습니다.

OpenSSL 3.5는 liboqs 같은 외부 라이브러리 없이도 ML-KEM을 완벽하게 지원합니다. oqs-provider를 별도로 설치할 필요가 없어 빌드 시간이 줄고 의존성도 단순해지며, 빌드 과정 전반이 가장 안정적이어서 프로덕션 환경에 적합하다고 판단했습니다.

OpenSSL 3.6에서는 LMS 알고리즘이 검증(verification) 전용으로 제한되어 있어, 인증서 생성이나 서명이 필요한 작업에는 사용할 수 없었고 일부 암호화 작업에서 `unrecognized algorithm` 오류가 발생했습니다.

OpenSSL 4.0은 SSLv3 등 레거시 API가 완전히 제거되어 기존 코드의 마이그레이션이 필요했고, ENGINE API 제거로 커스텀 엔진 통합이 불가능해졌습니다. 다만 ECH(Encrypted Client Hello)가 추가되어 향후 TLS 스택의 방향성을 보여주는 버전이기도 했습니다.

이러한 비교를 통해 OpenSSL 3.5를 기준으로 삼기로 결정했고, 이는 이후 회사 서버 구축의 근거가 되었습니다.

### 회사 서버에 PQC + strongSwan 파이프라인 구성

OpenSSL 3.5를 선택한 이후, 실제 회사 서버에 PQC 기반 VPN 인프라를 구축했습니다.

Linux 서버에 OpenSSL 3.5를 `/usr/local/openssl35`에 설치하고, strongSwan 6.0.6을 RPATH 설정을 통해 OpenSSL 3.5와 완전하게 통합하여 IKEv2 VPN 서버로 운영했습니다. Root CA, 서버, 클라이언트 인증서로 이어지는 3계층 PKI 구조를 구성했고, 모든 VPN 통신은 ML-KEM-768과 기존 타원곡선 암호화(ECP256)를 조합한 하이브리드 암호화 그룹으로 보호되도록 설정했습니다. 이렇게 하면 현재의 암호화 위협에 대한 보호와 향후 양자 컴퓨터 위협에 대한 대비를 동시에 달성할 수 있습니다.

구축 이후에는 Wireshark로 실제 IKEv2 협상 과정을 패킷 레벨에서 분석하여 SA negotiation, child SA 생성, 트래픽 암호화가 정상적으로 이루어지는지 확인했고, Windows·macOS·Linux 클라이언트에서 연결을 테스트하면서 각 플랫폼의 TLS 라이브러리 차이로 인한 호환성 문제를 함께 파악했습니다.

이 과정에서 가장 까다로웠던 부분은 라이브러리 의존성 관리였습니다. RPATH를 설정해 시스템에 이미 깔려 있는 libcrypto와의 충돌을 막고, PKG_CONFIG_PATH를 정확히 지정해 빌드 시 의도한 버전의 OpenSSL이 잡히도록 했으며, LD_LIBRARY_PATH를 런타임에 맞게 조정해 strongSwan의 openssl 플러그인이 항상 정확한 버전의 라이브러리를 로드하도록 했습니다. 단순히 소프트웨어를 설치하는 것을 넘어, 복잡한 의존성을 관리하고 프로덕션 환경에 맞게 최적화하는 실무 경험을 쌓을 수 있었습니다.

---

## 프로젝트에서 배운 것

**기술적 역량**
- Post-Quantum Cryptography의 실제 구현과 실무 적용 방법
- 여러 암호화 라이브러리 버전의 비교 평가 기준
- RPATH를 통한 고급 라이브러리 의존성 관리
- 크로스 플랫폼 TLS 라이브러리의 호환성 검증
- Wireshark를 이용한 VPN 프로토콜 패킷 분석
- 복잡한 오픈소스 소프트웨어의 빌드 및 통합 방법

**엔지니어링 역량**
- 단순한 설치를 넘어 프로덕션 레벨의 최적화
- 오류 발생 시 근본 원인 분석과 장기적 해결책 모색
- 기술 선택 과정에서의 체계적인 비교 평가
- 완벽한 기술 문서 작성 방법론

**협업 경험**
- 다민 로봇팀과의 기술 협업
- 다양한 배경의 개발자들과의 커뮤니케이션
- 기술적 문제를 비전문가에게 설명하는 능력

---

## 프로젝트 정보

작성자: 박재용 (TRENTO Systems)
진행 기간: 2026년 3월 ~ 6월
문서 작성: 2026-06-16

라이선스: Internal Documentation
