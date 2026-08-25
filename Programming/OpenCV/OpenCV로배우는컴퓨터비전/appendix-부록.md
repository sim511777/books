# 부록

[◀ 이전: 18장. 실전 프로젝트와 성능 최적화](ch18-실전프로젝트와성능최적화.md) | [📖 목차](00-목차.md)


이 부록은 본문에서 다룬 내용을 찾아보기 쉽게 정리한 Python 중심 참고 자료다. 새로운 개념을 설명하지는 않으며, 개발 환경 설치 요약, 이 책 전체에서 사용한 주요 `cv2` 함수의 레퍼런스, 자주 마주치는 오류의 해결 가이드, 그리고 앞으로 더 찾아볼 만한 공식 자료와 학습 방향을 모아두었다.

## A.1 Python 개발 환경 설치 요약

2장에서 다룬 설치 절차를 다시 정리한다. 패키지 이름과 버전은 시간이 지나며 바뀔 수 있으므로, 실제 설치 시에는 PyPI의 최신 안내를 함께 확인하는 것이 좋다.

| 패키지 | 용도 | 비고 |
|---|---|---|
| `opencv-python` | 기본 배포판, GUI 기능(`imshow` 등) 포함 | 데스크톱 환경에서 가장 흔히 사용 |
| `opencv-python-headless` | GUI 기능을 뺀 경량 배포판 | 서버/컨테이너/CI 환경에 적합, `libGL` 관련 오류를 피할 수 있다 |
| `opencv-contrib-python` | 추가·특허 관련 모듈(SIFT의 일부 구현, 일부 트래커 등) 포함 | 표준 배포판에 없는 기능이 필요할 때 |
| `opencv-contrib-python-headless` | contrib 모듈 + headless | 서버 환경에서 contrib 기능까지 필요할 때 |

같은 환경에 `opencv-python`과 `opencv-python-headless`처럼 서로 겹치는 배포판을 동시에 설치하면 내부 파일이 충돌해 예기치 않은 오류가 날 수 있으므로, **넷 중 하나만** 설치하는 것이 원칙이다.

```bash
# 가장 일반적인 데스크톱 개발 환경
pip install opencv-python numpy

# 가상환경을 새로 만들어 격리된 상태로 설치하는 것을 권장한다
python -m venv .venv
# Windows
.venv\Scripts\activate
# macOS / Linux
source .venv/bin/activate
pip install --upgrade pip
pip install opencv-python numpy

# 화면이 없는 서버(도커 컨테이너 등)라면 headless 배포판을 사용한다
pip install opencv-python-headless

# SIFT 등 contrib 전용 기능이 필요하다면
pip install opencv-contrib-python
```

버전을 고정해 재현 가능한 환경을 만들고 싶다면 `requirements.txt`에 다음과 같이 명시해둘 수 있다.

```text
opencv-python==4.10.0.84
numpy==1.26.4
```

```bash
pip install -r requirements.txt
```

## A.2 `cv2` 함수 레퍼런스

이 책 전체에서 사용한 주요 함수를 장별·기능별로 정리했다. "핵심 매개변수"는 실무에서 가장 자주 조정하는 값만 추렸으며, 전체 옵션은 각 함수의 공식 문서에서 확인할 수 있다.

### 입출력과 창 관리 (4, 16장)

| 함수 | 핵심 매개변수 | 반환값 | 사용 장 |
|---|---|---|---|
| `cv2.imread(filename, flags=cv2.IMREAD_COLOR)` | `filename`(경로), `flags`(컬러/그레이스케일 등 읽기 모드) | 이미지를 담은 NumPy 배열, 실패 시 `None` | 4 |
| `cv2.imwrite(filename, img, params=None)` | `filename`, `img`, 포맷별 압축 파라미터 | 성공 여부(`bool`) | 4, 18 |
| `cv2.imshow(winname, mat)` | `winname`(창 이름), `mat`(표시할 배열) | 없음 | 4 |
| `cv2.waitKey(delay=0)` | `delay`(ms, 0이면 무한 대기) | 눌린 키의 코드(`int`), 없으면 -1 | 4 |
| `cv2.destroyAllWindows()` | 없음 | 없음 | 4 |
| `cv2.VideoCapture(index_or_path, apiPreference=None)` | 카메라 인덱스 또는 파일 경로 | `VideoCapture` 객체 | 4, 16 |
| `cap.read()` | 없음 | `(성공 여부, 프레임 배열)` 튜플 | 16 |

