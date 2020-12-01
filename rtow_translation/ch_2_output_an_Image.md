> **이 글은 Peter Shirley의 [Ray Tracing in One Weekend](https://raytracing.github.io/books/RayTracingInOneWeekend.html)를 번역한 것입니다.
> Ray Tracing in One Weekend를 공부하면서 다시 한번 복습하는 느낌으로 번역을 해보려고 합니다. 영어가 서툴러 번역이 잘못되었을 수도 있으므로 잘못된 부분을 발견하신다면 지적해 주시면 감사하겠습니다.**

## 2.1 The PPM Image Format

---

일단 이미지를 보는 방법이 필요합니다. 가장 간단한 방법은 파일에 결과를 출력하는 것입니다. 하지만 문제는 파일 포맷의 종류가 너무 많다는 것입니다. 많은 수의 파일 포맷들은 복잡합니다. 저는 항상 plain text ppm file로 시작합니다. [위키피디아](https://en.wikipedia.org/wiki/Netpbm#File_formats)에 설명이 잘 되어있습니다.

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

    for (int j = image_height - 1; j >= 0; --j) {
        for (int i = 0; i < image_width; ++i) {
            auto r = double(i) / (image_width - 1);
            auto g = double(j) / (image_height - 1);
            auto b = 0.25;

            int ir = static_cast<int>(255.999 * r);
            int ig = static_cast<int>(255.999 * g);
            int ib = static_cast<int>(255.999 * b);

            std::cout << ir << ' ' << ig << ' ' << ib << '\n';
        }
    }
}
```

**<p align="center">Listing 1:** [main<span></span>.cc] Creating your first image</p>
</br>

위의 코드에서 알아야 할 것들이 있습니다.

1. 픽셀은 한 행의 왼쪽에서 오른쪽으로 입력됩니다.
2. 행은 위에서 아래로 입력됩니다.
3. 관례 상, red/green/blue 색상의 범위는 0.0부터 1.0입니다. 나중에 내부적으로 high dynamic range를 사용할 때 범위를 확장하지만 출력 전에 0부터 1로 톤 매핑(tone map)할 것이므로 이 코드는 바뀌지 않습니다.
4. 빨간색은 왼쪽에서 오른쪽으로 가면서 검은색에서 밝은 빨간색으로 바뀌고, 녹색은 아래에서 위로 가면서 검은색에서 밝은 녹색으로 바뀝니다. 빨간색과 녹색을 더하면 노란색이 되므로 이미지의 우측 상단은 노란색이라는 것을 알 수 있습니다.

---

## 2.2 Creating an Image File

---

프로그램의 출력이 파일에 쓰여지므로 이 출력을 이미지 파일로 리다이렉션해야합니다. 일반적으로 이 작업은 커맨드 라인에서 `>` 리다이렉션 연산자를 사용하여 수행합니다.

```
build\Release\inOneWeekend.exe > image.ppm
```

위의 예는 Windows에서 리다이렉션하는 것을 보여줍니다. Mac이나 Linux는 다음과 같습니다.

```
build/inOneWeekend > image.ppm
```

출력 파일을 열면 다음과 같은 결과를 확인할 수 있습니다.(제 Mac에는 `ToyViewer`가 있습니다만 여러분이 좋아하는 뷰어로 파일을 열어보시고 만약 ppm 포맷을 지원하지 않는다면 구글에서 "ppm viewer"를 검색해보십시오.)

<p align="center"><img src="https://raytracing.github.io/images/img-1.01-first-ppm-image.png"></p>

**<p align="center">Image 1:** First PPM image</p>
</br>

만세! 이게 바로 그래픽스 "hello world"입니다. 여러분의 이미지가 다르게 보인다면, 텍스트 에디터에서 파일을 열고 어떻게 입력되어 있는지 확인하십시오. 다음과 같은 형태로 시작해야합니다.

```
P3
256 256
255
0 255 63
1 255 63
2 255 63
3 255 63
4 255 63
5 255 63
6 255 63
7 255 63
8 255 63
9 255 63
...
```

**<p align="center">Listing 2:** First image output</p>
</br>

그렇지 않다면 줄바꿈이나 이미지 리더를 혼란스럽게하는 것이 포함되어 있을 수 있습니다.

PPM 이외의 다른 이미지 포맷을 만들고 싶다면, 저는 header-only 이미지 라이브러리인 `stb_image.h`를 좋아합니다. 여기 [링크](https://github.com/nothings/stb)가 있습니다.

---

## 2.3 Adding a Progress Indicator

---

계속하기 전에, 진행 상태 표시기(progress indicator)를 추가해봅시다. 오래 걸리는 렌더링 과정을 추적할 수 있는 간단한 방법입니다. 또한 무한 루프나 다른 문제로 인해 실행이 중단되는 것도 파악할 수 있습니다.

프로그램은 표준 출력 스트림(`std::cout`)에 이미지를 출력합니다. 그러므로 그것은 그대로 두고, 대신 에러 출력 스트림(`std::cerr`)에 진행 상태 표시기를 출력합니다.

```cpp
for (int j = image_height - 1; j >= 0; --j) {
    std::cerr << "\rScanlines remaining: " << j << ' ' << std::flush; // 추가
    for (int i = 0; i < image_width; ++i) {
        auto r = double(i) / (image_width - 1);
        auto g = double(j) / (image_height - 1);
        auto b = 0.25;

        int ir = static_cast<int>(255.999 * r);
        int ig = static_cast<int>(255.999 * g);
        int ib = static_cast<int>(255.999 * b);

        std::cout << ir << ' ' << ig << ' ' << ib << '\n';
    }
}

std::cerr << "\nDone.\n"; // 추가
```

**<p align="center">Listing 3:** [main<span></span>.cc] Main render loop with progress reporting</p>

---

## 출처

**Ray Tracing in One Weekend - Peter Shirley**
https://raytracing.github.io/books/RayTracingInOneWeekend.html#outputanimage
