# 부록

[◀ 이전: 18장. 실전 프로젝트와 성능 최적화](ch18-실전프로젝트와성능최적화.md) | [📖 목차](00-목차.md)


이 부록은 본문에서 다룬 내용을 찾아보기 쉽게 정리한 참고 자료다. 새로운 개념을 설명하지는 않으며, 2장에서 다룬 설치 절차의 요약, 이 책 전체에서 사용한 주요 함수들의 세 언어 대응표, 그리고 앞으로 더 찾아볼 만한 공식 자료와 학습 방향을 모아두었다.

## A.1 세 언어 개발 환경 설치 요약

2장에서 자세히 다룬 설치 절차를 표로 간단히 재정리한다. 버전이나 패키지명은 시간이 지나며 바뀔 수 있으므로, 실제 설치 시에는 각 배포처의 최신 안내를 함께 확인하는 것이 좋다.

| 언어 | 설치 방법 | 비고 |
|---|---|---|
| Python | `pip install opencv-python` | GUI 없는 서버 환경이라면 `opencv-python-headless`, SIFT 등 특허/추가 모듈이 필요하면 `opencv-contrib-python` |
| C++ | vcpkg(`vcpkg install opencv`) 또는 공식 소스를 CMake로 직접 빌드 | Windows에서는 Visual Studio, Linux/macOS에서는 GCC/Clang과 함께 사용 |
| C# | NuGet 패키지 `OpenCvSharp4` + 플랫폼별 런타임 패키지(`OpenCvSharp4.runtime.win`, `.runtime.ubuntu.20.04-x64` 등) | .NET 프로젝트(`dotnet add package OpenCvSharp4`)에 바로 추가 가능 |

```bash
# Python
pip install opencv-python

# C# (.NET CLI)
dotnet add package OpenCvSharp4
dotnet add package OpenCvSharp4.runtime.win
```

## A.2 API 대응표

이 책 전체에서 사용한 주요 함수를 기능별로 묶어 Python / C++ / C# 세 언어로 나란히 정리했다. 이름 규칙만 눈에 익혀두면, 처음 보는 OpenCV 함수라도 세 언어 사이를 오가며 유추하기가 훨씬 쉬워진다.

### 입출력과 창 관리

| 기능 | Python | C++ | C# |
|---|---|---|---|
| 이미지 읽기 | `cv2.imread(path)` | `cv::imread(path)` | `Cv2.ImRead(path)` |
| 이미지 저장 | `cv2.imwrite(path, img)` | `cv::imwrite(path, img)` | `Cv2.ImWrite(path, img)` |
| 창에 표시 | `cv2.imshow(name, img)` | `cv::imshow(name, img)` | `Cv2.ImShow(name, img)` |
| 키 입력 대기 | `cv2.waitKey(ms)` | `cv::waitKey(ms)` | `Cv2.WaitKey(ms)` |
| 비디오/카메라 캡처 | `cv2.VideoCapture(src)` | `cv::VideoCapture cap(src);` | `new VideoCapture(src)` |

### 색공간과 전처리

| 기능 | Python | C++ | C# |
|---|---|---|---|
| 색공간 변환 | `cv2.cvtColor(img, code)` | `cv::cvtColor(img, dst, code)` | `Cv2.CvtColor(img, dst, code)` |
| 가우시안 블러 | `cv2.GaussianBlur(img, k, s)` | `cv::GaussianBlur(img, dst, k, s)` | `Cv2.GaussianBlur(img, dst, k, s)` |
| 미디언 블러 | `cv2.medianBlur(img, k)` | `cv::medianBlur(img, dst, k)` | `Cv2.MedianBlur(img, dst, k)` |
| 히스토그램 평활화 | `cv2.equalizeHist(gray)` | `cv::equalizeHist(gray, dst)` | `Cv2.EqualizeHist(gray, dst)` |
| 색상 범위 마스킹 | `cv2.inRange(img, lo, hi)` | `cv::inRange(img, lo, hi, mask)` | `Cv2.InRange(img, lo, hi, mask)` |

### 기하 변환

