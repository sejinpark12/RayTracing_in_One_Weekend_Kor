# 6. Surface Normals and Multiple Objects

## 6.1 Shading with Surface Normals

먼저 셰이딩을 위해 표면 법선(surface normal) 벡터를 계산합니다. 이 벡터는 광선과 표면의 교차점에서 표면에 수직인 벡터입니다.

코드에서 법선 벡터를 어떤 형태로 표현할지 결정해야 합니다. 법선 벡터의 길이가 임의의 길이인지, 아니면 정규화된 단위 길이인지를 결정해야 합니다.

벡터 정규화가 꼭 필요하지 않을 수도 있으니, 벡터 정규화에 사용되는 연산 비용이 큰 제곱근 연산을 생략하고 싶어질 수가 있습니다. 그러나 여기에는 세 가지 중요한 포인트가 있습니다. 첫째, 단위 길이 법선 벡터가 _언젠가_ 필요하다면, 필요한 곳마다 매번 정규화하는 것보다 차라리 처음 한 번만 정규화하는 편이 낫습니다. 둘째, 여러 곳에서 단위 길이 법선 벡터가 _정말로_ 필요합니다. 셋째, 특정 지오메트리 클래스의 특성을 이용하면 생성자나 `hit()` 함수 안에서 단위 길이 법선 벡터를 효율적으로 만들어낼 수 있는 경우가 종종 있습니다. 예를 들어, 구의 법선 벡터는 구의 반지름으로 간단히 나누기만 하면 단위 길이가 되므로, 제곱근 연산을 전혀 사용하지 않을 수 있습니다.

이 모든 것을 바탕으로, 이제부터 모든 법선 벡터의 길이는 단위 길이가 되도록 하겠습니다.

구의 경우, 구의 바깥 방향 법선 벡터는 (교차점 - 중심점) 방향입니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.06-sphere-normal.jpg"></p>

**<p align="center">Figure 6**: _Sphere surface-normal geometry</p>_

지구를 예로 들면, 지구 중심에서 여러분까지의 벡터는 여러분 위치에서 정확히 위쪽 방향을 가리킨다는 것을 의미합니다. 이제 코드로 작성하여 셰이딩을 해보겠습니다. 아직 광원이나 다른 어떤 것도 존재하지 않으므로 컬러 매핑을 활용하여 법선 벡터를 시각화하겠습니다. $\mathbf{n}$ (법선 벡터)이 단위 길이 벡터라고 가정하면 각 성분은 -1 ~ 1사이의 값이므로, 법선 벡터의 각 성분을 0에서 1의 범위로 매핑한 다음, 매핑된 법선 벡터의 성분 $(x, y, z)$ 를 $(\mathit{red}, \mathit{green}, \mathit{blue})$ 로 다시 매핑하는 것입니다. 이게 법선 벡터를 시각화하는 일반적인 트릭입니다. 이 방법은 쉽고 직관적입니다. 법선 벡터를 구하기 위해서는 단지 교차 여부가 아닌 교차점이 필요합니다(현재는 교차 여부만 계산하고 있습니다). 현재 씬에는 카메라 앞에 구 하나만 존재하므로 음수 $t$ 값은 아직 생각하지 않겠습니다. 따라서 가장 가까운 교차점(가장 작은 $t$ )이 찾고자 하는 점이라고 가정하겠습니다. 아래의 코드로 $n$ 을 계산하고 시각화할 수 있습니다.

```cpp
///////////////////////// 수정 /////////////////////////////////////////////
double hit_sphere(const point3& center, double radius, const ray& r) {   //
///////////////////////////////////////////////////////////////////////////
  vec3 oc = center - r.origin();
  auto a = dot(r.direction(), r.direction());
  auto b = -2.0 * dot(r.direction(), oc);
  auto c = dot(oc, oc) - radius * radius;
  auto discriminant = b * b - 4 * a * c;

///////////////////////// 수정 ///////////////////////////////
  if (discriminant < 0) {                                  //
    return -1.0;                                           //
  } else {                                                 //
    return (-b - std::sqrt(discriminant)) / (2.0 * a);     //
  }                                                        //
/////////////////////////////////////////////////////////////
}

color ray_color(const ray& r) {
///////////////////////// 수정 //////////////////////////////////
  auto t = hit_sphere(point3(0, 0, -1), 0.5, r);              //
  if (t > 0.0) {                                              //
    // 법선 벡터 N을 단위 길이 벡터로 생성                            //
    vec3 N = unit_vector(r.at(t) - vec3(0, 0, -1));           //
    // 법선 벡터 N의 각 구성요소들을 -1 ~ 1 범위에서 0 ~ 1 범위로 매핑    //
    return 0.5 * color(N.x() + 1, N.y() + 1, N.z() + 1);      //
  }                                                           //
////////////////////////////////////////////////////////////////

  vec3 unit_direction = unit_vector(r.direction());
  auto a = 0.5 * (unit_direction.y() + 1.0);
  return (1.0 - a) * color(1.0, 1.0, 1.0) + a * color(0.5, 0.7, 1.0);
}
```

