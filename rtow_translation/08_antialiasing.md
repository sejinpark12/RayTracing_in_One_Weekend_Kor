# 8. Antialiasing
지금까지 렌더링한 이미지들을 확대해 보면, 이미지의 경계에서 거친 계단 모양(stair step)이 나타나는 것을 확인할 수 있습니다. 이런 계단 현상(stair-stepping)을 일반적으로 "aliasing" 또는 "jaggies"라고 합니다. 실제 카메라로 사진을 촬영하면 경계에서 계단 현상이 거의 발생하지 않습니다. 사진에서 경계에 걸친 픽셀은 전경(앞쪽에 있는 물체의 영역)과 배경(뒤쪽에 있는 배경의 영역)의 색이 섞이기 때문입니다. 렌더링된 이미지와 달리, 현실 세계를 담은 실제 사진은 연속적이라는 것을 한 번 생각해 보세요. 바꿔 말하면, 현실 세계와 현실 세계를 찍은 사진의 해상도는 사실상 무한하다는 것입니다. 레이 트레이서에서도 각 픽셀에서 샘플들의 평균을 계산한다면 이와 동일한 효과를 표현할 수 있습니다.

현재의 레이 트레이서는 한 개의 광선이 각 픽셀 중심을 통과하는 샘플링을 하고 있으며, 이러한 샘플링을 일반적으로 _포인트 샘플링(point sampling)_ 이라고 부릅니다. 포인트 샘플링 방식은 멀리 있는 작은 체커보드를 렌더링하는 상황에서 문제가 드러납니다. 체커보드의 패턴이 검은색과 흰색 타일로 구성된 8&times;8 격자라고 가정합니다. 그리고 오직 4개의 광선만 체커보드에 교차한다고 하면, 4개의 광선이 전부 흰색 타일과 교차할 수도 있고 전부 검은색 타일과 교차할 수도 있습니다. 또는 교차하는 타일의 색상이 이상한 조합으로 타일을 교차할 수 있습니다. 하지만 현실에서는 멀리 있는 체커보드를 바라보면 타일들은 검은색과 흰색의 뚜렷한 격자점들로 보이지 않고 체커보드 전체가 회색으로 인식됩니다. 이건 우리의 눈이 레이 트레이서에서 구현하고자 하는 기능과 같은 역할을 하는 작업을 이미 자연스럽게 하고 있기 때문입니다. 즉, 렌더링 이미지의 특정한 이산적 영역(각 픽셀 영역)으로 들어오는 연속 함수로 표현된 빛을 적분하고 있는 것입니다. 다시 말하면, 실제 세계의 빛은 연속적이지만 렌더링 이미지의 픽셀은 이산적입니다. 그래서 한 픽셀 영역에 들어오는 빛을 적분해 그 영역의 대표값을 구한다는 의미입니다.

단순히 동일한 광선이 같은 픽셀의 중심을 여러 번 통과하도록 리샘플링하는 것은 의미가 없습니다. 매번 같은 결과가 나올 뿐입니다. 대신, 픽셀 _주변 영역_ 의 빛을 샘플링하고 실제의 연속적인 결과를 근사하도록 그 샘플들을 적분해야 합니다. 그렇다면 픽셀 주변 영역에 들어오는 빛은 어떻게 적분해야 할까요?

