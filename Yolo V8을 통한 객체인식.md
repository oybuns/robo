# 

# Yolo V8을 통한 객체인식

이전까지 카메라 노드까지 완성해야한다.



필수 라이브러리 설치

```
pip install ultralytics opencv-python cvbridge3
```



실행 파이썬 파일을 만든다.

```
nano yolov8_node.py
```



파이썬 코드 내용

```
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import Image
from cv_bridge import CvBridge
from ultralytics import YOLO
import cv2
import numpy as np

class Yolov8Node(Node):
    def __init__(self):
        super().__init__('yolov8_node')
 
        self.bridge = CvBridge()

        self.get_logger().info('YOLOv8 모델을 불러오는 중...')
        self.model = YOLO('yolov8n.pt')
        self.get_logger().info('YOLOv8 모델 로드 완료!')

            # 1. 원본 이미지 구독 (Subscribe)
        self.subscription = self.create_subscription(
           Image,
           '/image_raw', # 원본 카메라 토픽
           self.image_callback,10)

    # 💡 2. [추가] 결과 이미지를 발행할 퍼블리셔(Publisher) 생성
        self.publisher_ = self.create_publisher(
           Image,
           '/yolo/result_image', # RViz에서 볼 새로운 토픽 이름
            10)

    def image_callback(self, msg):
        try:
        # yuv422_yuy2 수동 변환 유지
            if msg.encoding in ['yuv422_yuy2', 'yuyv']:
                img_data = np.frombuffer(msg.data, dtype=np.uint8).reshape(msg.height, msg.width, 2)
                cv_image = cv2.cvtColor(img_data, cv2.COLOR_YUV2BGR_YUYV)
            else:
                cv_image = self.bridge.imgmsg_to_cv2(msg, desired_encoding='bgr8')

        except Exception as e:
            self.get_logger().error(f'이미지 변환 에러: {e}')
            return

        if cv_image is None or cv_image.size == 0:
            return

        try:
            # YOLOv8 추론 실행
            results = self.model(cv_image, verbose=False)

            # 결과 이미지를 OpenCV 배열로 받기
            annotated_frame = results[0].plot()

            # 💡 3. [추가] OpenCV 이미지를 다시 ROS 2 Image 메시지로 변환
            result_msg = self.bridge.cv2_to_imgmsg(annotated_frame, encoding="bgr8")

            # 💡 4. [추가] RViz에서 시간 동기화를 위해 원본 이미지의 헤더(시간, 프레임 정보)를 복사
            result_msg.header = msg.header

            # 토픽 발행!
            self.publisher_.publish(result_msg)

            # (참고) 파이썬 기본 창은 이제 필요 없으므로 주석 처리합니다.
            # cv2.imshow("YOLOv8 Detection", annotated_frame)
            # cv2.waitKey(1)

        except Exception as e:
            self.get_logger().error(f'YOLO 추론 중 에러 발생: {e}')


def main(args=None):
    rclpy.init(args=args)
    yolov8_node = Yolov8Node()

    try:
        rclpy.spin(yolov8_node)
    except KeyboardInterrupt:
        yolov8_node.get_logger().info('사용자에 의해 노드가 종료되었습니다.')
    finally:
        yolov8_node.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
```



* numpy에 관련된 에러가 날 경우 아래 라이브러리를 재설치한다.

```
pip3 install --upgrade --force-reinstall numpy
```

```
pip3 install --upgrade ultralytics
```



- 계속 에러가 나면 버전을 낮춘다

```
pip3 install "numpy<2.0.0" --force-reinstall
```

```
pip3 install --ignore-installed matplotlib
```





## 파이썬 실행

```
python3 yolov8_node.py
```

 실행되면 토픽이 시작된다.



## 시작화 확인

```
ros2 run rqt_image_view rqt_image_view
```

객체 인식이 되는지 확인한다.



## rviz2 실행

```
rviz2
```

add > by topic > yolov8> image 확인


