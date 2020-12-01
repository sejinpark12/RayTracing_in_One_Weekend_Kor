>**이 글은 Peter Shirley의 [Ray Tracing in One Weekend](https://raytracing.github.io/books/RayTracingInOneWeekend.html)를 번역한 것입니다.
Ray Tracing in One Weekend를 공부하면서 다시 한번 복습하는 느낌으로 번역을 해보려고 합니다. 영어가 서툴러 번역이 잘못되었을 수도 있으므로 잘못된 부분을 발견하신다면 지적해 주시면 감사하겠습니다.**

유전체와 마찬가지로 카메라는 디버깅하기가 어렵습니다. 그래서 저는 항상 점진적으로 코드를 작성합니다. 첫 번째, 조절할 수 있는 시야(field of view : fov)를 적용합니다. fov는 문을 통해 보는 각도입니다. 우리가 다루는 이미지는 정사각형이 아니기 때문에, fov의 수직길이와 수평길이가 다릅니다. 저는 항상 수직 fov를 사용합니다. 또한 저는 주로 생성자 안에서 각도를 지정하여 라디안으로 변환합니다 - 개인적인 취향입니다.

## 11.1 Camera Viewing Geometry
---

처음에 광선이 원점에서 𝑧 = −1 평면으로 향하게 합니다. fov(𝜃)에 따른 𝑧 = −1 평면과 원점 사이의 거리에 대한 비율 ℎ를 구했다면 𝑧 = −2 평면이나 어떤 평면이든지 사용할 수 있습니다. 다음과 같이 설정합니다:

<p align="center"><img src="https://raytracing.github.io/images/fig-1.14-cam-view-geom.jpg"></p>

**<p align="center">Figure 14:** _Camera viewing geometry</p>_

여기서 ℎ = tan(𝜃 / 2)임을 알 수 있습니다. 우리의 카메라는 다음과 같습니다:

```c
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

```c
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
## 11.2 Positioning and Orienting the Camera
---

임의의 시점을 구하기 위해서, 우리가 관심 있는 점의 이름을 먼저 정하겠습니다. 카메라가 위치한 점을 _lookfrom_, 바라보는 점을 _lookat_ 이라고 부르겠습니다. (나중에, 원하신다면, 바라보는 점 대신에 바라보는 방향을 정의할 수 있습니다.)

카메라 롤(roll), 틸트(tilt)를 지정하는 방법이 필요합니다:

---

#### 출처 https://raytracing.github.io/books/RayTracingInOneWeekend.html#positionablecamera
