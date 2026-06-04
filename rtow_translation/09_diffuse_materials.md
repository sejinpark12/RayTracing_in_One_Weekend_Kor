# 9. Diffuse Materials
이제 오브젝트도 있고 픽셀당 여러 개의 광선을 쏘는 안티엘리어싱도 갖추었기 때문에, 사실적으로 보이는 머티리얼을 만들 수 있습니다. 먼저 디퓨즈 머티리얼부터 시작하겠습니다. 디퓨즈 머티리얼은 _매트_(_matte_) 라고도 부릅니다. 한 가지 설계적 고민은 지오메트리와 머티리얼을 이것저것 섞어서 조합할지 아니면 지오메트리와 머티리얼을 강하게 결합할지입니다. 전자의 방식에서는 하나의 머티리얼을 여러 개의 구에 적용하거나 그 반대도 가능합니다. 후자의 방식은 지오메트리와 머티리얼이 서로 연결된 절차적 객체에서 유용할 수 있습니다. 여기서는 전자의 방식을 따르겠습니다. 대부분의 렌더러에서 일반적으로 이 방식을 사용합니다. 그러나 다른 방식들도 있다는 것을 꼭 알아두세요.

## 9.1 A Simple Diffuse Material
스스로 빛을 방출하지 않는 디퓨즈 오브젝트는 단순히 주변의 색을 받아서 그대로 띠는 것이 아니라, 실제로는 받은 주변 색에 자기 고유의 색을 반영하여 색을 띱니다. 디퓨즈 표면에서 반사되는 빛은 랜덤한 방향으로 반사됩니다. 그래서 두 디퓨즈 표면 사이 틈으로 세 개의 광선을 쏘면 각 광선은 서로 다른 랜덤한 방향으로 향하게 됩니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.09-light-bounce.jpg"></p>

**<p align="center">Figure 9:** _Light ray bounces</p>_

또한 광선은 반사되지 않고 흡수될 수도 있습니다. 광선이 더 많이 흡수될수록 표면은 더 어둡게 보입니다. 사실, 랜덤 방향을 생성할 수 있는 알고리즘이라면 어떤 것이든, 표면을 매트하게 보이도록 할 수 있습니다. 가장 직관적인 방법부터 시작해 보겠습니다. 광선이 모든 방향으로 동일한 확률로 랜덤하게 반사되는 표면을 사용하는 방식입니다. 이 머티리얼에서는, 표면에 충돌하는 광선은 표면으로부터 반사되는 어떤 방향으로든 동일한 확률로 반사됩니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.10-random-vec-horizon.jpg"></p>

**<p align="center">Figure 10:** _Equal reflection above the horizon</p>_

이런 매우 직관적인 머티리얼이 디퓨즈 머티리얼의 가장 단순한 형태입니다. 그리고 뒤에서 구현할 좀 더 정확한 방식을 적용하기 전까지는, 실제로도 많은 초기 레이 트레이싱 논문들에서 이 디퓨즈 머티리얼을 사용했습니다. 현재 코드에는 아직 광선을 랜덤하게 반사하는 로직이 없으므로 벡터 유틸리티 헤더에 몇 가지 함수를 추가해야 합니다. 첫 번째로 필요한 것은 임의의 랜덤 벡터를 생성하는 기능입니다.

```cpp
class vec3 {
  public:
    ...

    double length_squared() const {
      return e[0] * e[0] + e[1] * e[1] + e[2] * e[2];
    }

///////////////////////// 추가 /////////////////////////////////////////////////////////////////////
    static vec3 random() {                                                                       //
      return vec3(random_double(), random_double(), random_double());                            //
    }                                                                                            //
                                                                                                 //
    static vec3 random(double min, double max) {                                                 //
      return vec3(random_double(min, max), random_double(min, max), random_double(min, max));    //
    }                                                                                            //
///////////////////////////////////////////////////////////////////////////////////////////////////
};
```

**<p align="center">Listing 47:** [vec3.h] _vec3 random utility functions</p>_

