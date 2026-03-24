# 2. Output an Image

## 2.1 The PPM Image Format

---

먼저, 이미지를 확인할 방법이 필요합니다. 가장 간단한 방법은 이미지를 파일로 저장하는 것입니다. 하지만 이미지 파일 포맷은 종류가 너무 많고, 대부분 복잡합니다. 그래서 저는 항상 일반 텍스트로 구성된 이미지 포맷인 ppm을 사용합니다. [위키피디아](https://en.wikipedia.org/wiki/Netpbm#PPM_example)에 설명이 잘 되어있습니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.01-ppm.jpg"></p>

**<p align="center">Figure 1:** PPM Example</p>
</br>

ppm 파일을 출력하는 C++ 코드를 만들어봅시다.

```cpp
#include <iostream>

int main() {
    // Image
    const int image_width = 256;
    const int image_height = 256;

    // Render
    std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";

    for (int j = 0; j < image_height; j++) {
        for (int i = 0; i < image_width; i++) {
            auto r = double(i) / (image_width - 1);
            auto g = double(j) / (image_height - 1);
            auto b = 0.0;

            int ir = int(255.999 * r);
            int ig = int(255.999 * g);
            int ib = int(255.999 * b);

            std::cout << ir << ' ' << ig << ' ' << ib << '\n';
        }
    }
}
```

**<p align="center">Listing 1:** [main<span></span>.cc] Creating your first image</p>
</br>

위의 코드에서 아래 내용들은 알고 넘어가야 합니다.

1. 픽셀은 행(row) 단위로 출력됩니다.
2. 각 행의 픽셀은 왼쪽에서 오른쪽으로 출력됩니다.
3. 각 행은 위에서 아래로 출력됩니다.
4. 일반적으로 red/green/blue 성분은 내부적으로 실수값(0.0 ~ 1.0)으로 표현됩니다. 이를 출력하기 전에 정수값(0 ~ 255)으로 스케일 해야 합니다.
5. 빨간색은 왼쪽에서 오른쪽으로 갈수록 완전히 꺼진 상태(검은색)에서 완전히 켜진 상태(밝은 빨간색)로 변하고, 녹색은 위에서 아래로 갈수록 완전히 꺼진 상태(검은색)에서 완전히 켜진 상태(밝은 녹색)로 변합니다. 빨간색과 녹색을 더하면 노란색이 되므로 이미지의 우측 하단은 노란색이라는 것을 예상할 수 있습니다.

---

## 2.2 Creating an Image File

---

파일은 표준 출력 스트림(standard output stream)으로 출력되므로, 이것을 이미지 파일로 리다이렉션해야 합니다. 일반적으로 이 작업은 커맨드 라인에서 `>` 리다이렉션 연산자를 사용하여 처리합니다.

Windows에서는 아래의 명령어를 실행하여 CMake로 디버그 빌드를 합니다.

``` bash
cmake -B build
cmake --build build
```

그다음 빌드 한 프로그램을 아래와 같이 실행합니다.

``` bash
build\Debug\inOneWeekend.exe > image.ppm
```

나중에는 최적화된 빌드(릴리즈 빌드)를 하는 것이 실행 속도 측면에서 좋습니다. 이 경우 아래와 같이 빌드 합니다.

```bash
cmake --build build --config release
```

최적화된 빌드는 아래와 같이 실행합니다.

```bash
build\Release\inOneWeekend.exe > image.ppm
```

위의 예시는 소스코드에 포함된 CMakeLists.txt 파일을 사용하여 CMake 빌드 하는 것을 가정하고 있지만, 빌드 환경과 언어는 여러분에게 가장 익숙한 것을 사용하시면 됩니다.


Mac이나 Linux에서는 아래와 같이 릴리즈 빌드를 실행합니다.

```bash
build/inOneWeekend > image.ppm
```

프로젝트의 [README](https://github.com/RayTracing/raytracing.github.io?tab=readme-ov-file#building-and-running) 에서 자세한 빌드/실행 방법을 확인할 수 있습니다.

출력 파일을 열어 보면 다음과 같은 결과를 확인할 수 있습니다.(저는 Mac에서 `ToyViewer` 를 사용하지만 여러분이 원하는 뷰어로 파일을 열어보시고 ppm 포맷을 지원하지 않는다면 구글에서 "ppm viewer"를 검색해 보세요.)

<p align="center"><img src="https://raytracing.github.io/images/img-1.01-first-ppm-image.png"></p>

**<p align="center">Image 1:** First PPM image</p>
</br>

만세! 이게 바로 그래픽스에서의 "hello world"입니다. 결과 이미지가 위와 다르게 보인다면, 텍스트 에디터로 파일을 열고 어떻게 되어 있는지 확인해 보세요. 아래와 같은 형식으로 시작해야 합니다.

```
P3
256 256
255
0 0 0
1 0 0
2 0 0
3 0 0
4 0 0
5 0 0
6 0 0
7 0 0
8 0 0
9 0 0
10 0 0
11 0 0
12 0 0
...
```

**<p align="center">Listing 2:** First image output</p>
</br>

만약 PPM 파일 내용이 위와 같은 형식이 아니라면, 해당 코드를 다시 점검해 보세요. 파일 내용에 문제가 없는데도 이미지가 제대로 렌더링되지 않는다면, 줄바꿈 방식 차이와 같은 원인으로 인해 뷰어가 제대로 동작하지 않을 수 있습니다. 디버깅에 사용할 test.ppm 파일을 GitHub 프로젝트의 images 디렉토리에 추가해 두었습니다. 이 파일을 사용하여 뷰어가 PPM 파일을 정상적으로 렌더링할 수 있는지 확인할 수 있습니다. 그리고 직접 생성한 PPM 파일과 비교해 볼 수도 있습니다.

Windows에서 생성한 PPM 파일을 뷰어로 열 때 발생하는 문제가 일부 독자들에게서 보고되었습니다. 이런 경우, PPM 파일이 UTF-16으로 저장되는 것이 원인인 경우가 많으며, PowerShell에서 자주 발생합니다. 이런 문제가 발생한다면, [Discussion 114](https://github.com/RayTracing/raytracing.github.io/discussions/1114) 를 참고하세요.

이미지가 정상적으로 렌더링된다면, 시스템이나 IDE 관련 문제는 거의 해결된 셈입니다. 앞으로도 렌더링하여 이미지를 생성할 때는 이와 같은 간단한 방식을 사용합니다.

결과물을 다른 이미지 포맷으로 생성하고 싶다면, header-only image library인 `stb_image.h` 를 추천합니다. https://github.com/nothings/stb 을 참고하세요.

---

## 2.3 Adding a Progress Indicator

---

계속 진행하기 앞서, 진행 상태 표시기(progress indicator)를 추가해 봅시다. 이것은 렌더링이 오래 걸리는 경우의 진행 상황을 추적할 수 있는 간단한 방법입니다. 또한 무한 루프나 다른 문제로 인해 실행이 중단되는 것도 파악할 수 있습니다.

기존 프로그램은 표준 출력 스트림(`std::cout`)에 이미지를 출력합니다. 그부분은 그대로 두고, 추가로 로그 출력 스트림(`std::log`)에 진행 상태 표시기를 출력하기로 합니다.

```cpp

    for (int j = 0; j < image_height; j++) {
        ///////////////////////// 추가 ///////////////////////////////
        std::clog << "\rScanlines remaining: " << (image_height - j) << ' ' << std::flush;
        /////////////////////////////////////////////////////////////
        for (int i = 0; i < image_width; i++) {
            auto r = double(i) / (image_width - 1);
            auto g = double(j) / (image_height - 1);
            auto b = 0.0;

            int ir = int(255.999 * r);
            int ig = int(255.999 * g);
            int ib = int(255.999 * b);

            std::cout << ir << ' ' << ig << ' ' << ib << '\n';
        }
    }

    ///////////////////////// 추가 ///////////////////////////////
    std::clog << "\rDone.                 \n";
    /////////////////////////////////////////////////////////////
```

**<p align="center">Listing 3:** [main<span></span>.cc] Main render loop with progress reporting</p>

이제 프로그램을 실행하면, 남은 scanline의 수를 실시간으로 확인할 수 있습니다. 하지만 실행 속도가 매우 빨라서 진행 상태를 볼 시간조차 없을 것입니다! 걱정하지 마세요. 앞으로 레이 트레이서를 확장해 나가며, 천천히 갱신되는 진행 상태 표시줄을 많이 보게 될 것입니다.

---

## 출처

**Ray Tracing in One Weekend - Peter Shirley**
https://raytracing.github.io/books/RayTracingInOneWeekend.html#outputanimage