| 기능 | Python | C++ | C# |
|---|---|---|---|
| 크기 조절 | `cv2.resize(img, size)` | `cv::resize(img, dst, size)` | `Cv2.Resize(img, dst, size)` |
| 회전 행렬 생성 | `cv2.getRotationMatrix2D(c, a, s)` | `cv::getRotationMatrix2D(c, a, s)` | `Cv2.GetRotationMatrix2D(c, a, s)` |
| 아핀 변환 적용 | `cv2.warpAffine(img, M, size)` | `cv::warpAffine(img, dst, M, size)` | `Cv2.WarpAffine(img, dst, M, size)` |
| 원근 변환 행렬 생성 | `cv2.getPerspectiveTransform(src, dst)` | `cv::getPerspectiveTransform(src, dst)` | `Cv2.GetPerspectiveTransform(src, dst)` |
| 원근 변환 적용 | `cv2.warpPerspective(img, M, size)` | `cv::warpPerspective(img, dst, M, size)` | `Cv2.WarpPerspective(img, dst, M, size)` |

### 임계처리와 모폴로지

| 기능 | Python | C++ | C# |
|---|---|---|---|
| 임계처리 | `cv2.threshold(img, t, max, type)` | `cv::threshold(img, dst, t, max, type)` | `Cv2.Threshold(img, dst, t, max, type)` |
| 적응형 임계처리 | `cv2.adaptiveThreshold(...)` | `cv::adaptiveThreshold(...)` | `Cv2.AdaptiveThreshold(...)` |
| 팽창 | `cv2.dilate(img, kernel)` | `cv::dilate(img, dst, kernel)` | `Cv2.Dilate(img, dst, kernel)` |
| 침식 | `cv2.erode(img, kernel)` | `cv::erode(img, dst, kernel)` | `Cv2.Erode(img, dst, kernel)` |
| 일반 모폴로지 연산 | `cv2.morphologyEx(img, op, kernel)` | `cv::morphologyEx(img, dst, op, kernel)` | `Cv2.MorphologyEx(img, dst, op, kernel)` |

### 엣지와 컨투어

| 기능 | Python | C++ | C# |
|---|---|---|---|
| 엣지 검출 | `cv2.Canny(img, t1, t2)` | `cv::Canny(img, dst, t1, t2)` | `Cv2.Canny(img, dst, t1, t2)` |
| 윤곽선 검출 | `cv2.findContours(img, mode, method)` | `cv::findContours(img, contours, mode, method)` | `Cv2.FindContours(img, out contours, ..., mode, method)` |
| 윤곽선 그리기 | `cv2.drawContours(img, cs, idx, color)` | `cv::drawContours(img, cs, idx, color)` | `Cv2.DrawContours(img, cs, idx, color)` |
| 윤곽선 면적 | `cv2.contourArea(c)` | `cv::contourArea(c)` | `Cv2.ContourArea(c)` |
| 다각형 근사 | `cv2.approxPolyDP(c, eps, closed)` | `cv::approxPolyDP(c, approx, eps, closed)` | `Cv2.ApproxPolyDP(c, eps, closed)` |

### 히스토그램과 특징점

| 기능 | Python | C++ | C# |
|---|---|---|---|
| 히스토그램 계산 | `cv2.calcHist([img], [0], None, [256], [0,256])` | `cv::calcHist(&img, 1, ..., hist, ...)` | `Cv2.CalcHist(imgs, ch, mask, hist, ...)` |
| ORB 생성 | `cv2.ORB_create()` | `cv::ORB::create()` | `ORB.Create()` |
| 특징점 검출+기술자 | `orb.detectAndCompute(img, None)` | `orb->detectAndCompute(img, noArray(), kp, desc)` | `orb.DetectAndCompute(img, null, out kp, desc)` |
| 특징점 매칭기 | `cv2.BFMatcher()` | `cv::BFMatcher matcher;` | `new BFMatcher()` |

### 도형 그리기와 검출

