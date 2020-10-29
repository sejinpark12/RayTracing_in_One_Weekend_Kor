>**이 글은 Peter Shirley의 [Ray Tracing in One Weekend](https://raytracing.github.io/books/RayTracingInOneWeekend.html)를 번역한 것입니다.
Ray Tracing in One Weekend를 공부하면서 다시 한번 복습하는 느낌으로 번역을 해보려고 합니다. 영어가 서툴러 번역이 잘못되었을 수도 있으므로 잘못된 부분을 발견하신다면 지적해 주시면 감사하겠습니다.**

레이 트레이서에 오브젝트 한 개를 추가해봅시다. 구(sphere)는 광선이 오브젝트를 교차하는지 계산이 매우 간단하기 때문에 레이 트레이서에서 구가 자주 사용됩니다.

## 5.1 Ray-Sphere Intersection
---
반지름이 𝑅이고 원점 중심인 구의 방정식은 𝑥<sup>2</sup> + 𝑦<sup>2</sup> + 𝑧<sup>2</sup> = 𝑅<sup>2</sup>입니다. 주어진 점(x, y, z)이 구 위에 있다면, 𝑥<sup>2</sup> + 𝑦<sup>2</sup> + 𝑧<sup>2</sup> = 𝑅<sup>2</sup>를 만족합니다. 주어진 점(x, y, z)이 구 안에 있다면 𝑥<sup>2</sup> + 𝑦<sup>2</sup> + 𝑧<sup>2</sup> < 𝑅<sup>2</sup>를 만족합니다. 주어진 점(x, y, z)이 구 밖에 있다면 𝑥<sup>2</sup> + 𝑦<sup>2</sup> + 𝑧<sup>2</sup> > 𝑅<sup>2</sup>를 만족합니다.

점(𝐶<sub>𝑥</sub>, 𝐶<sub>𝑦</sub>, 𝐶<sub>𝑧</sub>)가 구의 중심이라면 구의 방정식은 다음과 같습니다:

![image1](https://user-images.githubusercontent.com/19530862/95719074-482fd500-0caa-11eb-80ed-b6ddeeef09e3.png)

그래픽스에서는 거의 항상, 공식이 벡터 형식으로 표현되기를 원합니다. 그래서 `vec3` 클래스에는 내부적으로 x/y/z가 존재합니다. 중심 𝐂 = (𝐶<sub>𝑥</sub>, 𝐶<sub>𝑦</sub>, 𝐶<sub>𝑧</sub>)에서 점 𝐏 = (𝑥, 𝑦, 𝑧)까지의 벡터는 (𝐏 − 𝐂)입니다. 그러므로 다음 식이 성립합니다.

 ![image2](https://user-images.githubusercontent.com/19530862/95719080-49610200-0caa-11eb-925e-fb3ce9a63861.png)

벡터 형태의 구의 방정식은 다음과 같습니다:

![image3](https://user-images.githubusercontent.com/19530862/95719082-49f99880-0caa-11eb-99a3-e19285f45c4e.png)

"점 𝐏가 이 방정식을 만족한다면 점 𝐏는 구 위에 위치한다"라고 이해할 수 있습니다. 우리는 광선 𝐏(𝑡) = 𝐀 + 𝑡𝐛 가 구를 교차하는지 알고 싶습니다. 광선이 구를 교차한다면, 구의 방정식을 만족하는 𝐏(𝑡)의 𝑡가 존재합니다.

![image4](https://user-images.githubusercontent.com/19530862/95719102-5120a680-0caa-11eb-98ed-e041d88242bf.png)

광선 𝐏(𝑡)의 완전한 형태로 풀어쓰면:

![image5](https://user-images.githubusercontent.com/19530862/95719100-4fef7980-0caa-11eb-8a47-a35fc77d2d42.png)

벡터 대수 규칙은 여기서 우리가 원하는 전부입니다. 좌항을 풀어서 다시 정리하고 𝑟<sup>2</sup>을 좌항으로 이항시키면:

![image6](https://user-images.githubusercontent.com/19530862/95719104-52ea6a00-0caa-11eb-9b1d-75264e851c8e.png)

방정식에 있는 벡터와 𝑟은 모두 상수로 알고 있는 값이며 아직 모르는 값은 𝑡입니다. 이 방정식은 고등학교 수학 시간에 봤던 이차방정식입니다. 𝑡의 해를 구해보면 해의 루트 부분이 양수인 경우(2개의 실수 해 존재), 음수인 경우(실수 해 없음), 0인 경우(1개의 실수 해 존재) 이렇게 3가지 경우가 가능합니다. 그래픽스에서는 거의 항상, 대수학이 매우 직접적으로 기하학과 연관되어 있습니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.04-ray-sphere.jpg"></p>

**<p align="center">Figure 4**: _Ray-sphere intersection results</p>_

> ❗위의 이차방정식의 실수 해가 존재하지 않는다면 광선이 구와 교차하지 않는다.
> 실수 해가 1개 존재한다면 광선이 구의 한 점에서 접한다.
> 실수 해가 2개 존재한다면 광선이 구를 통과하여 두 점에서 교차한다.

---
## 5.2 Creating Our First Raytraced Image
---
아래와 같이 하드코딩한다면, 광선이 z 축의 -1에 위치한 작은 구를 교차하면 픽셀에 빨간색을 표시하도록 테스트할 수 있습니다.

```cpp
/* ************* 추가 ************ */
bool hit_sphere(const point3& center, double radius, const ray& r) {
  vec3 oc = r.origin() - center;
  auto a = dot(r.direction(), r.direction());
  auto b = 2.0 * dot(oc, r.direction());
  auto c = dot(oc, oc) - radius * radius;
  auto discriminant = b * b - 4 * a * c;
  return (discriminant > 0);
}
/* ******************************* */

color ray_color(const ray& r) {
/* ************* 추가 ************ */
  if (hit_sphere(point3(0, 0, -1), 0.5, r))
    return color(1, 0, 0);
/* ******************************* */
  vec3 unit_direction = unit_vector(r.direction());
  auto t = 0.5 * (unit_direction.y() + 1.0);
  return (1.0 - t) * color(1.0, 1.0, 1.0) + t * color(0.5, 0.7, 1.0);
}
```
**<p align="center">Listing 10:** [<span>main</span>.cc] _Rendering a red sphere</p>_

다음과 같은 결과를 얻을 수 있습니다.

<p align="center"><img src="https://raytracing.github.io/images/img-1.03-red-sphere.png"></p>

**<p align="center">Image 3:** _A simple red sphere</p>_

아직 많이 부족해 보입니다(셰이딩, 반사광 그리고 오브젝트 개수와 같은). 하지만 처음 시작에서 절반 정도 지나왔습니다! 주의할 점이 한 가지 있습니다. 우리는 광선이 모두 구를 교차하는지를 테스트한 것입니다. 하지만 𝑡가 음수인 경우도 동작합니다. 만약 구의 중심을 𝑧 = +1(구가 카메라의 뒤에 위치)로 옮긴다고 해도 기존의 𝑧 = -1 위치에 구가 있는 경우와 완전히 같은 이미지가 출력됩니다. 이것은 우리가 의도한 기능이 아닙니다! 다음에 이러한 문제들을 수정할 것입니다.

> ❗𝑡 < 0이면 광선 방향의 반대 방향이 된다

---

#### 출처 https://raytracing.github.io/books/RayTracingInOneWeekend.html#addingasphere
