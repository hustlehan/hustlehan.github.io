---
title: ROS1 URDF 수정하기
description: 
author: 
date: 2025-01-31 18:25:00 +0900
categories: [Robotics, ROS1]
tags: [ROS1, URP, Scout, Agilex]
pin: false
math: true
mermaid: true
---
## 사용할 로봇

제가 사용할 로봇 모델은 AGILEX Robotics사의 Scout Mini 입니다.
외형은 아래와 같이 생겼습니다.

![Desktop View](/assets/img/posts/urdf/scout_mini.png){: width="972" height="589" .w-75}



### 기존 URDF 살펴보기

제공된 xacro 파일을 실행시켜 rviz로 시각화를 우선해봅니다.

![Desktop View](/assets/img/posts/urdf/stockurdf.png){: width="972" height="589" .w-75}

또한 tf tree를 통해 확인해봅시다.

![Desktop View](/assets/img/posts/urdf/beforerp.png){: width="972" height="589" .w-75}

tf tree를 확인해보면 LiDAR의 /scan 토픽을 받아들일 수 있는 적절한 link가 없습니다.

따라서 xacro 파일에 LiDAR에 대응하는 child link를 직접 추가해주고 이를 /scan 토픽과 연결하는 것까지 이번 포스팅에서 해보고자 합니다.  

## XACRO 파일 수정
### LiDAR 링크 만들기

우선 LiDAR를 담당할 link의 설치 위치를 알아야 합니다. 

로봇의 중심이 되는 link는 바로 base_link입니다. 이 base_link에 여러 부품들을 각각 정의하는 child_link들이 joint를 통해 묶여 한 몸으로 움직임을 합니다. 만약 base_link에 묶이지 않은 채로 각 부품을 정의한다면, 로봇 Body는 가만히 있고 바퀴만 굴러가는 그러한 불상사가 생길 수도 있습니다. 

그렇기 때문에 base_link의 하위 link, 즉 parent_link를 base_link로, child_link를 LiDAR를 담당할 link로 만들어주고, 이를 joint로 연결해주면 됩니다.


```xml
    <link name="rp_lidar">
        <visual>
            <origin xyz="0 0 0.068999" rpy="0 0 0" />
            <geometry>
                <cylinder radius="0.1" length="0.05"/>
            </geometry>
            <material name="blue"/>
        </visual>
        <collision>
            <origin xyz="0 0 0.068999" rpy="0 0 0" />
            <geometry>
                <cylinder radius="0.1" length="0.05"/>
            </geometry>
        </collision>
    </link>
```
저는 LiDAR를 담당할 link의 이름을 rp_lidar로 지정해주었습니다. 

base_link의 cartesian 좌표가 (0, 0, 0)으로 설정되어있기 때문에, 실제 로봇에 LiDAR를 장착하는 높이를 고려하여 z값은 0.0689로 설정해주었습니다.(로봇 실제 사양 참고)

또한 rviz상에서 파랑색 원기둥으로 나타나게끔 시각화하였습니다.

### LiDAR 링크를 joint로 연결하기

앞서 언급했듯이, 추가한 link를 joint로 연결해주지 않는다면, 로봇이 앞으로 출발했는데, LiDAR만 똑 떼놓고 출발하게 됩니다. 따라서 이를 로봇 몸체에 fix 해주기 위해서 joint를 정의해주어야만 합니다.

```xml
    <joint name="rp_lidar_joint" type="fixed">
        <origin xyz="0 0 0.068999" rpy="0 0 0" />
        <parent link="base_link" />
        <child link="rp_lidar" />
    </joint>
```

보시면 아시겠지만 반드시 fixed 형태로 정의해주어야만 딱붙어서 움직이게 됩니다!!!

## xacro 수정 결과 확인

### RVIZ로 확인
![Desktop View](/assets/img/posts/urdf/urdfmod.png){: width="972" height="589" .w-75}

짜잔. 

앞서 rviz상에 나타난 것과 다르게 LiDAR link가 파란색 원기둥으로 추가된 것을 바로 알 수 있습니다.  
tf tree를 통해 확실하게 점검해 봅시다.

### TF_tree로 확인

![Desktop View](/assets/img/posts/urdf/afterrp.png){: width="972" height="589" .w-75}

예상대로 base_link 하단에 "rp_lidar"라는 새로운 녀석이 추가되어 child link로써 base_link에 연결된 것을 알 수 있습니다. tf tree에 이러한 결과값이 나타난다면 비로소 부품을 로봇에 제대로 연결한 것입니다.


## 토픽 연결하기

로봇에 LiDAR를 추가해줬으니, LiDAR가 스캔한 토픽을 앞서 만든 "rp_lidar"에서 받아오도록 설정해주어야 합니다.

```python
def combine_scans(scan1, scan2):
    combined_scan = LaserScan()
    
    combined_scan.header.stamp = rospy.Time.now()  
    combined_scan.header.frame_id = "rp_lidar"
```
제 publisher node의 일부를 발췌하였습니다.

이와 같이 header frame id에 새로 만든 link의 이름을 넣어주면 비로소 모든 절차가 끝이 납니다.
### 결과 예시
![Desktop View](/assets/img/posts/urdf/2dscan.png){: width="972" height="589" .w-75}

2D LiDAR의 LaserScan값을 제대로 받아오고 있는 모습을 확인할 수 있습니다.