### 그리기와 마우스 상호작용 (6, 18장)

| 함수 | 핵심 매개변수 | 반환값 | 사용 장 |
|---|---|---|---|
| `cv2.rectangle(img, pt1, pt2, color, thickness=1)` | 대각선 두 꼭짓점, 색상, 선 두께(-1이면 채우기) | 없음(제자리에서 img 수정) | 6, 17, 18 |
| `cv2.circle(img, center, radius, color, thickness=1)` | 중심, 반지름, 색상 | 없음 | 6, 18 |
| `cv2.line(img, pt1, pt2, color, thickness=1)` | 시작점, 끝점, 색상 | 없음 | 6 |
| `cv2.putText(img, text, org, fontFace, fontScale, color, thickness=1)` | 텍스트, 시작 좌표, 폰트, 크기, 색상 | 없음 | 6, 17 |
| `cv2.polylines(img, pts, isClosed, color, thickness=1)` | 점들의 배열 리스트, 닫힘 여부 | 없음 | 18 |
| `cv2.setMouseCallback(winname, onMouse, param=None)` | 창 이름, 이벤트 콜백 함수 | 없음 | 6, 18 |

### 색공간 변환과 전처리 (5, 9, 14장)

| 함수 | 핵심 매개변수 | 반환값 | 사용 장 |
|---|---|---|---|
| `cv2.cvtColor(src, code)` | `code`(예: `cv2.COLOR_BGR2GRAY`, `cv2.COLOR_BGR2HSV`) | 변환된 배열 | 5, 17, 18 |
| `cv2.split(m)` | 채널을 가진 배열 | 채널별 배열의 튜플 | 7 |
| `cv2.merge(mv)` | 합칠 채널 배열들의 리스트 | 합쳐진 다채널 배열 | 7 |
| `cv2.GaussianBlur(src, ksize, sigmaX)` | 커널 크기(홀수), 표준편차 | 블러 처리된 배열 | 9, 17, 18 |
| `cv2.medianBlur(src, ksize)` | 커널 크기(홀수) | 블러 처리된 배열 | 9 |
| `cv2.bilateralFilter(src, d, sigmaColor, sigmaSpace)` | 필터 지름, 색상/공간 표준편차 | 엣지를 보존한 블러 배열 | 9 |
| `cv2.equalizeHist(src)` | 8비트 단일 채널(그레이스케일) 입력 | 대비가 평활화된 배열 | 14, 17 |
| `cv2.calcHist(images, channels, mask, histSize, ranges)` | 이미지 리스트, 채널 인덱스, 빈 개수, 값 범위 | 히스토그램 배열 | 14 |
| `cv2.inRange(src, lowerb, upperb)` | 하한/상한 색상 범위 | 이진 마스크 배열 | 5 |

### 기하 변환 (8장)

| 함수 | 핵심 매개변수 | 반환값 | 사용 장 |
|---|---|---|---|
| `cv2.resize(src, dsize, fx=0, fy=0, interpolation=cv2.INTER_LINEAR)` | 목표 크기 또는 배율, 보간 방법 | 크기가 조절된 배열 | 8, 18 |
| `cv2.getRotationMatrix2D(center, angle, scale)` | 회전 중심, 각도(도), 배율 | 2×3 회전 행렬 | 8 |
| `cv2.warpAffine(src, M, dsize)` | 2×3 변환 행렬, 출력 크기 | 변환된 배열 | 8 |
| `cv2.getPerspectiveTransform(src, dst)` | 원본 4점, 목표 4점(각각 `float32` 배열) | 3×3 원근 변환 행렬 | 8, 18 |
| `cv2.warpPerspective(src, M, dsize)` | 3×3 변환 행렬, 출력 크기 | 변환된 배열 | 8, 18 |

### 임계처리와 모폴로지 (10, 11장)

