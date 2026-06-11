# 10. Metal

## 10.1 An Abstract Class for Materials
만약 오브젝트마다 서로 다른 머티리얼을 적용하려면 설계적으로 선택해야 할 사항이 있습니다. 한 가지 방법은 수많은 파라미터를 가진 범용 머티리얼 타입을 만드는 것입니다. 그리고 각 머티리얼 타입에서 자기와 무관한 파라미터들은 그냥 무시하면 됩니다. 이건 나쁜 접근 방식은 아닙니다. 또 다른 방법은 각 머티리얼의 고유한 동작을 캡슐화하는 추상 머티리얼 클래스를 만드는 것입니다. 저는 후자의 방식을 선호합니다. 우리 프로그램에서 머티리얼은 다음의 두 가지 일을 해야 합니다.

1. 광선이 산란되는지, 아니면 입사 광선을 흡수할지 정합니다.
2. 산란된 경우, 광선이 얼마나 감쇠되어야 하는지 정합니다.

이를 바탕으로 다음과 같은 추상 클래스를 정의할 수 있습니다.

```cpp
#ifndef MATERIAL_H
#define MATERIAL_H

#include "hittable.h"

class material {
  public:
    virtual ~material() = default;

    virtual bool scatter(
      const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered
    ) const {
      return false;
    }
};

#endif
```

**<p align="center">Listing 58:** [material.h] _The material class</p>_

---

## 10.2 A Data Structure to Describe Ray-Object Intersections
`hit_record` 는 인자가 많아지는 것을 줄이기 위한 목적의 클래스입니다. 그러므로 필요한 정보는 무엇이든 `hit_record` 클래스 안에 넣어 둘 수 있습니다. 캡슐화된 타입(hit_record) 대신에 인자들을 사용할 수도 있습니다만, 이것은 단지 취향의 문제입니다. hittable.h 과 material.h 은 코드에서 서로의 클래스를 참조해야 하므로 순환 참조가 발생합니다. C++에서는 `class material;` 코드를 추가하여 컴파일러에게 `material` 가 뒤에서 정의될 클래스라는 것을 알려줍니다. `material` 클래스를 가리키는 포인터를 지정해주기만 하면 순환 참조 문제를 해결할 수 있습니다. 컴파일러는 클래스의 세부 내용을 알 필요가 없습니다.

```cpp
///////////////////////// 추가 //////////////////////////
class material;                                       //
////////////////////////////////////////////////////////

class hit_record {
  public:
    point3 p;
    vec3 normal;
///////////////////////// 추가 //////////////////////////
    shared_ptr<material> mat;                         //
////////////////////////////////////////////////////////
    double t;
    bool front_face;

    void set_face_normal(const ray& r, const vec3& outward_normal) {
      front_face = dot(r.direction(), outward_normal) < 0;
      normal = front_face ? outward_normal : -outward_normal;
    }
};
```

**<p align="center">Listing 59:** [hittable.h] _Hit record with added material pointer</p>_

`hit_record` 는 많은 수의 인자들을 클래스 안에 담아서 그룹으로 묶어 보낼 수 있도록하는 한 가지 방법일 뿐입니다. 광선이 구의 표면에 충돌할 때, `hit_record` 안의 머티리얼 포인터는 그 구의 머티리얼 포인터를 가리키도록 설정됩니다. 구의 머티리얼은 프로그램 시작 시 `main()` 에서 설정됩니다. `ray_color()` 함수 안에서 `hit_record` 를 얻으면 `ray_color()` 함수는 머티리얼 포인터의 멤버 함수를 호출하여 어떤 광선이 산란되는지 알아낼 수 있습니다.

이를 위해, `hit_record` 는 구에 할당되어 있는 머티리얼 정보를 전달받아야 합니다.