랜덤 벡터를 어떻게 다뤄야 랜덤 벡터의 끝점이 반구의 표면 위에만(반구 표면 방향) 오도록 할 수 있을지 생각해야 합니다. 이를 위한 해석적 방법(analytical method)들이 있지만, 사실 이해하기 꽤 복잡하고 구현도 상당히 복잡합니다. 대신, 일반적으로 가장 쉬운 알고리즘인 기각법(rejection method)을 사용하겠습니다. 기각법은 원하는 기준을 만족하는 샘플이 나올 때까지 랜덤 샘플을 반복적으로 생성하는 방식으로 동작합니다. 다시 말해서, 기준에 맞는 샘플 하나를 찾을 때까지 그렇지 못한 샘플을 계속 버리라는 의미입니다.

기각법을 사용하여 반구 표면 위의 랜덤 벡터를 생성하는 방법은 여러 가지가 있습니다. 그렇지만, 여기서는 목적상 가장 단순한 방법을 선택하겠습니다. 그 방법은 다음과 같습니다.

1. 단위 구 내부의 랜덤 벡터 하나를 생성합니다.
2. 이 벡터를 정규화하여 벡터의 끝을 구 표면까지 연장합니다.
3. 정규화된 벡터가 잘못된 반구로 향한다면 벡터의 방향을 반대로 뒤집습니다.

먼저, 단위 구 내부에 위치한 랜덤 벡터를 생성하기 위해서 기각법을 사용합니다. 단위 구는 반지름이 1인 구입니다. 단위 구를 감싸고 있는 정육면체 내부에서 임의의 점 하나를 선택합니다. 다시 말해서, 이 점의 $x$, $y$, $z$ 성분들의 범위는 $[-1, +1]$ 입니다. 이 점이 단위 구 외부에 있다면, 단위 구 내부 또는 단위 구 표면에 위치하는 점이 나올 때까지 점을 생성합니다. 

<p align="center"><img src="https://raytracing.github.io/images/fig-1.11-sphere-vec.jpg"></p>

**<p align="center">Figure 11:** _Two vectors were rejected before finding a good one (pre-normalization)</p>_

<p align="center"><img src="https://raytracing.github.io/images/fig-1.12-sphere-unit-vec.jpg"></p>

**<p align="center">Figure 12:** _The accepted random vector is normalized to produce a unit vector</p>_

다음은 이 함수의 첫 번째 구현입니다.

```cpp
...

inline vec3 unit_vector(const vec3& v) {
  return v / v.length();
}

///////////////////////// 추가 //////////////////////
inline vec3 random_unit_vector() {                //
  while (true) {                                  //
    auto p = vec3::random(-1, 1);                 //
    auto lensq = p.length_squared();              //
    if (lensq <= 1)                               //
      return p / sqrt(lensq);                     //
  }                                               //
}                                                 //
////////////////////////////////////////////////////
```

**<p align="center">Listing 48:** [vec3.h] _The random\_unit\_vector() function, version one</p>_

안타깝게도, 이 코드에서는 부동소수점 때문에 발생하는 누수 문제를 해결해야 합니다. 부동소수점 숫자의 정밀도는 유한하기 때문에, 매우 작은 숫자를 제곱하면 언더플로우가 발생하여 0이 될 수 있습니다. 따라서 벡터 끝점의 세 성분이 모두 충분히 작아 구의 중심에 매우 가깝다면, 벡터의 크기는 0이 될 것이고 이 벡터를 정규화하면 $[\pm\infty, \pm\infty, \pm\infty]$ 의 잘못된 벡터가 됩니다. 이런 문제를 해결하기 위해서, 구 중심 근처의 "블랙홀"(언더플로우가 발생할 수 있는 위치) 안에 있는 점들도 사용하지 않을 것입니다. 배정밀도(double precision, 64-bit float)를 사용하면 $10^{-160}$ 보다 큰 값은 안전하게 다룰 수 있습니다.

다음은 이를 반영하여 더 안전하게 만든 함수입니다.

```cpp
inline vec3 random_unit_vector() {
  while (true) {
    auto p = vec3::random(-1, 1);
    auto lensq = p.length_squared();
///////////////////////// 수정 //////////////////////
    if (1e-160 < lensq && lensq <= 1)             //
////////////////////////////////////////////////////
      return p / sqrt(lensq);
  }
}
```

**<p align="center">Listing 49:** [vec3.h] _The random\_unit\_vector() function, version two</p>_

이제 랜덤 단위 벡터를 만들었으니, 표면 법선 벡터와 비교하여 올바른 반구에 위치하는지 판별할 수 있습니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.13-surface-normal.jpg"></p>

**<p align="center">Figure 13:** _The normal vector tells us which hemisphere we need</p>_

