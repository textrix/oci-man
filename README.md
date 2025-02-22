# oci-man

이 프로젝트는 WireGuard 설정 파일을 생성하고 Cloudflare DDNS를 관리하며, OCI 인스턴스를 관리하는 스크립트들을 포함하고 있습니다.

## 프로젝트 목적

이 스크립트들을 개발한 이유는 다음과 같습니다:

1. Oracle Cloud의 여러 계정의 인스턴스를 관리하는 것이 웹 인터페이스를 통해서는 번거로웠습니다.
2. CLI를 통해 인스턴스 목록과 IP 주소를 쉽게 얻고자 했습니다.
3. 얻은 IP 주소를 DNS에 등록하거나 업데이트하는 작업이 필요했습니다.
4. 인스턴스 간의 내부 연결을 WireGuard를 통해 구성하고자 했습니다.
5. Ansible을 사용하여 인스턴스 초기화 과정과 `wg0.conf` 파일을 배포하고자 했습니다.

## 준비 사항

- dig, jq
- oci CLI 및 oci용 설정파일 (~/.oci/config)

## 파일 설명

### `.env`

환경 변수 파일로, 다음과 같은 변수를 포함하고 있습니다:

- `DOMAIN`: 도메인 이름
- `API_TOKEN`: Cloudflare API 토큰
- `ZONE_ID`: Cloudflare Zone ID
- `WG_PORT`: WireGuard 포트

### `.wg-list.txt`

WireGuard 피어 리스트 파일로, 각 피어의 호스트 이름, IP 주소, 개인 키 및 공개 키를 포함하고 있습니다.

### `cf-ddns`

Cloudflare DDNS 업데이트 스크립트로, 다음과 같은 기능을 제공합니다:

- `.env` 파일을 로드하여 환경 변수를 설정합니다.
- 입력 인자를 검증합니다.
- Cloudflare API를 사용하여 DNS 레코드를 생성하거나 업데이트합니다.

사용법:

```sh
# 사용법
./cf-ddns <record_type (A/AAAA)> <host_name> <public_ip>

# A 레코드 업데이트
./cf-ddns A example 192.0.2.1

# AAAA 레코드 업데이트
./cf-ddns AAAA example 2001:db8::1

# wg ip update
cat .wg-list.txt | awk '{print "A", "wg."$1, $2}' | xargs -n3 bash -c './cf-ddns "$1" "$2" "$3"' _

# ip update
./oci-man . ip-list | xargs -P4 -n3 bash -c './cf-ddns "$1" "$2" "$3"' _

# ipv6 update
./oci-man . ipv6-list | xargs -P4 -n3 bash -c './cf-ddns "$1" "$2" "$3"' _
```

### mk-wg-conf

WireGuard 설정 파일 생성 스크립트로, 다음과 같은 기능을 제공합니다:

- .env 파일을 로드하여 환경 변수를 설정합니다.
- 입력 인자를 검증합니다.
- .wg-list.txt 파일에서 호스트 이름에 해당하는 정보를 찾아 WireGuard 설정 파일을 생성합니다.

```
# 사용법:
./mk-wg-conf <hostname>

# 호스트 이름이 example인 피어의 WireGuard 설정 파일 생성
./mk-wg-conf example

./oci-man . instance-list | awk -F, '{print $2}' | xargs -P8 -I % sh -c "./mk-wg-conf '%' > wg0.conf/'%'.conf"

cat .wg-list.txt | awk '{print $1}' | xargs -P8 -I % sh -c "./mk-wg-conf '%' > wg0.conf/'%'.conf"
```

### oci-man

OCI 인스턴스 관리 스크립트로, 다음과 같은 기능을 제공합니다:

- 명령어 목록
  - instance-list: OCI 인스턴스 리스트를 가져옵니다.
  - ip-list: 인스턴스의 IP 주소 리스트를 가져옵니다.
  - ipv6-list: 인스턴스의 IPv6 주소 리스트를 가져옵니다.
  - change-name: 인스턴스의 이름을 변경합니다.

```
# 사용법:
./oci-man . <command> <arg ...>

./oci-man . instance-list

./oci-man PROFILE_NAME ip-list
```

### ansible
```
eval $(ssh-agent -s)
ssh-add <ssh_private_key>
ansible-playbook -i hosts deplay_wg0.yml 
```