**<p align="center">Listing 12:** [<span>main</span>.cc] _Rendering surface normals on a sphere</p>_

다음과 같은 결과를 얻을 수 있습니다.

<p align="center"><img src="https://raytracing.github.io/images/img-1.04-normals-sphere.png"></p>

**<p align="center">Image 4:** _A sphere colored according to its normal vectors</p>_

---

## 6.2 Simplifying the Ray-Sphere Intersection Code

광선-구 함수를 다시 한 번 봅시다.

```cpp
double hit_sphere(const point3& center, double radius, const ray& r) {
  vec3 oc = center - r.origin();
  auto a = dot(r.direction(), r.direction());
  auto b = -2.0 * dot(r.direction(), oc);
  auto c = dot(oc, oc) - radius * radius;
  auto discriminant = b * b - 4 * a * c;

  if (discriminant < 0) {
    return -1.0;
  } else {
    return (-b - std::sqrt(discriminant)) / (2.0 * a);
  }
}
```

**<p align="center">Listing 13:** [<span>main</span>.cc] _Ray-sphere intersection code(before)</p>_

첫째, 동일한 두 벡터끼리의 내적은 해당 벡터 길이의 제곱과 같다는 것을 알고 있습니다.

둘째, 5장에서 만든 구의 방정식에는 `b` 에 -2가 포함되어 있다는 점에 주목합니다. 만약 $b = -2h$ 라면 이차방정식이 어떻게 변하는지 생각해봅시다.

$$ \frac{-b \pm \sqrt{b^2 - 4ac}}{2a} $$

$$ = \frac{-(-2h) \pm \sqrt{(-2h)^2 - 4ac}}{2a} $$

$$ = \frac{2h \pm 2\sqrt{h^2 - ac}}{2a} $$

$$ = \frac{h \pm \sqrt{h^2 - ac}}{a} $$

위와같이 이차방정식이 깔끔하게 정리되므로 이렇게 사용하겠습니다. 이제 $h$ 에 대하여 풀면

$$ b = -2 \mathbf{d} \cdot (\mathbf{C} - \mathbf{Q}) $$
$$ b = -2h $$
$$ h = \frac{b}{-2} = \mathbf{d} \cdot (\mathbf{C} - \mathbf{Q}) $$

이 성질을 이용하여 구-교차 코드를 다음과 같이 간단하게 작성할 수 있습니다.

```cpp
double hit_sphere(const point3& center, double radius, const ray& r) {
  vec3 oc = center - r.origin();
///////////////////////// 수정 ///////////////////////////////
  auto a = r.direction().length_squared();                 //
  auto h = dot(r.direction(), oc);                         //
  auto c = oc.length_squared() - radius * radius;          //
  auto discriminant = h * h - a * c;                       //
/////////////////////////////////////////////////////////////

  if (discriminant < 0) {
    return -1.0;
  } else {
///////////////////////// 수정 ///////////////////////////////
    return (h - std::sqrt(discriminant)) / a;              //
/////////////////////////////////////////////////////////////
  }
}
```

**<p align="center">Listing 14:** [<span>main</span>.cc] _Ray-sphere intersection code(after)</p>_

---

## 6.3 An Abstraction for Hittable Objects