| 함수 | 핵심 매개변수 | 반환값 | 사용 장 |
|---|---|---|---|
| `cv2.threshold(src, thresh, maxval, type)` | 임계값, 최댓값, 임계처리 방식(`cv2.THRESH_BINARY` 등) | `(사용된 임계값, 결과 배열)` | 10 |
| `cv2.adaptiveThreshold(src, maxValue, adaptiveMethod, thresholdType, blockSize, C)` | 지역 블록 크기, 보정값 `C` | 이진화된 배열 | 10 |
| `cv2.dilate(src, kernel, iterations=1)` | 구조 요소, 반복 횟수 | 팽창된 배열 | 11 |
| `cv2.erode(src, kernel, iterations=1)` | 구조 요소, 반복 횟수 | 침식된 배열 | 11 |
| `cv2.morphologyEx(src, op, kernel)` | 연산 종류(`cv2.MORPH_OPEN`, `cv2.MORPH_CLOSE` 등) | 처리된 배열 | 11 |

### 엣지와 컨투어 (12, 13, 18장)

| 함수 | 핵심 매개변수 | 반환값 | 사용 장 |
|---|---|---|---|
| `cv2.Canny(image, threshold1, threshold2)` | 하위/상위 임계값 | 엣지 이진 이미지 | 12, 18 |
| `cv2.findContours(image, mode, method)` | 검색 모드(`cv2.RETR_EXTERNAL` 등), 근사 방법 | `(contours, hierarchy)` | 13, 18 |
| `cv2.drawContours(image, contours, contourIdx, color, thickness=1)` | 윤곽선 리스트, 인덱스(-1이면 전체) | 없음 | 13 |
| `cv2.contourArea(contour)` | 윤곽선(점들의 배열) | 면적(`float`) | 13, 18 |
| `cv2.arcLength(curve, closed)` | 윤곽선, 폐곡선 여부 | 둘레 길이(`float`) | 13, 18 |
| `cv2.approxPolyDP(curve, epsilon, closed)` | 근사 정밀도(작을수록 원본에 가까움), 폐곡선 여부 | 근사된 꼭짓점 배열 | 13, 18 |
| `cv2.boundingRect(array)` | 윤곽선 또는 점 배열 | `(x, y, w, h)` | 13 |
| `cv2.minAreaRect(points)` | 점 배열 | 회전된 사각형 정보(중심, 크기, 각도) | 13 |

### 특징점 검출과 매칭 (15장)

| 함수 | 핵심 매개변수 | 반환값 | 사용 장 |
|---|---|---|---|
| `cv2.ORB_create(nfeatures=500)` | 검출할 최대 특징점 개수 | `ORB` 객체 | 15 |
| `orb.detectAndCompute(image, mask)` | 입력 이미지, 관심 영역 마스크 | `(키포인트 리스트, 기술자 배열)` | 15 |
| `cv2.BFMatcher(normType, crossCheck)` | 거리 척도, 상호 검증 여부 | `BFMatcher` 객체 | 15 |
| `matcher.match(desc1, desc2)` | 두 기술자 배열 | 매칭 결과(`DMatch`) 리스트 | 15 |
| `cv2.drawMatches(img1, kp1, img2, kp2, matches, outImg)` | 두 이미지와 각각의 키포인트, 매칭 결과 | 매칭을 시각화한 배열 | 15 |

### 비디오와 배경 제거 (16장)

| 함수 | 핵심 매개변수 | 반환값 | 사용 장 |
|---|---|---|---|
| `cv2.createBackgroundSubtractorMOG2(history, varThreshold, detectShadows)` | 학습에 사용할 프레임 수, 분산 임계값, 그림자 검출 여부 | 배경 제거기 객체 | 16 |
| `subtractor.apply(frame)` | 현재 프레임 | 전경 마스크 배열 | 16 |

### 얼굴/객체 검출과 DNN (17장)