```cpp
class sphere : public hittable {
  public:
///////////////////////// 수정 ////////////////////////////////////////////////////////////////////////
    sphere(const point3& center, double radius) : center(center), radius(std::fmax(0, radius)) {    //
      // TODO: Initialize the material pointer `mat`.                                               //
    }                                                                                               //
//////////////////////////////////////////////////////////////////////////////////////////////////////

    bool hit(const ray& r, interval ray_t, hit_record& rec) const override {
      ...

      rec.t = root;
      rec.p = r.at(rec.t);
      vec3 outward_normal = (rec.p - center) / radius;
      rec.set_face_normal(r, outward_normal);
///////////////////////// 추가 //////////////////////////
      rec.mat = mat;                                  //
////////////////////////////////////////////////////////

      return true;
    }

  private:
    point3 center;
    double radius;
///////////////////////// 추가 //////////////////////////
    shared_ptr<material> mat;                         //
////////////////////////////////////////////////////////
};
```

**<p align="center">Listing 60:** [sphere.h] _Ray-sphere intersection with added material information</p>_

---

## 10.3 Modeling Light Scatter and Reflectance
이 책에서는 앞으로 계속 _albedo_ (흰색의 정도(whiteness)를 의미하는 라틴어)라는 용어를 사용할 것입니다. 몇몇 분야에서는 albedo가 엄밀한 기술 용어이지만, 어떤 경우든 공통적으로는 _반사율을 비율로 나타내는 개념_ (_fractional reflectance_)을 정의하기 위해서 사용됩니다. albedo는 머티리얼 색상에 따라 달라집니다. 또한 유리 머티리얼에서 나중에 구현하겠지만 들어오는 광선의 방향에 따라서도 달라질 수 있습니다.

램버시안(디퓨즈) 반사율은 항상 산란시키되 반사율 $R$ 에 따라 빛이 감쇠되는 방식으로 구현할 수 있고, 또는 감쇠 없이 $1-R$ 확률로 특정 경우에만 산란시키는 방식으로 구현할 수도 있습니다. 이 경우 산란되지 않은 광선은 그냥 머티리얼 안으로 흡수됩니다. 또한 이 두 가지 방식을 모두 섞은 방식일 수도 있습니다. 여기서는 항상 산란시키는 방식을 사용하겠습니다. 그러면 램버시안 머티리얼 구현이 단순해집니다.

```cpp
class material {
  ...
};

///////////////////////// 추가 ////////////////////////////////////////////////////////////////////
class lambertian : public material {                                                            //
  public:                                                                                       //
    lambertian(const color& albedo) : albedo(albedo) {}                                         //
                                                                                                //
    bool scatter(const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered)    //
    const override {                                                                            //
      auto scatter_direction = rec.normal + random_unit_vector();                               //
      scattered = ray(rec.p, scatter_direction);                                                //
      attenuation = albedo;                                                                     //
      return true;                                                                              //
    }                                                                                           //
                                                                                                //
  private:                                                                                      //
    color albedo;                                                                               //
};                                                                                              //
//////////////////////////////////////////////////////////////////////////////////////////////////
```

**<p align="center">Listing 61:** [material.h] _The new lambertian material class</p>_

세 번째 방법도 있습니다. 광선을 고정된 확률 $p$ 로만 산란되게 하고, 산란이 일어났을 때의 감쇠를 $\mathit{albedo}/p$ 로 계산하는 방식입니다. 어떤 방식을 선택할지는 여러분의 선택입니다.

위의 코드를 자세히 살펴보면, 작은 문제점을 발견할 수 있습니다. 만약 생성한 랜덤 단위 벡터가 법선 벡터와 정확하게 반대 방향이라면 이 두 벡터의 합은 0이 됩니다. 그러면 결국 산란 방향 벡터는 0 벡터가 됩니다. 나중에 값이 무한대가 되거나 NaN가 되는 상황이 발생할 수 있으므로 조건을 추가하여 이런 상황을 방지해야 합니다.

이를 위해, 벡터의 모든 성분이 0에 아주 가까워지면 true를 리턴하는 새로운 벡터 메서드 `vec3::near_zero()` 를 정의합니다.

아래의 코드에서 C++ 표준 라이브러리 함수인 `std::fabs` 를 사용할 것입니다. 이 함수는 입력 값의 절댓값을 리턴합니다.

