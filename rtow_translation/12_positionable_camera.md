# 12. Positionable Camera

유전체와 마찬가지로 카메라는 디버깅하기가 어렵습니다. 그래서 저는 항상 점진적으로 코드를 작성합니다. 첫 번째, 조절할 수 있는 시야(field of view : fov)를 적용합니다. fov는 문을 통해 보는 각도입니다. 우리가 다루는 이미지는 정사각형이 아니기 때문에, fov의 수직길이와 수평길이가 다릅니다. 저는 항상 수직 fov를 사용합니다. 또한 저는 주로 생성자 안에서 각도를 지정하여 라디안으로 변환합니다 - 개인적인 취향입니다.

## 12.1 Camera Viewing Geometry

---

처음에 광선이 원점에서 𝑧 = −1 평면으로 향하게 합니다. fov(𝜃)에 따른 𝑧 = −1 평면과 원점 사이의 거리에 대한 비율 ℎ를 구했다면 𝑧 = −2 평면이나 어떤 평면이든지 사용할 수 있습니다. 다음과 같이 설정합니다:

<p align="center"><img src="https://raytracing.github.io/images/fig-1.14-cam-view-geom.jpg"></p>

**<p align="center">Figure 14:** _Camera viewing geometry</p>_

여기서 ℎ = tan(𝜃 / 2)임을 알 수 있습니다. 우리의 카메라는 다음과 같습니다:

```cpp
class camera {
public:
  camera(
    double vfov,    // vertical field-of-view in degrees
    double aspect_ratio
  ) {
    auto theta = degrees_to_radians(vfov);
    auto h = tan(theta / 2);
    auto viewport_height = 2.0 * h;
    auto viewport_width = aspect_ratio * viewport_height;

    auto focal_length = 1.0;

    origin = point3(0, 0, 0);
    horizontal = vec3(viewport_width, 0.0, 0.0);
    vertical = vec3(0.0, viewport_height, 0.0);
    lower_left_corner = origin - horizontal / 2 - vertical / 2 - vec3(0, 0, focal_length);
  }

  ray get_ray(double u, double v) const {
    return (ray(origin, lower_left_corner + u * horizontal + v * vertical - origin));
  }

private:
  point3 origin;
  point3 lower_left_corner;
  vec3 horizontal;
  vec3 vertical;
};
```

**<p align="center">Listing 62:** [camera.h] _Camera with adjustable field-of-view(fov)</p>_

아래와 같이 cam(90, aspect_ratio) 생성자로 카메라 객체를 생성하고 구를 추가합니다.

```cpp
int main() {
  ...
  // World

  auto R = cos(pi / 4);
  hittable_list world;

  auto material_left = make_shared<lambertian>(color(0, 0, 1));
  auto material_right = make_shared<lambertian>(color(1, 0, 0));

  world.add(make_shared<sphere>(point3(-R, 0, -1), R, material_left));
  world.add(make_shared<sphere>(point3( R, 0, -1), R, material_right));

  // Camera
  camera cam(90.0, aspect_ratio);

  // Render
  std::cout << "P3\n" << image_width << " " << image_height << "\n255\n";

  for (int j = image_height - 1; j >= 0; --j) {
  ...
}
```

**<p align="center">Listing 63:** [<span>main.</span>cc] _Scene with wide-angle camera</p>_

다음의 이미지를 얻을 수 있습니다:

<p align="center"><img src="https://raytracing.github.io/images/img-1.17-wide-view.png"></p>

**<p align="center">Image 17:** _A wide-angle view</p>_

---

## 12.2 Positioning and Orienting the Camera

---

임의의 시점을 구하기 위해서, 우리가 관심 있는 점의 이름을 먼저 정하겠습니다. 카메라가 위치한 점을 _lookfrom_, 바라보는 점을 _lookat_ 이라고 부르겠습니다. (나중에, 원하신다면, 바라보는 점 대신에 바라보는 방향을 정의할 수 있습니다.)

카메라 롤(roll) 또는 틸트(tilt)를 지정하는 방법이 필요합니다: lookat-lookfrom 축을 중심으로 회전하는 것입니다. 이 동작을 생각하는 다른 방법은 lookfrom과 lookat을 바꾸지 않고 코를 중심으로 머리를 회전하는 것입니다. 우리가 필요한 것은 카메라의 "up" 벡터를 지정하는 방법입니다. 이 up 벡터는 view 방향과 직각인 평면 상에 위치해야만 합니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.15-cam-view-dir.jpg"></p>

