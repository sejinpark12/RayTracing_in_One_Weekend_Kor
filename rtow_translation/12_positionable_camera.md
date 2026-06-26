# 12. Positionable Camera
카메라는 유전체와 마찬가지로 디버깅하기가 까다롭습니다. 따라서 저는 항상 카메라를 단계적으로 개발합니다. 먼저 조절 가능한 field of view( _fov_ )를 구현하겠습니다. fov는 렌더링 이미지의 한쪽 가장자리에서 반대쪽 가장자리까지의 시야각을 의미합니다. 우리의 렌더링 이미지는 정사각형이 아니기 때문에, 수평 fov와 수직 fov가 서로 다릅니다. 저는 항상 수직 fov를 사용합니다. 또한 fov를 도(degree) 단위로 지정한 다음에 생성자 내부에서 라디안으로 변환합니다. 하지만 이건 어디까지나 저의 개인적인 취향입니다.

## 12.1 Camera Viewing Geometry
먼저, 광선은 원점에서 출발하여 $z = -1$ 평면으로 향하도록 두겠습니다. $z = -1$ 평면 대신에 $z = -2$ 평면이나 다른 평면을 사용해도 됩니다. 단, 원점과 평면 사이의 거리에 맞춰서 같은 비율로 $h$ 를 조절해야 합니다. 다음과 같이 설정합니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.18-cam-view-geom.jpg"></p>

**<p align="center">Figure 18:** _Camera viewing geometry (from the side)</p>_

여기서 $h = \tan(\frac{\theta}{2})$ 임을 알 수 있습니다. 카메라 코드는 다음과 같습니다.

```cpp
class camera {
  public:
    double aspect_ratio      = 1.0;   // Ratio of image width over height
    int    image_width       = 100;   // Rendered image width in pixel count
    int    samples_per_pixel = 10;    // Count of random samples for each pixel
    int    max_depth         = 10;    // Maximum number of ray bounces into scene

///////////////////////// 추가 ////////////////////////////////////
    double vfov = 90; // Vertical view angle (field of view)    //
//////////////////////////////////////////////////////////////////

    void render(const hittable& world) {
      ...
    }

  private:
    ...

    void initialize() {
      image_height = int(image_width / aspect_ratio);
      image_height = (image_height < 1) ? 1 : image_height;

      pixel_samples_scale = 1.0 / samples_per_pixel;

      center = point3(0, 0, 0);

      // Determine viewport dimensions.
      auto focal_length = 1.0;
///////////////////////// 추가 ////////////////////////////////////
      auto theta = degrees_to_radians(vfov);                    //
      auto h = std::tan(theta / 2);                             //
      auto viewport_height = 2 * h * focal_length;              //
//////////////////////////////////////////////////////////////////
      auto viewport_width = viewport_height * (double(image_width) / image_height);

      // Calculate the vectors across the horizontal and down the vertical viewport edges.
      auto viewport_u = vec3(viewport_width, 0, 0);
      auto viewport_v = vec3(0, -viewport_height, 0);

      // Calculate the horizontal and vertical delta vectors from pixel to pixel.
      pixel_delta_u = viewport_u / image_width;
      pixel_delta_v = viewport_v / image_height;

      // Calculate the location of the upper left pixel.
      auto viewport_upper_left =
        center - vec3(0, 0, focal_length) - viewport_u / 2 - viewport_v / 2;
      pixel00_loc = viewport_upper_left + 0.5 * (pixel_delta_u + pixel_delta_v);
    }

    ...
};
```

**<p align="center">Listing 80:** [camera.h] _Camera with adjustable field-of-view (fov)</p>_

두 개의 구가 붙어있는 간단한 씬으로 90° field of view가 어떻게 적용되는지 테스트 해보겠습니다.

```cpp
int main() {
  hittable_list world;

///////////////////////// 수정 ////////////////////////////////////////////////
  auto R = std::cos(pi / 4);                                                //
                                                                            //
  auto material_left  = make_shared<lambertian>(color(0, 0, 1));            //
  auto material_right = make_shared<lambertian>(color(1, 0, 0));            //
                                                                            //
  world.add(make_shared<sphere>(point3(-R, 0, -1), R, material_left));      //
  world.add(make_shared<sphere>(point3( R, 0, -1), R, material_right));     //
//////////////////////////////////////////////////////////////////////////////  
  
  camera cam;

  cam.aspect_ratio      = 16.0 / 9.0;
  cam.image_width       = 400;
  cam.samples_per_pixel = 100;
  cam.max_depth         = 50;

///////////////////////// 추가 //////////////////////
  cam.vfov = 90;                                  //
////////////////////////////////////////////////////

  cam.render(world);
}
```