| 기능 | Python | C++ | C# |
|---|---|---|---|
| 사각형 그리기 | `cv2.rectangle(img, p1, p2, color)` | `cv::rectangle(img, p1, p2, color)` | `Cv2.Rectangle(img, p1, p2, color)` |
| 텍스트 그리기 | `cv2.putText(img, text, org, font, scale, color)` | `cv::putText(img, text, org, font, scale, color)` | `Cv2.PutText(img, text, org, font, scale, color)` |
| Haar Cascade 검출기 | `cv2.CascadeClassifier(path)` | `cv::CascadeClassifier` | `new CascadeClassifier(path)` |
| 다중 스케일 검출 | `clf.detectMultiScale(gray)` | `clf.detectMultiScale(gray, faces)` | `clf.DetectMultiScale(gray)` |
| DNN 모델 로드(ONNX) | `cv2.dnn.readNetFromONNX(path)` | `cv::dnn::readNetFromONNX(path)` | `CvDnn.ReadNetFromOnnx(path)` |

## A.3 더 찾아볼 공식 자료

이 책은 실전 예제 중심으로 구성되어 있어, 모든 함수의 모든 옵션을 다루지는 않는다. 특정 함수의 세부 파라미터나 최신 변경 사항이 궁금할 때는 다음 공식 자료를 확인하는 것이 가장 정확하다.

- **OpenCV 공식 웹사이트**: opencv.org — 프로젝트 소개, 다운로드, 뉴스
- **OpenCV 공식 레퍼런스 문서**: docs.opencv.org — C++ API의 원본 문서로, Python/기타 바인딩 문서도 여기서 함께 제공된다
- **opencv-python 패키지 정보**: pypi.org의 `opencv-python` 프로젝트 페이지 — 배포판 종류(headless, contrib 포함 여부)와 설치 안내
- **OpenCV 소스 저장소**: GitHub의 `opencv/opencv` 저장소 — Haar Cascade XML 등 데이터 파일은 이 저장소의 `data/` 폴더에서 받을 수 있다
- **OpenCvSharp 저장소**: GitHub의 `shimat/opencvsharp` 저장소 — C# 바인딩의 소스, 이슈 트래커, 샘플 코드
- **OpenCvSharp NuGet 페이지**: nuget.org에서 `OpenCvSharp4`로 검색하면 패키지와 버전별 변경 이력을 확인할 수 있다

## A.4 다음 학습 단계

이 책으로 OpenCV의 기본기를 다졌다면, 다음과 같은 방향으로 학습을 이어가는 것을 권한다.

- **딥러닝 기반 컴퓨터 비전**: 17장에서 개념만 살짝 짚었던 `dnn` 모듈을 실제 사전 학습 모델(객체 검출, 세그멘테이션, 포즈 추정 등)과 함께 다뤄보고, 필요하다면 PyTorch나 TensorFlow로 직접 모델을 학습시켜 ONNX로 내보내 OpenCV에서 추론해보는 흐름을 경험해본다.
- **GPU 가속과 대규모 처리**: CUDA 지원 빌드나 OpenCL 기반 `UMat`을 활용해 고해상도 영상이나 대량의 이미지를 실시간으로 처리하는 경험을 쌓아본다.
- **공개 데이터셋과 대회 플랫폼 활용**: 공개된 이미지 데이터셋으로 직접 검출·분류 파이프라인을 구성해보거나, 온라인 데이터 과학 대회 플랫폼에서 컴퓨터 비전 과제에 도전해보면 이 책에서 배운 전처리·후처리 기법들이 실전에서 어떻게 조합되는지 체감할 수 있다.
- **실시간 트래킹과 응용 프로젝트**: 웹캠이나 영상 스트림을 입력으로 받아 특정 객체를 지속적으로 추적하는 프로젝트, 또는 이 책의 문서 스캐너 예제를 확장해 실시간 모바일 스캐너 앱으로 발전시켜보는 것도 좋은 다음 단계다.

무엇을 선택하든, 이 책에서 반복해서 강조한 습관—"지금 다루는 배열의 모양과 타입은 무엇인가", "이 함수는 무엇을 입력받고 무엇을 돌려주는가"를 항상 확인하는 습관—은 어떤 언어, 어떤 프레임워크로 옮겨가더라도 그대로 유용하게 남는다.

---

[◀ 이전: 18장. 실전 프로젝트와 성능 최적화](ch18-실전프로젝트와성능최적화.md) | [📖 목차](00-목차.md)
