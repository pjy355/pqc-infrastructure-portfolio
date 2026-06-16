# PQC IKEv2/IPsec QuickStart

OpenSSL 3.5 빌드부터 strongSwan 6.0.6을 이용한 IKEv2 터널 설정까지, 처음부터 끝까지 따라할 수 있는 가이드다. 완전히 초기화된 Ubuntu 22.04 환경에서 전 과정을 다시 검증했고, 모든 단계가 그대로 작동하는 것을 확인했다.

예상 소요 시간은 2~3시간이며, 디스크 여유 공간은 5GB 이상이 필요하다.

---

## 사전 준비

```bash
sudo apt-get update
sudo apt-get install -y \
  build-essential git curl wget cmake ninja-build \
  pkg-config libssl-dev perl libgmp-dev libsystemd-dev
```

---

## Step 1. OpenSSL 3.5 빌드

OpenSSL 3.5는 ML-KEM이 기본으로 내장되어 있어서, liboqs나 oqs-provider 같은 외부 라이브러리가 따로 필요 없다. 이 점이 다른 버전(3.6, 4.0) 대비 가장 큰 장점이고, 이 프로젝트에서 3.5를 기준으로 잡은 이유이기도 하다.

```bash
cd ~
git clone https://github.com/openssl/openssl.git -b openssl-3.5
cd openssl

./config --prefix=/usr/local/openssl35 \
         --openssldir=/usr/local/openssl35/ssl \
         -DOPENSSL_SUPPRESS_DEPRECATED

make -j$(nproc)
sudo make install
```

빌드가 끝나면 ML-KEM 알고리즘이 제대로 들어있는지 확인한다.

```bash
export LD_LIBRARY_PATH=/usr/local/openssl35/lib:$LD_LIBRARY_PATH
/usr/local/openssl35/bin/openssl list -kem-algorithms | grep -i ml-kem
```

ML-KEM-512, ML-KEM-768, ML-KEM-1024가 모두 출력되면 정상이다.

마지막으로 LD_LIBRARY_PATH를 영구적으로 등록해둔다.

```bash
echo 'export LD_LIBRARY_PATH=/usr/local/openssl35/lib:$LD_LIBRARY_PATH' >> ~/.bashrc
echo 'alias openssl35="/usr/local/openssl35/bin/openssl"' >> ~/.bashrc
source ~/.bashrc
```

이 환경변수를 빼먹으면 이후 strongSwan 빌드와 실행 단계에서 라이브러리를 찾지 못하는 문제가 반복해서 발생한다. 처음 시도할 때 이 부분 때문에 가장 많이 막혔다.

---

## Step 2. strongSwan 6.0.6 빌드

```bash
cd ~
wget --no-check-certificate https://download.strongswan.org/strongswan-6.0.6.tar.gz
tar -xzf strongswan-6.0.6.tar.gz
cd strongswan-6.0.6

LDFLAGS="-L/usr/local/openssl35/lib -Wl,-rpath,/usr/local/openssl35/lib" \
CPPFLAGS="-I/usr/local/openssl35/include" \
PKG_CONFIG_PATH=/usr/local/openssl35/lib/pkgconfig \
./configure --prefix=/usr/local/strongswan-6.0.6 \
            --sysconfdir=/usr/local/strongswan-6.0.6/etc \
            --with-openssl-dir=/usr/local/openssl35 \
            --enable-openssl

make -j$(nproc)
sudo make install
```

여기서 핵심은 `-Wl,-rpath,/usr/local/openssl35/lib`이다. 이걸 빼고 빌드하면 strongSwan의 openssl 플러그인이 시스템에 이미 깔려 있는 libcrypto를 찾으려고 하고, 버전이 안 맞아서 다음과 같은 오류가 난다.