```cpp
class vec3 {
  ...

  double length_squared() const {
    return e[0] * e[0] + e[1] * e[1] + e[2] * e[2];
  }

///////////////////////// 추가 //////////////////////////////////////////////////////////
  bool near_zero() const {                                                            //
    // Return true if the vector is close to zero in all dimensions.                  //
    auto s = 1e-8;                                                                    //
    return (std::fabs(e[0]) < s) && (std::fabs(e[1]) < s) && (std::fabs(e[2]) < s);   //
  }                                                                                   //
////////////////////////////////////////////////////////////////////////////////////////

  ...
};
```

**<p align="center">Listing 62:** [vec3.h] _The vec3::near\_zero() method</p>_

```cpp
class lambertian : public material {
  public:
    lambertian(const color& albedo) : albedo(albedo) {}

    bool scatter(const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered)
    const override {
      auto scatter_direction = rec.normal + random_unit_vector();

///////////////////////// 추가 //////////////////////////////
      // Catch degenerate scatter direction               //
      if (scatter_direction.near_zero())                  //
        scatter_direction = rec.normal;                   //
////////////////////////////////////////////////////////////

      scattered = ray(rec.p, scatter_direction);
      attenuation = albedo;
      return true;
    }

  private:
    color albedo;
};
```

**<p align="center">Listing 63:** [material.h] _Lambertian scatter, bullet-proof</p>_

---

## 10.4 Mirrored Light Reflection
광선이 광택이 있는 금속에 반사될 경우에는 랜덤하게 산란되지 않습니다. 핵심 질문은 "금속 거울에서 광선이 어떻게 반사되는가?" 입니다. 이 문제를 푸는 데에는 벡터 수학이 유용합니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.15-reflection.jpg"></p>

**<p align="center">Figure 15:** _Ray reflection</p>_

빨간색으로 표시된 반사 광선의 방향 벡터는 $\mathbf{v} + 2\mathbf{b}$ 로 간단하게 구할 수 있습니다. 여기의 구현에서는 $\mathbf{n}$ 은 길이가 1인 단위 벡터이지만 $\mathbf{v}$ 은 그렇지 않을 수 있습니다. 벡터 $\mathbf{b}$ 를 구하기 위해서, $\mathbf{v}$ 를 $\mathbf{n}$ 로 투영한 다음, 그 투영한 길이로 법선 벡터를 스케일합니다. 투영 값은 내적 $\mathbf{v} \cdot \mathbf{n}$ 로 계산합니다. 만약 $\mathbf{n}$ 이 단위 벡터가 아니라면 내적 연산 결과를 $\mathbf{n}$ 의 길이로 나누는 과정도 필요합니다. 마지막으로 $\mathbf{v}$ 는 표면 안쪽을 향하고 있고 $\mathbf{b}$ 는 표면 바깥쪽을 향하도록 해야 하기 때문에 투영 길이의 부호를 반대로 해야 합니다.

지금까지의 내용을 모두 종합하면, 반사 벡터는 다음과 같이 계산할 수 있습니다.

```cpp
...

inline vec3 random_on_hemisphere(const vec3& normal) {
  ...
}

///////////////////////// 추가 //////////////////////////////
inline vec3 reflect(const vec3& v, const vec3& n) {       //
  return v - 2 * dot(v, n) * n;                           //
}                                                         //
////////////////////////////////////////////////////////////
```

**<p align="center">Listing 64:** [vec3.h] _vec3 reflection function</p>_

금속 머티리얼은 이 공식을 사용하여 광선을 반사합니다.

```cpp
...

class lambertian : public material {
  ...
};

///////////////////////// 추가 //////////////////////////////////////////////////////////////////
class metal : public material {                                                               //
  public:                                                                                     //
    metal(const color& albedo) : albedo(albedo) {}                                            //
                                                                                              //
    bool scatter(const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered)  //
    const override {                                                                          //
      vec3 reflected = reflect(r_in.direction(), rec.normal);                                 //
      scattered = ray(rec.p, reflected);                                                      //
      attenuation = albedo;                                                                   //
      return true;                                                                            //
    }                                                                                         //
                                                                                              //
  private:                                                                                    //
    color albedo;                                                                             //
};                                                                                            //
////////////////////////////////////////////////////////////////////////////////////////////////
```