**<p align="center">Figure 15:** _Camera view Direction</p>_

사실 여러분이 원하는 어떤 up 벡터도 사용할 수 있으며, 단순히 이 평면에 투영하여 카메라의 up 벡터를 얻을 수 있습니다. 저는 "view up" (vup) 벡터 네이밍 컨벤션을 따르겠습니다. 몇번의 외적 연산을 통해 우리 카메라의 방향을 표현하는 완전한 직교 기저 (𝑢, 𝑣, 𝑤) 얻을 수 있습니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.16-cam-view-up.jpg"></p>

**<p align="center">Figure 16:** _Camera view up direction</p>_

`vup`, `v`, `w`는 모두 동일한 평면에 존재하는 것을 기억하십시오. 이전의 고정 카메라가 -Z를 바라봤듯이, 임의 뷰 카메라는 -w를 바라봅니다. 그리고 vup을 지정하기 위해 world up(0, 1, 0)을 사용(하지만 반드시 그럴 필요는 없습니다.)할 수 있다는 것을 명심하십시오. 이 방식은 편리합니다. 그리고 여러분이 미친 카메라 각도로 실험하기 전까지 자연스럽게 카메라를 수평으로 유지할 수 있습니다.

```cpp
class camera {
public:
  camera(
    point3  lookfrom,
    point3  lookat,
    vec3    vup,
    double  vfov,     // vertical field-of-view in degrees
    double  aspect_ratio
  ) {
    auto theta = degrees_to_radians(vfov);
    auto h = tan(theta / 2);
    auto viewport_height = 2.0 * h;
    auto viewport_width = aspect_ratio * viewport_height;

    auto w = unit_vector(lookfrom - lookat);
    auto u = unit_vector(cross(vup, w));
    auto v = cross(w, u);

    origin = lookfrom;
    horizontal = viewport_width * u;
    vertical = viewport_height * v;
    lower_left_corner = origin - horizontal / 2 - vertical / 2 - w;
  }

  ray get_ray(double s, double t) const {
    return (ray(origin, lower_left_corner + s * horizontal + t * vertical - origin));
  }

private:
  point3  origin;
  point3  lower_left_corner;
  vec3    horizontal;
  vec3    vertical;
};
```

**<p align="center">Listing 64:** [camera.h] _Positionable and orientable camera</p>_

다시 원래의 씬으로 되돌리고, 새로운 시점을 사용합니다.

```cpp
hittable_list world;

auto material_ground = make_shared<lambertian>(color(0.8, 0.8, 0.0));
auto material_center = make_shared<lambertian>(color(0.1, 0.2, 0.5));
auto material_left = make_shared<dielectric>(1.5);
auto material_right = make_shared<metal>(color(0.8, 0.6, 0.2), 0.0);

world.add(make_shared<sphere>(point3( 0.0, -100.5, -1.0), 100.0, material_ground));
world.add(make_shared<sphere>(point3( 0.0,    0.0, -1.0),   0.5, material_center));
world.add(make_shared<sphere>(point3(-1.0,    0.0, -1.0),   0.5, material_left));
world.add(make_shared<sphere>(point3(-1.0,    0.0, -1.0), -0.45, material_left));
world.add(make_shared<sphere>(point3( 1.0,    0.0, -1.0),   0.5, material_right));

camera cam(point3(-2, 2, 1), point3(0, 0, -1), vec3(0, 1, 0), 90, aspect_ratio);
```

**<p align="center">Listing 65:** [<span>main.</span>cc] _Scene with alternate viewpoint</p>_

아래의 이미지를 얻을 수 있습니다:

<p align="center"><img src="https://raytracing.github.io/images/img-1.18-view-distant.png"></p>

**<p align="center">Image 18:** _A distant view</p>_

field of view를 변경할 수 있습니다:

```cpp
camera cam(pont3(-2, 2, 1), point3(0, 0, -1), vec3(0, 1, 0), 20, aspect_ratio);
```

**<p align="center">Listing 66:** [<span>main.</span>cc] _Change field of view</p>_

아래의 이미지를 얻을 수 있습니다:

<p align="center"><img src="https://raytracing.github.io/images/img-1.19-view-zoom.png"></p>

**<p align="center">Image 19:** _Zooming in</p>_

---

## 출처

**Ray Tracing in One Weekend - Peter Shirley**
https://raytracing.github.io/books/RayTracingInOneWeekend.html#positionablecamera