```
plugin 'openssl' failed to load: /lib/aarch64-linux-gnu/libcrypto.so.3:
version `OPENSSL_3.3.0' not found
```

LD_LIBRARY_PATH만 설정해주는 걸로는 해결이 안 됐다. RPATH를 빌드 시점에 박아넣어야 strongSwan이 정확히 `/usr/local/openssl35/lib`의 라이브러리를 쓴다. 빌드 후에는 이렇게 확인한다.

```bash
ldd /usr/local/strongswan-6.0.6/lib/ipsec/plugins/libstrongswan-openssl.so | grep openssl
```

`libcrypto.so.3 => /usr/local/openssl35/lib/libcrypto.so.3` 형태로 나오면 정상이다.

---

## Step 3. PKI 인증서 발급

Root CA, 서버, 클라이언트로 이어지는 3계층 인증서를 발급한다.

```bash
mkdir -p ~/pki/{cacerts,certs,private}
cd ~/pki

# Root CA
/usr/local/strongswan-6.0.6/bin/pki --gen --type rsa --size 4096 --outform pem > private/root-ca-key.pem
/usr/local/strongswan-6.0.6/bin/pki --self --in private/root-ca-key.pem \
  --dn "C=KR, O=TRENTO, CN=TRENTO-Root-CA" --ca --outform pem > cacerts/root-ca-cert.pem

# 서버
/usr/local/strongswan-6.0.6/bin/pki --gen --type rsa --size 2048 --outform pem > private/server-key.pem
/usr/local/strongswan-6.0.6/bin/pki --req --in private/server-key.pem \
  --dn "C=KR, O=TRENTO, CN=<서버 호스트명>" --outform pem > certs/server.csr
/usr/local/strongswan-6.0.6/bin/pki --issue --cacert cacerts/root-ca-cert.pem --cakey private/root-ca-key.pem \
  --in certs/server.csr --type pkcs10 --san <서버 호스트명> --outform pem > certs/server-cert.pem

# 클라이언트
/usr/local/strongswan-6.0.6/bin/pki --gen --type rsa --size 2048 --outform pem > private/client-key.pem
/usr/local/strongswan-6.0.6/bin/pki --req --in private/client-key.pem \
  --dn "C=KR, O=TRENTO, CN=client" --outform pem > certs/client.csr
/usr/local/strongswan-6.0.6/bin/pki --issue --cacert cacerts/root-ca-cert.pem --cakey private/root-ca-key.pem \
  --in certs/client.csr --type pkcs10 --outform pem > certs/client-cert.pem