이제, 구가 여러 개일 경우는 어떻게 해야 할까요? 구의 배열을 사용하고 싶은 생각이 들 수 있지만, 더 깔끔한 방법은 광선이 교차할 수 있는 모든 것에 대한 "추상 클래스(abstract class)"를 만드는 것입니다. 그리고 구와 구의 리스트 둘 다 광선이 교차할 수 있는 것으로 취급합니다. 이 클래스의 이름을 정하는 것은 꽤 어려운 문제입니다. "object"라는 이름도 괜찮을 수 있지만, "object oriented" 프로그래밍이라는 용어와 헷갈릴 수 있습니다.
 "Surface"라는 이름도 자주 사용되지만, 나중에 volume(안개, 구름 같은 것들)을 다루게 될 경우에는 적절한 클래스 이름이 아닙니다. "hittable"이라는 이름은 구나 구의 리스트처럼, 광선이 교차할 수 있는 모든 것들이 공통적으로 가지는 `hit` 멤버 함수를 가장 잘 드러내는 이름입니다. 이 클래스 이름들 중 어떤 것도 마음에 들지 않지만, 여기서는 "hittable"을 사용하겠습니다.

`hittable` 추상 클래스의 `hit` 멤버 함수는 매개변수로 광선을 받습니다. 대부분의 레이 트레이서에서 교차 유효 범위($t_{\mathit{min}}$ 에서 $t_{\mathit{max}}$ 까지의)를 설정하는 것이 편리하다는 사실이 알려져 있습니다. 따라서 $t_{\mathit{min}} < t < t_{\mathit{max}}$ 범위에서만 교차를 "계산"합니다. 

처음 광선을 쏠 때는 양수 $t$ 만 고려하면 됩니다. 하지만 뒤에서 보겠지만 $t_{\mathit{min}}$ 에서 $t_{\mathit{max}}$ 까지로 $t$ 의 범위를 설정하면 코드를 단순화할 수 있습니다. 한 가지 설계적 고려사항은 광선이 물체와 교차할 때, 법선 벡터 값을 함께 계산할지 여부입니다. 교차 대상을 탐색하는 중에는 더 가까운 교차 대상을 더 나중에 발견할 수 있습니다. 그리고 최종적으로 필요한 것은 가장 가까운 교차 대상에 대한 법선 벡터뿐입니다. 여기서는 필요한 여러 데이터를 구조체 안에 저장하는 단순한 방법을 사용하겠습니다. 아래의 코드가 추상 클래스 코드입니다.

```cpp
#ifndef HITTABLE_H
#define HITTABLE_H

#include "ray.h"

class hit_record {
  public:
    point3 p;
    vec3 normal;
    double t;
};

class hittable {
  public:
    virtual ~hittable() = default;

    virtual bool hit(const ray& r, double ray_tmin, double ray_tmax, hit_record& rec) const = 0;
};

#endif
```

**<p align="center">Listing 15:** [hittable.h] _The hittable class</p>_

구에 대한 코드입니다.

```cpp
#ifndef SPHERE_H
#define SPHERE_H

#include "hittable.h"
#include "vec3.h"

class sphere : public hittable {
  public:
    sphere(const point3& center, double radius) : center(center), radius(std::fmax(0, radius)) {}

    bool hit(const ray& r, double ray_tmin, double ray_tmax, hit_record& rec) const override {
      vec3 oc = center - r.origin();
      auto a = r.direction().length_squared();
      auto h = dot(r.direction(), oc);
      auto c = oc.length_squared() - radius * radius;

      auto discriminant = h * h - a * c;
      if (discriminant < 0)
        return false;

      auto sqrtd = std::sqrt(discriminant);

      // Find the nearest root that lies in the acceptable range.
      auto root = (h - sqrtd) / a;
      if (root <= ray_tmin || ray_tmax <= root) {
        root = (h + sqrtd) / a;
        if (root <= ray_tmin || ray_tmax <= root)
          return false;
      }

      rec.t = root;
      rec.p = r.at(rec.t);
      rec.normal = (rec.p - center) / radius;

      return true;
    }

  private:
    point3 center;
    double radius;
};

#endif
```

**<p align="center">Listing 16:** [sphere.h] _The sphere class</p>_

(여기서 C++ 표준 라이브러리 함수인 `std::fmax()` 를 사용하는 것에 주목하세요. 이 함수는 두 부동소수점 인자 중 큰 값을 리턴합니다. 비슷하게, 뒤에서 두 부동소수점 인자 중 작은 값을 리턴하는 `std::fmin()` 함수도 사용할 것입니다.)

---

## 6.4 Front Faces Versus Back Faces

