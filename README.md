### 환경 설정

- realsense github 주소 `git clone` 을 해준다.
    
- `librealsense2` 설치
    
    ```bash
    sudo mkdir -p /etc/apt/keyrings
    curl -fsSL https://librealsense.intel.com/Debian/librealsense.pgp | sudo gpg --dearmor -o /etc/apt/keyrings/librealsense.gpg
    echo "deb [signed-by=/etc/apt/keyrings/librealsense.gpg] https://librealsense.intel.com/Debian/apt-repo $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/librealsense.list
    sudo apt update
    sudo apt install -y librealsense2-dkms librealsense2-utils librealsense2-dev
    
    ```
    
- `librealsense2` 버전확인 
2.56.5면 오케이
    
    ```bash
    pkg-config --modversion realsense2
    ```
    
- `colcon build --symlink-install`
- 실행
    
    ```bash
    ros2 launch realsense2_camera rs_launch.py device_type:='d435(?!i)' pointcloud.enable:=true align_depth.enable:=true
    ```
    

### PointCloud2 분석

- `sensor_msgs/msg/PointCloud2` 의 topic 분석
    - message type `sensor_msgs/msg/PointCloud2`
        
        ```bash
        # This message holds a collection of N-dimensional points, which may
        # contain additional information such as normals, intensity, etc. The
        # point data is stored as a binary blob, its layout described by the
        # contents of the "fields" array.
        #
        # The point cloud data may be organized 2d (image-like) or 1d (unordered).
        # Point clouds organized as 2d images may be produced by camera depth sensors
        # such as stereo or time-of-flight.
        
        # Time of sensor data acquisition, and the coordinate frame ID (for 3d points).
        std_msgs/Header header
        	builtin_interfaces/Time stamp
        		int32 sec
        		uint32 nanosec
        	string frame_id
        
        # 2D structure of the point cloud. If the cloud is unordered, height is
        # 1 and width is the length of the point cloud.
        uint32 height
        uint32 width
        
        # Describes the channels and their layout in the binary data blob.
        PointField[] fields
        	uint8 INT8    = 1
        	uint8 UINT8   = 2
        	uint8 INT16   = 3
        	uint8 UINT16  = 4
        	uint8 INT32   = 5
        	uint8 UINT32  = 6
        	uint8 FLOAT32 = 7
        	uint8 FLOAT64 = 8
        	string name      #
        	uint32 offset    #
        	uint8  datatype  #
        	uint32 count     #
        
        bool    is_bigendian # Is this data bigendian?
        uint32  point_step   # Length of a point in bytes
        uint32  row_step     # Length of a row in bytes
        uint8[] data         # Actual point data, size is (row_step*height)
        
        bool is_dense        # True if there are no invalid points
        
        ```
        
        - Unix Time을 알려줌
            - 1970년 1월 1일을 기준으로 얼마나 시간이 지났는지
        - frame_id 기준 좌표계 이름
            - `camera_link` : 로봇 기준 좌표계 (카메라 중심 점 기준)
                - x축 : 전방
                - y축 : 왼쪽
                - Z축 : 상단
            - `camera_color_frame` :  RGB 렌즈의 물리적 위치
                - RGB 렌즈 : 일반 카메라
                - 축은 `camera_link` 와 동일
            - `camera_depth_frame` :  뎁스 렌즈의 물리적 위치
                - 뎁스 렌즈 : 레이저, 적외선을 쏴서 거리를 잼.
                - 축은 `camera_link` 와 동일
            - `camera_color_optical_frame` : RGB 렌즈의 컴퓨터 비전 좌표계
                - x축 : 이미지 오른쪽 방향
                - y축 : 이미지 아래쪽 방
                - z축 : 카메라가 바라보는 정면 깊이 방향
            - `depth_depth_optical_frame` : 뎁스 렌즈의 컴퓨터 비전 좌표계
                - 축은 `camera_color_optical_frame` 과 동일
        - 1차원이 CPU/GPU 계산 성능이 좋음 → 1차원으로 정렬
        - FLOAT32 = 4byte
        - fields 구성 = x,y,z, rgb
        - offset : filed가 몇 번지에서 시작하는지
        - is_bigendian : false ⇒ Little-endian :
            - 큰 번지의 숫자 부터 저장 = LSB 부터 저장
            - LSB(Little Significant Bit) : 가장 오른쪽에 위치한 데이터 (가장 작은 값)
            - MSB(Most Significant Bit) : 가장 왼쪽에 위치한 데이터 (가장 큰 값)
        - point_step  점 하나의 크기
        - row_step  전체 크기
            - width* point_step = row_step
        - x,y,z 점 계산법
            - IEEE 754를 이용
                - 인텔이 의뢰해서 윌리엄 카한(William Kahan) 아저씨가 개발한 32bit 실수 표현 공식
                - FLOAT32 = 32bit
                - 32bit 구조
                    - s(sign) : 1bit, 부호 비트
                    - e(Exponent) : 8bit, 지수 비트
                    - f(Rraction/Mantissa) 23bit, 가수 비트 (실제 유효 숫자 부분, 소수 점 아래 자리)
                - IEEE 754 공식
                    
                    $$
                    \text{Value} = (-1)^s \times (1 + f) \times 2^{(e - 127)}
                    $$
                    
                    - $2^{(e - 127)}$에서 -127를 하는 이유
                        - 지수의 범위가 0~255(8bit)
                        - 가장 작은 수 부터, 가장 큰 수를 표현하기 위해서
                - 예시
                    - 원본 x좌표 데이터
                        - [149, 5, 5, 192]
                        - Little-endian이기 때문에 192, 5, 5, 149가 들어감.
                    - 2진수 변환
                        - 11000000000001010000010110010101 (32bit)
                    - 32bit 구조 분석
                    - s = 1
                    - e = 10000000 → 128
                    - f = 00001010000010110010101
                        - 소수점 아래 자리를 의미
                        - 다 더하면 0.03923…
                    - 공식에 대입
                        
                        $$
                        \text{Value} = (-1)^1 \times (1 + 0.03923...) \times 2
                        $$
                        
                        - $\text{Value} \approx -2.07846$
        - is_dense 모든 점이 유효한 x,y,z 값을 가지고 있는지