여기서는 가장 단순한 모델을 사용할 것입니다. 픽셀 중심점을 기준으로 하고 상하좌우 네 이웃 픽셀 방향으로 각각 절반 거리까지의 정사각형 영역을 샘플링하는 방식입니다. 더 간단히 말하면, 이 정사각형 영역은 각 픽셀이 차지하는 영역과 같습니다. 이 방식이 최적의 방식은 아니지만, 가장 쉽고 직관적입니다.(이 주제에 대해 더 깊이 알고 싶다면 [A Pixel is Not a Little Square](https://www.researchgate.net/publication/244986797_A_Pixel_Is_Not_A_Little_Square_A_Pixel_Is_Not_A_Little_Square_A_Pixel_Is_Not_A_Little_Square) 를 참고하세요.)

<p align="center"><img src="https://raytracing.github.io/images/fig-1.08-pixel-samples.jpg"></p>

**<p align="center">Figure 8:** _Pixel samples</p>_

---

## 8.1 Some Random Number Utilities
이제 실수값 난수(real random numbers)를 리턴하는 난수발생기(random number generator)가 필요합니다. 이 함수는 실수값 난수인 표준 난수(canonical random number)를 리턴해야 하며, 범위는 관례적으로 $0 ≤ n < 1$ 입니다. 범위에서 1 미만($n < 1$)이라는 조건이 중요한데, 때때로 이 성질을 유용하게 활용할 수 있기 때문입니다.

간단한 방법은 `<cstdlib>` 라이브러리의 `std::rand()` 함수를 사용하는 것입니다. `rand()` 함수는 0부터 `RAND_MAX` 사이의 랜덤 정수를 리턴하는 함수입니다. 따라서 `rtweekend.h` 에 다음의 코드 스니펫 추가하여 원하는 실수 난수를 얻을 수 있습니다.

```cpp
#include <cmath>
///////////////////////// 추가 /////////////////////////
#include <cstdlib>                                   //
///////////////////////////////////////////////////////
#include <iostream>
#include <limits>
#include <memory>
...

// Utility Functions

inline double degrees_to_radians(double degrees) {
  return degrees * pi / 180.0;
}

///////////////////////// 추가 /////////////////////////////
inline double random_double() {                          //
  // Returns a random real in [0, 1).                    //
  return std::rand() / (RAND_MAX + 1.0);                 //
}                                                        //
                                                         //
inline double random_double(double min, double max) {    //
  // Returns a random real in [min, max).                //
  return min + (max - min) * random_double();            //
}                                                        //
///////////////////////////////////////////////////////////
```

**<p align="center">Listing 41:** [rtweekend.h] _random_double() functions</p>_

전통적으로 C++에는 표준 난수발생기가 없었습니다. 하지만 최신 C++ 버전에서는 `<random>` 헤더로 이 문제를 해결(일부 전문가에 따르면 불완전하게)했습니다. 만약 이 방식을 원하신다면, 다음 조건으로 난수를 얻을 수 있습니다.

```cpp
...

///////////////////////// 추가 /////////////////////////////
#include <random>                                        //
///////////////////////////////////////////////////////////

...

///////////////////////// 추가 //////////////////////////////////////////////
inline double random_double() {                                           //
  static std::uniform_real_distribution<double> distribution(0.0, 1.0);   //
  static std::mt19937 generator;                                          //
  return distribution(generator);                                         //
}                                                                         //
////////////////////////////////////////////////////////////////////////////

inline double random_double(double min, double max) {
  // Returns a random real in [min,max).
  return min + (max - min) * random_double();
}

...
```

**<p align="center">Listing 42:** [rtweekend.h] _random_double(), alternate implementation</p>_

---

## 8.2 Generating Pixels with Multiple Samples
여러 개의 샘플로 하나의 픽셀을 표현할 때는, 픽셀 주변 영역에서 샘플들을 선택하고 샘플마다 계산한 빛(색)들을 모아서 평균을 냅니다.

먼저, 사용할 샘플 수를 반영하도록 `write_color()` 함수를 수정하겠습니다. 즉, 우리가 사용할 샘플들의 평균을 구해야 합니다. 이를 위해 각 반복에서 계산한 전체 색을 누적합니다. 그리고 색을 출력하기 전 마지막에, 누적한 색을 샘플 수로 나누어 계산을 마무리합니다. 최종 결과의 색 성분 값들이 $[0, 1]$ 범위를 벗어나지 않도록, 간단한 헬퍼 함수 `interval::clamp()` 를 구현하여 사용하겠습니다.

```cpp
class interval {
  public:
    ...

    bool surrounds(double x) const {
      return min < x && x < max;
    }

///////////////////////// 추가 /////////////////////////////
    double clamp(double x) const {                       //
      if (x < min) return min;                           //
      if (x > max) return max;                           //
      return x;                                          //
    }                                                    //
///////////////////////////////////////////////////////////
    ...
};
```

**<p align="center">Listing 43:** [interval.h] _The interval::clamp() utility function</p>_

다음은 interval 헤더 파일의 clamp 함수를 반영하여 수정한 `write_color()` 함수입니다.

```cpp
///////////////////////// 추가 /////////////////////////////
#include "interval.h"                                    //
///////////////////////////////////////////////////////////
#include "vec3.h"

using color = vec3;

void write_color(std::ostream& out, const color& pixel_color) {
  auto r = pixel_color.x();
  auto g = pixel_color.y();
  auto b = pixel_color.z();

  // Translate the [0, 1] component values to the byte range [0, 255].
///////////////////////// 수정 /////////////////////////////
  static const interval intensity(0.000, 0.999);         //
  int rbyte = int(256 * intensity.clamp(r));             //
  int gbyte = int(256 * intensity.clamp(g));             //
  int bbyte = int(256 * intensity.clamp(b));             //
///////////////////////////////////////////////////////////

  // Write out the pixel color components.
  out << rbyte << ' ' << gbyte << ' ' << bbyte << '\n';
}
```

**<p align="center">Listing 44:** [color.h] _The multi-sample write\_color() function</p>_

이제 새로운 `camera::get_ray(i, j)` 함수를 정의하여 camera 클래스를 수정하겠습니다. 이 함수는 각 픽셀에서 서로 다른 여러 개의 샘플을 생성합니다. 그리고 함수 내부에서 새 헬퍼 함수인 `sample_square()` 를 사용합니다. `sample_square()` 함수는 원점 중심의 단위 정사각형 범위 안에서 임의의 샘플 포인트 하나를 생성하는 함수입니다. 그러고 나서 이 이상적인 단위 정사각형의 랜덤 샘플을 현재 샘플링하고 있는 각 픽셀에 맞게 변환합니다.

```cpp
class camera {
  public:
    double aspect_ratio      = 1.0; // Ratio of image width over height
    int    image_width       = 100; // Rendered image width in pixel count
///////////////////////// 추가 /////////////////////////////////////////////////////
    int    samples_per_pixel = 10;  // Count of random samples for each pixel    //
///////////////////////////////////////////////////////////////////////////////////

    void render(const hittable& world) {
      initialize();

      std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";

      for (int j = 0; j < image_height; j++) {
        std::clog << "\rScanlines remaining: " << (image_height - j) << ' ' << std::flush;
        for (int i = 0; i < image_width; i++) {
///////////////////////// 수정 /////////////////////////////////////////////////////
          color pixel_color(0, 0, 0);                                            //
          for (int sample = 0; sample < samples_per_pixel; sample++) {           //
            ray r = get_ray(i, j);                                               //
            pixel_color += ray_color(r, world);                                  //
          }                                                                      //
          write_color(std::cout, pixel_samples_scale * pixel_color);             //
///////////////////////////////////////////////////////////////////////////////////
        }
      }

      std::clog << "\rDone.                 \n";
    }
    ...
  private:
    int    image_height;        // Rendered image height
///////////////////////// 추가 /////////////////////////////////////////////////////
    double pixel_samples_scale; // Color scale factor for a sum of pixel samples //
///////////////////////////////////////////////////////////////////////////////////
    point3 center;              // Camera center
    point3 pixel00_loc;         // Location of pixel 0, 0
    vec3   pixel_delta_u;       // Offset to pixel to the right
    vec3   pixel_delta_v;       // Offset to pixel below

    void initialize() {
      image_height = int(image_width / aspect_ratio);
      image_height = (image_height < 1) ? 1 : image_height;

///////////////////////// 추가 /////////////////////////////////////////////////////
      pixel_samples_scale = 1.0 / samples_per_pixel;                             //
///////////////////////////////////////////////////////////////////////////////////

      center = point3(0, 0, 0);
      ...
    }

///////////////////////// 추가 ///////////////////////////////////////////////////////////////////
    ray get_ray(int i, int j) const {                                                          //
      // Construct a camera ray originating from the origin and directed at randomly sampled   //
      // point around the pixel location i, j.                                                 //
                                                                                               //
      auto offset = sample_square();                                                           //
      auto pixel_sample = pixel00_loc                                                          //
                        + ((i + offset.x()) * pixel_delta_u)                                   //
                        + ((j + offset.y()) * pixel_delta_v);                                  //
                                                                                               //
      auto ray_origin = center;                                                                //
      auto ray_direction = pixel_sample - ray_origin;                                          //
                                                                                               //
      return ray(ray_origin, ray_direction);                                                   //
    }                                                                                          //
                                                                                               //
    vec3 sample_square() const {                                                               //
      // Returns the vector to a random point in the [-.5, -.5] - [+.5, +.5] unit square.      //
      return vec3(random_double() - 0.5, random_double() - 0.5, 0);                            //
    }                                                                                          //
/////////////////////////////////////////////////////////////////////////////////////////////////

    color ray_color(const ray& r, const hittable& world) const {
      ...
    }
};

#endif
```

**<p align="center">Listing 45:** [camera.h] _Camera with samples-per-pixel parameter</p>_

(위에서 새로 추가한 `sample_square()` 함수와 더불어, `sample_disk()` 함수도 Github 소스 코드에서 확인할 수 있습니다. 이 함수는 정사각형이 아닌 픽셀에서 실험해 보고 싶은 경우를 위해 포함되어 있습니다. 그러나 이 책에서는 `sample_disk()` 함수를 사용하지 않을 것입니다. `sample_disk()` 함수는 나중에 구현할 `random_in_unit_disk()` 함수를 기반으로 동작합니다.)

새 카메라 파라미터를 설정하기 위해서 main을 수정합니다.

```cpp
int main() {
  ...

  camera cam;

  cam.aspect_ratio      = 16.0 / 9.0;
  cam.image_width       = 400;
///////////////////////// 추가 //////////////////////
  cam.samples_per_pixel = 100;                    //
////////////////////////////////////////////////////

  cam.render(world);
}
```

**<p align="center">Listing 46:** [main<span></span>.cc] _Setting the new samples-per-pixel parameter</p>_

렌더링 이미지를 확대해 보면, 경계 픽셀의 차이를 확인할 수 있습니다.

<p align="center"><img src="https://user-images.githubusercontent.com/19530862/96420143-1087d680-1230-11eb-8064-7343d973cda0.png"></p>

**<p align="center">Image 6:** _Before and after antialiasing</p>_

---

## 출처

**Ray Tracing in One Weekend - Peter Shirley**
https://raytracing.github.io/books/RayTracingInOneWeekend.html#antialiasing