올바른 반구에 랜덤 벡터가 위치하는지 판별하기 위해서 표면 법선 벡터와 랜덤 벡터의 내적을 구합니다. 내적 값이 양수라면, 랜덤 벡터는 올바른 반구에 위치합니다. 내적 값이 음수라면, 랜덤 벡터를 반대 방향으로 뒤집어야 합니다.

```cpp
...

inline vec3 random_unit_vector() {
  while (true) {
    auto p = vec3::random(-1, 1);
    auto lensq = p.length_squared();
    if (1e-160 < lensq && lensq <= 1)
      return p / sqrt(lensq);
  }
}

///////////////////////// 추가 ////////////////////////////////////////////////////////
inline vec3 random_on_hemisphere(const vec3& normal) {                              //
  vec3 on_unit_sphere = random_unit_vector();                                       //
  if (dot(on_unit_sphere, normal) > 0.0) // In the same hemisphere as the normal    //
    return on_unit_sphere;                                                          //
  else                                                                              //
    return -on_unit_sphere;                                                         //
}                                                                                   //
//////////////////////////////////////////////////////////////////////////////////////
```

**<p align="center">Listing 50:** [vec3.h] _The random\_on\_hemisphere() function</p>_

만약 광선이 어떤 머티리얼에 반사된 뒤에도 광선이 가지고 있던 색을 100% 유지한다면, 그 머티리얼은 _흰색_ 이라고 합니다. 만약 광선이 어떤 머티리얼에 반사되어 광선이 가지고 있던 색을 0% 유지한다면, 그 머티리얼은 _검은색_ 이라고 합니다. 새로운 디퓨즈 머티리얼의 첫 번째 예시로, `ray_color` 함수가 한 번 반사된 광선 색의 50%만 반환하도록 설정하겠습니다. 그러면 회색이 나올 것입니다.

```cpp
class camera {
  ...
  private:
    ...
    color ray_color(const ray& r, const hittable& world) const {
      hit_record rec;

      if (world.hit(r, interval(0, infinity), rec)) {
///////////////////////// 수정 ///////////////////////////////////////
        vec3 direction = random_on_hemisphere(rec.normal);         //
        return 0.5 * ray_color(ray(rec.p, direction), world);      //
/////////////////////////////////////////////////////////////////////
      }

      vec3 unit_direction = unit_vector(r.direction());
      auto a = 0.5 * (unit_direction.y() + 1.0);
      return (1.0 - a) * color(1.0, 1.0, 1.0) + a * color(0.5, 0.7, 1.0);
    }
};
```

**<p align="center">Listing 51:** [camera.h] _ray\_color() using a random ray direction</p>_

...실제로도 꽤 괜찮은 회색 구들이 렌더링됩니다.

<p align="center"><img src="https://raytracing.github.io/images/img-1.07-first-diffuse.png"></p>

**<p align="center">Image 7:** _First render of a diffuse sphere</p>_

---

## 9.2 Limiting the Number of Child Rays

여기에는 문제가 될 수 있는 부분이 하나 있습니다. `ray_color` 함수가 재귀함수라는 점에 주목하세요. 재귀는 언제 멈출까요? 광선이 더 이상 어떤 물체와도 충돌하지 못할 때 입니다. 다만 어떤 경우에 따라서는 그때까지 스택 오버플로우를 일으킬 정도로 오래 걸릴 수도 있습니다. 이 문제를 해결하기 위해, 최대 깊이에 도달하면 빛 기여를 0으로 하여 최대 재귀 깊이를 제한하겠습니다. 