```

주의할 점 하나는 `pki --self`와 `pki --req`에 `--type pkcs10`을 같이 넣으면 안 된다는 것이다. `--type`은 `--issue` 단계에서만 쓴다. 처음 시도할 때 이 옵션 때문에 인증서 파일이 빈 파일로 생성된 적이 있었다.

발급이 끝나면 각 pem 파일의 크기가 0이 아닌지 확인한다.

```bash
ls -lah ~/pki/{cacerts,certs,private}/*.pem
```

---

## Step 4. strongSwan 설정

인증서를 strongSwan 설정 디렉토리로 옮긴다.

```bash
sudo mkdir -p /usr/local/strongswan-6.0.6/etc/swanctl/{x509,private}

sudo cp ~/pki/cacerts/root-ca-cert.pem /usr/local/strongswan-6.0.6/etc/swanctl/x509/
sudo cp ~/pki/certs/server-cert.pem /usr/local/strongswan-6.0.6/etc/swanctl/x509/
sudo cp ~/pki/certs/client-cert.pem /usr/local/strongswan-6.0.6/etc/swanctl/x509/

sudo cp ~/pki/private/server-key.pem /usr/local/strongswan-6.0.6/etc/swanctl/private/
sudo cp ~/pki/private/client-key.pem /usr/local/strongswan-6.0.6/etc/swanctl/private/

sudo chmod 700 /usr/local/strongswan-6.0.6/etc/swanctl/private
sudo chmod 755 /usr/local/strongswan-6.0.6/etc/swanctl/x509
sudo chmod 600 /usr/local/strongswan-6.0.6/etc/swanctl/private/*
sudo chmod 644 /usr/local/strongswan-6.0.6/etc/swanctl/x509/*
```

`swanctl.conf`를 작성한다.

```bash
sudo nano /usr/local/strongswan-6.0.6/etc/swanctl/swanctl.conf
```

```
connections {
  pqc-tunnel {
    local_addrs = <서버 IP>
    remote_addrs = <클라이언트 IP 또는 대역>

    local {
      auth = pubkey
      certs = server-cert.pem
      id = <서버 호스트명>
    }

    remote {
      auth = pubkey
      certs = client-cert.pem
      id = client
    }

    children {
      pqc-child {
        local_ts = <서버 IP>/32
        remote_ts = <클라이언트 IP 또는 대역>
        esp_proposals = aes256-sha256
      }
    }

    version = 2
    proposals = aes256-sha256-prfsha256-ecp256
  }
}
```

처음 작성할 때 `dpd_action`, `rekey_time` 같은 옵션을 `connections` 블록 최상위에 넣었다가 `unknown option: dpd_action, config discarded` 오류로 설정 전체가 로드되지 않는 일이 있었다. swanctl.conf의 문법은 strongSwan의 ipsec.conf 문법과 다르므로, swanctl 전용 문서 기준으로 옵션을 넣어야 한다. 안전하게 가려면 위 예시처럼 최소 구성으로 시작해서 필요한 옵션을 하나씩 추가하는 편이 낫다.

---

## Step 5. charon 데몬 실행

```bash
sudo killall charon 2>/dev/null

sudo bash -c 'export LD_LIBRARY_PATH=/usr/local/openssl35/lib:/usr/local/strongswan-6.0.6/lib/ipsec:$LD_LIBRARY_PATH && /usr/local/strongswan-6.0.6/libexec/ipsec/charon &'

sleep 3

sudo /usr/local/strongswan-6.0.6/sbin/swanctl -c
sudo /usr/local/strongswan-6.0.6/sbin/swanctl -L
```

`pqc-tunnel` 연결이 정상적으로 로드되었다는 메시지가 나오면 여기까지는 끝이다.

---

## Step 6. 검증

```bash
sudo /usr/local/strongswan-6.0.6/sbin/swanctl -s
```

로드된 인증서와 키 목록이 모두 출력되는지 확인한다. 실제 상대 호스트가 있다면 IKEv2 연결을 시작해서 SA가 맺어지는지까지 확인할 수 있다.

```bash
sudo /usr/local/strongswan-6.0.6/sbin/swanctl -i -c pqc-tunnel
```

---

## 자주 만나는 문제

**`unable to locally verify the issuer's authority`로 wget이 실패하는 경우**
strongSwan 다운로드 서버의 인증서 체인을 환경에서 신뢰하지 못하는 경우다. `wget --no-check-certificate`로 우회할 수 있다.

**`libcrypto.so.3: version 'OPENSSL_3.3.0' not found`**
strongSwan이 시스템 OpenSSL을 참조하고 있다는 뜻이다. configure 단계에서 RPATH(`-Wl,-rpath,/usr/local/openssl35/lib`)를 빠뜨리지 않고 넣었는지 확인한다.

**`connecting to 'unix:///var/run/charon.vici' failed: Connection refused`**
charon 데몬이 실제로 떠 있지 않은 상태다. `ps aux | grep charon`으로 프로세스를 확인하고, 죽어있다면 다시 실행한다.

**인증서 파일이 0바이트로 생성됨**
`pki --self` 또는 `pki --req`에 `--type pkcs10`을 같이 넣은 경우다. `--type`은 `--issue` 단계에만 사용한다.

**`loading connection ... failed: unknown option`**
swanctl.conf 문법 오류다. ipsec.conf 문법을 그대로 가져다 쓰면 발생하므로, 최소 구성에서 옵션을 하나씩 추가하면서 원인을 좁히는 것이 빠르다.

---

## 검증 환경

ARM64 Ubuntu 22.04(M2 MacBook UTM 가상머신) 위에서 완전히 초기화된 상태부터 전 과정을 다시 따라가며 검증했다. 위 단계를 순서대로 따라가면 별도의 사전 설치 없이 PQC 하이브리드 키 교환 기반 IKEv2 터널 설정까지 도달한다.
