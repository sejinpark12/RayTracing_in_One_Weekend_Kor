# 4. Rays, a Simple Camera, and Background

## 4.1 The ray Class

---

모든 레이 트레이서는 ray 클래스와, 광선(ray)를 따라 어떤 색이 보이는지 계산하는 부분을 공통적으로 가지고 있습니다. 함수 $\mathbf{P}(t) = \mathbf{A} + t \mathbf{b}$ 로 광선을 표현할 수 있습니다. 위치 $\mathbf{P}$ 는 3차원 공간에서 해당 함수로 표현된 직선 상의 한 점입니다. $\mathbf{A}$ 는 광선의 원점(ray origin)이고 $\mathbf{b}$ 는 광선의 방향(ray direction)입니다. 광선 파라미터 $t$ 는 실수(real number)입니다(코드에서는 `double`). $t$ 의 값을 바꿔가며 함수 $\mathbf{P}(t)$ 에 대입하면 위치 $\mathbf{P}(t)$ 는 광선을 따라 이동합니다. 음수 $t$ 값까지 고려하면, 3차원 공간에서 그 직선 상의 어디든지 갈 수 있습니다. 양수 $t$ 값은 오로지 $\mathbf{A}$ (광선의 원점)의 앞쪽($\mathbf{b}$ 방향)으로만 이동할 수 있으며 이것을 반직선(half-line) 또는 광선(ray)라고 합니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.02-lerp.jpg"></p>

**<p align="center">Figure 2**: _Linear interpolation</p>_

아래와 같이 클래스로 광선을 구현할 수 있고, `ray::at(t)` 멤버 함수로 함수 $\mathbf{P}(t)$ 를 구현할 수 있습니다.

```cpp
#ifndef RAY_H
#define RAY_H

#include "vec3.h"

class ray {
  public:
    ray() {}

    ray(const point3& origin, const vec3& direction) : orig(origin), dir(direction) {}

    const point3& origin() const { return orig; }
    const vec3& direction() const { return dir; }

    point3 at(double t) const {
      return orig + t*dir;
    }

  private:
    point3 orig;
    vec3 dir;
};

#endif
```

**<p align="center">Listing 7**: [ray.h] _The ray class</p>_

(C++에 익숙하지 않은 분들을 위해 설명하자면, `ray::origin()` 과 `ray::direction()` 두 함수 모두 const 레퍼런스(immutable reference)를 리턴합니다. 호출하는 쪽에서는 필요에 따라서 const 레퍼런스를 그대로 사용할 수도 있고, 수정 가능한 복사본을 만들어 사용할 수도 있습니다.)

## 4.2 Sending Rays Into the Scene

---

이제 본격적으로 레이 트레이서를 만들기 위한 준비가 끝났습니다. 레이 트레이서의 핵심은 카메라 위치에서 픽셀을 향해 광선을 보내고, 그 광선 방향에서 보이는 색을 계산하는 것입니다. 과정은 다음과 같습니다.

1. 눈(카메라)에서 해당 픽셀을 통과하는 광선을 계산합니다.
2. 그 광선이 어떤 오브젝트와 교차하는지 판별합니다.
3. 가장 가까운 교차점의 색을 계산합니다.

항상 저는 레이 트레이서를 만드는 초기 단계에서는 코드가 일단 동작하도록 카메라를 단순하게 만듭니다.

정사각형 이미지는 가로와 세로의 크기가 같기 때문에 x와 y를 뒤바꿔도 티가 덜 나서 디버깅할 때 자주 헷갈리는 경우가 많습니다. 그렇기 때문에 여기서는 직사각형 이미지를 사용하겠습니다. 정사각형 이미지의 종횡비(aspect ratio)는 가로, 세로의 길이가 같기 때문에 1&ratio;1 종횡비입니다. 위에서 스크린으로 정사각형이 아닌 이미지를 사용하기로 했으므로, 매우 일반적으로 쓰이는 16&ratio;9 종횡비를 사용하겠습니다.

$$ \text{width} / \text{height} = 16 / 9 = 1.7778 $$

실제의 예를 들면, 가로 800 픽셀, 세로 400 픽셀인 이미지의 종횡비는 2&ratio;1 종횡비가 됩니다.

