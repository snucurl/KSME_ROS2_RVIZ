# 🚀 Rviz 기초 및 시뮬레이션 소개  
  

**[Environment]**  
***TurtleBot3 Burger + Gazebo + RViz  
Ubuntu 22.04 / ROS2 Humble***

📌 **강의 목표**  
이번 실습에서는 다음에 대하여 학습합니다.
- ROS2 설치 및 환경 설정
- TurtleBot3 Burger 구조 이해
- Gazebo 가상 환경 실행
- LiDAR 데이터 /scan 시각화
- Differential Drive 제어 (/cmd_vel)
- SLAM을 통한 지도 생성 (/map)

<br>

##  🖥 1️⃣ 전체 구조 이해
**Gazebo**에서 **TurtleBot3**을 움직이고,  
그 과정에서 생성되는 **센서/좌표/지도 데이터**를 ROS2 토픽으로 주고받아,  
**RViz에서 시각화**하고, **SLAM**으로 `/map`을 만들 수 있습니다.
```js
             ┌───────────────────────┐
             │      Gazebo World     │
             │  (벽, 장애물, 물리엔진) │
             └────────────┬──────────┘
                          │
                          ▼
               Gazebo LiDAR Plugin
                          │
                          ▼
                 /scan  (LaserScan)
                          │
                          ▼
                     ROS2 DDS
                          │
      ┌──────────────┬──────────────┐
      ▼              ▼              ▼
    RViz          SLAM Node        TF Tree
 (시각화)        (지도 생성)     (좌표변환)
                      │
                      ▼
                   /map
```

<br>  

### 1-1. 전체 데이터 흐름
```js
[Keyboard] 
   ↓ (teleop node)
 /cmd_vel  (geometry_msgs/Twist)
   ↓
[TurtleBot3 base controller in simulation]
   ↓
/odom  +  /tf   (로봇이 얼마나 움직였는지, 좌표계가 어떻게 연결되는지)
   ↓
[Virtual LiDAR in Gazebo]
   ↓
/scan  (sensor_msgs/LaserScan)
   ↓
[SLAM node]
   ↓
/map (nav_msgs/OccupancyGrid)
   ↓
[RViz]  ←  /scan, /tf, /odom, /map 등을 “보여줌”
```

<br>

### 1-2. 실습 환경
- OS : Ubuntu 22.04
- ROS2 : Humble
- 모델 : TurtleBot3 Burger

<br>

### 1-3. 핵심 토픽/프레임  
1. `/cmd_vel` : 로봇을 움직이는 "운전대"
    - 메시지 타입 : `geometry_msgs/Twist`
    - 자주 쓰는 필드
      - linear.x : 전진/후진 속도
      -  angular.z : 회전 속도
2. `/scan` : LiDAR가 보는 "거리 정보"
    - 메시지 타입 : `sensor_msgs/LaserScan`
    - 자주 쓰는 필드
      - ranges[] : 각도별 거리 배열
      -  angle_min, angle_max, angle_increment
3. `/tf`와 `/tf_static` : “좌표계 변환 트리”
    - `RViz/SLAM`에서 매우 중요
    - 로봇에는 여러 좌표계(frame)가 있고, 서로 어떻게 연결되는지 알려줌
      - `map` : SLAM이 만든 전역 지도 좌표계
      - `odom`: 바퀴/적분 기반 누적 위치(드리프트 있음)
      - `base_link`: 로봇 몸체 기준 좌표계
      - `base_scan`(또는 `laser`): 라이다 센서 기준 좌표계

4. `/map` : SLAM이 만든 “2D 지도”
      - 메시지 타입 : `nav_msgs/OccupancyGrid`
      - RViz에서 Map 디스플레이로 확인
      - `/scan` + TF(로봇 자세/이동) 정보가 합쳐져서 만들어짐

##  🖥 2️⃣ ROS2 설치 확인

### 2-1. ROS2 버전 확인  

아래 명령이 `humble`을 출력해야 정상입니다.
```js
echo $ROS_DISTRO
```

정상 출력 예시 :
```js
humble
```