**<p align="center">Listing 81:** [<span>main.</span>cc] _Scene with wide-angle camera</p>_

렌더링 결과는 다음과 같습니다.

<p align="center"><img src="https://raytracing.github.io/images/img-1.19-wide-view.png"></p>

**<p align="center">Image 19:** _A wide-angle view</p>_

---

## 12.2 Positioning and Orienting the Camera
임의의 시점을 구하기 위해서 필요한 점들의 이름을 먼저 지정하겠습니다. 카메라가 위치한 점을 _lookfrom_, 카메라가 바라보는 점을 _lookat_ 이라고 하겠습니다. 나중에 원한다면, 바라보는 점 대신에 바라보는 방향을 정의할 수도 있습니다.

또한 카메라의 롤(roll) 다시 말해, 좌우 기울기를 지정할 방법이 필요합니다. 이 동작은 lookat-lookfrom 축 중심의 회전을 의미합니다. 다른 방식으로 생각해보면 `lookfrom` 과 `lookat` 을 고정한 상태 즉, 코를 축으로 하여 머리를 좌우로 기울이는 것을 의미합니다. 마지막으로 카메라 롤을 정의하기 위해서는 lookfrom, lookat 뿐만 아니라, 카메라의 "up" 벡터도 지정해야 합니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.19-cam-view-dir.jpg"></p>

**<p align="center">Figure 19:** _Camera view direction</p>_

카메라가 바라보는 방향과 평행하지만 않는다면 어떤 벡터든지 up 벡터로 지정할 수 있습니다. 카메라 기준의 up 벡터를 구하기 위해서 카메라가 바라보는 방향에 수직인 평면 위로 up 벡터를 투영합니다. 여기서는 일반적인 관례에 따라 "view up" ( _vup_ ) 벡터라고 하겠습니다. 몇 번의 외적과 벡터 정규화를 거쳐서 카메라의 방향을 표현할 수 있는 완전한 정규직교기저 $(u,v,w)$ 를 구합니다. $u$ 는 카메라의 오른쪽 방향을 가리키는 단위 벡터이고, $v$ 는 카메라의 위쪽 방향을 가리키는 단위 벡터이고, $w$ 는 카메라가 바라보는 방향의 반대 방향을 가리키는 단위벡터입니다. 그리고 카메라 중심이 카메라 좌표계의 원점이 됩니다. $w$ 가 카메라가 바라보는 방향의 반대 방향을 가리키는 이유는 오른손 좌표계를 사용하기 때문입니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.20-cam-view-up.jpg"></p>

**<p align="center">Figure 20:** _Camera view up direction</p>_

이전의 고정 카메라가 $-Z$ 방향을 바라봤듯이, 임의 시점 카메라는 $-w$ 방향을 바라봅니다. 그리고 vup 벡터를 지정하기 위해 월드 up 벡터 $(0,1,0)$ 을 사용할 수 있지만 반드시 그럴 필요는 없습니다. 다만 이렇게 하는 것이 편리하고, 극단적인 카메라 각도로 실험하기 전까지는 자연스럽게 카메라를 수평으로 유지할 수 있습니다.