이미지의 종횡비는 이미지의 너비와 높이의 비로 결정할 수 있지만, 여기서는 사용할 종횡비를 이미 위에서 선택했습니다. 그러므로 이미지의 너비를 정한 다음, 너비와 해당 종횡비를 기준으로 높이를 계산하는 편이 더 쉽습니다. 이렇게 하면 이미지의 너비만 변경하여 이미지를 확대/축소할 수 있고 원하는 종횡비도 유지할 수 있습니다. 또한, 이미지의 높이를 계산할 때는 높이가 최소 1 이상인지 반드시 확인해야 합니다.

렌더링 이미지의 픽셀 해상도(pixel dimensions)를 결정하는 것 뿐만 아니라, 씬으로 광선을 통과시킬 가상의 뷰포트(virtual viewport)도 설정해야 합니다. 뷰포트는 3차원 공간에 놓인 가상의 직사각형입니다. 그리고 그 위에는 이미지 픽셀에 대응하는 중심점 위치들의 그리드가 배치됩니다. 픽셀이 가로, 세로 방향으로 동일한 간격으로 배치되어 있다면, 이 픽셀들로 이루어진 뷰포트도 렌더링 이미지와 같은 종횡비를 갖습니다. 인접한 두 픽셀 사이의 거리를 픽셀 간격(pixel spacing)이라고 하며, 픽셀의 형태는 정사각형(square pixels)이 일반적입니다.

먼저, 뷰포트의 높이를 임의로 2.0으로 정한 다음, 원하는 종횡비가 되도록 뷰포트 너비를 계산합니다. 다음은 해당 동작의 코드 스니펫입니다.

```cpp
auto aspect_ratio = 16.0 / 9.0;
int image_width = 400;

// Calculate the image height, and ensure that it's at least 1.
int image_height = int(image_width / aspect_ratio);
image_height = (image_height < 1) ? 1 : image_height;

// Viewport widths less than one are ok since they are real valued.
auto viewport_height = 2.0;
auto viewport_width = viewport_height * (double(image_width) / image_height);
```

**<p align="center">Listing 8**: _Rendered image setup</p>_

`viewport_width` 를 계산할 때, 왜 `aspect_ratio` 를 그대로 사용하지 않는지 궁금하실겁니다. 그 이유는 `aspect_ratio` 에 저장된 값은 이상적인 비율일 뿐, `image_width` 와 `image_height` 사이의 _실제_ 비율과는 다를 수 있기 때문입니다. `image_height` 에 정수뿐만 아니라 실수도 허용한다면, `aspect_ratio` 를 그대로 사용할 수도 있습니다. 하지만 `image_width` 와 `image_height` 의 실제 비율은 위 코드 스니펫의 두 부분으로 인해 달라질 수 있습니다. 첫째, `image_height` 는 정수로 내림되므로 이로 인해 비율이 커질 수 있습니다. 둘째, `image_height` 가 1보다 작아지는 것을 허용하지 않으므로 이 또한 실제 종횡비에 영향을 줄 수 있습니다.

`aspect_ratio` 는 이상적인 비율일 뿐이고, 실제 이미지의 종횡비는 이미지의 너비를 이미지의 높이로 나눈 정수 기반 비율로 `aspect_ratio` 에 가능한 한 가깝게 근사한다는 점에 유의하세요. 최종 뷰포트의 너비를 결정할 때는 뷰포트의 비율을 정확히 이미지의 비율에 일치시키기 위해, 계산된 실제 이미지 종횡비를 사용합니다.

다음으로 카메라 중심(camera center)을 정의하겠습니다. 3차원 공간에서 모든 씬 광선은 카메라 중심에서 출발합니다.(이 점을 흔히 eye point라고도 합니다.) 카메라 중심에서 출발하여 뷰포트 중심으로 향하는 벡터는 뷰포트와 수직입니다. 일단, 뷰포트와 카메라 중심 사이의 거리를 단위 길이 1로 설정하겠습니다. 이 거리는 흔히 초점 거리(focal length)라고 합니다.

