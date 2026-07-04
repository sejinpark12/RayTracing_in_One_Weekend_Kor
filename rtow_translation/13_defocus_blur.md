# 13. Defocus Blur
이제 마지막으로 구현할 기능은 _defocus blur_ 입니다. 사진사들은 이 기능을 _depth of field_ 라고 부르기 때문에, _defocus blur_ 라는 용어는 레이 트레이싱 분야의 사람들끼리만 사용해야 됩니다.

실제 카메라는 구멍이 아주 작은 핀홀이 아닌 빛을 더 많이 모으기 위한 큰 구멍이 있기 때문에 defocus blur가 발생합니다. 큰 구멍만 있으면 빛이 퍼져 들어와서 모든 물체에 초점이 맞지 않습니다. 하지만 필름/센서 앞에 렌즈를 두면, 초점이 정확히 맞는 특정한 거리가 생깁니다. 이 거리에 놓인 물체들은 초점이 맞아 선명하게 보이고 이 거리에서 멀어질수록 물체들은 선형적으로 더 흐리게 보입니다. focus distance만큼 떨어진 _특정한 지점으로부터_ 출발하여 렌즈에 닿는 모든 광선은 렌즈에서 굴절되어 _이미지 센서의 한 점으로_ 다시 모인다고 생각할 수 있습니다.

카메라 중심부터 focus plane(초점이 완벽히 맞는 평면)까지의 거리를 _focus distance_ 라고 부릅니다. 여기서 일반적으로 focus distance는 focal length와 다르다는 점을 주의하세요. _focal length_ 는 카메라 중심과 이미지 평면 사이의 길이입니다. 그러나 여기서 다루는 카메라 모델은 이 두 값이 동일합니다. 그 이유는 픽셀 그리드를 focus plane 위에 둘 것이고 focus plane은 카메라 중심에서 _focus distance_ 만큼 떨어져 있기 때문입니다.

실제 카메라에서는 렌즈와 필름/센서 사이의 거리를 조절해서 focus distance를 바꿉니다. 이것이 어떤 물체에 초점을 맞출 때 카메라 기준으로 렌즈를 앞뒤로 움직이는 이유입니다. 휴대폰 카메라에서도 마찬가지입니다. 하지만 이 경우에는 센서가 움직입니다. "조리개(aperture)"는 렌즈가 빛을 얼마나 넓게 받아들일지를 조절하는 구멍입니다. 더 많은 빛이 필요할 경우, 실제 카메라에서는 조리개를 더 크게 엽니다. 그러면 focus distance에서 멀리 떨어진 초점 밖의 물체들은 더 흐려지게 됩니다. 가상의 카메라에서는 완벽한 센서를 가정할 수 있어서 더 많은 빛이 필요하지 않습니다. 따라서 가상 카메라의 조리개는 오직 defocus blur를 표현하기 위해서만 사용합니다.

## 13.1 A Thin Lens Approximation
실제 카메라는 복잡한 복합 렌즈로 이루어져 있습니다. 센서, 렌즈, 조리개의 순서 그대로 카메라를 시뮬레이션할 수도 있습니다. 그렇게 하면 광선을 어디로 보내야 할지 계산할 수 있고, 계산이 끝난 다음에는 이미지를 뒤집는 처리를 합니다. 필름에는 위아래가 뒤집힌 이미지가 투영되기 때문입니다. 그러나 그래픽스에서는 이 방법 대신에 보통 얇은 렌즈 근사(Thin Lens Approximation)를 사용합니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.21-cam-lens.jpg"></p>

**<p align="center">Figure 21:** _Camera lens model</p>_

카메라 바깥에 존재하는 씬의 이미지를 렌더링하기 위한 목적이라면 카메라 내부는 전혀 시뮬레이션할 필요가 없습니다. 이런 내부 시뮬레이션은 불필요하게 복잡하기만 하기 때문입니다. 대신에 무한히 얇은 원형 "렌즈"를 가정합니다. 렌즈로부터 광선을 출발시켜, focus plane 위의 원하는 픽셀을 향해 그 광선을 보냅니다. 여기서 focus plane은 렌즈로부터 `focal_length` 만큼 떨어져 있고, 3D 공간에서 focus plane에 있는 모든 것들은 완벽하게 초점이 맞습니다.

실제 구현은 뷰포트를 focus plane 위에 위치시키는 방식으로 구현합니다. 지금까지의 내용을 정리하면 다음과 같습니다.