법선 벡터에 대한 두 번째 설계 결정사항은 법선 벡터가 항상 구의 바깥 방향을 향하도록 할지 여부입니다. 현재까지 구현한 법선 벡터는 항상 구의 중심에서 교차점으로 향하므로, 바깥 방향을 향하는 법선 벡터입니다. 광선이 구의 바깥에서 안으로 들어오면서 교차하는 경우, 법선 벡터(바깥 방향)는 광선의 방향과 반대입니다. 광선이 구의 안에서 밖으로 나가면서 교차하는 경우, 법선 벡터는 항상 바깥쪽을 향하므로 광선의 방향과 같은 방향입니다. 아니면 다른 방법으로, 법선 벡터가 항상 광선의 반대 방향을 가리키도록 만들 수도 있습니다. 광선이 구의 바깥에서 안으로 들어오면서 교차한다면 법선 벡터는 바깥 방향을 가리키게 됩니다. 반대로, 광선이 구 안에서 밖으로 나가면서 교차한다면 법선 벡터는 안쪽 방향을 가리키게 됩니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.07-normal-sides.jpg"></p>

**<p align="center">Figure 7:** _Possible directions for sphere surface-normal geometry</p>_

이 두 방법 중 하나를 선택해야 합니다. 왜냐하면 광선이 구의 안에서 밖으로 나가면서 교차하는지 구의 바깥에서 안으로 들어오면서 교차하는지를 판단해야 하기 때문입니다. 이것은 양면 종이의 글자처럼 양쪽 면이 다르게 렌더링되는 물체나, 유리 공처럼 안과 밖이 모두 존재하는 물체를 다룰 때 중요합니다.

만약 법선 벡터가 항상 바깥쪽을 향하도록 한다면, 색을 계산할 때 광선이 표면의 어느 쪽에 있는지를 판별해야 합니다. 이것은 광선과 법선 벡터를 비교하여 구할 수 있습니다. 광선과 법선 벡터가 같은 방향이라면 광선은 물체 안쪽에 있습니다. 그리고 광선과 법선 벡터가 반대 방향이라면 광선은 물체 바깥쪽에 있습니다. 이것은 두 벡터의 내적(dot product)을 통해 구할 수 있습니다. 두 벡터의 내적이 양수라면 광선은 구의 안쪽에 있습니다.

```cpp
if (dot(ray_direction, outward_normal) > 0.0) {
  // ray is inside the sphere
  ...
} else {
  // ray is outside the sphere
  ...
}
```

**<p align="center">Listing 17:** _Comparing the ray and the normal</p>_

반대로 법선 벡터가 항상 광선의 반대 방향을 가리키도록 한다면, 광선이 표면의 안쪽에 있는지 바깥쪽에 있는지를 판별하는 데 내적을 사용할 수 없게 됩니다. 대신, 아래의 추가 정보를 저장해야합니다.

```cpp
bool front_face;
if (dot(ray_direction, outward_normal) > 0.0) {
  // ray is inside the sphere
  normal = -outward_normal;
  front_face = false;
} else {
  // ray is outside the sphere
  normal = outward_normal;
  front_face = true;
}
```

**<p align="center">Listing 18:** _Remembering the side of the surface</p>_

즉, 법선 벡터가 항상 표면의 바깥쪽을 향하도록 설정할 수도 있고, 항상 입사 광선(incident ray)과 반대 방향을 향하도록 설정할 수도 있습니다. 이 결정은 표면의 안팎이 어느 쪽인지 판별하는 시점을 지오메트리 교차 시점으로 할지, 아니면 색을 계산하는 시점으로 할지에 따라 결정됩니다. 이 책에서는 지오메트리 타입보다 머티리얼 타입이 더 많습니다. 그 많은 머티리얼 코드에서 표면 판별을 반복하는 것보다 지오메트리 코드에서 표면을 처리하는 편이 작업량이 더 적으므로 지오메트리 작업 시점에서 판별을 하겠습니다. 이것은 단순히 선호도의 문제이고, 관련 자료들을 보면 두 방식이 모두 사용되는 것을 확인할 수 있습니다.

`hit_record` 구조체에 bool형 `front_face` 변수를 추가합니다. 또한 표면의 안팎과 법선 벡터를 계산하는 `set_face_normal()` 함수도 추가합니다. 편의성을 위해, `set_face_normal()` 함수에 매개변수로 전달되는 벡터는 단위 길이(unit length) 라고 가정하겠습니다. 이 함수에서 매번 매개변수를 직접 정규화할 수도 있지만, 이 작업은 지오메트리 코드에서 처리하는 편이 더 효율적입니다. 보통은 특정 지오메트리 정보를 더 잘 알고 있는 쪽에서 더 쉽게 처리할 수 있기 때문입니다.

