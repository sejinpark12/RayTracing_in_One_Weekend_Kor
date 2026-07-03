# 5. Adding a Sphere
이제, 레이 트레이서에 오브젝트 한 개를 추가해봅시다. 가장 먼저 추가할 수 있는 오브젝트는 일반적으로 구(Sphere)입니다. 광선이 구에 충돌하는지 판별하는 계산은 비교적 간단하기 때문입니다.

## 5.1 Ray-Sphere Intersection
반지름이 $r$ 이고 원점 중심인 구의 방정식은 중요한 방정식입니다.

$$ x^2 + y^2 + z^2 = r^2 $$

이 식은 이렇게도 생각할 수 있습니다. 어떤 점 $(x,y,z)$ 가 구의 표면에 위치한다면 $x^2 + y^2 + z^2 = r^2$ 를 만족합니다. 어떤 점 $(x,y,z)$ 가 구의 내부에 위치한다면 $x^2 + y^2 + z^2 < r^2$ 를 만족하고, 어떤 점 $(x,y,z)$ 가 구의 외부에 위치한다면 $x^2 + y^2 + z^2 > r^2$ 를 만족합니다.

구의 중심이 임의의 점 $(C_x, C_y, C_z)$ 에 위치한다면 구의 방정식은 더 복잡해집니다.

$$ (C_x - x)^2 + (C_y - y)^2 + (C_z - z)^2 = r^2 $$

그래픽스에서는 대부분의 수식을 벡터 형태로 표현하고자 합니다. 그러면 $x$ / $y$ / $z$ 같은 항들 모두를 `vec3` 클래스로 간단히 나타낼 수 있기 때문입니다. 점 $\mathbf{P} = (x,y,z)$ 에서 중심 $\mathbf{C} = (C_x, C_y, C_z)$ 까지의 벡터는 $(\mathbf{C} - \mathbf{P})$ 로 표현합니다.

내적(dot product)의 정의를 사용하면 다음과 같습니다.

$$ (\mathbf{C} - \mathbf{P}) \cdot (\mathbf{C} - \mathbf{P})
  = (C_x - x)^2 + (C_y - y)^2 + (C_z - z)^2
$$

그렇다면 벡터 형태의 구의 방정식을 아래와 같이 나타낼 수 있습니다.

$$ (\mathbf{C} - \mathbf{P}) \cdot (\mathbf{C} - \mathbf{P}) = r^2 $$

이것을 "이 방정식을 만족하는 모든 점 $\mathbf{P}$ 는 구 표면에 위치한다"라고 생각할 수 있습니다. 우리는 광선 $\mathbf{P}(t) = \mathbf{Q} + t\mathbf{d}$ 가 구와 충돌하는지 알고 싶습니다. 만약 광선이 구와 충돌한다면, 구의 방정식을 만족하는 $\mathbf{P}(t)$ 에 대한 $t$ 가 존재합니다. 따라서 아래의 식이 참이 되는 $t$ 를 찾는 것이 목표입니다.

$$ (\mathbf{C} - \mathbf{P}(t)) \cdot (\mathbf{C} - \mathbf{P}(t)) = r^2 $$

$\mathbf{P}(t)$ 를 전개한 형태로 바꾸면 아래와 같이 됩니다.

$$ (\mathbf{C} - (\mathbf{Q} + t \mathbf{d}))
    \cdot (\mathbf{C} - (\mathbf{Q} + t \mathbf{d})) = r^2 $$

내적의 왼쪽 괄호 안에 벡터 세 개와 오른쪽 괄호 안에 벡터 세 개가 존재하므로 내적을 전개하면 아홉 개의 항이 생깁니다. 물론 하나하나 모두 전개할 수 있지만, 그렇게까지 할 필요는 없습니다. 여기서 구하려는 것은 $t$ 에 대한 해이므로, $t$ 가 포함된 항과 포함되지 않은 항으로 분리하여 식을 정리합니다.

$$ (-t \mathbf{d} + (\mathbf{C} - \mathbf{Q})) \cdot (-t \mathbf{d} + (\mathbf{C} - \mathbf{Q}))
    = r^2
$$

이제 벡터 대수의 규칙에 따라 내적을 전개합니다.

$$ t^2 \mathbf{d} \cdot \mathbf{d}
    - 2t \mathbf{d} \cdot (\mathbf{C} - \mathbf{Q})
    + (\mathbf{C} - \mathbf{Q}) \cdot (\mathbf{C} - \mathbf{Q}) = r^2
$$

반지름 제곱을 좌변으로 이항시키면 아래의 식이 됩니다.