1. focus plane은 카메라가 바라보는 방향에 수직입니다.
2. focus distance는 카메라 중심과 focus plane 사이의 거리입니다.
3. 뷰포트는 focus plane 위에 위치하고, 뷰포트의 중심은 카메라가 바라보는 방향 벡터 축 위에 있습니다.
4. 픽셀 그리드는 뷰포트 안에 위치합니다. 뷰포트는 3D 공간 상에 있습니다.
5. 랜덤 이미지 샘플 위치는 현재 픽셀 위치의 주변 영역에서 선택됩니다.
6. 카메라는 렌즈 위의 랜덤한 점에서 출발하여 현재 이미지 샘플 위치를 통과하도록 광선을 발사합니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.22-cam-film-plane.jpg"></p>

**<p align="center">Figure 22:** _Camera focus plane</p>_

---

## 13.2 Generating Sample Rays
defocus blur가 적용되지 않은 경우, 씬의 모든 광선은 카메라 중심(즉 `lookfrom`)에서 출발합니다. defocus blur를 구현하기 위에서는 카메라 중심 위치를 중심으로 하는 원반이 필요합니다. 원반의 반지름이 증가하면 defocus blur 효과도 더 커집니다. 기존 카메라는 원반 반지름이 0인 경우와 같다고 생각할 수 있습니다. 이 경우에는 defocus blur가 전혀 나타나지 않으므로 모든 광선이 원반 중심(`lookfrom`)에서 출발합니다.

그렇다면 defocus disk는 얼마나 커야 될까요? 원반의 크기가 defocus blur의 세기를 결정하므로, 원반의 크기를 카메라 클래스의 파라미터로 설정할 수 있습니다. 이렇게 원반의 반지름 자체를 카메라 클래스의 파라미터로 설정할 수도 있지만, 그러면 projection distance가 바뀔 때 defocus blur도 달라집니다. 그래서 조금 더 쉬운 방법은 꼭짓점이 뷰포트 중심에 있고 밑면이 defocus disk인 원뿔의 각도를 파라미터로 두는 것입니다. 이렇게 하면 같은 구도에서 focus distance를 바꾸더라도 defocus blur가 더 일관성 있게 적용됩니다.

defocus disk 위의 랜덤한 점들을 선택하기 위한 함수인 `random_in_unit_disk()` 가 필요합니다. 이 함수는 `random_unit_vector()` 함수와 동일한 방식으로 동작합니다. 단지 3차원 대신 2차원으로 동작한다는 것만 다릅니다.

```cpp
...

inline vec3 unit_vector(const vec3& v) {
  return v / v.length();
}

///////////////////////// 추가 //////////////////////////////////////////
inline vec3 random_in_unit_disk() {                                   //
  while (true) {                                                      //
    auto p = vec3(random_double(-1, 1), random_double(-1, 1), 0);     //
    if (p.length_squared() < 1)                                       //
      return p;                                                       //
  }                                                                   //
}                                                                     //
////////////////////////////////////////////////////////////////////////

...
```

**<p align="center">Listing 85:** [vec3.h] _Generate random point inside unit disk</p>_

이제 광선이 defocus disk에서 출발하도록 카메라를 수정하겠습니다.

