# Bocker
Docker implemented in around 100 lines of bash.

  * [Prerequisites](#prerequisites)
  * [Example Usage](#example-usage)
  * [Functionality: Currently Implemented](#functionality-currently-implemented)
  * [Functionality: Not Yet Implemented](#functionality-not-yet-implemented)
  * [License](#license)

## Prerequisites

The following packages are needed to run bocker.

* btrfs-progs
* curl
* iproute2
* iptables
* libcgroup-tools
* util-linux >= 2.25.2
* coreutils >= 7.5

Because most distributions do not ship a new enough version of util-linux you will probably need to grab the sources from [here](https://www.kernel.org/pub/linux/utils/util-linux/v2.25/) and compile it yourself.

Additionally your system will need to be configured with the following:

* A btrfs filesystem mounted under `/var/bocker`
* A network bridge called `bridge0` and an IP of 10.0.0.1/24
* IP forwarding enabled in `/proc/sys/net/ipv4/ip_forward`
* A firewall routing traffic from `bridge0` to a physical interface.

For ease of use a Vagrantfile is included which will build the needed environment.

Even if you meet the above prerequisites you probably still want to **run bocker in a virtual machine**. Bocker runs as root and among other things needs to make changes to your network interfaces, routing table, and firewall rules. **I can make no guarantees that it won't trash your system**.

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

## Functionality: Currently Implemented

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

† `bocker init` provides a very limited implementation of `docker build`

## Functionality: Not Yet Implemented

* Data Volume Containers
* Data Volumes
* Port Forwarding

## 코드 로직 단순나열
다음은 `bocker` 스크립트의 각 줄에 대한 설명입니다:

```
1. Bash 인터프리터를 사용합니다.
2. 오류 발생 시 종료, 정의되지 않은 변수 사용 시 오류 발생, 파이프 실패 시 오류 발생; nullglob을 활성화합니다.
3. Btrfs 경로와 cgroups를 설정합니다.
4. 명령줄 옵션을 파싱하고 변수를 설정합니다.
5. (빈 줄)
6. Btrfs 서브볼륨이 존재하는지 확인하는 함수를 정의합니다.
7. Btrfs 서브볼륨을 나열하고 일치하는지 확인합니다.
8. 함수 끝.
9. (빈 줄)
10. 디렉토리에서 이미지를 초기화하는 함수를 정의합니다.
11. 이미지에 대한 랜덤 UUID를 생성합니다.
12. 디렉토리가 존재하는지 확인합니다.
13. 이미지가 이미 존재하면 실행합니다.
14. 새로운 Btrfs 서브볼륨을 생성합니다.
15. 디렉토리 내용을 새로운 서브볼륨으로 복사합니다.
16. 소스 디렉토리가 기록되지 않았다면 기록합니다.
17. 성공 메시지를 출력합니다.
18. 디렉토리가 존재하지 않으면 오류를 출력합니다.
19. 함수 끝.
20. (빈 줄)
21. Docker Hub에서 이미지를 가져오는 함수를 정의합니다.
22. 인증 토큰을 가져옵니다.
23. Docker 레지스트리 URL을 설정합니다.
24. 지정된 태그의 이미지 ID를 가져옵니다.
25. 이미지 ID가 유효하지 않으면 오류를 출력하고 종료합니다.
26. 이미지의 계보를 가져옵니다.
27. 임시 디렉토리를 초기화하고 계보를 처리합니다.
28. 이미지의 각 레이어를 다운로드하고 추출합니다.
29. 이미지 소스를 기록합니다.
30. 이미지를 초기화하고 정리합니다.
31. 함수 끝.
32. (빈 줄)
33. 이미지나 컨테이너를 삭제하는 함수를 정의합니다.
34. 컨테이너가 존재하지 않으면 오류를 출력하고 종료합니다.
35. Btrfs 서브볼륨을 삭제합니다.
36. cgroup을 삭제합니다.
37. 성공 메시지를 출력합니다.
38. 함수 끝.
39. (빈 줄)
40. 이미지를 나열하는 함수를 정의합니다.
41. 헤더를 출력합니다.
42. 이미지 디렉토리를 반복합니다.
43. 이미지 ID와 소스를 출력합니다.
44. 함수 끝.
45. (빈 줄)
46. 컨테이너를 나열하는 함수를 정의합니다.
47. 헤더를 출력합니다.
48. 컨테이너 디렉토리를 반복합니다.
49. 컨테이너 ID와 명령을 출력합니다.
50. 함수 끝.
51. (빈 줄)
52. 컨테이너를 생성하는 함수를 정의합니다.
53. 컨테이너에 대한 랜덤 UUID를 생성합니다.
54. 이미지가 존재하지 않으면 오류를 출력하고 종료합니다.
55. UUID가 충돌하면 재시도합니다.
56. 네트워크 인터페이스를 설정합니다.
57. 네트워크 네임스페이스를 설정합니다.
58. 네트워크 네임스페이스를 구성합니다.
59. 이미지의 Btrfs 스냅샷을 생성합니다.
60. DNS를 설정합니다.
61. 명령을 기록합니다.
62. cgroup을 생성합니다.
63. CPU와 메모리 제한을 설정합니다.
64. 컨테이너에서 명령을 실행합니다.
65. 네트워크를 정리합니다.
66. 함수 끝.
67. (빈 줄)
68. 실행 중인 컨테이너에서 명령을 실행하는 함수를 정의합니다.
69. 컨테이너가 존재하지 않으면 오류를 출력하고 종료합니다.
70. 컨테이너의 PID를 가져옵니다.
71. 컨테이너가 실행 중이지 않으면 오류를 출력하고 종료합니다.
72. 컨테이너의 네임스페이스에 들어가 명령을 실행합니다.
73. 함수 끝.
74. (빈 줄)
75. 컨테이너 로그를 보는 함수를 정의합니다.
76. 컨테이너가 존재하지 않으면 오류를 출력하고 종료합니다.
77. 로그를 출력합니다.
78. 함수 끝.
79. (빈 줄)
80. 컨테이너를 이미지로 커밋하는 함수를 정의합니다.
81. 컨테이너나 이미지가 존재하지 않으면 오류를 출력하고 종료합니다.
82. 기존 이미지를 제거하고 스냅샷을 생성합니다.
83. 성공 메시지를 출력합니다.
84. 함수 끝.
85. (빈 줄)
86. 도움말을 표시하는 함수를 정의합니다.
87. 도움말 메시지를 파싱하고 형식을 지정합니다.
88. 함수 끝.
89. (빈 줄)
90. 인자가 제공되지 않으면 도움말을 표시합니다.
91. 명령을 적절한 함수에 전달합니다.
92. 명령을 알 수 없는 경우 도움말을 표시합니다.
93. 스크립트 끝.
```

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