만약 아무것도 나오지 않는다면, ROS 환경이 source되지 않은 상태입니다.

<br>

### 2-2. ROS2 환경 source 적용 확인  

터미널에서 아래를 실행하세요.
```js
source /opt/ros/humble/setup.bash
```
‼️ 매번 터미널을 실행시킬 때 마다, 위의 source를 적용해야합니다.


<details><summary> ⚠️매번 터미널에 입력시키는 과정을 스킵하고싶다면 클릭   

</summary>

<br>
매번 수동으로 하기 싫으면 ~/.bashrc에 추가합니다(일반 사용자 기준) :  

```js
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source ~/.bashrc
```
</details>

<br>

### 2-3. ROS2 CLI 동작 확인

```js
ros2 -h
```
도움말이 뜬다면 OK.

<br>

### 2-4. 설치된 ROS 패키지 목록 설치 확인  
ROS2의 패키지 목록이 보이면 환경이 제대로 적용된 것 입니다.

설치를 확인해야 하는 패키지 리스트 (아래 명령어 입력)
- ros-humble-desktop 
```js
dpkg -l | grep ros-humble-desktop
```
- ros-humble-turtlebot3
```js
ros2 pkg list | grep turtlebot3
```

설치 상태가 정상이라면 아래와 같이 보여야 함 : 
```js
turtlebot3_bringup
turtlebot3_description
turtlebot3_gazebo
turtlebot3_teleop
turtlebot3_cartographer
```

<br>

### 2-4. SLAM 설치 확인
cartographer의 실행 테스트 : 
```js
ros-humble-turtlebot3-cartographer
```

<details><summary> ⚠️만약, cartographer가 없다면 클릭
</summary>

<br>

```js
sudo apt update
sudo apt install -y ros-humble-turtlebot3-cartographer
```
설치 확인 :
```js
ros2 pkg list | grep cartographer
```
정상이라면 다음이 보여야 합니다.
```js
turtlebot3_cartographer
cartographer_ros
```
</details>

<br>

### 2-5. Gazebo 관련 패키지 설치 확인
ROS2 Desktop에 대부분 포함되지만 확인 권장 :
- gazebo
- ros-humble-gazebo-ros-pkgs  

확인 방법 :
```js
gazebo --version
```

<br>

### 2-6. RViz 설치 확인
- rviz
```js
ros2 pkg list | grep rviz
```

또는 실행 테스트 :
```js
rviz2
```

<details><summary> ⚠️만약, RViz가 없다면 클
</summary>

<br>

```js
sudo apt install ros-humble-rviz2
```
</details>


<br>

##  🖥 3️⃣ Gazebo 설치 확인 및 실행

### 3-1. Gazebo 설치 확인
ROS2 Desktop을 설치했다면 대부분 Gazebo가 포함되어 있습니다.
```js
gazebo --version
```
정상 예시 :
``` js
Gazebo multi-robot simulator, version 11.x.x
```

<br>

### 3-2. Gazebo ROS 연동 패키지 확인
```js
ros2 pkg list | grep gazebo
```
최소한 다음이 보여야 정상입니다.
- `gazebo_ros`
- `gazebo_ros_pkgs`


<details><summary> ⚠️ 만약 보이지 않는다면 클릭
</summary>

<br>

패키지 설치하기 :
```js
sudo apt update
sudo apt install -y ros-humble-gazebo-ros-pkgs
```

</details>

<br>

### 3-3. TurtleBot3 Gazebo 패키지 확인
```js
ros2 pkg list | grep turtlebot3_gazebo
```
정상이라면 :
```
turtlebot3_gazebo
```
없다면 설치 :
```js
sudo apt install ros-humble-turtlebot3-gazebo
```

<br>

### 3-4. 모델 환경변수 설정
TurtleBot3는 모델을 환경변수로 지정해야 합니다.
```js
echo $TURTLEBOT3_MODEL
```
아무 것도 나오지 않는다면 설정 필요 :
```js
export TURTLEBOT3_MODEL=burger
```
영구 설정하는 방법 :
```js
echo "export TURTLEBOT3_MODEL=burger" >> ~/.bashrc
source ~/.bashrc
```

