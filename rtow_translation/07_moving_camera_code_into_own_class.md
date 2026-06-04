# 7. Moving Camera Code Into Its Own Class
계속 진행하기 전에, 지금은 카메라와 씬 렌더링 코드를 하나의 새로운 클래스로 정리하기에 좋은 시점입니다. 새로운 클래스의 이름은 `camera` 입니다. camera 클래스는 두 가지의 중요한 역할을 합니다.
1. 광선을 생성하여 월드로 쏘아 보냅니다.
2. 그 광선들의 결과를 사용하여 렌더링 이미지를 생성합니다.

이 리펙토링에서는 기존의 main 프로그램에 있었던 image, camera, render 관련 코드들을 `ray_color` 함수와 함께 모을 것입니다. 새 camera 클래스에서는 public 멤버 함수로 `initialize()` 와 `render()` 를, 그리고 private 멤버 함수로는 `get_ray()` 와 `ray_color()` 를 구현할 것입니다.

궁극적으로, 카메라는 우리가 생각할 수 있는 가장 단순한 사용 패턴을 따르게 될 것입니다. 먼저 기본 생성자로 아무 인자 없이 객체를 생성할 것입니다. 그다음 카메라 객체를 사용하는 코드 쪽에서 단순 대입으로 public 변수들을 수정할 것입니다. 그리고 마지막으로 `initialize()` 함수를 호출함으로써 모든 초기화를 완료합니다. 이 패턴은 카메라 객체를 사용하는 코드 쪽에서 엄청 많은 매개변수를 가진 생성자를 호출하거나, setter 메서드를 여러개 정의하고 호출하는 패턴들 보다도 단순합니다. 대신 카메라 객체를 사용하는 코드 쪽에서 명시적으로 필요한 변수들만 설정하면 되기 때문입니다. 마지막으로, 카메라 객체를 사용하는 코드 쪽에서 `initialize()` 함수를 직접 호출하게 할 수도 있고, 아니면 `render()` 함수가 시작될 때 카메라가 `initialize()` 함수를 자동으로 호출하게 할 수도 있습니다. 여기서는 자동으로 호출되는 방식을 사용하겠습니다.

main에서 카메라를 생성하고 디폴트 값을 설정한 후에, `render()` 함수를 호출할 것입니다. `render()` 함수는 렌더링을 할 수 있도록 카메라 준비를 마친 뒤 렌더링 루프를 실행합니다.

다음은 새 `camera` 클래스의 기본 골격입니다.

```cpp
#ifndef CAMERA_H
#define CAMERA_H

#include "hittable.h"

class camera {
    public:
        /* Public Camera Parameters Here */

        void render(const hittable& world) {
            ...
        }
    
    private:
        /* Private Camera Variables Here */

        void initialize() {
            ...
        }

        color ray_color(const ray& r, const hittable& world) const {
            ...
        }
};

#endif
```

**<p align="center">Listing 37:** [<span>camera</span>.h] _The camera class skeleton</p>_

우선, main.cc에 있던 `ray_color()` 함수를 camera 클래스로 옮기겠습니다.

```cpp
class camera {
    ...

    private:
        ...

        color ray_color(const ray& r, const hittable& world) const {
///////////////////////// 추가 ///////////////////////////////////////////////////////
            hit_record rec;                                                        //
                                                                                   //
            if (world.hit(r, interval(0, infinity), rec)) {                        //
                return 0.5 * (rec.normal + color(1, 1, 1));                        //
            }                                                                      //
                                                                                   //
            vec3 unit_direction = unit_vector(r.direction());                      //
            auto a = 0.5 * (unit_direction.y() + 1.0);                             //
            return (1.0 - a) * color(1.0, 1.0, 1.0) + a * color(0.5, 0.7, 1.0);    //
/////////////////////////////////////////////////////////////////////////////////////
        }
};

#endif
```

**<p align="center">Listing 38:** [<span>camera</span>.h] _The camera::ray\_color function</p>_

이제 main() 함수에 있던 대부분의 코드를 camera 클래스로 옮기겠습니다. main() 함수에는 월드를 구성하는 코드만 남게 됩니다. 아래는 새로 옮긴 코드가 추가된 camera 클래스입니다.

