> **이 글은 Peter Shirley의 [Ray Tracing in One Weekend](https://raytracing.github.io/books/RayTracingInOneWeekend.html)를 번역한 것입니다.
> Ray Tracing in One Weekend를 공부하면서 다시 한번 복습하는 느낌으로 번역을 해보려고 합니다. 영어가 서툴러 번역이 잘못되었을 수도 있으므로 잘못된 부분을 발견하신다면 지적해 주시면 감사하겠습니다.**

## 9.1 An Abstract Class for Materials

---

만약 다른 메테리얼을 가진 또다른 오브젝트를 원한다면, 설계 결정사항을 가지고 있습니다. 많은 매개 변수와 다른 메테리얼 타입을 가진 범용적인 메테리얼을 만들 수 있습니다. 이건 나쁜 접근 방식이 아닙니다. 또는 동작을 캡슐화하는 추상 메테리얼 클래스를 만들 수 있습니다. 저는 후자의 접근 방식을 좋아합니다. 우리의 프로그램의 경우 메테리얼은 두 가지 작업을 필요로 합니다:

1. 산란 광선을 생성한다(또는 입사 광선을 흡수한다).
2. 산란된 경우, 광선이 얼마나 감쇠되어야 하는지 설정한다.

다음이 추상 클래스입니다:

```cpp
#ifndef MATERIAL_H
#define MATERIAL_H

#include "rtweekend.h"

struct hit_record;

class material {
  public:
    virtual bool scatter(
      const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered
    ) const = 0;
};

#endif
```

**<p align="center">Listing 41:** [material.h] _The material class</p>_

---

## 9.2 A Data Structure to Describe Ray-Object Intersections

---
`hit_record`로 인수를 많이 사용하는 것을 피할 수 있습니다. 원하는 정보를 무엇이든지 거기에 넣을 수 있습니다. 인수를 사용할 수도 있습니다; 취항의 문제입니다. Hittables와 materials는 서로를 알아야 합니다. 그러므로 순환 참조 상태를 가지고 있습니다. C++에서는 포인터가 클래스를 가리킨다고 컴파일러에게 알려주기만 하면 됩니다. hittable 클래스 안의 "class material"는 다음과 같습니다:

```cpp
/* ************* 추가 ************ */
#include "rtweekend.h"

class material;
/* ******************************* */

struct hit_record {
  point3 p;
  vec3 normal;
/* ************* 추가 ************ */
  shared_ptr<material> mat_ptr;
/* ******************************* */
  double t;
  bool front_face;

  inline void set_face_normal(const ray& r, const vec3& outward_normal) {
    front_face = dot(r.direction(), outward_normal) < 0;
    normal = front_face ? outward_normal : -outward_normal;
  }
};
```

**<p align="center">Listing 42:** [hittable.h] _Hit record with added material pointer</p>_

여기서 메테리얼을 설정한 것은 광선이 표면과 어떻게 상호작용하는지를 알려줍니다. `hit_record`는 여러 인수를 단지 구조체에 넣은 방식입니다. 그러므로 그룹의 형태로 인수들을 보낼 수 있습니다. 광선이 표면을 교차할 경우(예를 들어, 특정 구), `hit_record`의 메테리얼 포인터는 `main()`를 실행했을 때 설정된 구의 메테리얼 포인터를 가리키도록 설정됩니다. `ray_color()`이 `hit_record`를 가져오면 메테리얼 포인터의 멤버 함수를 호출하여 어떤 광선이 산란되었는지 알 수 있습니다.

이것을 구현하기 위해, sphere 클래스에는 `hit_record`로 리턴될 메테리얼의 참조를 반드시 가지고 있어야 합니다. 아래의 하이라이트 된 줄을 보십시오:

```cpp
class sphere : public hittable {
public:
  sphere() {}
/* ************* 수정 ************ */
  sphere(point3 cen, double r, shared_ptr<material> m)
    : center(cen), radius(r), mat_ptr(m) {};
/* ******************************* */

  virtual bool hit(
    const ray& r, double tmin, double tmax, hit_record& rec) const override;

public:
  point3 center;
  double radius;
/* ************* 추가 ************ */
  shared_ptr<material> mat_ptr;
/* ******************************* */
};

bool sphere::hit(const ray& r, double t_min, double t_max, hit_record& rec) const {
  ...

  rec.t = root;
  rec.p = r.at(rec.t);
  vec3 outward_normal = (rec.p - center) / radius;
  rec.set_face_normal(r, outward_normal);
/* ************* 추가 ************ */
  rec.mat_ptr = mat_ptr;
/* ******************************* */

  return true;
}
```

**<p align="center">Listing 43:** [sphere.h] _Ray-sphere intersection with added material information</p>_

---

## 9.3 Modeling Light Scatter and Reflectance

---

이미 구현한 램버시안(디퓨즈) 케이스의 경우, 항상 산란하고 반사율 𝑅에 의해 감쇠하거나 감쇠없이 산란하지만 광선의 1 - 𝑅을 흡수하거나 이 전략의 혼합이 될 수 있습니다. 램버시안 메테리얼의 경우 간단한 클래스를 만들 수 있습니다:

```cpp
class lambertian : public material {
public:
  lambertian(const color& a) : albedo(a) {}

  virtual bool scatter(
      const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered
  ) const override {
    auto scatter_direction = rec.normal + random_unit_vector();
    scattered = ray(rec.p, scatter_direction);
    attenuation = albedo;
    return true;
  }
public:
  color albedo;
};
```

**<p align="center">Listing 44:** [material.h] _The lambertian material class</p>_

𝑝확률로만 산란할 수 있고 감쇠는 𝑎𝑙𝑏𝑒𝑑𝑜/𝑝가 됩니다. 여러분의 선택입니다.

위의 코드를 자세히 살펴보면, 문제가 발생할 가능성을 발견할 것입니다. 우리가 생성한 랜덤 단위 벡터는 법선 벡터와 정확히 반대 방향이고, 두 벡터의 합은 0이 됩니다. 결국 산란 방향 벡터로 0이 리턴됩니다. 이로 인해 나중에 잘못된 상황(infinities, NaNs)가 발생하므로 조건을 전달하기 전에 차단해야합니다.

이것을 위해, 새로운 벡터 메소드 `vec3::near_zero()`를 만들 것입니다. 이 벡터는 모든 성분이 0에 아주 가까울 경우 true를 리턴합니다.

```cpp
class vec3 {
  ...
  bool near_zero() const {
    // Return true if the vector is close to zero in all dimensions.
    const auto s = 1e-8;
    return (fabs(e[0]) < s) && (fabs(e[1]) < s) && (fabs(e[2]) < s);
  }
  ...
};
```

**<p align="center">Listing 45:** [vec3.h] _The vec3::near_zero() method</p>_

```cpp
class lambertian : public material {
public:
  lambertian(const color& a) : albedo(a) {}

  virtual bool scatter(
      const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered
  ) const override {
    auto scatter_direction = rec.normal + random_unit_vector();

/* ************* 추가 ************ */
    // Catch degenerate scatter direciton
    if (scatter_direction.near_zero())
      scatter_direction = rec.normal;
/* ******************************* */

    scattered = ray(rec.p, scatter_direction);
    attenuation = albedo;
    return true;
  }
public:
  color albedo;
};
```

**<p align="center">Listing 46:** [material.h] _Lambertian scatter, bullet-proof</p>_

---

## 9.4 Mirrored Light Reflection

---

---

## 9.5 A Scene with Metal Spheres

---

---

## 9.6 Fuzzy Reflection

---

---

#### 출처 https://raytracing.github.io/books/RayTracingInOneWeekend.html#metal