```cpp
class camera {
  public:
    double aspect_ratio = 1.0;     // Ratio of image width over height
    int    image_width = 100;      // Rendered image width in pixel count
    int    samples_per_pixel = 10; // Count of random samples for each pixel
///////////////////////// 추가 //////////////////////////////////////////////////////
    int    max_depth = 10;         // Maximum number of ray bounces into scene    //
////////////////////////////////////////////////////////////////////////////////////

    void render(const hittable& world) {
      initialize();

      std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";

      for (int j = 0; j < image_height; j++) {
        std::clog << "\rScanlines remaining: " << (image_height - j) << ' ' << std::flush;
        for (int i = 0; i < image_width; i++) {
          color pixel_color(0, 0, 0);
          for (int sample = 0; sample < samples_per_pixel; sample++) {
            ray r = get_ray(i, j);
///////////////////////// 수정 //////////////////////////////////////////////
            pixel_color += ray_color(r, max_depth, world);                //
////////////////////////////////////////////////////////////////////////////
          }
          write_color(std::cout, pixel_samples_scale * pixel_color);
        }
      }

      std::clog << "\rDone.                 \n";
    }
    ...
  private:
    ...
///////////////////////// 수정 ///////////////////////////////////////////////////
    color ray_color(const ray& r, int depth, const hittable& world) const {    //
      // If we've exceeded the ray bounce limit, no more light is gathered.    //
      if (depth <= 0)                                                          //
        return color(0, 0, 0);                                                 //
/////////////////////////////////////////////////////////////////////////////////

      hit_record rec;

      if (world.hit(r, interval(0, infinity), rec)) {
        vec3 direction = random_on_hemisphere(rec.normal);
///////////////////////// 수정 ///////////////////////////////////////////////////
        return 0.5 * ray_color(ray(rec.p, direction), depth - 1, world);       //
/////////////////////////////////////////////////////////////////////////////////
      }

      vec3 unit_direction = unit_vector(r.direction());
      auto a = 0.5 * (unit_direction.y() + 1.0);
      return (1.0 - a) * color(1.0, 1.0, 1.0) + a * color(0.5, 0.7, 1.0);
    }
};
```

**<p align="center">Listing 52:** [camera.h] _camera::ray\_color() with depth limiting</p>_

새로 추가한 깊이 제한 변수(max_depth)를 사용하도록 main() 함수를 수정합니다.

```cpp
int main() {
  ...

  camera cam;

  cam.aspect_ratio      = 16.0 / 9.0;
  cam.image_width       = 400;
  cam.samples_per_pixel = 100;
///////////////////////// 추가 ////////////////////////
  cam.max_depth         = 50;                       //
//////////////////////////////////////////////////////

  cam.render(world);
}
```

**<p align="center">Listing 53:** [main<span></span>.cc] _Using the new ray depth limiting</p>_

이런 아주 단순한 씬에서는 기본적으로 같은 결과가 나와야 합니다.

<p align="center"><img src="https://raytracing.github.io/images/img-1.08-second-diffuse.png"></p>

**<p align="center">Image 8:** _Second render of a diffuse sphere with limited bounces</p>_

---

## 9.3 Fixing Shadow Acne

여기에는 해결해야 할 숨은 버그가 하나 더 있습니다. 광선은 표면과 교차할 때, 교차점을 정확하게 계산하려고 시도합니다. 하지만 안타깝게도 이 계산은 부동소수점 반올림 오차에 영향을 받기 쉽고, 이 오차로 인해 교차점이 아주 약간씩 어긋날 수 있습니다. 이 말은 표면에서 랜덤하게 산란되어 나가는 다음 광선의 시작점이 표면과 완벽하게 딱 맞닿아 있을 가능성이 낮다는 의미입니다. 그 시작점은 표면 바로 위에 있을 수도 있고, 바로 아래에 있을 수도 있습니다. 만약 광선의 시작점이 표면 바로 아래에 있다면, 광선은 동일한 표면과 다시 교차할 수 있습니다. 이는 광선이 찾는 가장 가까운 표면을 hit 함수가 계산해 낸 부동소수점 근삿값인 $t=0.00000001$ 같은 거의 0에 가까운 값에서 찾게 된다는 뜻입니다. 이 문제를 해결하는 가장 간단한 방법은 계산된 교차점에 아주 가까운 hit들을 단순히 무시하는 것입니다.

```cpp
class camera {
  ...
  private:
    ...
    color ray_color(const ray& r, int depth, const hittable& world) const {
      // If we've exceeded the ray bounce limit, no more light is gathered.
      if (depth <= 0)
        return color(0, 0, 0);

      hit_record rec;

///////////////////////// 수정 //////////////////////////////////
      if (world.hit(r, interval(0.001, infinity), rec)) {     //
////////////////////////////////////////////////////////////////
        vec3 direction = random_on_hemisphere(rec.normal);
        return 0.5 * ray_color(ray(rec.p, direction), depth - 1, world);
      }

      vec3 unit_direction = unit_vector(r.direction());
      auto a = 0.5 * (unit_direction.y() + 1.0);
      return (1.0 - a) * color(1.0, 1.0, 1.0) + a * color(0.5, 0.7, 1.0);
    }
};
```