<br>

### 3-5. Gazebo 실행
<u>**‼️터미널 1에서 실행해야 합니다.**</u>  
(Gazebo, RViz, 토픽 확인, TurtleBot3 Burger 제어가 일어나는 터미널이 개별적으로 작동하므로, 각각 다른 터미널에서 실행해야함)

```js
source /opt/ros/humble/setup.bash
export TURTLEBOT3_MODEL=burger
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py
```

정상적으로 실행되었다면 다음과 같은 화면을 확인할 수 있음 :

![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/Gazebo.JPG?raw=true)   

<br>

### 3-6. 실행 후 확인해야 할 것

Gazebo 창이 뜨면:  
- TurtleBot3 Burger 로봇이 보임
- 벽/환경이 있는 월드가 로드됨
- LiDAR 센서가 상단에 장착된 형태

<details><summary> ⚠️ 만약 보이지 않는다면 클릭
</summary>

<br>

다음과 같이 Gazebo 구성 요소 중 TurtleBot3 Burger가 보이지 않는다면 :
![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/Gazebo2.JPG?raw=true)   

좌측 Insert 의 **TurtleBot3(Burger)** 를 Gazebo의 지도 위에 드래그 인 드롭하여 로봇을 생성해주십시오.
 
![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/insert.JPG?raw=true)   

<br>

</details>

<br>

### 3-7. ROS2 토픽 생성 확인
<u>**‼️터미널 2에서 실행해야 합니다.**</u>  
Gazebo가 실행된 상태에서 새 터미널을 열고 :
```js
source /opt/ros/humble/setup.bash
ros2 topic list
```
정상이라면 다음과 같은 토픽들이 추가로 보입니다.
```js
/scan
/cmd_vel
/odom
/tf
/tf_static
/clock
⁞
```
![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/topic2.JPG?raw=true)   


<br>

### 3-8. LiDAR 토픽 확인
ros2 topic echo /clock
```js
ros2 topic echo /scan
```
출력 예:
```js
angle_min: -3.14
angle_max: 3.14
ranges: [1.23, 1.21, 1.18, ...]
⁞
```

![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/echo_scan.JPG?raw=true)   

<br>

### 3-9. 시뮬레이션 시간 확인
<u>**‼️터미널 2에서 실행해야 합니다.**</u>  
Gazebo는 ROS2의 시뮬레이션 시간을 사용합니다.
```js
ros2 topic echo /clock
```
값이 계속 변하면 정상입니다.

<br>


##  🖥 4️⃣ RViz2 실행
### 4-1. RViz2 설치 확인
```js
ros2 pkg list | grep rviz2
```
또는 실행 테스트 :
```js
rviz2
```
만약 `rviz2: command not found` 라면 설치 :
```js
sudo apt update
sudo apt install -y ros-humble-rviz2
```

<br>

### 4-2. RViz2 실행 (권장 : TurtleBot3 기본 설정 사용)
<u>**‼️터미널 2에서 실행해야 합니다.**</u>  
```js
source /opt/ros/humble/setup.bash
export TURTLEBOT3_MODEL=burger
ros2 launch turtlebot3_bringup rviz2.launch.py
```
이 런치는 TurtleBot3에 맞춘 RViz 기본 설정을 같이 불러오기 위한 것 입니다.

![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/rviz1.JPG?raw=true)   


<br>

### 4-3. RViz 기본 설정
RViz에서 좌측 상단 **Global Option** 의 **Fixed Frame** 을 설정합니다.

✅ **Fixed Frame 설정**
- `odom`으로 설정

![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/fixedframe.JPG?raw=true)   

❗ Fixed Frame이 잘못되면 화면이 비거나 “No transform” 오류가 뜹니다.

<br>

### 4-4. RViz에 디스플레이 추가하기

좌측 **Display 패널**에서 **Add 버튼** 클릭하여 아래 항목들을 추가합니다.

![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/add1.JPG?raw=true)   