**<p align="center">Listing 65:** [material.h] _Metal material with reflectance function</p>_

지금까지의 모든 변경 사항에 맞게 `ray_color()` 함수를 수정해야 합니다.

```cpp
#include "hittable.h"
///////////////////////// 추가 //////////////////////
#include "material.h"                             //
////////////////////////////////////////////////////
...

class camera {
  ...
  private:
    ...
    color ray_color(const ray& r, int depth, const hittable& world) const {
      // If we've exceeded the ray bounce limit, no more light is gathered.
      if (depth <= 0)
        return color(0, 0, 0);

      hit_record rec;

      if (world.hit(r, interval(0.001, infinity), rec)) {
///////////////////////// 수정 ////////////////////////////////////////////////
        ray scattered;                                                      //
        color attenuation;                                                  //
        if (rec.mat->scatter(r, rec, attenuation, scattered))               //
          return attenuation * ray_color(scattered, depth - 1, world);      //
        return color(0, 0, 0);                                              //
//////////////////////////////////////////////////////////////////////////////
      }

      vec3 unit_direction = unit_vector(r.direction());
      auto a = 0.5 * (unit_direction.y() + 1.0);
      return (1.0 - a) * color(1.0, 1.0, 1.0) + a * color(0.5, 0.7, 1.0);
    }
};
```

**<p align="center">Listing 66:** [camera.h] _Ray color with scattered reflectance</p>_

이제 머티리얼 포인터 `mat` 를 초기화 하도록 `sphere` 클래스 생성자를 수정합니다.

```cpp
class sphere : public hittable {
  public:
///////////////////////// 수정 ////////////////////////////////////////////////
    sphere(const point3& center, double radius, shared_ptr<material> mat)   //
      : center(center), radius(std::fmax(0, radius)), mat(mat) {}           //
//////////////////////////////////////////////////////////////////////////////

    ...
};
```

**<p align="center">Listing 67:** [sphere.h] _Initializing sphere with a material</p>_

---

## 10.5 A Scene with Metal Spheres
이제 씬에 금속 구들을 추가해보겠습니다.

```cpp
#include "rtweekend.h"

#include "camera.h"
#include "hittable.h"
#include "hittable_list.h"
///////////////////////// 추가 //////////////////////
#include "material.h"                             //
////////////////////////////////////////////////////
#include "sphere.h"

int main() {
  hittable_list world;

///////////////////////// 추가 ////////////////////////////////////////////////////////////
  auto material_ground = make_shared<lambertian>(color(0.8, 0.8, 0.0));                 //
  auto material_center = make_shared<lambertian>(color(0.1, 0.2, 0.5));                 //
  auto material_left   = make_shared<metal>(color(0.8, 0.8, 0.8));                      //
  auto material_right  = make_shared<metal>(color(0.8, 0.6, 0.2));                      //
                                                                                        //
  world.add(make_shared<sphere>(point3( 0.0, -100.5, -1.0), 100.0, material_ground));   //
  world.add(make_shared<sphere>(point3( 0.0,    0.0, -1.2),   0.5, material_center));   //
  world.add(make_shared<sphere>(point3(-1.0,    0.0, -1.0),   0.5, material_left));     //
  world.add(make_shared<sphere>(point3( 1.0,    0.0, -1.0),   0.5, material_right));    //
//////////////////////////////////////////////////////////////////////////////////////////

  camera cam;

  cam.aspect_ratio      = 16.0 / 9.0;
  cam.image_width       = 400;
  cam.samples_per_pixel = 100;
  cam.max_depth         = 50;

  cam.render(world);
}
```