```cpp
class camera {
  public:
    double aspect_ratio      = 1.0;   // Ratio of image width over height
    int    image_width       = 100;   // Rendered image width in pixel count
    int    samples_per_pixel = 10;    // Count of random samples for each pixel
    int    max_depth         = 10;    // Maximum number of ray bounces into scene

    double vfov     = 90;               // Vertical view angle (field of view)
///////////////////////// 추가 ////////////////////////////////////////////////
    point3 lookfrom = point3(0, 0, 0);  // Point camera is looking from     //
    point3 lookat   = point3(0, 0, -1); // Point camera is looking at       //
    vec3 vup        = vec3(0, 1, 0);    // Camera-relative "up" direction   //
//////////////////////////////////////////////////////////////////////////////

    ...

  private:
    int    image_height;        // Rendered image height
    double pixel_samples_scale; // Color scale factor for a sum of pixel samples
    point3 center;              // Camera center
    point3 pixel00_loc;         // Location of pixel 0, 0
    vec3   pixel_delta_u;       // Offset to pixel to the right
    vec3   pixel_delta_v;       // Offset to pixel below
///////////////////////// 추가 //////////////////////////////////////
    vec3   u, v, w;             // Camera frame basis vectors     //
////////////////////////////////////////////////////////////////////

    void initialize() {
      image_height = int(image_width / aspect_ratio);
      image_height = (image_height < 1) ? 1 : image_height;

      pixel_samples_scale = 1.0 / samples_per_pixel;

///////////////////////// 수정 //////////////////////////////////////
      center = lookfrom;                                          //
////////////////////////////////////////////////////////////////////

      // Determine viewport dimensions.
///////////////////////// 수정 //////////////////////////////////////
      auto focal_length = (lookfrom - lookat).length();           //
////////////////////////////////////////////////////////////////////
      auto theta = degrees_to_radians(vfov);
      auto h = std::tan(theta / 2);
      auto viewport_height = 2 * h * focal_length;
      auto viewport_width = viewport_height * (double(image_width) / image_height);

///////////////////////// 추가 ////////////////////////////////////////////////////////
      // Calculate the u, v, w unit basis vectors for the camera coordinate frame.  //
      w = unit_vector(lookfrom - lookat);                                           //
      u = unit_vector(cross(vup, w));                                               //
      v = cross(w, u);                                                              //
//////////////////////////////////////////////////////////////////////////////////////

      // Calculate the vectors across the horizontal and down the vertical viewport edges.
///////////////////////// 수정 ////////////////////////////////////////////////////////////////
      vec3 viewport_u = viewport_width * u;   // Vector across viewport horizontal edge     //
      vec3 viewport_v = viewport_height * -v; // Vector down viewport vertical edge         //
//////////////////////////////////////////////////////////////////////////////////////////////

      // Calculate the horizontal and vertical delta vectors from pixel to pixel.
      pixel_delta_u = viewport_u / image_width;
      pixel_delta_v = viewport_v / image_height;

      // Calculate the location of the upper left pixel.
///////////////////////// 수정 //////////////////////////////////////////////////////////////////////
      auto viewport_upper_left = center - (focal_length * w) - viewport_u / 2 - viewport_v / 2;   //
////////////////////////////////////////////////////////////////////////////////////////////////////
      pixel00_loc = viewport_upper_left + 0.5 * (pixel_delta_u + pixel_delta_v);
    }

    ...

  private:
};
```

**<p align="center">Listing 82:** [camera.h] _Positionable and orientable camera</p>_

다시 원래의 씬으로 되돌리고, 새로운 시점을 적용하겠습니다.

```cpp
int main() {
  hittable_list world;

///////////////////////// 수정 //////////////////////////////////////////////////////////////
  auto material_ground = make_shared<lambertian>(color(0.8, 0.8, 0.0));                   //
  auto material_center = make_shared<lambertian>(color(0.1, 0.2, 0.5));                   //
  auto material_left   = make_shared<dielectric>(1.50);                                   //
  auto material_bubble = make_shared<dielectric>(1.00 / 1.50);                            //
  auto material_right  = make_shared<metal>(color(0.8, 0.6, 0.2), 1.0);                   //
                                                                                          //
  world.add(make_shared<sphere>(point3( 0.0, -100.5, -1.0), 100.0, material_ground));     //
  world.add(make_shared<sphere>(point3( 0.0,    0.0, -1.2),   0.5, material_center));     //
  world.add(make_shared<sphere>(point3(-1.0,    0.0, -1.0),   0.5, material_left));       //
  world.add(make_shared<sphere>(point3(-1.0,    0.0, -1.0),   0.4, material_bubble));     //
  world.add(make_shared<sphere>(point3( 1.0,    0.0, -1.0),   0.5, material_right));      //
////////////////////////////////////////////////////////////////////////////////////////////

  camera cam;

  cam.aspect_ratio      = 16.0 / 9.0;
  cam.image_width       = 400;
  cam.samples_per_pixel = 100;
  cam.max_depth         = 50;

  cam.vfov     = 90;
///////////////////////// 추가 //////////////////////////
  cam.lookfrom = point3(-2, 2, 1);                    //
  cam.lookat   = point3(0, 0, -1);                    //
  cam.vup      = vec3(0, 1, 0);                       //
////////////////////////////////////////////////////////

  cam.render(world);
}
```

**<p align="center">Listing 83:** [<span>main.</span>cc] _Scene with alternate viewpoint</p>_

아래의 이미지를 얻을 수 있습니다.

<p align="center"><img src="https://raytracing.github.io/images/img-1.20-view-distant.png"></p>

**<p align="center">Image 20:** _A distant view</p>_

field of view를 변경할 수 있습니다.

```cpp
///////////////////////// 수정 //////////////////////////
  cam.vfov = 20;                                      //
////////////////////////////////////////////////////////
```

**<p align="center">Listing 84:** [<span>main.</span>cc] _Change field of view</p>_

아래의 이미지를 얻을 수 있습니다.

<p align="center"><img src="https://raytracing.github.io/images/img-1.21-view-zoom.png"></p>

**<p align="center">Image 21:** _Zooming in</p>_

---

## 출처

**Ray Tracing in One Weekend - Peter Shirley**
https://raytracing.github.io/books/RayTracingInOneWeekend.html#positionablecamera