```cpp
class hit_record {
  public:
    point3 p;
    vec3 normal;
    double t;
///////////////////////// 추가 ///////////////////////////////////////////////////
    bool front_face;                                                           //
                                                                               //
    void set_face_normal(const ray& r, const vec3& outward_normal) {           //
      // Sets the hit record normal vector.                                    //
      // NOTE: the parameter `outward_normal` is assumed to have unit length.  //
                                                                               //
      front_face = dot(r.direction(), outward_normal) < 0;                     //
      normal = front_face ? outward_normal : -outward_normal;                  //
    }                                                                          //
/////////////////////////////////////////////////////////////////////////////////
};
```

**<p align="center">Listing 19:** [hittable.h] _Adding front-face tracking to hit_record</p>_

그 다음, 표면의 안팎을 판별하는 로직을 클래스에 추가합니다.

```cpp
class sphere : public hittable {
  public:
    ...
    bool hit(const ray& r, double ray_tmin, double ray_tmax, hit_record& rec) const {
      ...

      rec.t = root;
      rec.p = r.at(rec.t);
///////////////////////// 수정 ///////////////////////////////
      vec3 outward_normal = (rec.p - center) / radius;     //
      rec.set_face_normal(r, outward_normal);              //
/////////////////////////////////////////////////////////////

      return true;
    }
    ...
};
```

**<p align="center">Listing 20:** [sphere.h] _The sphere class with normal determination</p>_

---

## 6.5 A List of Hittable Objects

우리는 광선이 교차할 수 있는 일반적인 객체인 `hittable` 를 가지고 있습니다. 이제 `hittable` 의 리스트를 저장하는 클래스를 추가할 것입니다.

```cpp
#ifndef HITTABLE_LIST_H
#define HITTABLE_LIST_H

#include "hittable.h"

#include <memory>
#include <vector>

using std::make_shared;
using std::shared_ptr;

class hittable_list : public hittable {
  public:
    std::vector<shared_ptr<hittable>> objects;

    hittable_list() {}
    hittable_list(shared_ptr<hittable> object) { add(object); }

    void clear() { objects.clear(); }

    void add(shared_ptr<hittable> object) {
      objects.push_back(object);
    }

    bool hit(const ray& r, double ray_tmin, double ray_tmax, hit_record& rec) const override {
      hit_record temp_rec;
      bool hit_anything = false;
      auto closest_so_far = ray_tmax;

      for (const auto& object : objects) {
        if (object->hit(r, ray_tmin, closest_so_far, temp_rec)) {
          hit_anything = true;
          closest_so_far = temp_rec.t;
          rec = temp_rec;
        }
      }

      return hit_anything;
    }
};

#endif
```

**<p align="center">Listing 21:** [hittable_list.h] _The hittable_list class</p>_

---

## 6.6 Some New C++ Features

C++ 프로그래머가 아니라면 실수할 수 있는 몇 가지 C++ 기능을 `hittable_list` 클래스 코드에서 사용합니다: `vector` 와 `shared_ptr` 그리고 `make_shared` 입니다.

`shared_ptr<type>` 은 참조 카운팅(reference-counting)를 활용하여 할당된 특정 타입을 가리키는 포인터입니다. 이 포인터를 다른 shared pointer에 대입(대개 단순 대입)할 때마다 참조 카운트가 증가합니다. shared pointer가 스코프 범위 밖(블록이나 함수의 끝)으로 벗어나면, 참조 카운트가 감소합니다. 참조 카운트가 0이 되면 포인터가 참조했던 객체가 안전하게 삭제됩니다.

다음과 같이 일반적으로, shared pointer는 새로 할당되는 객체와 함께 초기화됩니다. 예를 들면 다음과 같습니다.

```cpp
shared_ptr<double> double_ptr = make_shared<double>(0.37);
shared_ptr<vec3>   vec3_ptr   = make_shared<vec3>(1.414214, 2.718281, 1.618034);
shared_ptr<sphere> sphere_ptr = make_shared<sphere>(point3(0, 0, 0), 1.0);
```

**<p align="center">Listing 22:** _An example allocation using shared_ptr</p>_