**<p align="center">Listing 68:** [main<span></span>.cc] _Scene with metal spheres</p>_

렌더링 결과는 다음과 같습니다.

<p align="center"><img src="https://raytracing.github.io/images/img-1.13-metal-shiny.png"></p>

**<p align="center">Image 13:** _Shiny metal</p>_

---

## 10.6 Fuzzy Reflection
반사 광선의 끝점에 작은 구를 두고 그 구 위의 점 하나를 광선의 새로운 끝점으로 선택함으로써, 반사 방향을 랜덤하게 만들 수도 있습니다. 광선의 원래 끝점을 중심으로 하는 구의 표면에서 랜덤한 점을 선택하고, 반사 퍼짐(fuzz)의 세기는 fuzz factor로 조절합니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.16-reflect-fuzzy.jpg"></p>

**<p align="center">Figure 16:** _Generating fuzzed reflection rays</p>_

fuzz 구가 더 클수록 반사는 더 퍼지게 됩니다. 따라서 fuzz 구의 반지름 자체를 fuzziness 파라미터로 사용하면 됩니다. fuzziness 파라미터가 0이면 반사 방향의 퍼짐이 전혀 없게 됩니다. 문제는 구가 크거나 광선이 표면에 거의 평행하게 들어오는 경우, 산란 방향이 표면 아래로 향할 수 있다는 점입니다. 그러나 그런 광선들은 그냥 표면에 흡수된 것으로 처리하면 됩니다.

또한 주의할 점은 fuzz 구가 제대로 동작하려면 구의 크기가 반사 벡터에 대해 일관되게 해석되어야 합니다. 그러나 반사 벡터의 길이는 얼마든지 달라질 수가 있습니다. 따라서 이를 해결하기 위해서는 먼저 반사 광선을 정규화해야 합니다.

```cpp
class metal : public material {
  public:
///////////////////////// 수정 //////////////////////////////////////////////////////////////////
    metal(const color& albedo, double fuzz) : albedo(albedo), fuzz(fuzz < 1 ? fuzz : 1) {}    //
////////////////////////////////////////////////////////////////////////////////////////////////

    bool scatter(const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered)
    const override {
      vec3 reflected = reflect(r_in.direction(), rec.normal);
///////////////////////// 추가 ////////////////////////////////////////////////
      reflected = unit_vector(reflected) + (fuzz * random_unit_vector());   //
//////////////////////////////////////////////////////////////////////////////
      scattered = ray(rec.p, reflected);
      attenuation = albedo;
///////////////////////// 수정 //////////////////////////////////
      return (dot(scattered.direction(), rec.normal) > 0);    //
////////////////////////////////////////////////////////////////
    }

  private:
    color albedo;
///////////////////////// 추가 //////////////////////////////////
    double fuzz;                                              //
////////////////////////////////////////////////////////////////
};
```

**<p align="center">Listing 69:** [material.h] _Metal material fuzziness</p>_

금속 머티리얼에 fuzziness를 0.3과 1.0으로 적용하여 어떻게 보이는지 테스트할 수 있습니다.

```cpp
int main() {
  ...
  auto material_ground = make_shared<lambertian>(color(0.8, 0.8, 0.0));
  auto material_center = make_shared<lambertian>(color(0.1, 0.2, 0.5));
///////////////////////// 수정 //////////////////////////////////////////////
  auto material_left   = make_shared<metal>(color(0.8, 0.8, 0.8), 0.3);   //
  auto material_right  = make_shared<metal>(color(0.8, 0.6, 0.2), 1.0);   //
////////////////////////////////////////////////////////////////////////////
  ...
}
```

**<p align="center">Listing 70:** [main<span></span>.cc] _Metal spheres with fuzziness</p>_

<p align="center"><img src="https://raytracing.github.io/images/img-1.14-metal-fuzz.png"></p>

**<p align="center">Image 14:** _Fuzzed metal</p>_

---

## 출처

**Ray Tracing in One Weekend - Peter Shirley**
https://raytracing.github.io/books/RayTracingInOneWeekend.html#metal
