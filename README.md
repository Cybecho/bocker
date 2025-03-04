# Bocker
약 100줄의 배쉬로 구현된 Docker.

- [Bocker](#bocker)
  - [Prerequisites](#prerequisites)
  - [Example Usage](#example-usage)
  - [기능: 현재 구현됨](#기능-현재-구현됨)
  - [기능을 매우 제한적으로 구현합니다: 아직 구현되지 않음](#기능을-매우-제한적으로-구현합니다-아직-구현되지-않음)
  - [각 함수 설명](#각-함수-설명)
      - [초기 설정과 기본 구조](#초기-설정과-기본-구조)
      - [이미지 관리 기능](#이미지-관리-기능)
      - [컨테이너 실행 및 관리](#컨테이너-실행-및-관리)
      - [도움말 및 유틸리티 기능](#도움말-및-유틸리티-기능)
      - [전체 워크플로우 요약](#전체-워크플로우-요약)
  - [License](#license)

## Prerequisites

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

## Example Usage

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

## 기능을 매우 제한적으로 구현합니다: 아직 구현되지 않음

* 데이터 볼륨 컨테이너
* 데이터 볼륨
* 포트 포워딩

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