#### (1) RobotModel (로봇 외형)
- Add → RobotModel
- 로봇 URDF를 기반으로 모델이 보입니다.  

✅ 정상 확인 : 로봇 형태가 3D로 보임

![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/robotmodel.JPG?raw=true)   



#### (2) TF (좌표계)
- Add  → TF  

✅ 정상 확인 : 좌표축들이 로봇 주변에 표시됨  
✅ 정상 확인 : 로봇이 움직이면 좌표축도 이동/회전


![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/TF.JPG?raw=true)   


#### (3) LaserScan (LiDAR 시각화)
- Add   → LaserScan

![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/laserscan.JPG?raw=true)   

![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/laserscan2.JPG?raw=true)

- Topic → `/scan` 선택  

![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/lasertopic1.JPG?raw=true)   

![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/lasertopic2.JPG?raw=true)   


✅ 정상 확인 : 벽/장애물이 점 또는 선 형태로 보임  
✅ 정상 확인 : 로봇을 움직이면 점들이 상대적으로 변화  
✅ 정상 확인 : 로봇이 움직이면 좌표축도 이동/회전  

![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/laser.JPG?raw=true)   



#### (4) Odometry (이동 궤적 확인)
- Add → Odometry
- Topic → `/odom`  

✅ 정상 확인 : 로봇이 이동하면 궤적(화살표/선)이 나타남

![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/odometry.JPG?raw=true)   

<br>

### 4-5. RViz 설정 저장하기
RViz 상단 메뉴 :
- File → Save Config As…

![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/saverviz.jpg?raw=true)   

- 예 : `rviz_config/tb3_slam.rviz`  

다음부터는 이렇게 실행 가능 :
```js
rviz2 -d rviz_config/tb3_slam.rviz
```

<br>

##  🖥 5️⃣ TurtleBot3 Burger 제어

### 5-1. ` /cmd_vel` 토픽 이해
TurtleBot3는 `/cmd_vel` 토픽을 구독하여 이동합니다.  
메시지 타입 :
```js
geometry_msgs/Twist
```
주요 필드 :
```js
linear.x → 전진/후진 속도 (m/s)
angular.z → 회전 속도 (rad/s)
```
예시 :
```
linear.x = 0.2
angular.z = 0.0
→ 직진

linear.x = 0.0
angular.z = 0.5
→ 제자리 회전
```

<br>

### 5-2. 키보드 제어 실행 (Teleop)
<u>**‼️터미널 3에서 실행해야 합니다.**</u>
```js
source /opt/ros/humble/setup.bash
export TURTLEBOT3_MODEL=burger
ros2 run turtlebot3_teleop teleop_keyboard
```
기본 키 설명
```js
w : 전진
x : 후진
a : 좌회전
d : 우회전
s : 정지
```

<br>

### 5-3. `/cmd_vel` 토픽 실시간 확인
<u>**‼️터미널 4에서 실행해야 합니다.**</u>  
새 터미널을 열고 :
```js
ros2 topic echo /cmd_vel
```
키를 누르면 다음과 같은 메세지가 출력됩니다.
```
linear:
  x: 0.2
angular:
  z: 0.0
```
👉 키보드 입력이 ROS2 토픽으로 publish되는 것을 확인하세요.

<br>

### 5-4. RViz에서 수집 데이터 관찰
#### 1. Gazebo에서 TurtleBot3 이동 확인
#### 2. RViz에서 LaserScan 변화 관찰
#### 3. Odometry 확인
#### 4. TF 변화 확인 

<br>

##  🖥 6️⃣ SLAM 실행 (지도 생성)  
**LaserScan 시각화(A)** 와 **SLAM** 의 차이  

✅ LaserScan 시각화 (간단/직관)  
- 필요한 것: /scan + 적절한 Fixed Frame
- RViz에서 벽이 점으로 찍히는 것을 확인

✅ SLAM: 지도 생성 (한 단계 더)  
- 필요한 것: /scan + /tf(+ 종종 /odom)
- 결과: /map 생성 → RViz에서 격자 지도가 누적됨