`make_shared<thing>(thing의 생성자 매개변수 ...)` 는 생성자 매개변수를 사용하여 `thing` 타입의 새 인스턴스를 할당합니다. 그리고 `shared_ptr<thing>` 가 리턴됩니다.

`make_shared<type>(...)` 의 리턴 타입으로 타입을 자동으로 추론할 수 있으므로, 위의 코드는 C++의 `auto` 타입 지정자를 사용하여 더 간단히 표현할 수 있습니다.

```cpp
auto double_ptr = make_shared<double>(0.37);
auto vec3_ptr   = make_shared<vec3>(1.414214, 2.718281, 1.618034);
auto sphere_ptr = make_shared<sphere>(point3(0, 0, 0), 1.0);
```

**<p align="center">Listing 23:** _An example allocation using shared_ptr with auto type</p>_

여기서는 shared pointer를 사용할 것입니다. 그 이유는 다수의 지오메트리가 하나의 공통 인스턴스를 공유할 수 있게 해주기 때문입니다(예를 들어, 여러 개의 구가 모두 동일한 색상 머티리얼을 사용하는 경우). 또한 메모리 관리를 자동화해 주어, 객체의 수명 관리를 더 쉽게 파악할 수 있기 때문입니다.

`std::shared_ptr` 은 `<memory>` 헤더에 포함되어 있습니다.

익숙하지 않은 두 번째 C++ 기능은 `std::vector` 입니다. 이것은 임의 타입을 요소로 가지는 일반적인 배열 같은 컬렉션입니다. 위에서, `hittable` 을 참조하는 포인터의 컬렉션을 사용했습니다. `std::vector` 는 새 값을 추가할 경우 자동으로 배열의 길이가 증가합니다: `objects.push_back(object)` 는 `std::vector` 멤버 변수 `objects` 의 끝에 값을 추가합니다.

`std::vector` 은 `<vector>` 헤더에 포함되어 있습니다.

마지막으로, listing 21에서 `using` 선언은 `std` 라이브러리의 `shared_ptr과` `make_shared` 를 사용한다고 컴파일러에게 알려줍니다. 그래서 이것들을 참조할 때마다 앞에 `std::` 를 붙일 필요가 없습니다.

---

## 6.7 Common Constants and Utility Functions

몇 가지 수학 상수들을 편리하게 사용할 수 있도록 별도의 헤더 파일에 정의해 둘 필요가 있습니다. 지금 당장은 무한대(infinity)만 필요하지만 나중에 필요하게 될 pi 또한 정의할 것입니다. 유용한 공통 상수들과 앞으로 사용할 유틸리티 함수들도 여기에 같이 넣어 둘 것입니다. 이 헤더 파일의 이름을 `rtweekend.h` 이라고 하겠습니다. `rtweekend.h` 헤더 파일은 앞으로 공통 기반 헤더 파일 역할을 할 것입니다.


```cpp
#ifndef RTWEEKEND_H
#define RTWEEKEND_H

#include <cmath>
#include <iostream>
#include <limits>
#include <memory>


// C++ Std Usings

using std::make_shared;
using std::shared_ptr;

// Constants

const double infinity = std::numeric_limits<double>::infinity();
const double pi = 3.1415926535897932385;

// Utility Functions

inline double degrees_to_radians(double degrees) {
  return degrees * pi / 180.0;
}

// Common Headers

#include "color.h"
#include "ray.h"
#include "vec3.h"

#endif
```

**<p align="center">Listing 24:** [rtweekend.h] _The rtweekend.h common header</p>_

프로그램 파일들이 `rtweekend.h` 를 가장 먼저 인클루드할 것이므로, 다른 헤더 파일들은 `rtweekend.h` 가 이미 인클루드 되었다고 암묵적으로 가정할 수 있습니다. 그렇다고 해도 여전히 헤더 파일들에서 각자 필요한 다른 헤더 파일들은 명시적으로 인클루드해야 합니다. 이러한 가정을 바탕으로 코드를 몇 가지 수정하겠습니다.

```diff
- #include <iostream>
```

**<p align="center">Listing 25:** [color.h] _Assume rtweekend.h inclusion for color.h</p>_

```diff
- #include <ray.h>
```

**<p align="center">Listing 26:** [hittable.h] _Assume rtweekend.h inclusion for hittable.h</p>_