**<p align="center">Listing 54:** [camera.h] _Calculating reflected ray origins with tolerance</p>_

이렇게 하면 그림자 여드름(shadow acne) 문제가 사라집니다. 네, 실제로 사용되는 용어입니다. 결과는 다음과 같습니다.

<p align="center"><img src="https://raytracing.github.io/images/img-1.09-no-acne.png"></p>

**<p align="center">Image 9:** _Diffuse sphere with no shadow acne</p>_

---

## 9.4 True Lambertian Reflection

위의 방식으로 반사 광선을 반구 전체에 고르게 산란시키면 멋진 부드러운 디퓨즈 모델을 만들 수 있습니다. 하지만 더 나은 방법이 분명히 존재합니다. _램버시안_ 분포(_Lambertian_ distribution)를 사용하면 실제의 디퓨즈 오브젝트를 더 정확하게 표현할 수 있습니다. 이 분포는 반사된 광선을 $\cos(\phi)$ 에 비례하여 산란시킵니다. 여기서 $\phi$ 는 반사된 광선과 표면 법선 벡터 사이의 각도입니다. 이 말은 반사 광선이 표면 법선 벡터 근처 방향으로 산란될 가능성이 높고, 법선에서 멀어지는 방향으로 반사될 가능성은 낮다는 의미입니다. 비균일 램버시안 분포는 이전의 균일 산란 모델보다 실제 세계의 머티리얼 반사를 더 잘 모델링합니다.

법선 벡터에 랜덤 단위 벡터를 더함으로써 램버시안 분포를 만들 수 있습니다. 표면 교차점에는 교차점 $\mathbf{p}$ 와 표면 법선 벡터 $\mathbf{n}$ 이 존재합니다. 교차점에서 표면은 정확히 두 면(앞면, 뒷면)을 가지므로, 그 교차점에 접할 수 있는 단위 구는 오직 두 개뿐입니다. 각 면마다 하나씩 있는 셈입니다. 이 두 단위 구는 반지름만큼 표면에서 떨어져 있으며, 단위 구의 반지름은 정확히 1입니다.

한 구는 표면의 법선 벡터 ($\mathbf{n}$) 방향으로 떨어져 있고, 다른 한 구는 그 반대인 ($\mathbf{-n}$) 방향으로 떨어져 있습니다. 그 결과, 교차점에서 단위 구 두 개가 표면에 _딱_ 접해 있는 상태가 됩니다. 이로 인해, 한 구의 중심은 $(\mathbf{P} + \mathbf{n})$ 이고 다른 한 구의 중심은 $(\mathbf{P} - \mathbf{n})$ 이 됩니다. 중심이 $(\mathbf{P} - \mathbf{n})$ 인 구는 표면의 안쪽에 있는 것으로 보고, 반면 중심이 $(\mathbf{P} + \mathbf{n})$ 인 구는 표면의 바깥쪽에 있는 것으로 봅니다.

광선의 원점과 같은 쪽에 있는 면에 접한 단위 구를 선택합니다. 그리고 단위 구 위에서 랜덤으로 점 $\mathbf{S}$ 를 뽑고, 교차점 $\mathbf{P}$ 에서 점 $\mathbf{S}$ 를 향해 광선을 보냅니다. 그 결과, 방향 벡터는 $(\mathbf{S}-\mathbf{P})$ 가 됩니다. 

<p align="center"><img src="https://raytracing.github.io/images/fig-1.14-rand-unitvec.jpg"></p>

**<p align="center">Figure 14:** _Randomly generating a vector according to Lambertian distribution</p>_

실제로 바뀌는 부분은 많지 않습니다.

```cpp
class camera {
  ...
  color ray_color(const ray& r, int depth, const hittable& world) const {
    // If we've exceeded the ray bounce limit, no more light is gathered.
    if (depth <= 0)
      return color(0, 0, 0);

    hit_record rec;

    if (world.hit(r, interval(0.001, infinity), rec)) {
///////////////////////// 수정 //////////////////////////////////////////////
      vec3 direction = rec.normal + random_unit_vector();                 //
////////////////////////////////////////////////////////////////////////////
      return 0.5 * ray_color(ray(rec.p, direction), depth - 1, world);
    }

    vec3 unit_direction = unit_vector(r.direction());
    auto a = 0.5 * (unit_direction.y() + 1.0);
    return (1.0 - a) * color(1.0, 1.0, 1.0) + a * color(0.5, 0.7, 1.0);
  }
};
```

