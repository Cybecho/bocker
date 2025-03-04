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
      - [이미지 관리 기능](#이미지-관리-기능)
      - [컨테이너 실행 및 관리](#컨테이너-실행-및-관리)
      - [도움말 및 유틸리티 기능](#도움말-및-유틸리티-기능)
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

---

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

> 1. 자기문서화 도움말 시스템

```bash
# 함수 정의에 도움말 포함
function bocker_run() { #HELP Create a container:\nBOCKER run <image_id> <command>
    # 코드...
}

# 도움말 추출
function bocker_help() {
    sed -n "s/^.*#HELP\\s//p;" < "$1" | sed "s/\\\\n/\n\t/g;s/$/\n/;s!BOCKER!${1/!/\\!}!g"
}
```

이 접근법은 코드와 문서화를 함께 유지하는 우아한 방식입니다. `#HELP` 주석을 파싱하여 도움말 텍스트를 동적으로 생성합니다.

> 2. 재귀적 UUID 충돌 해결

```bash
[[ "$(bocker_check "$uuid")" == 0 ]] && echo "UUID conflict, retrying..." && bocker_run "$@" && return
```

UUID 충돌이 발생하면(매우 드문 경우), 함수는 재귀적으로 자신을 호출하여 새 UUID를 생성합니다. 이는 간결하지만 깊은 재귀가 발생할 위험이 있습니다.

> 3. 파이프라인과 명령 체인

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

#### 이미지 관리 기능

> 1. 이미지 확인 (bocker_check)
```bash
function bocker_check() {
    btrfs subvolume list "$btrfs_path" | grep -qw "$1" && echo 0 || echo 1
}
```
- **시나리오**: 모든 작업 전에 이미지/컨테이너가 존재하는지 확인합니다.
- **작업**: Btrfs 서브볼륨 목록에서 ID를 검색합니다.
- **이유**: 오류를 방지하고 사용자에게 명확한 피드백을 제공하기 위함입니다.

> 2. 이미지 목록 표시 (bocker_images)
```bash
function bocker_images() {
    echo -e "IMAGE_ID\t\tSOURCE"
    for img in "$btrfs_path"/img_*; do
        img=$(basename "$img")
        echo -e "$img\t\t$(cat "$btrfs_path/$img/img.source")"
    done
}
```
- **시나리오**: 사용자가 시스템에 있는 모든 이미지를 확인하려고 할 때
- **작업**: 모든 이미지 서브볼륨을 순회하며 ID와 출처를 표시합니다.
- **이유**: 사용자가 사용 가능한 이미지를 빠르게 파악할 수 있게 합니다.

#### 컨테이너 실행 및 관리

> 3. 컨테이너 실행 (bocker_run)
```bash
function bocker_run() {
    uuid="ps_$(shuf -i 42002-42254 -n 1)"
    # ... 복잡한 설정 및 실행 코드
}
```
- **시나리오**: 사용자가 이미지로부터 새 컨테이너를 실행하려고 할 때
- **작업**:
  1. 고유한 컨테이너 ID를 생성합니다.
  2. 이미지 서브볼륨의 스냅샷을 만들어 컨테이너 파일시스템을 준비합니다.
  3. 네트워크 인터페이스와 네임스페이스를 설정합니다(가상 이더넷, IP, MAC 주소).
  4. cgroup으로 CPU와 메모리 사용량을 제한합니다.
  5. 네임스페이스 격리와 chroot로 컨테이너 환경을 만들고 명령을 실행합니다.
- **이유**: 격리된 환경에서 애플리케이션을 실행하면서 자원 사용량을 제어하기 위함입니다.

> 4. 컨테이너 목록 조회 (bocker_ps)
```bash
function bocker_ps() {
    echo -e "CONTAINER_ID\t\tCOMMAND"
    for ps in "$btrfs_path"/ps_*; do
        ps=$(basename "$ps")
        echo -e "$ps\t\t$(cat "$btrfs_path/$ps/$ps.cmd")"
    done
}
```
- **시나리오**: 사용자가 현재 시스템의 모든 컨테이너를 확인하려고 할 때
- **작업**: 모든 컨테이너 서브볼륨을 순회하며 ID와 실행 명령을 표시합니다.
- **이유**: 실행 중이거나 종료된 컨테이너를 쉽게 식별할 수 있게 합니다.

> 5. 컨테이너에서 명령 실행 (bocker_exec)
```bash
function bocker_exec() {
    # 컨테이너 PID 찾고 명령 실행
}
```
- **시나리오**: 사용자가 실행 중인 컨테이너에 추가 명령을 실행하려고 할 때
- **작업**:
  1. 컨테이너의 PID를 찾습니다.
  2. nsenter로 컨테이너의 네임스페이스에 진입합니다.
  3. chroot로 컨테이너의 파일시스템에서 명령을 실행합니다.
- **이유**: 실행 중인 컨테이너와 상호작용하여 디버깅이나 구성 변경을 위함입니다.

> 6. 컨테이너 로그 조회 (bocker_logs)
```bash
function bocker_logs() {
    cat "$btrfs_path/$1/$1.log"
}
```
- **시나리오**: 사용자가 컨테이너의 출력 로그를 확인하려고 할 때
- **작업**: 컨테이너가 생성한 로그 파일을 출력합니다.
- **이유**: 문제 해결이나 컨테이너 동작 확인을 위함입니다.

> 7. 컨테이너를 이미지로 변환 (bocker_commit)
```bash
function bocker_commit() {
    bocker_rm "$2" && btrfs subvolume snapshot "$btrfs_path/$1" "$btrfs_path/$2" > /dev/null
}
```
- **시나리오**: 사용자가 컨테이너 상태를 새 이미지로 저장하려고 할 때
- **작업**: 
  1. 기존 이미지가 있다면 삭제합니다.
  2. 컨테이너 서브볼륨의 스냅샷을 이미지로 생성합니다.
- **이유**: 변경된 컨테이너 상태를 재사용 가능한 이미지로 만들기 위함입니다.

#### 도움말 및 유틸리티 기능

> 8. 도움말 표시 (bocker_help)
```bash
function bocker_help() {
    sed -n "s/^.*#HELP\\s//p;" < "$1" | sed "s/\\\\n/\n\t/g;s/$/\n/;s!BOCKER!${1/!/\\!}!g"
}
```
- **시나리오**: 사용자가 도움말을 보려고 할 때
- **작업**: 소스 코드에서 #HELP 주석을 추출하여 사용법을 보여줍니다.
- **이유**: 사용자에게 사용 방법을 안내하기 위함입니다.

#### 전체 워크플로우 요약

> Bocker는 다음과 같은 작업 흐름을 지원합니다:

1. **이미지 준비**: `bocker_pull`이나 `bocker_init`으로 이미지를 가져오거나 만듭니다.
2. **컨테이너 생성**: `bocker_run`으로 이미지 기반 컨테이너를 실행합니다.
3. **컨테이너 관리**: `bocker_exec`, `bocker_logs`로 컨테이너와 상호작용합니다.
4. **변경사항 저장**: `bocker_commit`으로 컨테이너를 새 이미지로 저장합니다.
5. **정리**: `bocker_rm`으로 불필요한 컨테이너나 이미지를 제거합니다.

이 모든 기능들이 Linux 커널 기능(Btrfs, 네임스페이스, cgroups)을 활용하여 Docker와 유사한 경험을 제공하면서도 단일 Bash 스크립트로 구현되어 있습니다.

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
