---
title: micro-ROS 사용 전 bashrc 환경 설정하기
description: ROS2 환경에서 micro-ROS 통신  환경변수 설정
author: hustlehan
date: 2025-10-11 01:40:00 +0900
categories: [Robotics, ROS2]
tags: [ROS, ROS2, micro-ROS, DDS]
pin: false
math: false
mermaid: false
---

## micro-ROS란?

micro-ROS는 마이크로컨트롤러와 같은 리소스가 제한된 임베디드 시스템에서 ROS2를 실행할 수 있게 해주는 프레임워크. 
ESP32, Arduino 등의 소형 보드에서 ROS2 노드를 구동하고 일반 ROS2 시스템과 통신이 가능.(~~획기적인 시스템~~)

## bashrc 파일 수정하기

micro-ROS와 ROS2 간의 원활한 통신을 위해서는 몇 가지 환경변수를 설정해주어야 합니다.

### bashrc 파일 열기

터미널에서 다음 명령어로 bashrc 파일을 엽니다:

```bash
nano ~/.bashrc
```

또는 VSCode에서:

```bash
code ~/.bashrc
```

### 환경변수 추가하기

bashrc 파일의 맨 아래에 다음 내용을 추가합니다:

```bash
# micro-ROS 설정
export ROS_DOMAIN_ID=0
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
export ROS_AUTOMATIC_DISCOVERY_RANGE=SUBNET
```

> **주의**: 등호(=) 앞뒤로 공백이 없어야 함!!!
{: .prompt-warning }

### 변경사항 적용하기

파일을 저장한 후, 터미널에서 다음 명령어로 변경사항을 적용합니다:

```bash
source ~/.bashrc
```

## 각 환경변수의 의미

### ROS_DOMAIN_ID

```bash
export ROS_DOMAIN_ID=0
```

**역할:**
- ROS2 네트워크를 논리적으로 분리하는 식별자
- 0부터 101까지의 값 ㄱㄴ
- 같은 네트워크 상에서 여러 ROS2 시스템이 있을 때, 서로 다른 Domain ID를 사용하면 간섭 없이 독립적으로 운영 ㄱㄴ

**사용 예시:**
- 연구실에서 여러 팀이 작업할 때, 팀 A는 Domain ID 0, 팀 B는 Domain ID 1 사용
- 같은 Domain ID를 가진 노드들끼리만 통신 가능

### RMW_IMPLEMENTATION

```bash
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
```

**역할:**
- ROS2의 미들웨어(RMW, ROS MiddleWare) 구현체를 지정합니다
- DDS(Data Distribution Service) 프로토콜을 어떤 방식으로 사용할지 결정합니다

**주요 옵션:**
- `rmw_fastrtps_cpp`: Fast DDS를 사용 (기본값, 가장 널리 사용됨)
- `rmw_cyclonedds_cpp`: Cyclone DDS를 사용
- `rmw_connextdds`: RTI Connext DDS를 사용

**왜 Fast DDS인가?**
- micro-ROS와의 호환성이 가장 우수
- 안정적이고 성능이 검증
- 대부분의 micro-ROS 예제와 튜토리얼에서 사용

### ROS_AUTOMATIC_DISCOVERY_RANGE

```bash
export ROS_AUTOMATIC_DISCOVERY_RANGE=SUBNET
```

**역할:**
- ROS2 노드들이 서로를 자동으로 발견하는 범위를 설정합니다

**옵션:**
- `SUBNET`: 같은 서브넷(로컬 네트워크) 내의 노드만 발견 (권장)
- `LOCALHOST`: 같은 컴퓨터 내의 노드만 발견
- `SYSTEM_DEFAULT`: 시스템 기본 설정 사용
- `OFF`: 자동 발견 기능 비활성화

**SUBNET을 사용하는 이유:**
- 불필요한 외부 네트워크 트래픽을 차단
- micro-ROS 보드가 같은 Wi-Fi 네트워크에 연결되어 있다면 자동으로 통신 가능(~~하지만 저는 serial 통신까지만...~~)

## bashrc 설정 확인하기

![micro-ROS bashrc 설정](/assets/img/posts/micro_ros_bashrc/micro_ros_bashrc.png){: width="972" height="589" .w-75}
```bash
echo $ROS_DOMAIN_ID
echo $RMW_IMPLEMENTATION
echo $ROS_AUTOMATIC_DISCOVERY_RANGE
```

각각 설정한 값이 출력되면 성공


### micro-ROS agent와 통신 테스트

micro-ROS agent를 실행하고:

-UDP4 프로토콜로 wifi 무선 통신을 활용할 경우
 
```bash
ros2 run micro_ros_agent micro_ros_agent udp4 --port 8888
```

-Serial 통신으로 활용할 경우(보드레이트 115200)

```bash
ros2 run micro_ros_agent micro_ros_agent serial --dev /dev/ttyUSB0
```

-> 디버그 모드를 사용하고 싶은경우, verbose 6 옵션 추가
-> 옵션을 추가하지 않을 경우 자동으로 verbose 4로 실행

```bash
ros2 run micro_ros_agent micro_ros_agent udp4 --port 8888 -v6
```

```bash
ros2 run micro_ros_agent micro_ros_agent serial --dev /dev/ttyUSB0 -v6
```

MCU단에서 micro-ROS 노드를 실행한 후, 다른 터미널에서:

```bash
ros2 node list
ros2 topic list
```

micro-ROS 노드가 목록에 나타나면 설정 완료!

## 트러블슈팅

### 노드가 발견되지 않는 경우

1. **방화벽 확인**
   ```bash
   sudo ufw status
   ```
   필요시 DDS 통신 포트(7400-7500)를 허용

2. **Domain ID 일치 확인**
   - PC와 마이크로컨트롤러의 Domain ID가 같은지 

3. **네트워크 연결 확인**
   - 같은 Wi-Fi 네트워크에 연결되어 있는지 확인

### 설정이 적용되지 않는 경우

새 터미널을 열거나, 다시 한 번 source 명령어를 실행:

```bash
source ~/.bashrc
```

## 마무리

이제 micro-ROS 디바이스와 ROS2 시스템 간의 통신을 위한 기본 환경 설정이 완료되었습니다. 이 설정을 통해 ESP32, STM32 등의 마이크로컨트롤러에서 실행되는 micro-ROS 노드와 원활하게 통신할 수 있습니다.