| 함수 | 핵심 매개변수 | 반환값 | 사용 장 |
|---|---|---|---|
| `cv2.CascadeClassifier(filename)` | Haar Cascade XML 경로 | `CascadeClassifier` 객체 | 17 |
| `classifier.detectMultiScale(image, scaleFactor, minNeighbors, minSize, maxSize)` | 스케일 비율, 최소 이웃 수, 검출 크기 범위 | 사각형 `(x, y, w, h)`의 배열 | 17 |
| `cv2.dnn.readNetFromCaffe(prototxt, caffeModel)` | 구조 설명 파일, 가중치 파일 경로 | `Net` 객체 | 17 |
| `cv2.dnn.readNetFromONNX(onnxFile)` | ONNX 모델 파일 경로 | `Net` 객체 | 17 |
| `cv2.dnn.blobFromImage(image, scalefactor, size, mean, swapRB, crop)` | 정규화 배율, 입력 크기, 채널 평균, RGB 변환 여부 | 4차원 블롭 배열 | 17 |
| `net.setInput(blob)` | 블롭 배열 | 없음 | 17 |
| `net.forward()` | 없음 | 모델별로 형식이 다른 출력 배열 | 17 |

### 성능 측정 유틸리티 (18장)

| 함수 | 핵심 매개변수 | 반환값 | 사용 장 |
|---|---|---|---|
| `cv2.getTickCount()` | 없음 | 현재 시각의 틱 카운트(`int`) | 18 |
| `cv2.getTickFrequency()` | 없음 | 초당 틱 수(`float`) | 18 |
| `time.perf_counter()` (표준 라이브러리) | 없음 | 고정밀 기준 시각(초, `float`) | 18 |

`cv2.getTickCount()`와 `cv2.getTickFrequency()`를 조합하면 `(끝 틱 - 시작 틱) / cv2.getTickFrequency()`로 경과 시간을 초 단위로 구할 수 있다. 18장의 예제처럼 순수 Python 코드 구간의 시간을 측정할 때는 표준 라이브러리의 `time.perf_counter()`를 쓰는 것이 더 간단하고 일반적이다.

## A.3 개발 환경 문제 해결 가이드

Python으로 OpenCV를 다루다 보면 반복적으로 마주치는 환경 문제들이 있다. 자주 겪는 순서대로 정리했다.

### 1. `pip install opencv-python`이 실패한다

- pip 자체가 오래된 버전이면 최신 배포판의 휠(wheel) 파일을 제대로 찾지 못하는 경우가 있다. 먼저 `python -m pip install --upgrade pip`로 pip을 갱신한다.
- 사내망이나 방화벽 환경이라면 사내 프록시 설정(`pip install --proxy`)이나 미러 저장소 지정이 필요할 수 있다.
- 32비트 Python 환경에 64비트 전용 휠을 설치하려 하면 실패한다. `python -c "import platform; print(platform.architecture())"`로 현재 Python의 아키텍처를 확인하고, 필요하면 64비트 Python을 새로 설치한다.
- 가상환경 없이 시스템 Python에 설치하면 다른 패키지와 충돌하는 경우가 있으므로, 가능하면 `venv`나 `conda` 가상환경 안에서 설치한다.

### 2. Windows에서 `ImportError: DLL load failed while importing cv2`

- 대부분 Visual C++ 재배포 패키지(Visual C++ Redistributable)가 시스템에 없어서 발생한다. Microsoft 공식 배포 페이지에서 최신 x64 재배포 패키지를 설치하면 해결되는 경우가 많다.
- 설치된 Python의 비트(32/64)와 `opencv-python` 휠의 비트가 서로 다르면 같은 오류가 난다. `pip uninstall opencv-python` 후, 현재 Python 환경에 맞는 배포판을 다시 설치해본다.
- 시스템에 여러 버전의 Python이 설치되어 있고 `PATH` 환경변수가 뒤섞여 있으면, 의도한 것과 다른 Python/pip이 실행되어 문제가 생길 수 있다. `where python`(Windows) 또는 `which python`(macOS/Linux)으로 실제 실행되는 인터프리터 경로를 확인한다.

### 3. Linux(특히 headless 서버·도커 컨테이너)에서 `ImportError: libGL.so.1: cannot open shared object file`