```cpp
class camera {
  public:
    double aspect_ratio   = 1.0;   // Ratio of image width over height
    int image_width       = 100;   // Rendered image width in pixel count
    int samples_per_pixel = 10;    // Count of random samples for each pixel
    int max_depth         = 10;    // Maximum number of ray bounces into scene

    double vfov     = 90;                 // Vertical view angle (field of view)
    point3 lookfrom = point3(0, 0, 0);    // Point camera is looking from
    point3 lookat   = point3(0, 0, -1);   // Point camera is looking at
    vec3 vup        = vec3(0, 1, 0);      // Camera-relative "up" direction

///////////////////////// 추가 ////////////////////////////////////////////////////////////////////////
    double defocus_angle = 0;   // Variation angle of rays through each pixel                       //
    double focus_dist = 10;     // Distance from camera lookfrom point to plane of perfect focus    //
//////////////////////////////////////////////////////////////////////////////////////////////////////

    ...

  private:
    int    image_height;        // Rendered image height
    double pixel_samples_scale; // Color scale factor for a sum of pixel samples
    point3 center;              // Camera center
    point3 pixel00_loc;         // Location of pixel 0, 0
    vec3   pixel_delta_u;       // Offset to pixel to the right
    vec3   pixel_delta_v;       // Offset to pixel below
    vec3   u, v, w;             // Camera frame basis vectors
///////////////////////// 추가 //////////////////////////////////////////
    vec3   defocus_disk_u;      // Defocus disk horizontal radius     //
    vec3   defocus_disk_v;      // Defocus disk vertical radius       //
////////////////////////////////////////////////////////////////////////

    void initialize() {
      image_height = int(image_width / aspect_ratio);
      image_height = (image_height < 1) ? 1 : image_height;

      pixel_samples_scale = 1.0 / samples_per_pixel;

      center = lookfrom;

      // Determine viewport dimensions.
///////////////////////// 삭제 ////////////////////////////////
//      auto focal_length = (lookfrom - lookat).length();   //
//////////////////////////////////////////////////////////////
      auto theta = degrees_to_radians(vfov);
      auto h = std::tan(theta / 2);
///////////////////////// 수정 ////////////////////////////////
      auto viewport_height = 2 * h * focus_dist;            //
//////////////////////////////////////////////////////////////
      auto viewport_width = viewport_height * (double(image_width) / image_height);

      // Calculate the u, v, w unit basis vectors for the camera coordinate frame.
      w = unit_vector(lookfrom - lookat);
      u = unit_vector(cross(vup, w));
      v = cross(w, u);

      // Calculate the vectors across the horizontal and down the vertical viewport edges.
      vec3 viewport_u = viewport_width * u;     // Vector across viewport horizontal edge
      vec3 viewport_v = viewport_height * -v;   // Vector down viewport vertical edge
      
      // Calculate the horizontal and vertical delta vectors to the next pixel.
      pixel_delta_u = viewport_u / image_width;
      pixel_delta_v = viewport_v / image_height;

      // Calculate the location of the upper left pixel.
///////////////////////// 수정 ////////////////////////////////////////////////////////////////////
      auto viewport_upper_left = center - (focus_dist * w) - viewport_u / 2 - viewport_v / 2;   //
//////////////////////////////////////////////////////////////////////////////////////////////////
      pixel00_loc = viewport_upper_left + 0.5 * (pixel_delta_u + pixel_delta_v);

///////////////////////// 추가 ////////////////////////////////////////////////////////////////////
      // Calculate the camera defocus disk basis vectors.                                       //
      auto defocus_radius = focus_dist * std::tan(degrees_to_radians(defocus_angle / 2));       //
      defocus_disk_u = u * defocus_radius;                                                      //
      defocus_disk_v = v * defocus_radius;                                                      //
//////////////////////////////////////////////////////////////////////////////////////////////////
    }

    ray get_ray(int i, int j) const {
///////////////////////// 수정 ////////////////////////////////////////////////////////////////////
      // Construct a camera ray originating from the defocus disk and directed at a randomly    //
      // sampled point around the pixel location i, j.                                          //
//////////////////////////////////////////////////////////////////////////////////////////////////

      auto offset = sample_square();
      auto pixel_sample = pixel00_loc
                        + ((i + offset.x()) * pixel_delta_u)
                        + ((j + offset.y()) * pixel_delta_v);

///////////////////////// 수정 ////////////////////////////////////////////////////////
      auto ray_origin = (defocus_angle <= 0) ? center : defocus_disk_sample();      //
//////////////////////////////////////////////////////////////////////////////////////
      auto ray_direction = pixel_sample - ray_origin;

      return ray(ray_origin, ray_direction);
    }

    vec3 sample_square() const {
      ...
    }

///////////////////////// 추가 ////////////////////////////////////////////////
    point3 defocus_disk_sample() const {                                    //
      // Returns a random point in the camera defocus disk.                 //
      auto p = random_in_unit_disk();                                       //
      return center + (p[0] * defocus_disk_u) + (p[1] * defocus_disk_v);    //
    }                                                                       //
//////////////////////////////////////////////////////////////////////////////

    color ray_color(const ray& r, int depth, const hittable& world) const {
      ...
    }
};
```

**<p align="center">Listing 86:** [camera.h] _Camera with adjustable depth-of-field</p>_

defocus angle을 크게 설정하여 조리개를 크게 연 효과를 주면

```cpp
int main() {
  ...

  camera cam;

  cam.aspect_ratio      = 16.0 / 9.0;
  cam.image_width       = 400;
  cam.samples_per_pixel = 100;
  cam.max_depth         = 50;

  cam.vfov     = 20;
  cam.lookfrom = point3(-2, 2, 1);
  cam.lookat   = point3(0, 0,-1);
  cam.vup      = vec3(0, 1, 0);

///////////////////////// 추가 ////////////////////
  cam.defocus_angle = 10.0;                     //
  cam.focus_dist    = 3.4;                      //
//////////////////////////////////////////////////

  cam.render(world);
}
```

**<p align="center">Listing 87:** [<span>main.</span>cc] _Scene camera with depth-of-field</p>_

다음과 같은 결과를 얻을 수 있습니다.

<p align="center"><img src="https://raytracing.github.io/images/img-1.22-depth-of-field.png"></p>

**<p align="center">Image 22:** _Spheres with depth-of-field</p>_

---

## 출처

**Ray Tracing in One Weekend - Peter Shirley**
https://raytracing.github.io/books/RayTracingInOneWeekend.html#defocusblur