단순하게 표현하기 위해서, 카메라 중심을 $(0, 0, 0)$ 좌표에 위치시킵니다. 그리고 x축은 오른쪽, y축은 위쪽, z축은 카메라가 바라보는 방향의 반대 방향인 좌표계를 사용하겠습니다.(흔히 오른손 좌표계(right-handed coordinates)라고 합니다.)

<p align="center"><img src="https://raytracing.github.io/images/fig-1.03-cam-geom.jpg"></p>

**<p align="center">Figure 3:** _Camera geometry_</p>

이제부터 까다로운 부분으로 넘어가겠습니다. 위에서는 3차원 공간 좌표계에 대해 설명했습니다. 하지만 이것은 이미지 좌표계와 충돌합니다. 이미지 좌표계는 0번째 픽셀을 왼쪽 위에 두고 오른쪽 아래에 위치한 마지막 픽셀까지 차근차근 내려가며 처리하는 방식이기 때문입니다. 이말은 이미지 좌표계의 Y축이 카메라 중심 3차원 좌표계의 Y축과 반전되어 있다는 뜻입니다. 이미지에서는 아래로 내려갈 수록 Y값이 증가합니다.

이미지를 스캔할 때는 왼쪽 위 픽셀($0,0$ 의 픽셀)에서 시작하여, 각 행 안에서는 왼쪽에서 오른쪽으로 스캔하고, 행 단위로는 위에서 아래로 내려가며 각 행을 스캔합니다. 픽셀 그리드를  따라가기 쉽도록, 뷰포트의 왼쪽 가장자리에서 오른쪽 가장자리로 향하는 벡터($\mathbf{V_u}$)와 뷰포트의 위쪽 가장자리에서 아래쪽 가장자리로 향하는 벡터($\mathbf{V_v}$)를 사용할 것입니다.

픽셀 그리드는 뷰포트의 가장자리로부터 픽셀 간 거리의 절반만큼 안쪽으로 들어간 위치에 배치됩니다. 이렇게 하면 뷰포트의 영역이 너비 &times; 높이 개의 영역으로 균등하게 나뉩니다. 아래 그림은 뷰포트와 픽셀 그리드가 어떻게 배치되는지를 보여 줍니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.04-pixel-grid.jpg"></p>

**<p align="center">Figure 4:** _Viewport and pixel grid_</p>

이 그림에서는 뷰포트, 7 &times; 5 해상도 이미지에 대한 픽셀 그리드, 뷰포트의 왼쪽 위 모서리 $\mathbf{Q}$, 픽셀 $\mathbf{P_{0,0}}$ 의 위치, 뷰포트 벡터 $\mathbf{V_u}$ (`viewport_u`), 뷰포트 벡터 $\mathbf{V_v}$ (`viewport_v`), 픽셀 델타 벡터 $\mathbf{\Delta u}$, $\mathbf{\Delta v}$ 가 표시되어 있습니다.

이 모든 내용을 바탕으로, 이제 카메라를 구현하는 코드를 작성하겠습니다. 주어진 씬 광선에  대한 색을 반환하는 `ray_color(const ray& r)` 함수를 일단 임시로 만들어 둡니다. 지금은 이 함수가 항상 검은색을 반환하도록 해 두겠습니다.