<br>

### 6-1. SLAM이 동장하는 조건
SLAM은 `/scan`만으로는 동작하지 않습니다.  
최소 아래 3가지가 준비되어야 합니다.  
- `/scan` : LiDAR 거리 데이터
- `/tf` (및 `/tf_static`) : 좌표계 변환
- `/odom` : 로봇 이동 정보

✅ 준비 상태 확인 :
```js
source /opt/ros/humble/setup.bash
ros2 topic list | grep -E "scan|tf|odom|clock"
```
정상이라면 최소 아래가 보여야 합니다 :
- `/scan`
- `/tf`
- `/tf_static`
- `/odom`
- `/clock` (Gazebo 시뮬레이션 시간)

‼️ /clock이 존재하면, SLAM 실행 시 use_sim_time:=True 설정이 필수입니다  

<br>

### 6-2. SLAM 방식 선택
TurtleBot3 + ROS2 Humble 환경에서 SLAM은 보통 다음 중 하나를 사용합니다.  

✅ A) Cartographer
- Robotis(TurtleBot3) 제공 런치와 호환이 좋음
  
✅ B) slam_toolbox (Nav2 확장에 유리)
- 추후 Navigation2까지 확장할 계획이면 유리합니다.

<br>

### 6-3. Cartographer SLAM 설치 확인
Cartographer SLAM 설치 방법 :
```js
sudo apt update
sudo apt install -y ros-humble-turtlebot3-cartographer
```
설치 확인 :
```js
ros2 pkg list | grep cartographer
```

<br>

### 6-4. Cartographer SLAM 실행  
⚠️ 전제: Gazebo가 이미 실행 중이어야 합니다.  
(Gazebo 실행은 터미널 1에서 계속 켜 둔 상태)

<u>**‼️터미널 5에서 실행해야 합니다.**</u>

```js
source /opt/ros/humble/setup.bash
export TURTLEBOT3_MODEL=burger
ros2 launch turtlebot3_cartographer cartographer.launch.py use_sim_time:=True
```
성공하면 SLAM 노드가 `/map`을 publish하기 시작합니다.

RViz가 실행되며 다음과 같은 화면을 확인할 수 있습니다.

![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/slam1.jpg?raw=true)   

<br>

### 6-5.  RViz에서 지도 `/map` 표시
RViz에서 아래 설정을 합니다.  
1. Fixed Frame 변경
    - SLAM 전 : `odom`
     - SLAM 후 지도 기준으로 보기 위해 `map`으로 변경  

📌 RViz 좌측 상단 : Global Options → Fixed Frame → map

![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/slam2.JPG?raw=true)   



2. Map 디스플레이 추가
- Add → Map

![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/slam3.JPG?raw=true)   

- Topic → `/map`  

![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/slam4.JPG?raw=true)   

![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/slam5.JPG?raw=true)   


<br>

## 6-6. 로봇을 움직여서 지도 채우기
<u>**‼️터미널 4에서 실행해야 합니다.**</u>  

Telecop을 활용해 Gazebo의 TurtleBot3를 움직였던 터미널에서 키보드 값을 입력하여 로봇을 천천히 이동시키며 지도를 생성합니다.

📌 잘 작동한다면 로봇이 이동할수록 지도(격자)가 점점 채워집니다.


<details><summary> ⚠️ 키보드 입력하는 터미널을 종료했을 경우 클릭
</summary>

<br>

만약, 기존 Telecop이 활성화되었던 터미널을 종료시켰다면, 아래 명령어를 입력하여 새로 활성화하십시오.
```js
source /opt/ros/humble/setup.bash
ros2 run turtlebot3_teleop teleop_keyboard
```

<br>


</details>


<br>

**주행 팁(지도 품질 개선)**
- 갑자기 빠르게 회전하지 말기  
- 벽을 따라 천천히 이동하기  
- 한 구역을 원형으로 크게 돌아보며 닫힌 루프 만들기  


![title](https://github.com/snucurl/KSME_ROS2_RVIZ/blob/main/readme/slam6.JPG?raw=true)   