**<p align="center">Listing 55:** [camera.h] _ray\_color() with replacement diffuse</p>_

렌더링하면 이전과 비슷한 이미지가 렌더링됩니다.

<p align="center"><img src="https://raytracing.github.io/images/img-1.10-correct-lambertian.png"></p>

**<p align="center">Image 10:** _Correct rendering of Lambertian spheres</p>_

두 개의 구로 구성된 씬은 너무 단순해서 여기서 두 디퓨즈 방식의 차이점을 구분하는 것은 어렵습니다. 하지만 그렇더라도 중요한 두 가지 시각적 차이점은 확인할 수 있습니다.

1. 변경 후에 그림자가 더 뚜렷해졌습니다.
2. 변경 후에 두 구 모두 하늘의 영향을 받아 푸른색을 띱니다.

이 두 변화 모두 광선이 덜 균일하게 산란되어 발생합니다. 즉, 더 많은 광선이 법선 벡터 방향으로 산란되기 때문입니다. 이 말은 디퓨즈 오브젝트에서는 카메라 쪽으로 반사되는 빛이 더 적기 때문에 더 어둡게 보인다는 의미입니다. 그림자의 경우에는 더 많은 빛이 표면 바로 위쪽 방향(법선 벡터 방향)으로 반사되기 때문에 구 아래 영역이 더 어두워집니다.

일상에서 보는 흔한 오브젝트들 중에 완벽하게 디퓨즈한 것들은 많지 않습니다. 그래서 이런 오브젝트들이 빛 아래에서 어떻게 보이는지에 대한 시각적 직관은 부정확할 수 있습니다. 앞으로 씬이 더 복잡해짐에 따라, 여기서 소개된 서로 다른 디퓨즈 렌더러들을 서로 바꿔 가며 사용해 보는 것이 좋습니다. 대부분의 흥미로운 씬에는 많은 디퓨즈 머티리얼이 포함되어 있습니다. 서로 다른 디퓨즈 방식이 씬의 라이팅에 어떤 영향을 주는지 이해함으로써 의미 있는 통찰을 얻을 수 있습니다.

## 9.5 Using Gamma Correction for Accurate Color Intensity

구 아래의 그림자에 주목하세요. 이 이미지는 매우 어둡습니다. 하지만 구들은 각 반사마다 에너지의 절반만 흡수하므로 이 구들은 50% 반사체입니다. 따라서 구들은 꽤 밝게 보여야 하고 현실에서는 연한 회색 정도로 보이는 것이 자연스럽습니다. 하지만 렌더링 결과는 더 어둡게 보입니다. 디퓨즈 머티리얼의 전체 밝기 영역을 단계적으로 살펴보면, 이 어둡게 보이는 현상을 더 명확하게 이해할 수 있습니다. 먼저 `ray_color` 함수의 반사율을 `0.5` (50%)에서 `0.1` (10%)로 낮춰 설정하겠습니다.

```cpp
class camera {
  ...
  color ray_color(const ray& r, int depth, const hittable& world) const {
    // If we've exceeded the ray bounce limit, no more light is gathered.
    if (depth <= 0)
      return color(0, 0, 0);

    hit_record rec;

    if (world.hit(r, interval(0.001, infinity), rec)) {
      vec3 direction = rec.normal + random_unit_vector();
///////////////////////// 수정 ////////////////////////////////////////////////
      return 0.1 * ray_color(ray(rec.p, direction), depth - 1, world);      //
//////////////////////////////////////////////////////////////////////////////
    }

    vec3 unit_direction = unit_vector(r.direction());
    auto a = 0.5 * (unit_direction.y() + 1.0);
    return (1.0 - a) * color(1.0, 1.0, 1.0) + a * color(0.5, 0.7, 1.0);
  }
};
```

**<p align="center">Listing 56:** [camera.h] _ray\_color() with 10% reflectance</p>_

