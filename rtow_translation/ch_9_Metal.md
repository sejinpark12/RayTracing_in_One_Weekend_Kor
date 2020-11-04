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

---

## 9.3 Modeling Light Scatter and Reflectance

---

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