```cpp
#include "color.h"
///////////////////////// 추가 ///////////////////////////////
#include "ray.h"
/////////////////////////////////////////////////////////////
#include "vec3.h"

#include <iostream>

///////////////////////// 추가 ///////////////////////////////
color ray_color(const ray& r) {
  return color(0, 0, 0);
}
/////////////////////////////////////////////////////////////

int main() {

  // Image

///////////////////////// 추가 ///////////////////////////////
  auto aspect_ratio = 16.0 / 9.0;
  int image_width = 400;

  // Calculate the image height, and ensure that it's at least 1.
  int image_height = int(image_width / aspect_ratio);
  image_height = (image_height < 1) ? 1 : image_height;

  // Camera

  auto focal_length = 1.0;
  auto viewport_height = 2.0;
  auto viewport_width = viewport_height * (double(image_width) / image_height);
  auto camera_center = point3(0, 0, 0);

  // Calculate the vectors across the horizontal and down the vertical viewport edges.
  auto viewport_u = vec3(viewport_width, 0, 0);
  auto viewport_v = vec3(0, -viewport_height, 0);

  // Calculate the horizontal and vertical delta vectors from pixel to pixel.
  auto pixel_delta_u = viewport_u / image_width;
  auto pixel_delta_v = viewport_v / image_height;

  // Calculate the location of the upper left pixel.
  auto viewport_upper_left = camera_center
                           - vec3(0, 0, focal_length) - viewport_u/2 - viewport_v/2;
  auto pixel00_loc = viewport_upper_left + 0.5 * (pixel_delta_u + pixel_delta_v);
/////////////////////////////////////////////////////////////

  // Render

  std::cout << "P3\n" << image_width << " " << image_height << "\n255\n";

  for (int j = 0; j < image_height; j++) {
    std::clog << "\rScanlines remaining: " << (image_height - j) << ' ' << std::flush;
    for (int i = 0; i < image_width; i++) {
///////////////////////// 추가 ///////////////////////////////
      auto pixel_center = pixel00_loc + (i * pixel_delta_u) + (j * pixel_delta_v);
      auto ray_direction = pixel_center - camera_center;
      ray r(camera_center, ray_direction);

      color pixel_color = ray_color(r);
/////////////////////////////////////////////////////////////
      write_color(std::cout, pixel_color);
    }
  }

  std::clog << "\rDone.                 \n";
}
```

**<p align="center">Listing 9**: [<span>main</span>.cc] _Creating scene rays</p>_

위 코드에서 `ray_direction` 을 단위 벡터(unit vector)로 만들지 않은 부분에 주목하세요. 그렇게 하지 않는 편이 코드가 더 간단하고, 속도도 조금 더 빠르기 때문입니다.

이제 `ray_color(ray)` 함수를 작성하여 간단한 그라데이션을 구현하겠습니다. 이 함수는 광선 방향을 단위 길이로 스케일링 한 _후에_ $y$ 좌표의 높이에 따라 선형적으로 흰색과 파란색을 섞습니다. 스케일링된 $y$ 좌표의 범위는 $-1.0 < y < 1.0$ 가 됩니다. 벡터를 정규화한 다음에 $y$ 높이를 보고 있기 때문에, 수직 방향 그라데이션뿐만 아니라 수평 방향 그라데이션도 나타나는 것을 확인할 수 있습니다.

여기서는 $0.0 ≤ a ≤ 1.0$ 범위로 선형 스케일링 하는 그래픽스의 표준 기법을 사용합니다. $a = 1.0$ 일 때는 파란색, $a = 0.0$ 일 때는 흰색으로 표현되어야 합니다. 그 사이의 $a$ 값에서는 두 색을 섞습니다. 이것을 선형 블랜딩(linear blend) 또는 선형 보간(linear interpolation) 이라고 합니다. 보통은 두 값을 _lerp_ 한다고 말합니다. lerp는 항상 다음과 같은 형태입니다.

$$ \mathit{blendedValue} = (1-a)\cdot\mathit{startValue} + a\cdot\mathit{endValue}, $$

위 수식의 $a$ 는 0부터 1까지의 값을 가집니다.

위 내용을 적용하면, 다음과 같은 결과를 확인할 수 있습니다.

```cpp
#include "color.h"
#include "ray.h"
#include "vec3.h"

#include <iostream>


color ray_color(const ray& r) {
///////////////////////// 추가 ///////////////////////////////
  vec3 unit_direction = unit_vector(r.direction());
  auto a = 0.5 * (unit_direction.y() + 1.0);

  return (1.0 - a) * color(1.0, 1.0, 1.0) + a * color(0.5, 0.7, 1.0);
/////////////////////////////////////////////////////////////
}

...
```

**<p align="center">Listing 10**: [<span>main</span>.cc] _Rendering a blue-to-white gradient</p>_

<p align="center"><img src="https://raytracing.github.io/images/img-1.02-blue-to-white.png"></p>

**<p align="center">Image 2**: _A blue-to-white gradient depeding on ray Y coordinate</p>_

---

## 출처

**Ray Tracing in One Weekend - Peter Shirley**
https://raytracing.github.io/books/RayTracingInOneWeekend.html#rays,asimplecamera,andbackground