- GUI가 없는 서버 환경에 GUI 기능이 포함된 `opencv-python`을 설치하면, 그 안에 딸려오는 Qt/GTK 관련 공유 라이브러리(`libGL.so.1` 등)를 시스템이 찾지 못해 발생한다.
- 가장 깔끔한 해결책은 `pip uninstall opencv-python` 후 `pip install opencv-python-headless`로 바꾸는 것이다. headless 배포판은 `cv2.imshow` 같은 창 표시 기능을 아예 포함하지 않으므로 이 오류가 원천적으로 발생하지 않는다.
- 창 표시 기능이 꼭 필요한데 서버 환경이라 GUI가 없다면, `apt-get install -y libgl1`(Debian/Ubuntu 계열)처럼 필요한 시스템 라이브러리를 직접 설치해 해결할 수도 있다.

### 4. Jupyter Notebook/JupyterLab에서 `cv2.imshow`가 뜨지 않거나 커널이 멈춘다

- `cv2.imshow`는 OS의 네이티브 GUI 창을 띄우는 함수인데, Jupyter의 커널(특히 원격 서버나 Docker 위에서 실행되는 커널)에는 이 창을 표시할 화면 자체가 없는 경우가 많아 창이 뜨지 않거나 커널이 응답하지 않는 것처럼 보인다.
- 로컬 데스크톱에서 실행 중인 Jupyter라도 `cv2.imshow` + `cv2.waitKey`의 이벤트 루프가 노트북 환경과 잘 맞물리지 않아 창이 멈춘 것처럼 보이는 경우가 있다.
- 가장 안정적인 대안은 `matplotlib`으로 이미지를 셀 안에 바로 표시하는 것이다. OpenCV는 BGR 순서를 쓰므로 표시 전에 RGB로 바꿔줘야 한다.

```python
import cv2
import matplotlib.pyplot as plt

img = cv2.imread("photo.jpg")
img_rgb = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)

plt.imshow(img_rgb)
plt.axis("off")
plt.show()
```

### 5. `cv2.VideoCapture(0)`이 카메라를 열지 못하거나 빈 프레임만 반환한다

- 다른 프로그램(화상 회의 앱 등)이 이미 카메라를 점유하고 있으면 열리지 않는다. 다른 프로그램을 먼저 종료해본다.
- 카메라 인덱스가 예상과 다를 수 있다. `0` 대신 `1`, `2` 등을 시도해보거나, 여러 인덱스를 순회하며 `cap.isOpened()`가 `True`인 것을 찾는다.
- Windows에서는 백엔드를 명시적으로 지정하면 안정성이 좋아지는 경우가 있다. `cv2.VideoCapture(0, cv2.CAP_DSHOW)`처럼 `apiPreference`를 지정해본다.
- `cap.read()`가 `(False, None)`을 반환하는지 항상 확인하는 방어적 코드를 넣어, 카메라가 준비되기 전 프레임을 읽으려다 발생하는 오류를 예방한다.

### 6. `opencv-python`, `opencv-contrib-python`, `opencv-python-headless`를 함께 설치해 생기는 알 수 없는 오류

- 이 배포판들은 같은 `cv2` 패키지 이름을 공유하기 때문에, 여러 개를 동시에 설치하면 파일이 서로 덮어써지거나 일부만 남아 예측할 수 없는 오류(속성이 없다는 `AttributeError` 등)가 발생할 수 있다.
- `pip uninstall opencv-python opencv-python-headless opencv-contrib-python opencv-contrib-python-headless`로 관련 패키지를 모두 제거한 뒤, 필요한 배포판 하나만 다시 설치하는 것이 가장 확실한 해결책이다.

### 7. `cv2` 임포트 시 NumPy 관련 오류(`numpy.core.multiarray failed to import` 등)

- `opencv-python`이 컴파일 시 가정한 NumPy 버전과 현재 설치된 NumPy 버전이 크게 어긋나면 발생한다.
- `pip install --upgrade numpy opencv-python`으로 두 패키지를 함께 최신 버전으로 갱신하면 대체로 해결된다.

## A.4 더 찾아볼 공식 자료

이 책은 실전 예제 중심으로 구성되어 있어, 모든 함수의 모든 옵션을 다루지는 않는다. 특정 함수의 세부 파라미터나 최신 변경 사항이 궁금할 때는 다음 자료를 확인하는 것이 가장 정확하다.