```diff
- #include <memory>
#include <vector>

- using std::make_shared;
- using std::shared_ptr;
```

**<p align="center">Listing 27:** [hittable_list.h] _Assume rtweekend.h inclusion for hittable\_list.h</p>_

```diff
- #include "vec3.h"
```

**<p align="center">Listing 28:** [sphere.h] _Assume rtweekend.h inclusion for sphere.h</p>_

```diff
- #include <cmath>
- #include <iostream>
```

**<p align="center">Listing 29:** [vec3.h] _Assume rtweekend.h inclusion for vec3.h</p>_

새로운 main입니다:

```cpp
///////////////////////// 추가 ///////////////////////////////
#include "rtweekend.h"                                     //
/////////////////////////////////////////////////////////////

///////////////////////// 삭제 ///////////////////////////////
// #include "color.h"                                      //
// #include "ray.h"                                        //
// #include "vec3.h"                                       //
/////////////////////////////////////////////////////////////
///////////////////////// 추가 ///////////////////////////////
#include "hittable.h"                                      //
#include "hittable_list.h"                                 //
#include "sphere.h"                                        //
/////////////////////////////////////////////////////////////

///////////////////////// 삭제 ///////////////////////////////
// #include <iostream>                                     //
/////////////////////////////////////////////////////////////

///////////////////////// 삭제 ///////////////////////////////////////////////
// double hit_sphere(const point3& center, double radius, const ray& r) {  //
//     ...                                                                 //
// }                                                                       //
/////////////////////////////////////////////////////////////////////////////

///////////////////////// 수정 ///////////////////////////////
color ray_color(const ray& r, const hittable& world) {     //
  hit_record rec;                                          //
  if (world.hit(r, 0, infinity, rec)) {                    //
    return 0.5 * (rec.normal + color(1, 1, 1));            //
  }                                                        //
/////////////////////////////////////////////////////////////

  vec3 unit_direction = unit_vector(r.direction());
  auto a = 0.5 * (unit_direction.y() + 1.0);
  return (1.0 - a) * color(1.0, 1.0, 1.0) + a * color(0.5, 0.7, 1.0);
}

int main() {

  // Image

  auto aspect_ratio = 16.0 / 9.0;
  int image_width = 400;

  // Calculate the image height, and ensure that it's at least 1.
  int image_height = int(image_width / aspect_ratio);
  image_height = (image_height < 1) ? 1 : image_height;

///////////////////////// 추가 ////////////////////////////////////
  // World                                                      //
                                                                //
  hittable_list world;                                          //
                                                                //
  world.add(make_shared<sphere>(point3(0, 0, -1), 0.5));        //
  world.add(make_shared<sphere>(point3(0, -100.5, -1), 100));   //
//////////////////////////////////////////////////////////////////

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
                           - vec3(0, 0, focal_length) - viewport_u / 2 - viewport_v / 2;
  auto pixel00_loc = viewport_upper_left + 0.5 * (pixel_delta_u + pixel_delta_v);

  // Render

  std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";

  for (int j = 0; j < image_height; j++) {
    std::clog << "\rScanlines remaining: " << (image_height - j) << ' ' << std::flush;
    for (int i = 0; i < image_width; i++) {
      auto pixel_center = pixel00_loc + (i * pixel_delta_u) + (j * pixel_delta_v);
      auto ray_direction = pixel_center - camera_center;
      ray r(camera_center, ray_direction);

///////////////////////// 수정 ///////////////////////////////
      color pixel_color = ray_color(r, world);             //
/////////////////////////////////////////////////////////////
      write_color(std::cout, pixel_color);
    }
  }

  std::clog << "\rDone.                 \n";
}
```

**<p align="center">Listing 30:** [<span>main.</span>cc] _The new main with hittables</p>_

이렇게 하면 구들의 위치와 그 표면 법선 벡터를 함께 시각화한 이미지를 얻을 수 있습니다. 이 방법은 종종 지오메트리 모델의 결함이나 특징을 확인하는데 매우 좋습니다.

<p align="center"><img src="https://raytracing.github.io/images/img-1.05-normals-sphere-ground.png"></p>

**<p align="center">Image 5:** _Resulting render of normals-colored sphere with ground</p>_

## 6.8 An Interval Class

계속 진행하기 전에, 실수 구간(real-valued intervals)을 다루기 위한 interval 클래스를 구현하겠습니다. 실수 구간은 최솟값과 최댓값으로 표현됩니다. 앞으로 interval 클래스를 꽤 자주 사용하게 될 것입니다.

