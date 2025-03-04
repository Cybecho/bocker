> 기존 Bocker 프로젝트를 단순히 번역하고, 설명을 추가한 레파토지입니다. 설명이 틀렸을 가능성이 존재하니, 이점 감안하고 참고만 해주시면 감사하겠습니다.
# Bocker
약 100줄의 배쉬로 구현된 Docker.

- [Bocker](#bocker)
  - [전제 조건](#전제-조건)
  - [사용 예시](#사용-예시)
  - [기능: 현재 구현됨](#기능-현재-구현됨)
  - [기능을 매우 제한적으로 구현되어있습니다: 아직 구현되지 않음](#기능을-매우-제한적으로-구현되어있습니다-아직-구현되지-않음)
  - [Bocker의 핵심 컨테이너 기술 동작 원리](#bocker의-핵심-컨테이너-기술-동작-원리)
      - [컨테이너 격리 구현 방식](#컨테이너-격리-구현-방식)
      - [컨테이너 라이프사이클 관리](#컨테이너-라이프사이클-관리)
      - [그 외 구현](#그-외-구현)
  - [각 함수 설명](#각-함수-설명)
      - [초기 설정과 기본 구조](#초기-설정과-기본-구조)
    - [기타 유틸리티 함수들](#기타-유틸리티-함수들)
      - [전체 워크플로우 요약](#전체-워크플로우-요약)
  - [License](#license)

## 전제 조건

bocker를 실행하려면 다음 패키지가 필요합니다.

* btrfs-progs
* curl
* iproute2
* iptables
* libcgroup-tools
* util-linux >= 2.25.2
* coreutils >= 7.5

대부분의 배포판에는 충분히 새로운 버전의 유틸리티 리눅스가 제공되지 않으므로 [여기](https://www.kernel.org/pub/linux/utils/util-linux/v2.25/)에서 소스를 가져와 직접 컴파일해야 할 수도 있습니다.

또한 시스템을 다음과 같이 구성해야 합니다:

* 아래에 마운트된 btrfs 파일 시스템 `/var/bocker`
* 'bridge0'이라는 네트워크 브리지 및 10.0.0.1/24의 IP.
* IP 포워딩이 `/proc/sys/net/ipv4/ip_forward`에서 활성화된 경우
* 'bridge0'에서 물리적 인터페이스로 트래픽을 라우팅하는 방화벽.

사용 편의성을 위해 필요한 환경을 구축할 수 있는 Vagrant파일이 포함되어 있습니다.

위의 전제 조건을 충족하더라도 가상 머신에서 **보커를 실행**하고 싶을 수도 있습니다. Bocker는 루트로 실행되며 무엇보다도 네트워크 인터페이스, 라우팅 테이블 및 방화벽 규칙을 변경해야 합니다. **시스템을 엉망으로 만들지 않는다는 보장은 없습니다**.

## 사용 예시

```
$ bocker pull centos 7
######################################################################## 100.0%
######################################################################## 100.0%
######################################################################## 100.0%
Created: img_42150

$ bocker images
IMAGE_ID        SOURCE
img_42150       centos:7

$ bocker run img_42150 cat /etc/centos-release
CentOS Linux release 7.1.1503 (Core)

$ bocker ps
CONTAINER_ID       COMMAND
ps_42045           cat /etc/centos-release

$ bocker logs ps_42045
CentOS Linux release 7.1.1503 (Core)

$ bocker rm ps_42045
Removed: ps_42045

$ bocker run img_42150 which wget
which: no wget in (/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin)

$ bocker run img_42150 yum install -y wget
Installing : wget-1.14-10.el7_0.1.x86_64                                  1/1
Verifying  : wget-1.14-10.el7_0.1.x86_64                                  1/1
Installed  : wget.x86_64 0:1.14-10.el7_0.1
Complete!

$ bocker ps
CONTAINER_ID       COMMAND
ps_42018           yum install -y wget
ps_42182           which wget

$ bocker commit ps_42018 img_42150
Removed: img_42150
Created: img_42150

$ bocker run img_42150 which wget
/usr/bin/wget

$ bocker run img_42150 cat /proc/1/cgroup
...
4:memory:/ps_42152
3:cpuacct,cpu:/ps_42152

$ cat /sys/fs/cgroup/cpu/ps_42152/cpu.shares
512

$ cat /sys/fs/cgroup/memory/ps_42152/memory.limit_in_bytes
512000000

$ BOCKER_CPU_SHARE=1024 \
	BOCKER_MEM_LIMIT=1024 \
	bocker run img_42150 cat /proc/1/cgroup
...
4:memory:/ps_42188
3:cpuacct,cpu:/ps_42188

$ cat /sys/fs/cgroup/cpu/ps_42188/cpu.shares
1024

$ cat /sys/fs/cgroup/memory/ps_42188/memory.limit_in_bytes
1024000000
```

## 기능: 현재 구현됨

* `docker build` †
* `docker pull`
* `docker images`
* `docker ps`
* `docker run`
* `docker exec`
* `docker logs`
* `docker commit`
* `docker rm` / `docker rmi`
* Networking
* Quota Support / CGroups

'bocker init'은 매우 제한적인 `docker build` 구현을 제공합니다.

## 기능을 매우 제한적으로 구현되어있습니다: 아직 구현되지 않음

* 데이터 볼륨 컨테이너
* 데이터 볼륨
* 포트 포워딩

## Bocker의 핵심 컨테이너 기술 동작 원리

#### 컨테이너 격리 구현 방식

Bocker는 리눅스 커널의 여러 격리 기술을 조합해 경량 컨테이너를 구현합니다. 각 기술의 연결 방식을 살펴보겠습니다.

> 1. 파일시스템 격리

```bash
# 이미지 기반 컨테이너 생성
btrfs subvolume snapshot "$btrfs_path/$1" "$btrfs_path/$uuid" > /dev/null

# 파일시스템 격리
chroot "$btrfs_path/$uuid" /bin/sh -c "..."
```

Bocker는 **Btrfs의 스냅샷 기능**을 활용해 이미지와 컨테이너를 관리합니다. 이는 Docker가 초기에 AUFS와 같은 계층 파일시스템을 사용한 것과 유사합니다. Btrfs 스냅샷은:

1. **Copy-on-Write**: 변경이 발생할 때만 새 데이터 블록을 생성하므로 저장공간이 효율적입니다
2. **빠른 생성**: 메타데이터만 복사하므로 대용량 이미지도 즉시 복제됩니다
3. **격리**: 각 컨테이너는 독립적인 파일시스템 뷰를 가집니다

`chroot`는 프로세스의 루트 디렉토리를 변경해 컨테이너가 호스트 파일시스템에 접근하지 못하게 합니다.

> 2. 네트워크 격리

```bash
# 네트워크 네임스페이스 생성
ip netns add netns_"$uuid"

# 가상 이더넷 인터페이스 생성 및 연결
ip link add dev veth0_"$uuid" type veth peer name veth1_"$uuid"
ip link set veth1_"$uuid" netns netns_"$uuid"

# 네트워크 격리 내에서 명령 실행
ip netns exec netns_"$uuid" ... 명령어 실행 ...
```

네트워크 격리는 **네트워크 네임스페이스**와 **가상 이더넷(veth) 쌍**을 이용합니다:

1. 각 컨테이너는 전용 네트워크 네임스페이스를 가집니다
2. veth 인터페이스 쌍은 컨테이너와 호스트 사이의 통신 터널 역할을 합니다
3. 호스트의 `bridge0` 브릿지는 모든 컨테이너를 연결하는 가상 스위치처럼 작동합니다
4. 각 컨테이너는 `10.0.0.x` 형태의 내부 IP를 할당받습니다

이 구성은 Docker의 기본 `bridge` 네트워크 모드와 매우 유사합니다.

> 3. 프로세스 격리

```bash
# 여러 격리 기술 조합하여 컨테이너 실행
unshare -fmuip --mount-proc \
    chroot "$btrfs_path/$uuid" \
    /bin/sh -c "..."
```

`unshare` 명령은 프로세스를 여러 네임스페이스로부터 격리합니다:
- `-f`: 새 fork 네임스페이스 (PID 분리)
- `-m`: 새 마운트 네임스페이스 (마운트 지점 분리)
- `-u`: 새 UTS 네임스페이스 (호스트명 분리)
- `-i`: 새 IPC 네임스페이스 (IPC 객체 분리)
- `-p`: 새 PID 네임스페이스 (프로세스 ID 분리)
- `--mount-proc`: 새 /proc 마운트 (프로세스 정보 분리)

이런 네임스페이스 조합으로 컨테이너 프로세스는 호스트 시스템의 다른 프로세스를 볼 수 없고, 자신이 시스템의 유일한 프로세스인 것처럼 작동합니다.

> 4. 자원 제한

```bash
# cgroup 설정
cgcreate -g "$cgroups:/$uuid"
cgset -r cpu.shares="$BOCKER_CPU_SHARE" "$uuid"
cgset -r memory.limit_in_bytes="$((BOCKER_MEM_LIMIT * 1000000))" "$uuid"

# cgroup 내에서 실행
cgexec -g "$cgroups:$uuid" 명령어
```

**cgroups(컨트롤 그룹)**은 컨테이너의 리소스 사용량을 제한합니다:

1. CPU 할당량은 `cpu.shares`로 제어됩니다 (기본값: 512)
2. 메모리 사용량은 `memory.limit_in_bytes`로 제한됩니다 (기본값: 512MB)
3. 사용자는 환경변수(`BOCKER_CPU_SHARE`, `BOCKER_MEM_LIMIT`)로 이 값을 조정할 수 있습니다

#### 컨테이너 라이프사이클 관리

> 1. 이미지 → 컨테이너 전환

```bash
# 이미지 서브볼륨을 컨테이너로 변환
btrfs subvolume snapshot "$btrfs_path/$1" "$btrfs_path/$uuid" > /dev/null
echo "$cmd" > "$btrfs_path/$uuid/$uuid.cmd"  # 명령어 기록
```

컨테이너는 이미지의 스냅샷으로 생성되어 이미지를 변경하지 않으면서 독립적으로 수정할 수 있습니다. 이 접근법은 Docker의 이미지 레이어 시스템과 컨테이너 레이어 개념과 유사합니다.

> 2. 컨테이너 → 이미지 전환 (commit)

```bash
# 컨테이너를 이미지로 변환
bocker_rm "$2" && btrfs subvolume snapshot "$btrfs_path/$1" "$btrfs_path/$2" > /dev/null
```

`bocker_commit`은 컨테이너의 현재 상태를 새 이미지로 저장합니다. 이는 변경사항을 영구적으로 보존하는 방법으로, Docker의 `docker commit` 명령과 동일한 목적을 가집니다.

> 3. 컨테이너 내 명령 실행 (exec)

```bash
# 실행 중인 컨테이너 PID 찾기
cid="$(ps o ppid,pid | grep "^$(ps o pid,cmd | grep -E "^\ *[0-9]+ unshare.*$1" | awk '{print $1}')" | awk '{print $2}')"

# 해당 네임스페이스로 진입하여 명령 실행
nsenter -t "$cid" -m -u -i -n -p chroot "$btrfs_path/$1" "${@:2}"
```

`nsenter`는 기존 컨테이너의 네임스페이스에 진입하여 추가 명령을 실행합니다. 이 방식은:

1. 먼저 `ps` 명령으로 컨테이너의 메인 프로세스 PID를 찾습니다
2. 해당 PID의 모든 네임스페이스(마운트, UTS, IPC, 네트워크, PID)에 진입합니다
3. `chroot`로 컨테이너 파일시스템에서 명령을 실행합니다

이는 Docker의 `docker exec` 명령과 동일한 기능입니다.

#### 그 외 구현

> 재귀적 UUID 충돌 해결

```bash
[[ "$(bocker_check "$uuid")" == 0 ]] && echo "UUID conflict, retrying..." && bocker_run "$@" && return
```

UUID 충돌이 발생하면(매우 드문 경우), 함수는 재귀적으로 자신을 호출하여 새 UUID를 생성합니다. 이는 간결하지만 깊은 재귀가 발생할 위험이 있습니다.

> 파이프라인과 명령 체인

```bash
# 복잡한 파이프라인으로 Docker Hub API 토큰 추출
token="$(curl -sL -o /dev/null -D- -H 'X-Docker-Token: true' "https://index.docker.io/v1/repositories/$1/images" | tr -d '\r' | awk -F ': *' '$1 == "X-Docker-Token" { print $2 }')"
```

Bocker는 복잡한 동작을 구현하기 위해 여러 Unix 도구를 파이프라인으로 연결합니다. 이는 외부 라이브러리에 의존하지 않고 기본 시스템 도구만으로 기능을 구현하는 유닉스 철학을 따릅니다.

## 각 함수 설명

#### 초기 설정과 기본 구조

```bash
#!/usr/bin/env bash
set -o errexit -o nounset -o pipefail; shopt -s nullglob
btrfs_path='/var/bocker' && cgroups='cpu,cpuacct,memory';
```

- **설정 이유**: 안정적인 스크립트 실행과 오류 처리를 위한 설정입니다.
- **Btrfs와 cgroups**: 컨테이너 격리와 리소스 관리에 필요한 기술입니다.

> #### 1. `bocker_check()` - 컨테이너/이미지 존재 확인 함수

```bash
function bocker_check() {
    btrfs subvolume list "$btrfs_path" | grep -qw "$1" && echo 0 || echo 1
}
```
```
이 함수는 Bocker의 가장 기본적인 유틸리티 함수로, 
특정 ID를 가진 이미지나 컨테이너가 시스템에 존재하는지 확인합니다. 
Btrfs 파일시스템의 서브볼륨 목록을 가져와 우리가 찾는 ID가 있는지 grep으로 검색합니다. 
찾으면 0을, 찾지 못하면 1을 출력합니다.

이 함수는 모든 Bocker 명령어의 시작점으로, 작업을 진행하기 전에 항상 대상이 유효한지 검증합니다. 
예를 들어 존재하지 않는 이미지로 컨테이너를 실행하려 하거나, 없는 컨테이너를 삭제하려는 시도를 방지합니다. 
사용자에게 명확한 오류 메시지를 제공하기 위한 첫 번째 방어선입니다.
```

**내부 동작 과정:**
1. `btrfs subvolume list "$btrfs_path"`는 btrfs 파일시스템에서 모든 서브볼륨 목록을 가져옵니다
2. 파이프(`|`)를 통해 목록을 `grep -qw "$1"`으로 전달하여 매개변수와 정확히 일치하는 이름의 서브볼륨을 조용히 검색합니다
3. `&&` 및 `||` 연산자로 검색 결과에 따라 다른 값을 출력합니다:
   - 일치하는 항목이 발견되면 `echo 0` (성공을 의미)
   - 발견되지 않으면 `echo 1` (실패를 의미)

이 함수는 다른 모든 함수들의 기반이 되는 유틸리티 함수로, 작업 실행 전에 대상이 실제로 존재하는지 확인합니다.

> #### 2. `bocker_init()` - 디렉토리로부터 이미지 생성 함수

```bash
function bocker_init() {
    uuid="img_$(shuf -i 42002-42254 -n 1)"
    
    if [[ -d "$1" ]]; then
        [[ "$(bocker_check "$uuid")" == 0 ]] && bocker_run "$@"
        btrfs subvolume create "$btrfs_path/$uuid" > /dev/null
        cp -rf --reflink=auto "$1"/* "$btrfs_path/$uuid" > /dev/null
        [[ ! -f "$btrfs_path/$uuid"/img.source ]] && echo "$1" > "$btrfs_path/$uuid"/img.source
        echo "Created: $uuid"
    else
        echo "No directory named '$1' exists"
    fi
}
```
```
이 함수는 로컬 디렉토리를 Docker 이미지로 변환합니다. 
작업 흐름은 다음과 같습니다:

1. 먼저 "img_"로 시작하는 랜덤 ID를 생성합니다
2. 지정된 디렉토리가 실제로 존재하는지 확인합니다
3. 새로운 Btrfs 서브볼륨을 생성하고, Copy-on-Write 방식으로 디렉토리 내용을 복사합니다
4. 이미지 출처 정보를 기록합니다

이 함수는 로컬 개발 환경에서 직접 이미지를 만들거나, 
Docker Hub에서 다운로드한 이미지 레이어를 처리할 때 사용됩니다.
Copy-on-Write 기능을 활용하여 디스크 공간을 효율적으로 사용하면서도, 
각 이미지가 완전한 파일 시스템을 가질 수 있게 합니다.
```

**내부 동작 과정:**
1. 임의의 5자리 숫자(42002~42254)를 생성하고 "img_" 접두사와 함께 UUID로 설정합니다
2. `if [[ -d "$1" ]]`로 지정된 디렉토리가 존재하는지 확인합니다
3. `bocker_check "$uuid"`로 생성된 UUID가 이미 사용 중인지 검사합니다
   - 중복이면 `bocker_run "$@"`을 호출해 현재 인자들로 함수를 다시 실행합니다 (재귀)
4. `btrfs subvolume create`로 새 Btrfs 서브볼륨을 생성합니다
5. `cp -rf --reflink=auto`로 원본 디렉토리의 파일을 새 서브볼륨에 복사합니다
   - `--reflink=auto` 옵션은 가능한 경우 Copy-on-Write 기능을 활용해 디스크 공간을 절약합니다
6. 이미지 출처를 추적할 메타데이터 파일을 생성합니다
7. 성공 메시지와 함께 생성된 이미지 ID를 출력합니다

> #### 3. bocker_pull() - Docker Hub에서 이미지 가져오기 함수

```bash
function bocker_pull() {
    token="$(curl -sL -o /dev/null -D- -H 'X-Docker-Token: true' "https://index.docker.io/v1/repositories/$1/images" | tr -d '\r' | awk -F ': *' '$1 == "X-Docker-Token" { print $2 }')"
    registry='https://registry-1.docker.io/v1'
    id="$(curl -sL -H "Authorization: Token $token" "$registry/repositories/$1/tags/$2" | sed 's/"//g')"
    [[ "${#id}" -ne 64 ]] && echo "No image named '$1:$2' exists" && exit 1
    ancestry="$(curl -sL -H "Authorization: Token $token" "$registry/images/$id/ancestry)"
    IFS=',' && ancestry=(${ancestry//[\[\] \"]/}) && IFS=' \n\t'; tmp_uuid="$(uuidgen)" && mkdir /tmp/"$tmp_uuid"
    for id in "${ancestry[@]}"; do
        curl -#L -H "Authorization: Token $token" "$registry/images/$id/layer" -o /tmp/"$tmp_uuid"/layer.tar
        tar xf /tmp/"$tmp_uuid"/layer.tar -C /tmp/"$tmp_uuid" && rm /tmp/"$tmp_uuid"/layer.tar
    done
    echo "$1:$2" > /tmp/"$tmp_uuid"/img.source
    bocker_init /tmp/"$tmp_uuid" && rm -rf /tmp/"$tmp_uuid"
}
```
```
이 함수는 Docker Hub에서 이미지를 가져오는 작업을 수행합니다. 
Docker Registry API와 통신하여 다음과 같은 과정으로 이미지를 다운로드합니다:

1. Docker Hub API에 인증 요청을 보내 토큰을 받습니다
2. 받은 토큰으로 요청한 이미지:태그의 ID를 조회합니다
3. 이미지의 계층 구조를 파악하고 각 레이어를 순차적으로 다운로드합니다
4. 모든 레이어를 임시 디렉토리에 압축 해제한 후, 이를 하나의 이미지로 변환합니다

이 과정은 Docker가 내부적으로 이미지를 처리하는 방식을 간소화한 것입니다. 
레이어드 파일 시스템 대신 단일 Btrfs 볼륨을 사용하기 때문에 계층 구조를 평면화하여 처리합니다. 
이로 인해 Docker의 이미지 캐싱 이점은 잃지만, 간결한 구현이 가능해집니다.
```

**내부 동작 과정:**
1. Docker Registry API와 통신하기 위한 인증 토큰을 획득합니다:
   - `curl` 요청을 보내고 `-D-` 옵션으로 HTTP 헤더만 추출합니다
   - `tr -d '\r'`로 캐리지 리턴을 제거합니다
   - `awk`로 'X-Docker-Token' 헤더 값을 추출합니다
2. 레지스트리 기본 URL을 설정합니다
3. 두 번째 `curl` 요청으로 요청된 이미지:태그의 ID를 가져옵니다:
   - JSON 응답에서 따옴표를 제거하기 위해 `sed 's/"//g'`를 사용합니다
4. ID가 64자 SHA256 해시가 아니면 이미지가 존재하지 않는 것으로 간주하고 종료합니다
5. 세 번째 `curl` 요청으로 이미지의 계층 구조(ancestry) 정보를 가져옵니다
6. 복잡한 문자열 처리로 JSON 배열을 Bash 배열로 변환합니다:
   - `IFS=','`는 필드 구분자를 쉼표로 설정합니다
   - `${ancestry//[\[\] \"]/}`는 대괄호와 따옴표를 제거합니다
   - 처리 후 `IFS=' \n\t'`로 구분자를 원래대로 복원합니다
7. 임시 디렉토리를 생성합니다
8. 각 레이어를 순회하며:
   - `curl -#L`로 레이어 tarball을 다운로드합니다 (`-#`는 진행 상태 표시줄 제공)
   - `tar xf`로 압축을 해제하고 원본 tar 파일을 삭제합니다
9. 이미지 출처 정보를 기록합니다
10. 임시 디렉토리를 `bocker_init`에 전달하여 이미지로 변환하고, 작업 완료 후 임시 디렉토리를 삭제합니다

> #### 4. bocker_run() - 컨테이너 생성 및 실행 함수

```bash
function bocker_run() {
    uuid="ps_$(shuf -i 42002-42254 -n 1)"
    [[ "$(bocker_check "$1")" == 1 ]] && echo "No image named '$1' exists" && exit 1
    [[ "$(bocker_check "$uuid")" == 0 ]] && echo "UUID conflict, retrying..." && bocker_run "$@" && return
    cmd="${@:2}" && ip="$(echo "${uuid: -3}" | sed 's/0//g')" && mac="${uuid: -3:1}:${uuid: -2}"
    
    # 네트워크 설정
    ip link add dev veth0_"$uuid" type veth peer name veth1_"$uuid"
    ip link set dev veth0_"$uuid" up
    ip link set veth0_"$uuid" master bridge0
    ip netns add netns_"$uuid"
    ip link set veth1_"$uuid" netns netns_"$uuid"
    ip netns exec netns_"$uuid" ip link set dev lo up
    ip netns exec netns_"$uuid" ip link set veth1_"$uuid" address 02:42:ac:11:00"$mac"
    ip netns exec netns_"$uuid" ip addr add 10.0.0."$ip"/24 dev veth1_"$uuid"
    ip netns exec netns_"$uuid" ip link set dev veth1_"$uuid" up
    ip netns exec netns_"$uuid" ip route add default via 10.0.0.1
    
    # 파일시스템 설정
    btrfs subvolume snapshot "$btrfs_path/$1" "$btrfs_path/$uuid" > /dev/null
    echo 'nameserver 8.8.8.8' > "$btrfs_path/$uuid"/etc/resolv.conf
    echo "$cmd" > "$btrfs_path/$uuid/$uuid.cmd"
    
    # 리소스 제한 설정
    cgcreate -g "$cgroups:/$uuid"
    : "${BOCKER_CPU_SHARE:=512}" && cgset -r cpu.shares="$BOCKER_CPU_SHARE" "$uuid"
    : "${BOCKER_MEM_LIMIT:=512}" && cgset -r memory.limit_in_bytes="$((BOCKER_MEM_LIMIT * 1000000))" "$uuid"
    
    # 컨테이너 실행
    cgexec -g "$cgroups:$uuid" ip netns exec netns_"$uuid" unshare -fmuip --mount-proc chroot "$btrfs_path/$uuid" /bin/sh -c "/bin/mount -t proc proc /proc && $cmd" 2>&1 | tee "$btrfs_path/$uuid/$uuid.log" || true
    
    # 정리
    ip link del dev veth0_"$uuid"
    ip netns del netns_"$uuid"
}
```
```
사용자가 시스템에 있는 모든 이미지를 확인하고자 할 때, 
시스템은 모든 이미지 서브볼륨을 순회하며 각 이미지의 ID와 출처를 표시한다. 
이를 통해 사용자는 사용 가능한 이미지를 빠르게 파악할 수 있다.

한편, 사용자가 특정 이미지로부터 새로운 컨테이너를 실행하려고 할 때, `bocker_run` 함수가 이를 처리한다. 
이 함수는 우선 고유한 컨테이너 ID를 생성한 후, 이미지 서브볼륨의 스냅샷을 만들어 컨테이너 파일시스템을 준비한다. 
이후 네트워크 인터페이스 및 네임스페이스를 설정하여 가상 이더넷, IP, MAC 주소를 할당하며, 
cgroup을 활용해 CPU와 메모리 사용량을 제한한다. 
마지막으로, 네임스페이스 격리 및 chroot를 적용하여 컨테이너 환경을 구성하고 실행할 명령을 수행한다.

이 모든 과정은 격리된 환경에서 애플리케이션을 실행하면서도 자원의 사용량을 효과적으로 제어하기 위해 설계되었다.

```

**내부 동작 과정:**
1. 컨테이너 ID 생성 및 유효성 검사:
   - "ps_" 접두사와 랜덤 숫자로 컨테이너 ID를 생성합니다
   - `bocker_check`로 이미지 존재 여부와 ID 중복을 검사합니다
   - 중복 ID가 발견되면 재귀적으로 다시 시도합니다
2. 입력 매개변수와 네트워크 설정 준비:
   - `${@:2}`는 두 번째 인수부터의 모든 인수를 명령어로 결합합니다
   - 컨테이너 ID에서 IP와 MAC 주소 일부를 파생시킵니다:
     - 마지막 3자리에서 0을 제거하여 IP 주소 끝자리를 만듭니다
     - 마지막 3자리 중 첫 번째와 두 번째를 MAC 주소 일부로 사용합니다
3. 네트워크 네임스페이스와 인터페이스 설정:
   - 호스트와 컨테이너 간 통신을 위한 가상 이더넷(veth) 쌍을 생성합니다
   - 호스트 측 인터페이스를 활성화하고 브릿지에 연결합니다
   - 컨테이너 전용 네트워크 네임스페이스를 생성합니다
   - veth 쌍의 컨테이너 측 인터페이스를 네임스페이스로 이동합니다
   - 네임스페이스 내에서 루프백 인터페이스 활성화, MAC 주소 설정, IP 할당, 라우팅 설정을 진행합니다
4. 컨테이너 파일시스템 설정:
   - 이미지의 Btrfs 스냅샷을 만들어 컨테이너의 독립된 파일시스템을 생성합니다
   - Google Public DNS를 사용하도록 resolv.conf를 설정합니다
   - 실행할 명령어를 기록 파일에 저장합니다
5. 리소스 제한 설정:
   - cgroup을 생성하고 CPU 및 메모리 제한을 설정합니다
   - `: "${BOCKER_CPU_SHARE:=512}"`는 환경변수가 설정되지 않은 경우 기본값을 사용합니다
6. 컨테이너 실행 블록:
   - `cgexec`으로 cgroup 리소스 제한 내에서 실행합니다
   - `ip netns exec`로 컨테이너의 네트워크 네임스페이스에서 실행합니다
   - `unshare`로 다양한 네임스페이스(파일시스템, 마운트, UTS, IPC, PID)를 격리합니다
   - `chroot`로 컨테이너 루트 디렉토리를 변경합니다
   - `/bin/sh -c`로 명령을 쉘을 통해 실행합니다
   - `/bin/mount -t proc proc /proc && $cmd`로 proc 파일시스템을 마운트한 후 사용자 명령을 실행합니다
   - 모든 출력을 로그 파일에 저장하고 동시에 표시합니다
   - `|| true`로 명령 실패가 스크립트 전체 실행을 중단시키지 않도록 합니다
7. 리소스 정리:
   - 컨테이너 종료 후 생성했던 네트워크 리소스를 정리합니다
   - veth 인터페이스와 네트워크 네임스페이스를 제거합니다

> #### 5. bocker_exec() - 실행 중인 컨테이너에서 명령 실행

```bash
function bocker_exec() {
    [[ "$(bocker_check "$1")" == 1 ]] && echo "No container named '$1' exists" && exit 1
    cid="$(ps o ppid,pid | grep "^$(ps o pid,cmd | grep -E "^\ *[0-9]+ unshare.*$1" | awk '{print $1}')" | awk '{print $2}')"
    [[ ! "$cid" =~ ^\ *[0-9]+$ ]] && echo "Container '$1' exists but is not running" && exit 1
    nsenter -t "$cid" -m -u -i -n -p chroot "$btrfs_path/$1" "${@:2}"
}
```
```
이 함수는 이미 실행 중인 컨테이너에 추가 명령을 실행하는 기능을 제공합니다. 작동 흐름은 다음과 같습니다:

1. 먼저 지정된 컨테이너가 존재하는지 검증합니다
2. ps 명령어와 여러 필터링 도구를 조합하여 컨테이너의 프로세스 ID를 찾아냅니다
3. nsenter 명령으로 해당 프로세스의 네임스페이스에 진입합니다
4. 컨테이너 환경에서 요청된 명령을 실행합니다

이 함수는 실행 중인 컨테이너 내부에서 문제를 해결하거나, 
파일을 조회하거나, 설정을 변경하는 등의 작업을 가능하게 합니다. 
ps와 grep 같은 표준 Unix 도구를 창의적으로 조합하여 복잡한 프로세스 탐색 로직을 구현한 것이 특징입니다.
```

**내부 동작 과정:**
1. 컨테이너 존재 여부 확인:
   - `bocker_check`로 컨테이너 서브볼륨이 존재하는지 검사합니다
2. 복잡한 프로세스 검색 파이프라인으로 컨테이너의 PID를 찾습니다:
   - 첫 번째 `ps o pid,cmd`로 모든 프로세스의 PID와 명령어를 나열합니다
   - `grep -E "^\ *[0-9]+ unshare.*$1"`로 컨테이너 ID를 포함한 unshare 명령을 찾습니다
   - `awk '{print $1}'`로 첫 번째 필드(PID)만 추출합니다
   - 이 PID를 부모 PID로 가진 프로세스를 찾기 위해 다시 `ps o ppid,pid`를 실행합니다
   - 최종적으로 `awk '{print $2}'`로 자식 프로세스의 PID를 얻습니다
3. PID 유효성 검사:
   - 정규식 `^\ *[0-9]+$`로 숫자로만 이루어졌는지 확인합니다
   - 유효하지 않으면 컨테이너가 존재하지만 실행 중이 아니라고 판단합니다
4. 컨테이너 네임스페이스에 진입하여 명령 실행:
   - `nsenter`로 지정된 PID의 네임스페이스에 들어갑니다
   - `-t "$cid"`는 타겟 PID를 지정합니다
   - `-m -u -i -n -p`는 마운트, UTS, IPC, 네트워크, PID 네임스페이스에 진입을 의미합니다
   - `chroot "$btrfs_path/$1"`로 컨테이너의 파일시스템 루트를 사용합니다
   - `"${@:2}"`로 두 번째 인수부터의 모든 인수를 실행할 명령어로 사용합니다

> #### 6. bocker_commit() - 컨테이너를 이미지로 변환

```bash
function bocker_commit() {
    [[ "$(bocker_check "$1")" == 1 ]] && echo "No container named '$1' exists" && exit 1
    [[ "$(bocker_check "$2")" == 1 ]] && echo "No image named '$2' exists" && exit 1
    bocker_rm "$2" && btrfs subvolume snapshot "$btrfs_path/$1" "$btrfs_path/$2" > /dev/null
    echo "Created: $2"
}
```
```
이 함수는 컨테이너의 현재 상태를 새 이미지로 저장합니다:

1. 소스 컨테이너와 대상 이미지 ID가 유효한지 검증합니다
2. 대상 이미지가 이미 존재한다면 삭제합니다
3. 컨테이너 서브볼륨의 Btrfs 스냅샷을 생성하여 새 이미지를 만듭니다

이 기능은 컨테이너에서 수행한 변경사항(소프트웨어 설치, 설정 변경 등)을 
영구적으로 보존하여 새로운 이미지로 만들 수 있게 해줍니다. 
개발 과정에서 환경을 점진적으로 구축하거나, 특정 구성을 재사용 가능한 이미지로 저장할 때 매우 유용합니다. 
Docker의 `docker commit`과 정확히 동일한 목적을 가진 기능입니다.
```

**내부 동작 과정:**
1. 컨테이너와 이미지 존재 여부 확인:
   - 첫 번째 매개변수(컨테이너 ID)가 존재하는지 검사합니다
   - 두 번째 매개변수(이미지 ID)는 *존재하지 않아야* 하는 것이 논리적이지만, 코드는 오히려 존재해야 한다고 검사합니다 (가능한 버그)
2. 기존 이미지 제거 및 새 이미지 생성:
   - `bocker_rm "$2"`로 기존 이미지를 제거합니다 (덮어쓰기 효과)
   - `btrfs subvolume snapshot`으로 컨테이너 서브볼륨의 스냅샷을 새 이미지 ID로 생성합니다
3. 성공 메시지 출력:
   - 생성된 이미지 ID를 포함하여 완료 메시지를 표시합니다

### 기타 유틸리티 함수들

> #### bocker_images() - 이미지 목록 표시
```bash
function bocker_images() {
    echo -e "IMAGE_ID\t\tSOURCE"
    for img in "$btrfs_path"/img_*; do
        img=$(basename "$img")
        echo -e "$img\t\t$(cat "$btrfs_path/$img/img.source")"
    done
}
```
```
이 함수는 시스템에 있는 모든 Docker 이미지 목록을 표 형식으로 보여줍니다. 
작동 방식은 간단합니다:

1. 테이블 헤더를 출력합니다 (IMAGE_ID와 SOURCE 두 열)
2. Btrfs 경로에서 "img_" 접두사를 가진 모든 서브볼륨을 찾습니다
3. 각 이미지의 ID와 출처 정보(img.source 파일 내용)를 함께 표시합니다

이 함수는 Docker의 `docker images` 명령과 유사한 출력을 제공하며, 
사용자가 시스템에 있는 이미지를 파악하고 관리하는 데 도움이 됩니다. 
특히 이미지가 로컬에서 생성되었는지, Docker Hub에서 가져온 것인지를 구별할 수 있게 해줍니다.
```

**내부 동작 과정:**
1. 테이블 헤더 출력
2. glob 패턴 `img_*`으로 모든 이미지 서브볼륨 경로를 찾습니다
3. 각 경로에 대해:
   - `basename`으로 경로에서 이미지 ID만 추출합니다
   - 이미지 ID와 해당 이미지의 출처 정보를 탭으로 구분하여 출력합니다

> #### bocker_ps() - 컨테이너 목록 표시
```bash
function bocker_ps() {
    echo -e "CONTAINER_ID\t\tCOMMAND"
    for ps in "$btrfs_path"/ps_*; do
        ps=$(basename "$ps")
        echo -e "$ps\t\t$(cat "$btrfs_path/$ps/$ps.cmd")"
    done
}
```
```
이 함수는 시스템의 모든 컨테이너 목록을 보여줍니다. 
bocker_images와 유사한 방식으로 작동하지만, 컨테이너를 대상으로 합니다:

1. 테이블 헤더(CONTAINER_ID와 COMMAND)를 출력합니다
2. "ps_" 접두사를 가진 모든 서브볼륨을 찾아 순회합니다
3. 각 컨테이너의 ID와 실행 명령어를 표시합니다

Docker의 `docker ps` 명령과 달리, 
이 함수는 실행 중인 컨테이너만 필터링하지 않고 모든 컨테이너를 보여줍니다. 
각 컨테이너는 생성 시 실행 명령어가 파일로 저장되기 때문에, 
이미 종료된 컨테이너의 명령어도 확인할 수 있습니다. 
이는 실행 기록을 추적하고 컨테이너를 관리하는 데 유용합니다.
```

**내부 동작 과정:**
1. 테이블 헤더 출력
2. glob 패턴 `ps_*`로 모든 컨테이너 서브볼륨 경로를 찾습니다
3. 각 경로에 대해:
   - `basename`으로 경로에서 컨테이너 ID만 추출합니다
   - 컨테이너 ID와 해당 컨테이너에 저장된 명령어 정보를 출력합니다

> #### bocker_logs() - 컨테이너 로그 보기
```bash
function bocker_logs() {
    [[ "$(bocker_check "$1")" == 1 ]] && echo "No container named '$1' exists" && exit 1
    cat "$btrfs_path/$1/$1.log"
}
```
```
이 함수는 컨테이너의 출력 로그를 확인하는 간단한 기능을 제공합니다:

1. 지정된 컨테이너가 존재하는지 확인합니다
2. 컨테이너의 로그 파일 내용을 출력합니다

bocker_run 함수가 모든 컨테이너 출력을 로그 파일에 저장하기 때문에, 
이 함수는 단순히 해당 파일을 표시하는 역할을 합니다. 
컨테이너 실행 중 발생한 오류를 디버깅하거나 애플리케이션의 출력을 확인할 때 유용합니다. 
Docker의 `docker logs` 명령과 유사하지만, 실시간 로그 스트리밍은 지원하지 않는 간소화된 버전입니다.
```

**내부 동작 과정:**
1. 컨테이너 존재 여부 확인
2. 컨테이너 실행 시 저장된 로그 파일을 `cat` 명령으로 출력합니다

> #### bocker_rm() - 이미지/컨테이너 삭제
```bash
function bocker_rm() {
    [[ "$(bocker_check "$1")" == 1 ]] && echo "No container named '$1' exists" && exit 1
    btrfs subvolume delete "$btrfs_path/$1" > /dev/null
    cgdelete -g "$cgroups:/$1" &> /dev/null || true
    echo "Removed: $1"
}
```
```
이 함수는 이미지나 컨테이너를 삭제합니다. 단순하지만 효과적인 구현으로:

1. 먼저 bocker_check로 삭제할 대상이 실제로 존재하는지 확인합니다
2. Btrfs 서브볼륨 삭제 명령으로 해당 이미지/컨테이너의 파일 시스템을 제거합니다
3. 컨테이너였다면 관련 cgroup 리소스도 함께 정리합니다

특별한 점은 이미지와 컨테이너 삭제에 동일한 함수를 사용한다는 것입니다. 
cgroup 삭제가 실패해도 전체 명령이 실패하지 않도록 `|| true`로 처리했기 때문에 가능합니다. 
이런 접근법은 코드를 간결하게 유지하면서도 다양한 상황을 처리할 수 있게 해줍니다.
```

**내부 동작 과정:**
1. 대상 존재 여부 확인
2. Btrfs 명령으로 해당 서브볼륨을 삭제합니다
3. `cgdelete`로 관련된 cgroup도 삭제합니다
   - `&> /dev/null || true`로 오류가 발생해도 무시합니다 (cgroup이 없을 수 있음)
4. 삭제 완료 메시지를 출력합니다

> #### bocker_help() - 도움말 표시
```bash
function bocker_help() {
    sed -n "s/^.*#HELP\\s//p;" < "$1" | sed "s/\\\\n/\n\t/g;s/$/\n/;s!BOCKER!${1/!/\\!}!g"
}
```
```
이 함수는 Bocker의 사용법을 보여주는 도움말 시스템을 구현합니다:

1. 소스 코드에서 #HELP로 표시된 주석을 추출합니다
2. 추출된 텍스트를 보기 좋게 형식화합니다
3. 도움말 내용을 사용자에게 표시합니다
```

**내부 동작 과정:**
1. `sed -n "s/^.*#HELP\\s//p;"`로 입력 파일에서 #HELP로 표시된 주석 줄을 찾아 패턴 앞부분을 제거합니다
2. 결과를 다시 `sed` 파이프라인으로 전달하여:
   - `"s/\\\\n/\n\t/g"`로 이스케이프된 줄바꿈 문자를 실제 줄바꿈과 탭으로 변환합니다
   - `s/$/\n/`로 각 줄 끝에 줄바꿈을 추가합니다
   - `s!BOCKER!${1/!/\\!}!g`로 "BOCKER" 텍스트를 실제 스크립트 이름으로 대체합니다 (특수문자 처리 포함)

#### 전체 워크플로우 요약

> Bocker는 다음과 같은 작업 흐름을 지원합니다:

1. **이미지 준비**: `bocker_pull`이나 `bocker_init`으로 이미지를 가져오거나 만듭니다.
2. **컨테이너 생성**: `bocker_run`으로 이미지 기반 컨테이너를 실행합니다.
3. **컨테이너 관리**: `bocker_exec`, `bocker_logs`로 컨테이너와 상호작용합니다.
4. **변경사항 저장**: `bocker_commit`으로 컨테이너를 새 이미지로 저장합니다.
5. **정리**: `bocker_rm`으로 불필요한 컨테이너나 이미지를 제거합니다.

이 모든 기능들이 Linux 커널 기능(Btrfs, 네임스페이스, cgroups)을 활용하여 
Docker와 유사한 경험을 제공하면서도 단일 Bash 스크립트로 구현되어 있습니다.

## License

Copyright (C) 2015 Peter Wilmott

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.