- **OpenCV 공식 웹사이트**: opencv.org — 프로젝트 소개, 다운로드, 뉴스
- **OpenCV 공식 레퍼런스 문서**: docs.opencv.org — 모든 모듈·함수의 원본 문서(Python 바인딩 시그니처도 함께 제공)
- **opencv-python 패키지 저장소**: GitHub의 `opencv/opencv-python` 저장소 — 배포판 종류(headless, contrib 포함 여부)와 빌드·설치 관련 이슈를 확인할 수 있다
- **OpenCV 소스 저장소**: GitHub의 `opencv/opencv` 저장소 — Haar Cascade XML, DNN 예제용 설정 파일 등 데이터 파일은 이 저장소의 `data/`, `samples/dnn/` 폴더에서 받을 수 있다
- **PyImageSearch**: OpenCV·Python 기반 컴퓨터 비전 튜토리얼을 꾸준히 제공하는 커뮤니티로, 이 책에서 다룬 개념들을 실전 프로젝트 형태로 확장한 자료가 많다
- **LearnOpenCV**: OpenCV의 다양한 모듈(DNN 포함)을 다루는 튜토리얼과 예제 코드를 제공하는 커뮤니티
- **Stack Overflow의 opencv 태그**: 특정 오류 메시지나 예외 상황을 검색하면 비슷한 사례와 해결책을 빠르게 찾을 수 있다

## A.5 다음 학습 단계

이 책으로 OpenCV의 기본기를 다졌다면, 다음과 같은 방향으로 학습을 이어가는 것을 권한다.

- **딥러닝 기반 컴퓨터 비전**: 17장에서 다룬 `cv2.dnn` 모듈을 더 다양한 사전 학습 모델(객체 검출, 세그멘테이션, 포즈 추정 등)과 함께 다뤄보고, 필요하다면 PyTorch나 TensorFlow로 직접 모델을 학습시켜 ONNX로 내보내 OpenCV에서 추론해보는 흐름을 경험해본다. `ultralytics` 패키지처럼 YOLO 계열 모델을 몇 줄의 Python 코드로 바로 사용할 수 있게 해주는 고수준 라이브러리도 함께 살펴볼 만하다.
- **GPU 가속과 대규모 처리**: CUDA 지원 빌드의 `cv2.cuda` 모듈이나 OpenCL 기반 `cv2.UMat`을 활용해 고해상도 영상이나 대량의 이미지를 실시간으로 처리하는 경험을 쌓아본다.
- **공개 데이터셋과 대회 플랫폼 활용**: 공개된 이미지 데이터셋으로 직접 검출·분류 파이프라인을 구성해보거나, 온라인 데이터 과학 대회 플랫폼에서 컴퓨터 비전 과제에 도전해보면 이 책에서 배운 전처리·후처리 기법들이 실전에서 어떻게 조합되는지 체감할 수 있다.
- **실시간 트래킹과 응용 프로젝트**: 웹캠이나 영상 스트림을 입력으로 받아 특정 객체를 지속적으로 추적하는 프로젝트, 또는 이 책의 문서 스캐너 예제를 확장해 실시간 모바일 스캐너 앱으로 발전시켜보는 것도 좋은 다음 단계다.
- **성능 최적화 심화**: 18장에서 이름 수준으로만 짚은 Numba·Cython을 실제로 적용해보거나, `cv2.parallel_for_`에 대응하는 Python 쪽 병렬화 패턴(멀티프로세싱, 비동기 I/O)을 더 깊이 실험해본다.

무엇을 선택하든, 이 책에서 반복해서 강조한 습관—"지금 다루는 배열의 모양(shape)과 타입(dtype)은 무엇인가", "이 함수는 무엇을 입력받고 무엇을 돌려주는가", "자동화가 실패할 수 있는 지점은 어디인가"를 항상 확인하는 습관—은 어떤 프레임워크로 옮겨가더라도 그대로 유용하게 남는다.

---

[◀ 이전: 18장. 실전 프로젝트와 성능 최적화](ch18-실전프로젝트와성능최적화.md) | [📖 목차](00-목차.md)
