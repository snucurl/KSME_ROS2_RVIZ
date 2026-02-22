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
             │  (벽, 장애물, 물리엔진)│
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
      ┌──────────────┬──────────────┬──────────────┐
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

<details><summary> ⚠️만약, RViz가 없다면
</summary>

<br>

```js
sudo apt install ros-humble-rviz2
```
</details>



##  🖥 3️⃣ Gazebo 설치 및 실행
##  🖥 4️⃣ 토픽 발생 확인
##  🖥 5️⃣ RViz 실행
##  🖥 6️⃣ TurtleBot3 Burger 제어
##  🖥 7️⃣ SLAM 실행 (지도 생성)
4) “LaserScan 시각화(A)”와 “SLAM”의 차이
✅ A: LaserScan 시각화 (간단/직관)

필요한 것: /scan + 적절한 Fixed Frame(대개 odom 또는 base_link)

RViz에서 벽이 점으로 찍히는 것을 확인

✅ SLAM: 지도 생성 (한 단계 더)

필요한 것: /scan + /tf(+ 종종 /odom)

결과: /map 생성 → RViz에서 격자 지도가 누적됨



## 🙌 안녕하세요. EASYME.md를 만든 원아입니다!
![easyme](/assets/readme/cartoon.png)   

## ❓ EASYME.md가 뭐예요?   
- **EASYME.md**는 **<u>개발자가 README.md를 좀 더 쉽게 작성할 수 있도록</u>** 하기 위해 만들었어요.   
- 블로그에서 글을 쓰는 것처럼 쉽게 글을 작성하고 스타일을 적용하면 오른쪽(👉)에 미리보기로 확인하실 수 있어요.   
- 스타일을 적용하면 마크다운 문법 및 md 파일에서 인식할 수 있는 소스코드가 자동으로 적용돼요.   
- 커서 위치, 드래그한 영역 등에 따라 스타일을 적용할 수 있으니 자유롭게 사용해보세요!
- 복사하기를 통해 본문 내용을 복사하고 여러분의 README에 적용해보세요.   

## 🙋‍♀️ 좀 더 구체적으로 가르쳐주세요!   
1. 왼쪽 공간에서 블로그에 글을 쓰는 것처럼 README를 작성해주세요!   
2. 👆 위에 툴바창에 보이는 다양한 스타일을 적용해보세요!   
3. 다 작성하셨나요? 예쁘게 잘 나왔는지 오른쪽 미리보기 화면에서 확인해보세요.   
4. 오른쪽에 작성한 글 전체를 복사하세요!   
(저장을 원할 경우 `Ctrl + S` / `Command + S` 또는 툴바창 제일 오른쪽에 `공유하기 아이콘`을 클릭해주세요.)   
5. 이제 여러분의 **README.md** 에 붙여넣으세요!   
(저장 또는 공유를 할 경우 링크를 다른 사람에게 전달할 수 있어요! 😀)  

## 🛠 기능 엿보기   

1. [❓ EASYME.md가 뭐예요?  ](#-easymemd가-뭐예요)
2. [🙋‍♀️ 좀 더 구체적으로 가르쳐주세요!](#-좀-더-구체적으로-가르쳐주세요)
3. [🛠 기능 엿보기](#-기능-엿보기)
    - [Header](#header)   
    - [Text Style1](#text-style1)   
    - [Text Stlye2](#text-style2)   
    - [List](#list)      
    - [Link](#link)   
    - [Code Block](#code-block)   
    - [Table](#table)   
   
## Header
- # H1 Header   
- ## H2 Header   
- ### H3 Header   
- #### H4 Header   
- ##### H5 Header   
- ###### H6 Header   

<br>   

## Text Style1
- **진하게** (`Ctrl(Command) + B`)   
- *기울이기* (`Ctrl(Command) + I`)   
- <s>취소선</s> (`Ctrl(Command) + D`)   
- <u>밑줄</u> (`Ctrl(Command) + U`)   

<br>   
   
## Text Style2

>인용문   
   
<details><summary>접고 펴는 기능
</summary>

*Write here!*
</details>

- EASYME.md를 드래그하고 상단에 `Aa` 아이콘을 누르면? 👉 Easyme.md   
- EASYME.md를 드래그하고 상단에 `A` 아이콘을 누르면? 👉 EASYME.MD   
- EASYME.md를 드래그하고 상단에 `a` 아이콘을 누르면? 👉 easyme.md   
   
<br>   
   
## List   
### Table of contents
1. [title1](#write-title-here!)   
2. [title2](#only-lowercase)   
3. [title3](#use"-"instead-of-spacing-words)   
4. [title4](#example)   
    - [❓ EASYME.md가 뭐예요?](#-easymemd가-뭐예요)   
    - [🛠 기능 엿보기](#-기능-엿보기)
   
### Unordered list   
- unordered list1   
- unordered list2   
- unordered list3   
- unordered list4   
   
### Ordered list   
1. ordered list1   
2. ordered list2   
3. ordered list3   
4. ordered list4   
   
<br>   
   
## Link   
### General link
- [🚗 Visit EASYME.md's Repo](https://github.com/EASYME-md/client)   
- [🙋‍♂️ Visit ONE:A's Github](https://github.com/onealog)

### Image link
![onealog](/assets/readme/easyme.png)   
   
<br>   
   
## Code Block   
### Code inline
- `console.log('Hello EASYME.md!');`   
   
### Code block
```js
function makeDeveloper(name, language) {
  if (name === 'ONE:A' && language === 'JavaScript') {
    return 'perfect!';
  }

  return false;
}

makeDeveloper('ONE:A', 'JavaScript');
```

<br>   
   
## Table   


| title1 | title2 | title3 |
| --- | --- | --- |
| 1 | 2 | 3 |
| 4 | 5 | 6 |
| 7 | 8 | 9 |


<br>   

