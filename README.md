# realsense_RSUSB_Backend_Build
리얼센스 RSUSB 백엔드 설정 구조. 젯슨등의 ARM 기반 기기들에서는 커널의 유효성이 안보이기에, RSUSB를 통해 커널을 우회하게 빌드. 그과정에서 필수 사항들까지 추가



# ROS 2 RealSense 소스 빌드 및 설치 가이드 (RSUSB 백엔드)

리눅스 커널 패치 문제나 USB 권한 충돌(`RS2_USB_STATUS_ACCESS`, `failed to set power state` 등)을 피하기 위해, `apt` 패키지 대신 **RSUSB 백엔드를 강제 활성화하여 소스 코드로 직접 빌드하는 방법**입니다. 본 가이드는 ROS 2 Humble을 기준으로 작성되었습니다.

## 1. 기존 패키지 삭제 및 필수 의존성 설치

이전에 `apt`로 설치한 리얼센스 패키지가 있다면 충돌 방지를 위해 삭제하고, 빌드에 필요한 도구들과 USB 권한 설정에 필요한 `v4l-utils`를 설치합니다.

```bash
# 기존 패키지 삭제
sudo apt remove ros-humble-librealsense2* ros-humble-realsense2-* -y

# 패키지 목록 업데이트
sudo apt update

# 빌드 필수 도구 및 v4l-utils 설치
sudo apt install -y git libssl-dev libusb-1.0-0-dev libudev-dev pkg-config cmake build-essential v4l-utils

```

## 2. librealsense (SDK) 소스 빌드

인텔 리얼센스 공식 SDK를 다운로드하고, RSUSB 옵션을 켜서 빌드합니다.

```bash
cd ~
git clone https://github.com/IntelRealSense/librealsense.git
cd librealsense

# 빌드 디렉토리 생성
mkdir build && cd build

# RSUSB 백엔드 강제 활성화 설정으로 CMake 실행
cmake ../ -DFORCE_RSUSB_BACKEND=ON -DCMAKE_BUILD_TYPE=Release -DBUILD_EXAMPLES=false -DBUILD_GRAPHICAL_EXAMPLES=false

# 컴파일 및 설치 (코어 수에 맞춰 병렬 빌드)
make -j$(nproc)
sudo make install

```

## 3. Udev 권한 설정 및 하드웨어 인식 테스트

일반 사용자가 카메라 USB 디바이스에 접근할 수 있도록 권한을 부여합니다.

```bash
cd ~/librealsense
sudo ./scripts/setup_udev_rules.sh

```

> **⚠️ 매우 중요한 주의사항:**
> 스크립트 실행이 완료되면, **반드시 카메라의 USB 케이블을 PC에서 완전히 분리했다가 3초 뒤에 다시 연결**해야 권한이 정상적으로 적용됩니다. (파란색 USB 3.0 포트 사용 권장)

카메라가 정상적으로 인식되었는지 확인합니다.

```bash
rs-enumerate-devices -c

```

*성공 시 카메라의 Extrinsic Parameter(자이로, 뎁스, 컬러 등의 캘리브레이션 데이터)가 출력됩니다.*

## 4. ROS 2 realsense-ros 래퍼 빌드

ROS 2와 카메라를 연결해 주는 노드 패키지를 워크스페이스에 다운로드하고 빌드합니다. 이미 소스로 설치한 `librealsense2`를 `rosdep`이 덮어쓰지 못하도록 예외 처리하는 것이 핵심입니다.

```bash
# 워크스페이스 및 src 폴더 생성
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src

# ROS 2 래퍼 소스 코드 다운로드
git clone -b ros2-development https://github.com/IntelRealSense/realsense-ros.git

# 워크스페이스 루트로 이동
cd ~/ros2_ws

# 의존성 설치 (소스 빌드한 SDK 제외)
rosdep install -i --from-path src --rosdistro humble --skip-keys=librealsense2 -y

# 전체 패키지 빌드
colcon build

```

## 5. 실행 및 데이터 확인

빌드가 완료되면 환경 변수를 적용하고 카메라 노드를 실행합니다.

**카메라 노드 실행 (터미널 1):**

```bash
source ~/ros2_ws/install/local_setup.bash
ros2 launch realsense2_camera rs_launch.py

```

**토픽 확인 및 시각화 (터미널 2):**

```bash
source ~/ros2_ws/install/local_setup.bash

# 발행 중인 토픽 리스트 확인
ros2 topic list

# rqt_image_view를 통한 실시간 2D 영상 확인
ros2 run rqt_image_view rqt_image_view

```

*(rqt 화면 좌측 상단 드롭다운에서 `/camera/camera/color/image_raw`를 선택하면 컬러 화면을 볼 수 있습니다.)*