```cpp
class camera {
    public:
///////////////////////// 추가 //////////////////////////////////////////////////////////////////////////
        double aspect_ratio = 1.0; // Ratio of image width over height                                //
        int    image_width  = 100; // Rendered image width in pixel count                             //
                                                                                                      //
        void render(const hittable& world) {                                                          //
            initialize();                                                                             //
                                                                                                      //
            std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";                   //
                                                                                                      //
            for (int j = 0; j < image_height; j++) {                                                  //
                std::clog << "\rScanlines remaining: " << (image_height - j) << ' ' << std::flush;    //
                for (int i = 0; i < image_width; i++) {                                               //
                    auto pixel_center = pixel00_loc + (i * pixel_delta_u) + (j * pixel_delta_v);      //
                    auto ray_direction = pixel_center - center;                                       //
                    ray r(center, ray_direction);                                                     //
                                                                                                      //
                    color pixel_color = ray_color(r, world);                                          //
                    write_color(std::cout, pixel_color);                                              //
                }                                                                                     //
            }                                                                                         //
                                                                                                      //
            std::clog << "\rDone.                 \n";                                                //
        }                                                                                             //
////////////////////////////////////////////////////////////////////////////////////////////////////////
        
    private:
///////////////////////// 추가 //////////////////////////////////////////////////////////////////////////
        int    image_height;     // Rendered image height                                             //
        point3 center;           // Camera center                                                     //
        point3 pixel00_loc;      // Location of pixel 0, 0                                            //
        vec3   pixel_delta_u;    // Offset to pixel to the right                                      //
        vec3   pixel_delta_v;    // Offset to pixel below                                             //
                                                                                                      //
        void initialize() {                                                                           //
            image_height = int(image_width / aspect_ratio);                                           //
            image_height = (image_height < 1) ? 1 : image_height;                                     //
                                                                                                      //
            center = point3(0, 0, 0);                                                                 //
                                                                                                      //
            // Determine viewport dimensions.                                                         //
            auto focal_length = 1.0;                                                                  //
            auto viewport_height = 2.0;                                                               //
            auto viewport_width = viewport_height * (double(image_width) / image_height);             //
                                                                                                      //
            // Calculate the vectors across the horizontal and down the vertical viewport edges.      //
            auto viewport_u = vec3(viewport_width, 0, 0);                                             //
            auto viewport_v = vec3(0, -viewport_height, 0);                                           //
                                                                                                      //
            // Calculate the horizontal and vertical delta vectors from pixel to pixel.               //
            pixel_delta_u = viewport_u / image_width;                                                 //
            pixel_delta_v = viewport_v / image_height;                                                //
                                                                                                      //
            // Calculate the location of the upper left pixel.                                        //
            auto viewport_upper_left =                                                                //
                center - vec3(0, 0, focal_length) - viewport_u / 2 - viewport_v / 2;                  //
            pixel00_loc = viewport_upper_left + 0.5 * (pixel_delta_u + pixel_delta_v);                //
        }                                                                                             //
////////////////////////////////////////////////////////////////////////////////////////////////////////

        color ray_color(const ray& r, const hittable& world) const {
            ...
        }
};

#endif
```

**<p align="center">Listing 39:** [<span>camera</span>.h] _The working camera class</p>_

그리고 아래는 훨씬 간결해진 main입니다.

```cpp
#include "rtweekend.h"

///////////////////////// 추가 ///////////////////////////////
#include "camera.h"                                        //
/////////////////////////////////////////////////////////////
#include "hittable.h"
#include "hittable_list.h"
#include "sphere.h"

///////////////////////// 삭제 ///////////////////////////////
// color ray_color(const ray& r, const hittable& world) {  //
//     ...                                                 //
// }                                                       //
/////////////////////////////////////////////////////////////

int main() {
///////////////////////// 수정 //////////////////////////////////////
    hittable_list world;                                          //
                                                                  //
    world.add(make_shared<sphere>(point3(0, 0, -1), 0.5));        //
    world.add(make_shared<sphere>(point3(0, -100.5, -1), 100));   //
                                                                  //
    camera cam;                                                   //
                                                                  //
    cam.aspect_ratio = 16.0 / 9.0;                                //
    cam.image_width = 400;                                        //
                                                                  //
    cam.render(world);                                            //
////////////////////////////////////////////////////////////////////
}

```

**<p align="center">Listing 40:** [<span>main</span>.cc] _The new main, using the new camera</p>_

새로 리펙토링한 프로그램을 실행하여도 이전과 같은 렌더링 이미지가 출력되어야 합니다.

---

## 출처

**Ray Tracing in One Weekend - Peter Shirley**
https://raytracing.github.io/books/RayTracingInOneWeekend.html#movingcameracodeintoitsownclass