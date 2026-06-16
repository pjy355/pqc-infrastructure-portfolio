# TLS/mTLS 다중 플랫폼 PQC 검증

회사 서버(리눅스)를 고정 클라이언트로 두고, macOS, Windows, Linux 클라이언트가 각각 어떤 TLS 라이브러리를 쓰는지에 따라 PQC 적용 가능 여부가 달라지는지 확인한 검증이다.

---

## 배경

OpenSSL 3.5로 PQC 환경을 구축했다고 해도, 실제로 클라이언트가 macOS나 Windows라면 그 운영체제가 기본으로 쓰는 TLS 스택이 OpenSSL이 아닐 수 있다. 그래서 같은 서버에 여러 플랫폼의 클라이언트를 연결해보면서, 어떤 조합에서 PQC 하이브리드 키 교환이 성립하는지를 직접 확인했다.

대상으로 삼은 라이브러리는 NSS, GnuTLS, wolfSSL, BoringSSL, Secure Transport, SChannel 여섯 가지다.

---

## 라이브러리별 확인 내용

**NSS (Mozilla/Firefox 계열)**
ML-KEM 지원이 추가되고 있는 라이브러리지만, 버전에 따라 차이가 있다. OpenSSL 기반 서버와의 mTLS에서 하이브리드 그룹 협상이 항상 매끄럽게 되는 것은 아니었다.

**GnuTLS (GNU/Linux 계열)**
리눅스 환경에서는 비교적 PQC 지원이 빠르게 들어오는 편이다. OpenSSL 서버와의 연동에서 큰 문제는 없었다.

**wolfSSL (IoT/임베디드)**
경량 라이브러리임에도 PQC 지원에 적극적이다. 다만 임베디드 타깃 빌드 옵션에 따라 ML-KEM이 빠지는 경우가 있어, 빌드 설정을 확인하는 과정이 필요했다.

**BoringSSL (Google/gRPC 계열)**
Google 인프라에서 PQC 도입이 비교적 앞서 있는 만큼, 하이브리드 키 교환 자체는 잘 동작했다. 다만 OpenSSL과는 별개의 fork이기 때문에 옵션 이름이나 동작 방식에 미묘한 차이가 있었다.

**Secure Transport (macOS/iOS)**
macOS 클라이언트가 기본으로 쓰는 스택이라 의미가 컸는데, 결과적으로 여기서는 PQC를 적용할 수 없었다. OS 레벨 TLS 스택이 ML-KEM을 지원하지 않기 때문이고, macOS에서 PQC 통신을 하려면 OpenSSL 기반 클라이언트를 별도로 띄워야 한다.

**SChannel (Windows)**
Windows의 기본 TLS 스택도 마찬가지였다. 같은 Windows 환경이라도 SChannel을 그대로 쓰면 PQC가 적용되지 않고, OpenSSL 기반 라이브러리(혹은 OpenSSL을 사용하는 별도 클라이언트)를 쓰는 경우에만 하이브리드 키 교환이 성립했다.

---

## 핵심 발견

PQC 적용 가능 여부는 운영체제가 아니라 클라이언트가 실제로 사용하는 TLS 라이브러리에 의해 결정된다. macOS냐 Windows냐가 기준이 아니라, 그 위에서 Secure Transport/SChannel 같은 OS 기본 스택을 쓰는지, 아니면 OpenSSL 계열 라이브러리를 쓰는지가 기준이다. 즉 같은 Windows 머신이라도 애플리케이션이 SChannel을 호출하면 PQC가 적용되지 않고, OpenSSL을 직접 링크해서 쓰는 애플리케이션이라면 적용된다.

이 발견은 회사의 다중 플랫폼 클라이언트 환경(macOS, Windows, Linux)에서 PQC를 도입할 때, OS별로 별도의 전략이 아니라 "어떤 TLS 라이브러리를 쓰는 애플리케이션인가"를 기준으로 적용 범위를 판단해야 한다는 결론으로 이어졌다.

---

## 비교 요약

| 라이브러리 | 플랫폼 | ML-KEM 지원 | 비고 |
|-----------|--------|------------|------|
| NSS | Firefox 계열 | 지원 (버전에 따라 차이) | 하이브리드 협상이 항상 안정적이지는 않음 |
| GnuTLS | Linux | 지원 | OpenSSL 서버와 연동 무난 |
| wolfSSL | IoT/임베디드 | 지원 (빌드 옵션에 따라 다름) | 임베디드 타깃 빌드 시 확인 필요 |
| BoringSSL | Google/gRPC | 지원 | OpenSSL과 옵션 명명 차이 있음 |
| Secure Transport | macOS/iOS | 미지원 | OS 기본 스택, PQC 불가 |
| SChannel | Windows | 미지원 | OS 기본 스택, PQC 불가 |

---

## 결론

서버 측 OpenSSL 3.5 + strongSwan 구성은 그대로 유지하되, 클라이언트가 OS 기본 TLS 스택(Secure Transport, SChannel)을 사용하는 경우에는 PQC 하이브리드 키 교환이 적용되지 않는다는 점을 명확히 했다. 따라서 다중 플랫폼 환경에서 PQC를 실제로 적용하려면 클라이언트 애플리케이션이 OpenSSL 계열 라이브러리를 사용하도록 별도로 구성해야 한다.