```cpp
#ifndef INTERVAL_H
#define INTERVAL_H

class interval {
  public:
    double min, max;

    interval() : min(+infinity), max(-infinity) {} // Default interval is empty

    interval(double min, double max) : min(min), max(max) {}

    double size() const {
      return max - min;
    }

    bool contains(double x) const {
      return min <= x && x <= max;
    }

    bool surrounds(double x) const {
      return min < x && x < max;
    }

    static const interval empty, universe;
};

const interval interval::empty    = interval(+infinity, -infinity);
const interval interval::universe = interval(-infinity, +infinity);

#endif
```

**<p align="center">Listing 31:** [<span>interval.</span>h] _Introducing the new interval class</p>_

```cpp
// Common Headers

#include "color.h"
///////////////////////// 추가 ///////////////////////////////
#include "interval.h"                                      //
/////////////////////////////////////////////////////////////
#include "ray.h"
#include "vec3.h"
```

**<p align="center">Listing 32:** [<span>rtweekend.</span>h] _Including the new interval class</p>_

```cpp
class hittable {
  public:
    ...
///////////////////////// 수정 ///////////////////////////////////////////////////////////
    virtual bool hit(const ray& r, interval ray_t, hit_record& rec) const = 0;        //
/////////////////////////////////////////////////////////////////////////////////////////
};
```

**<p align="center">Listing 33:** [<span>hittable.</span>h] _hittable::hit() using interval</p>_

```cpp
class hittable_list : public hittable {
  public:
    ...
///////////////////////// 수정 /////////////////////////////////////////////////////
    bool hit(const ray& r, interval ray_t, hit_record& rec) const override {     //
///////////////////////////////////////////////////////////////////////////////////
      hit_record temp_rec;
      bool hit_anything = false;
///////////////////////// 수정 /////////////////////////////////////////////////////
      auto closest_so_far = ray_t.max;                                           //
///////////////////////////////////////////////////////////////////////////////////

      for (const auto& object : objects) {
///////////////////////// 수정 /////////////////////////////////////////////////////
        if (object->hit(r, interval(ray_t.min, closest_so_far), temp_rec)) {     //
///////////////////////////////////////////////////////////////////////////////////
          hit_anything = true;
          closest_so_far = temp_rec.t;
          rec = temp_rec;
        }
      }

      return hit_anything;
    }
    ...
};
```

**<p align="center">Listing 34:** [<span>hittable_list.</span>h] _hittable\_list::hit() using interval</p>_

```cpp
class sphere : public hittable {
  public:
    ...
///////////////////////// 수정 /////////////////////////////////////////////////////
    bool hit(const ray& r, interval ray_t, hit_record& rec) const override {     //
///////////////////////////////////////////////////////////////////////////////////
      ...

      // Find the nearest root that lies in the acceptable range.
      auto root = (h - sqrtd) / a;
///////////////////////// 수정 /////////////////////////////////////////////////////
      if (!ray_t.surrounds(root)) {                                              //
///////////////////////////////////////////////////////////////////////////////////
        root = (h + sqrtd) / a;
///////////////////////// 수정 /////////////////////////////////////////////////////
        if (!ray_t.surrounds(root))                                              //
///////////////////////////////////////////////////////////////////////////////////
          return false;
      }
      ...
    }
    ...
};
```

**<p align="center">Listing 35:** [<span>sphere.</span>h] _sphere using interval</p>_

```cpp
color ray_color(const ray& r, const hittable& world) {
  hit_record rec;
///////////////////////// 수정 //////////////////////////
  if (world.hit(r, interval(0, infinity), rec)) {     //
////////////////////////////////////////////////////////
    return 0.5 * (rec.normal + color(1, 1, 1));
  }

  vec3 unit_direction = unit_vector(r.direction());
  auto a = 0.5 * (unit_direction.y() + 1.0);
  return (1.0 - a) * color(1.0, 1.0, 1.0) + a * color(0.5, 0.7, 1.0);
}
```

**<p align="center">Listing 36:** [<span>main.</span>cc] _The new main using interval</p>_

---

## 출처

**Ray Tracing in One Weekend - Peter Shirley**
https://raytracing.github.io/books/RayTracingInOneWeekend.html#surfacenormalsandmultipleobjects