새로 설정한 10% 반사율로 렌더링을 합니다. 그리고 30% 반사율로 설정하고 다시 렌더링을 합니다. 50%, 70%, 90%까지 렌더링을 반복합니다. 원하는 사진 편집기를 사용하여 렌더링 이미지들을 왼쪽에서 오른쪽 순서로 겹쳐 볼 수 있습니다. 그러면 설정된 영역에서 밝기가 증가하는 아주 멋진 시각적 결과를 얻을 수 있습니다. 다음의 이미지가 지금까지 작업해 온 것입니다.

<p align="center"><img src="https://raytracing.github.io/images/img-1.11-linear-gamut.png"></p>

**<p align="center">Image 11:** _The gamut of our renderer so far</p>_

이미지를 자세히 살펴보거나 컬러 피커를 사용해 보면, 50% 반사율 렌더링(가운데 결과)이 흰색과 검은색의 중간값인 중간 회색(middle-gray)으로 보기에는 너무 어두운 것을 눈치챌 수 있어야 합니다. 실제로 70% 반사체가 중간 회색에 더 가깝습니다. 그 이유는 대부분의 컴퓨터 프로그램은 이미지가 파일에 저장되기 전에 감마 보정(gamma-corrected) 된다고 가정하기 때문입니다. 이 말은 0부터 1까지의 값들이 바이트로 저장되기 전에 특정 변환이 적용된다는 의미입니다. 변환이 적용되지 않은 데이터로 구성된 이미지는 _선형 공간_ (_linear space_)에 있다고 하고, 반면에 변환이 적용된 이미지는 _감마 공간_ (_gamma space_)에 있다고 합니다. 여러분이 사용하는 이미지 뷰어는 감마 공간의 이미지를 전제로 할 가능성이 큽니다. 하지만 여기서 렌더링한 이미지는 선형 공간에 있는 이미지이기 때문에 이미지가 실제보다 어둡게 보이는 것입니다.

왜 감마 공간으로 이미지를 저장하는 것이 좋은지에 대한 여러 가지 이유가 있지만, 여기서는 그냥 그런 것이 있다는 정도로만 알고 넘어가겠습니다. 이미지 데이터를 감마 공간으로 변환하여 이미지 뷰어에서 더 정확하게 보이도록 하겠습니다. 단순한 근사인 "gamma 2" 변환을 사용하겠습니다. 이것은 감마 공간에서 선형 공간으로 변환시킬 때 사용하는 거듭제곱 지수에 해당합니다. 여기서는 반대로 선형 공간에서 감마 공간으로 변환해야 하므로 "gamma 2"의 역을 적용해야 합니다. 다시 말해 지수가 $1/\mathit{gamma}$ 가 되고, 결국 제곱근을 취하는 것과 같습니다. 음수 값이 들어오는 경우에도 안전하게 처리되도록 해야 합니다.

```cpp
///////////////////////// 추가 //////////////////////////////////
inline double linear_to_gamma(double linear_component)        //
{                                                             //
  if (linear_component > 0)                                   //
    return std::sqrt(linear_component);                       //
                                                              //
  return 0;                                                   //
}                                                             //
////////////////////////////////////////////////////////////////

void write_color(std::ostream& out, const color& pixel_color) {
  auto r = pixel_color.x();
  auto g = pixel_color.y();
  auto b = pixel_color.z();

///////////////////////// 추가 //////////////////////////////////
  // Apply a linear to gamma transform for gamma 2            //
  r = linear_to_gamma(r);                                     //
  g = linear_to_gamma(g);                                     //
  b = linear_to_gamma(b);                                     //
////////////////////////////////////////////////////////////////

  // Translate the [0, 1] component values to the byte range [0, 255].
  static const interval intensity(0.000, 0.999);
  int rbyte = int(256 * intensity.clamp(r));
  int gbyte = int(256 * intensity.clamp(g));
  int bbyte = int(256 * intensity.clamp(b));

  // Write out the pixel color components.
  out << rbyte << ' ' << gbyte << ' ' << bbyte << '\n';
}
```

**<p align="center">Listing 57:** [color.h] _write\_color(), with gamma correction</p>_

감마 보정을 적용하면, 밝기 변화가 더 일관되게 나타납니다.

<p align="center"><img src="https://raytracing.github.io/images/img-1.12-gamma-gamut.png"></p>

**<p align="center">Image 12:** _The gamut of our renderer, gamma-corrected</p>_

---

## 출처

**Ray Tracing in One Weekend - Peter Shirley**
https://raytracing.github.io/books/RayTracingInOneWeekend.html#diffusematerials