$$ t^2 \mathbf{d} \cdot \mathbf{d}
    - 2t \mathbf{d} \cdot (\mathbf{C} - \mathbf{Q})
    + (\mathbf{C} - \mathbf{Q}) \cdot (\mathbf{C} - \mathbf{Q}) - r^2 = 0
$$

이 방정식이 정확히 어떤 방정식인지 한번에 알아보기는 어렵지만, 방정식 안의 벡터와 $r$ 은 모두 상수이고 이미 값을 알고 있습니다. 더욱이, 벡터들은 내적 연산으로 인해 숫자 하나인 스칼라로 바뀝니다. 미지수는 $t$ 뿐이고, 방정식에 $t^2$ 항이 존재하므로 이차방정식이라는 것을 알 수 있습니다. 이차방정식의 근의 공식을 사용하여 이차방정식 $ax^2 + bx + c = 0$ 의 해를 구할 수 있습니다.

$$ \frac{-b \pm \sqrt{b^2 - 4ac}}{2a} $$

따라서 광선-구 교차 방정식에서 $t$ 에 대한 풀이는 다음과 같은 $a$, $b$, $c$ 값을 가집니다.

$$ a = \mathbf{d} \cdot \mathbf{d} $$
$$ b = -2 \mathbf{d} \cdot (\mathbf{C} - \mathbf{Q}) $$
$$ c = (\mathbf{C} - \mathbf{Q}) \cdot (\mathbf{C} - \mathbf{Q}) - r^2 $$

위의 내용들을 바탕으로 $t$ 의 해를 구할 수 있습니다. 루트 안의 판별식은 양수(실수 해가 두 개), 음수(실수 해가 없음), 0(실수 해가 하나)가 될 수 있습니다. 그래픽스에서는 대수적인 결과가 거의 언제나 기하학적인 의미와 직접적으로 대응되어 있습니다. 그러므로 다음과 같은 결과를 확인할 수 있습니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.05-ray-sphere.jpg"></p>

**<p align="center">Figure 5**: _Ray-sphere intersection results</p>_

---

## 5.2 Creating Our First Raytraced Image
위의 내용을 하드코딩해 보겠습니다. z축의 -1에 작은 구 하나를 놓고 광선이 구와 교차하면 픽셀에 빨간색을 칠하도록 테스트합니다.

```cpp
///////////////////////// 추가 ////////////////////////////////////////////
bool hit_sphere(const point3& center, double radius, const ray& r) {    //
  vec3 oc = center - r.origin();                                        //
  auto a = dot(r.direction(), r.direction());                           //
  auto b = -2.0 * dot(r.direction(), oc);                               //
  auto c = dot(oc, oc) - radius * radius;                               //
  auto discriminant = b * b - 4 * a * c;                                //
  return (discriminant >= 0);                                           //
}                                                                       //
//////////////////////////////////////////////////////////////////////////

color ray_color(const ray& r) {
///////////////////////// 추가 ////////////////////////////////////////////
  if (hit_sphere(point3(0, 0, -1), 0.5, r))                             //
    return color(1, 0, 0);                                              //
//////////////////////////////////////////////////////////////////////////

  vec3 unit_direction = unit_vector(r.direction());
  auto a = 0.5 * (unit_direction.y() + 1.0);
  return (1.0 - a) * color(1.0, 1.0, 1.0) + a * color(0.5, 0.7, 1.0);
}
```

**<p align="center">Listing 11:** [<span>main</span>.cc] _Rendering a red sphere</p>_

다음과 같은 결과를 얻을 수 있습니다.

<p align="center"><img src="https://raytracing.github.io/images/img-1.03-red-sphere.png"></p>

**<p align="center">Image 3:** _A simple red sphere</p>_

현재 구현에는 많은 요소가 빠져 있습니다(셰이딩, 반사광 그리고 오브젝트 개수와 같은). 하지만 이제 처음 상태보다는 절반 정도 완성된 상태에 더 가까워졌습니다! 한 가지 주의할 점이 있습니다. 여기서는 광선이 구와 교차하는지 확인하기 위해 이차방정식의 해가 존재하는지 확인하고 있습니다. 따라서 $t$ 가 음수인 경우에도 해로 인정하고 있습니다. 그러므로 구의 중심을 $z = +1$ 로 바꿔도 구의 중심이 $z = -1$ 인 이미지와 완전히 동일한 이미지가 출력됩니다. 그 이유는 이 방식이 _카메라 앞에 있는_ 오브젝트와 _카메라 뒤에 있는_ 오브젝트를 구분하지 못하기 때문입니다. 이건 의도한 기능이 아닙니다! 이런 문제들은 뒤에서 수정하겠습니다.

---

## 출처

**Ray Tracing in One Weekend - Peter Shirley**
https://raytracing.github.io/books/RayTracingInOneWeekend.html#addingasphere
