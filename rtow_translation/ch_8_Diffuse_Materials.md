> **이 글은 Peter Shirley의 [Ray Tracing in One Weekend](https://raytracing.github.io/books/RayTracingInOneWeekend.html)를 번역한 것입니다.
> Ray Tracing in One Weekend를 공부하면서 다시 한번 복습하는 느낌으로 번역을 해보려고 합니다. 영어가 서툴러 번역이 잘못되었을 수도 있으므로 잘못된 부분을 발견하신다면 지적해 주시면 감사하겠습니다.**

이제, 물체와 여러 개의 픽셀 당 광선을 만들었으므로, 사실적인 메테리얼을 만들 수 있습니다. 디퓨즈(diffuse(matte)) 메테리얼부터 시작하겠습니다. 한 가지 질문은 지오메트리와 메테리얼을 다양하게 조합(메테리얼을 여러 구에 할당할 수 있습니다. 그 반대도 경우도 가능)할 것인지 아니면 지오메트리와 메테리얼을 강하게 연결(지오메트리와 메테리얼을 연결하는 것은 절차적 객체에 유용합니다.)할 것인지 입니다. 여기서는 지오메트리와 메테리얼을 구분하여 진행하겠습니다(대부분의 렌더러에서 일반적임). 하지만 그 한계점을 인지해야 합니다.

## 8.1 A Simple Diffuse Material

---

빛을 발산하지 않는 디퓨즈 객체는 오직 주변의 빛만 받아들이지만, 받아들인 빛을 객체 고유의 색으로 변조합니다. 디퓨즈 표면에 반사되는 빛은 랜덤한 방향으로 반사됩니다. 만약 광선 세 개를 두 디퓨즈 표면 사이로 쏜다면, 각 빛은 랜덤한 방향으로 서로 다른 움직임을 나타낼 것입니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.08-light-bounce.jpg"></p>

**<p align="center">Figure 8:** _Light ray bounces</p>_

그 빛들은 반사보다는 흡수되는 경향이 있습니다. 더 어두운 표면에서는 흡수가 보다 많이 발생합니다(어두운게 그 이유입니다!). 랜덤한 방향을 지정하는 어떤 알고리즘으로도 무광(matte) 표면을 구현할 수 있습니다. 이것을 구현하는 가장 간단한 방법 중 하나는 이상적인 디퓨즈 표면과 정확히 동일하다고 알려져있습니다.(저는 수학적으로 이상적인 램버시안(Lambertian)과 유사한 방법을 사용했습니다.)

(독자 Vassilen Chizhov님이 위의 방법은 말 그대로 유사한 방법이고 부정확하다는 것을 증명했습니다. 이상적인 램버시안의 올바른 표현은 그다지 많은 작업을 요구하지 않으며, 이 장의 마지막에서 다룰 것입니다.)

표면의 교차점 𝑝에 접하는 2개의 단위구(unit sphere)가 존재합니다. 두 구의 중점은 (𝐏 + 𝐧)과 (𝐏 − 𝐧)이며 𝐧은 표면의 법선 벡터입니다. 중점이 (𝐏 − 𝐧)인 구는 표면의 안쪽에 위치하는 반면, 중점이 (𝐏 + 𝐧)인 구는 표면의 바깥에 위치한다고 생각할 수 있습니다. 표면에 접하는 단위구 중 광선 원점과 같은 쪽에 있는 구를 선택합니다. 선택한 단위구 안의 랜덤한 점 𝐒를 지정하고 교차점 𝐏에서 점 𝐒로 광선을 보냅니다(이 벡터는 (𝐒 − 𝐏)입니다.):

<p align="center"><img src="https://raytracing.github.io/images/fig-1.09-rand-vec.jpg"></p>

**<p align="center">Figure 9:** _Generating a random diffuse bounce ray</p>_

단위구 안의 랜덤한 점을 지정하는 방법이 필요합니다. 일반적으로 가장 쉬운 알고리즘을 사용할 것입니다: 기각 메소드(Rejection method). 먼저, x, y, z 좌표의 범위가 모두 -1부터 +1인 단위 정육면체 안에서 랜덤한 점을 지정합니다. 만약 점이 구의 바깥에 위치한다면 그 점을 기각(Rejection)하고 다시 시도합니다.

```cpp
class vec3 {
public:
  ...
  inline static vec3 random() {
    return vec3(random_double(), random_double(), random_double());
  }

  inline static vec3 random(double min, double max) {
    return vec3(random_double(min, max), random_double(min, max),
    random_double(min, max));
  }
```

**<p align="center">Listing 31:** [vec3.h] _vec3 random utility functions</p>_

```cpp
vec3 random_in_unit_sphere() {
  while (true) {
    auto p = vec3::random(-1, 1);
    if (p.length_squared() >= 1) continue;
    return p;
  }
}
```

**<p align="center">Listing 32:** [vec3.h] _The random_in_unit_sphere() function</p>_

새로운 랜덤 방향 생성기를 사용하기 위해 `ray_color()` 함수를 업데이트합니다.

```cpp
color ray_color(const ray& r, const hittable& world) {
  hit_record rec;

  if (world.hit(r, 0, infinity, rec)) {
/* ************* 수정 ************ */
    point3 target = rec.p + rec.normal + random_in_unit_sphere();
    return 0.5 * ray_color(ray(rec.p, target - rec.p), world);
/* ******************************* */
  }

  vec3 unit_direction = unit_vector(r.direction());
  auto t = 0.5 * (unit_direction.y() + 1.0);
  return (1.0 - t) * color(1.0, 1.0, 1.0) + t * color(0.5, 0.7, 1.0);
}
```

**<p align="center">Listing 33:** [<span>main.</span>cc] _ray_color() using a random ray direction</p>_

---

## 8.2 Limiting the Number of Child Rays

---

한 가지 문제점이 숨어있습니다. `ray_color` 함수는 재귀적입니다. 언제쯤 재귀 호출을 멈출까요? 어떤 것과도 교차하지 못할 때입니다. 하지만 어떤 경우에는, 스택을 파괴시킬만큼 긴 시간이 걸릴 수 있습니다. 이 상황을 방지하기 위해, 최대 재귀 깊이(maximum recursion depth)로 제한을 두어 최대 깊이에서 빛 기여도(light contribution)를 리턴하지 않습니다.

```cpp
/* ************* 수정 ************ */
color ray_color(const ray& r, const hittable& world, int depth) {
/* ******************************* */
  hit_record rec;

/* ************* 추가 ************ */
  // 광선 반사 제한을 초과한다면, 빛이 더 이상 모이지 않습니다.
  if (depth <= 0)
    return color(0, 0, 0);
/* ******************************* */

  if (world.hit(r, 0, infinity, rec)) {
    point3 target = rec.p + rec.normal + random_in_unit_sphere();
/* ************* 수정 ************ */
    return 0.5 * ray_color(ray(rec.p, target - rec.p), world, depth - 1);
/* ******************************* */
  }

  vec3 unit_direction = unit_vector(r.direction());
  auto t = 0.5 * (unit_direction.y() + 1.0);
  return (1.0 - t) * color(1.0, 1.0, 1.0) + t * color(0.5, 0.7, 1.0);
}

...

int main() {

  // Image
  const auto aspect_ratio = 16.0 / 9.0;
  const int image_width = 400;
  const int image_height = static_cast<int>(image_width / aspect_ratio);
  const int samples_per_pixel = 100;
/* ************* 추가 ************ */
  const int max_depth = 50;
/* ******************************* */

  ...
  // Render

  std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";

  for (int j = image_height - 1; j >= 0; --j) {
    std::cerr << "\rScanlines remaining: " << j << ' ' << std::flush;
    for (int i = 0; i < image_width; ++i) {
      color pixel_color(0, 0, 0);
      for (int s = 0; s < samples_per_pixel; ++s) {
        auto u = (i + random_double()) / (image_width - 1);
        auto v = (j + random_double()) / (image_height - 1);
        ray r = cam.get_ray(u, v);
/* ************* 수정 ************ */
        pixel_color += ray_color(r, world, max_depth);
/* ******************************* */
      }
      write_color(std::cout, pixel_color, samples_per_pixel);
    }
  }

  std::cerr << "\nDone.\n";
}
```

**<p align="center">Listing 34:** [<span>main.</span>cc] _ray_color() with depth limiting</p>_

다음과 같은 이미지를 얻을 수 있습니다:

<p align="center"><img src="https://raytracing.github.io/images/img-1.07-first-diffuse.png"></p>

**<p align="center">Image 7:** _First render of a diffuse sphere</p>_

---

## 8.3 Using Gamma Correction for Accurate Color Intensity

---

구 아래의 그림자를 주목하십시오. 이 이미지는 매우 어둡습니다. 구는 각 반사마다 절반의 에너지를 흡수합니다. 그러므로 구는 50% 반사체입니다. 만약 그림자가 보이지 않아도 걱정하지 마십시오. 지금부터 그 부분을 수정하겠습니다. 이 구는 굉장히 밝게 보여야 합니다(실제로는 밝은 회색). 그렇기 때문에 거의 모든 이미지 뷰어는 이미지가 "감마 보정(gamma corrected) : 0부터 1까지의 값이 바이트로 저장되기 전에 약간의 변환이 있음을 의미"된다고 가정합니다. 감마 보정을 하는 여러 이유가 있지만, 우리는 우리의 목적을 위해 그것을 인지하고만 있으면 됩니다. 첫 번째 근삿값으로 "감마 2(gamma 2)"를 사용할 수 있습니다. 색상을 1/𝑔𝑎𝑚𝑚𝑎로 제곱한다는 의미입니다. 간단한 경우인 ½는 제곱근을 의미합니다:

```cpp
void write_color(std::ostream &out, color pixel_color, int samples_per_pixel) {
  auto r = pixel_color.x();
  auto g = pixel_color.y();
  auto b = pixel_color.z();

/* ************* 수정 ************ */
  // Divide the color by the number of samples and gamma-correct for gamma=2.0
  auto scale = 1.0 / samples_per_pixel;
  r = sqrt(scale * r);
  g = sqrt(scale * g);
  b = sqrt(scale * b);
/* ******************************* */

  // Write the translated [0, 255] value of each color component.
  out << static_cast<int>(256 * clamp(r, 0.0, 0.999)) << ' '
    << static_cast<int>(256 * clamp(g, 0.0, 0.999)) << ' '
    << static_cast<int>(256 * clamp(b, 0.0, 0.999)) << '\n';
}
```

**<p align="center">Listing 35:** [color.h] _write_color(), with gamma correction</p>_

원하는 밝은 회색의 이미지를 얻을 수 있습니다:

<p align="center"><img src="https://raytracing.github.io/images/img-1.08-gamma-correct.png"></p>

**<p align="center">Image 8:** _Diffuse sphere, with gamma correction</p>_

---

## 8.4 Fixing Shadow Acne

---

위의 이미지에는 버그가 조금 있습니다. 반사된 광선 중 일부는 정확한 𝑡 = 0에서 교차가 아닌, 𝑡 = −0.0000001 이나 𝑡 = 0.00000001 또는 구 교차점의 부동 소수점 근삿값에서 물체와 교차합니다. 그러므로 0에 매우 가까운 교차들은 무시할 필요가 있습니다.

```cpp
if (world.hit(r, 0.001, infinity, rec)) {
```

**<p align="center">Listing 36:** [<span>main.</span>cc] _Calculating reflected ray origins with tolerance</p>_

이렇게 하면 그림자 결함(Shadow acne) 문제를 해결할 수 있습니다. 네, 정말로 Shadow acne라고 불립니다.

> <p align="center"><img src="https://user-images.githubusercontent.com/19530862/97144711-46850780-17a8-11eb-96f3-c47952dc1322.png"></p>
> <p align="center">❗그림자 결함을 수정한 이미지입니다. 원문에 없어서 추가했습니다. 그림자가 좀 더 깔끔해진 것을 확인할 수 있습니다.</p>

---

## 8.5 True Lambertian Reflection

---

여기서 제공된 기각 메소드(rejection method)는 표면 법선 벡터를 따라 오프셋 된 단위 구의 랜덤한 점을 생성합니다.
법선 벡터에 가까운
This corresponds to picking directions on the hemisphere with high probability close to the normal, and a lower probability of scattering rays at grazing angles. 이 분포는 cos<sup>3</sup>(𝜙)로 스케일 됩니다. 𝜙는 법선 벡터로부터 각도입니다. 얕은 각도에 도달하는 빛은 더 넓은 영역에 퍼지므로 최종 색상에 기여도가 낮기 때문에 유용합니다.

하지만, 우리는 cos(𝜙)의 분포를 가진 램버시안 분포에 관심이 있습니다. 진짜 램버시안은 법선 벡터에 가까울수록 광선 산란율이 더 크지만 분포는 일정합니다. 이는 표면 법선 벡터를 따라 오프셋 된 단위구 표면의 점을 선택하여 이루어집니다. 구에서 점을 선택하는 것은 단위구에서 점은 선택한 다음, 정규화하여 구할 수 있습니다.

```cpp
vec3 random_unit_vector() {
  auto a = random_double(0, 2 * pi);
  auto z = random_double(-1, 1);
  auto r = sqrt(1 - z * z);
  return vec3(r * cos(a), r * sin(a), z);
}
```

**<p align="center">Listing 37:** [vec3.h] _The random_unit_vector() function</p>_

<p align="center"><img src="https://raytracing.github.io/images/fig-1.10-rand-unitvec.png"></p>

**<p align="center">Figure 10:** _Generating a random unit vector</p>_

random_unit_vector() 함수는 기존 random_in_unit_sphere() 함수를 손쉽게 대체합니다.

```cpp
color ray_color(const ray& r, const hittable& world, int depth) {
  hit_record rec;

  // 광선 반사 제한을 초과한다면, 빛이 더 이상 모이지 않습니다.
  if (depth <= 0)
    return color(0, 0, 0);

  if (world.hit(r, 0.001, infinity, rec)) {
/* ************* 수정 ************ */
    point3 target = rec.p + rec.normal + random_unit_vector();
/* ******************************* */
    return 0.5 * ray_color(ray(rec.p, target - rec.p), world, depth - 1);
  }

  vec3 unit_direction = unit_vector(r.direction());
  auto t = 0.5 * (unit_direction.y() + 1.0);
  return (1.0 - t) * color(1.0, 1.0, 1.0) + t * color(0.5, 0.7, 1.0);
}
```

**<p align="center">Listing 38:** [<span>main.</span>cc] _ray_color() with replacement diffuse</p>_

렌더링 결과, 기존 이미지와 비슷한 이미지를 얻을 수 있습니다:

<p align="center"><img src="https://raytracing.github.io/images/img-1.09-correct-lambertian.png"></p>

**<p align="center">Image 9:** _Correct rendering of Lambertian spheres</p>_

씬의 구가 매우 간단해서 두 디퓨즈 메소드의 차이점을 구분하기 어렵습니다. 하지만 두 가지 중요한 시각적 차이점을 눈치챌 수 있어야 합니다.

1. 그림자가 기존보다 옅어졌습니다.
2. 구가 기존보다 밝아졌습니다.

이러한 차이점은 빛 광선의 일정한 산란때문이며, 더 적은 광선이 법선 벡터 방향으로 산란됩니다. 디퓨즈 오브젝트의 경우, 더 많은 빛이 카메라를 향해 반사되므로 더 밝게 나타납니다. 그림자의 경우, 더 적은 빛이 수직으로 반사되므로 큰 구의 부분 중 작은 구 바로 아래 부분이 더 밝습니다.

---

## 8.6 An Alternative Diffuse Formulation

---

이 책의 초기 램버시안 계산은 이상적인 램버시안 디퓨즈의 틀린 근사라는 것이 증명되기 전까지 오랜 시간 동안 유지되었습니다. 오랜 시간 동안 오류가 수정되지 않았던 큰 이유는, 다음과 같은 작업이 어려울 수 있기 때문입니다:

1. 확률 분포가 부정확하다는 사실을 수학적으로 증명하는 것
2. cos(𝜙) 분포가 바람직하다는(그리고 그것이 어떻게 생겼는지) 이유를 직관적으로 설명하는 것

모든 물체들이 완벽하게 디퓨즈 메테리얼인 상황이 흔치 않으므로, 이러한 물체들이 빛 아래에서 어떻게 나타나는지 우리의 시각적인 직관이 제대로 형성되지 않을 수 있습니다.

학습의 흥미를 위해, 직관적이고 이해하기 쉬운 디퓨즈 메소드를 담고 있습니다. 위의 두 가지 메소드에 대해, 우리는 랜덤 길이와 단위 길이의 랜덤 벡터를 가지고 있었으며, 교차점에서 법선 벡터에 의해 오프셋 됩니다. 그 벡터들이 법선 벡터로 대체되는 이유가 당장은 명확하지 않을 수 있습니다.

보다 더 직관적인 접근은 법선 벡터로부터의 각도에 의존하지 않고 교차점에서 모든 각도에 대해 균일한 산란 방향을 가지는 것입니다. 많은 초기 레이 트레이싱 논문들은 이 디퓨즈 메소드를 사용했었습니다(램버시안 디퓨즈를 적용하기 전에).

```cpp
vec3 random_in_hemisphere(const vec3 &normal) {
  vec3 in_unit_sphere = random_in_unit_sphere();
  if (dot(in_unit_sphere, normal) > 0.0) // In the same hemisphere as the normal
    return in_unit_sphere;
  else
    return -in_unit_sphere;
}
```

**<p align="center">Listing 39:** [vec3.h] _The random_in_hemisphere(normal) function</p>_

`ray_color()` 함수에 새 공식을 적용합니다:

```cpp
color ray_color(const ray& r, const hittable& world, int depth) {
  hit_record rec;

  // 광선 반사 제한을 초과한다면, 빛이 더 이상 모이지 않습니다.
  if (depth <= 0)
    return color(0, 0, 0);

  if (world.hit(r, 0.001, infinity, rec)) {
/* ************* 수정 ************ */
    point3 target = rec.p + random_in_hemisphere(rec.normal);
/* ******************************* */
    return 0.5 * ray_color(ray(rec.p, target - rec.p), world, depth - 1);
  }

  vec3 unit_direction = unit_vector(r.direction());
  auto t = 0.5 * (unit_direction.y() + 1.0);
  return (1.0 - t) * color(1.0, 1.0, 1.0) + t * color(0.5, 0.7, 1.0);
}
```

**<p align="center">Listing 40:** [<span>main.</span>cc] _ray_color() with hemispherical scattering</p>_

다음과 같은 이미지를 얻을 수 있습니다:

<p align="center"><img src="https://raytracing.github.io/images/img-1.10-rand-hemispherical.png"></p>

**<p align="center">Image 9:** _Rendering of diffuse spheres with hemispherical scattering</p>_

씬은 이 책의 과정 동안 더 복잡해질 것입니다. 여기서 제공되는 서로 다른 디퓨즈 렌더러들 사이의 전환을 권장합니다. 대부분의 장면에는 불균형한 수의 디퓨즈 메테리얼이 포함됩니다. 씬의 조명에 대한 다양한 디퓨즈 메소드 효과를 이해하면 귀중한 통찰력을 얻을 수 있습니다.

---

## 출처

**Ray Tracing in One Weekend - Peter Shirley**
https://raytracing.github.io/books/RayTracingInOneWeekend.html#diffusematerials
