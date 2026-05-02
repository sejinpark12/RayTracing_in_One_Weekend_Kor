# 3. The vec3 Class

거의 모든 그래픽스 프로그램은 기하 벡터와 색상을 저장하는 클래스들을 가지고 있습니다. 많은 시스템에서 이 벡터는 4차원(기하에서는 3D position(xyz) + 동차좌표, 색상에서는 RGB + 알파 투명도)으로 표현됩니다. 하지만 여기서는 3차원 좌표만으로도 충분합니다. `vec3` 클래스를 사용하여 색상, 위치, 방향, 오프셋 그리고 그 이외의 모든 것을 표현합니다. 색상에서 위치를 빼는 바보같은 동작을 방지할 수 없다는 이유로 이 방식을 좋아하지 않는 사람들도 있습니다. 타당한 지적이지만, 명백히 틀린 경우가 아니라면 언제나 "더 적은 코드"를 사용하는 쪽을 지향할 것이므로 `vec3` 클래스의 타입 별칭(type alias)을 `point3` 과 `color` 로 선언해 줍니다. 두 타입은 단지 `vec3` 클래스의 타입 별칭이기 때문에, `point3` 을 파라미터로 받는 함수에 `color` 를 입력한다고 해도 경고가 발생하지 않습니다. 따라서 `point3` 과 `color` 를 더하는 것도 방지하지 못합니다. 하지만 이런 타입 별칭은 코드를 좀 더 읽기 쉽고 이해하기 쉽게 만듭니다.

새로 생성한 `vec3.h` 헤더파일에서 `vec3` 클래스를 정의하고 유용한 벡터 유틸리티 함수들도 정의할 것입니다.

```cpp
#ifndef VEC3_H
#define VEC3_H

#include <cmath>
#include <iostream>

class vec3 {
    public:
        double e[3];

        vec3() : e{0, 0, 0} {}
        vec3(double e0, double e1, double e2) : e{e0, e1, e2} {}

        double x() const { return e[0]; }
        double y() const { return e[1]; }
        double z() const { return e[2]; }

        vec3 operator-() const { return vec3(-e[0], -e[1], -e[2]); }
        double operator[](int i) const { return e[i]; }
        double& operator[](int i) { return e[i]; }

        vec3& operator+=(const vec3& v) {
            e[0] += v.e[0];
            e[1] += v.e[1];
            e[2] += v.e[2];
            return *this;
        }

        vec3& operator*=(double t) {
            e[0] *= t;
            e[1] *= t;
            e[2] *= t;
            return *this;
        }

        vec3& operator/=(double t) {
            return *this *= 1/t;
        }

        double length() const {
            return std::sqrt(length_squared());
        }

        double length_squared() const {
            return e[0]*e[0] + e[1]*e[1] + e[2]*e[2];
        }
};

// point3 is just an alias for vec3, but useful for geometric clarity in the code.
using point3 = vec3;

// Vector Utility Functions

inline std::ostream& operator<<(std::ostream& out, const vec3& v) {
    return out << v.e[0] << ' ' << v.e[1] << ' ' << v.e[2];
}

inline vec3 operator+(const vec3& u, const vec3& v) {
    return vec3(u.e[0] + v.e[0], u.e[1] + v.e[1], u.e[2] + v.e[2]);
}

inline vec3 operator-(const vec3& u, const vec3& v) {
    return vec3(u.e[0] - v.e[0], u.e[1] - v.e[1], u.e[2] - v.e[2]);
}

inline vec3 operator*(const vec3& u, const vec3& v) {
    return vec3(u.e[0] * v.e[0], u.e[1] * v.e[1], u.e[2] * v.e[2]);
}

inline vec3 operator*(double t, const vec3& v) {
    return vec3(t * v.e[0], t * v.e[1], t * v.e[2]);
}

inline vec3 operator*(const vec3& v, double t) {
    return t * v;
}

inline vec3 operator/(const vec3& v, double t) {
    return (1/t) * v;
}

inline double dot(const vec3& u, const vec3& v) {
    return u.e[0] * v.e[0]
         + u.e[1] * v.e[1]
         + u.e[2] * v.e[2];
}

inline vec3 cross(const vec3& u, const vec3& v) {
    return vec3(u.e[1] * v.e[2] - u.e[2] * v.e[1],
                u.e[2] * v.e[0] - u.e[0] * v.e[2],
                u.e[0] * v.e[1] - u.e[1] * v.e[0]);
}

inline vec3 unit_vector(const vec3& v) {
    return v / v.length();
}

#endif
```
**<p align="center">Listing 4:** [vec3<span></span>.h] vec3 definitions and helper functions</p>

여기서는 `double` 을 사용하지만, 일부 레이 트레이서에서는 `float` 를 사용하기도 합니다. `double` 은 `float` 보다 높은 정밀도와 넓은 범위를 가지고 있지만, 사이즈도 `float` 의 두 배입니다. 이런 사이즈 증가는 제한적인 메모리 조건에서 프로그래밍할 때(예를 들어 GPU에서 실행되는 셰이더) 중요할 수 있습니다. 둘 중에 어떤 것을 사용해도 괜찮습니다. 마음에 드는 것을 사용하세요.

---

## 3.1 Color Utility Functions

---

`color.h` 헤더 파일을 생성하고, `vec3` 클래스를 사용하여 한 픽셀의 색상을 표준 출력 스트림으로 출력하는 유틸리티 함수를 만들 것입니다.

```cpp
#ifndef COLOR_H
#define COLOR_H

#include "vec3.h"

#include <iostream>

using color = vec3;

void write_color(std::ostream& out, const color& pixel_color) {
    auto r = pixel_color.x();
    auto g = pixel_color.y();
    auto b = pixel_color.z();

    // Translate the [0, 1] component values to the byte range [0, 255].
    int rbyte = int(255.999 * r);
    int gbyte = int(255.999 * g);
    int bbyte = int(255.999 * b);

    // Write out the pixel color components.
    out << rbyte << ' ' << gbyte << ' ' << bbyte << '\n';
}

#endif
```

**<p align="center">Listing 5:** [color<span></span>.h] color utility functions</p>

이제 이 두 가지를 모두 사용하도록 main을 수정합니다.

```cpp
///////////////////////// 추가 ///////////////////////////////
#include "color.h"
#include "vec3.h"
/////////////////////////////////////////////////////////////

#include <iostream>

int main() {

    // Image

    int image_width = 256;
    int image_height = 256;

    // Render

    std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";

    for (int j = 0; j < image_height; j++) {
        std::clog << "\rScanlines remaining: " << (image_height - j) << ' ' << std::flush;
        for (int i = 0; i < image_width; i++) {
            ///////////////////////// 수정 ///////////////////////////////
            auto pixel_color = color(double(i) / (image_width - 1), double(j) / (image_height - 1), 0);
            write_color(std::cout, pixel_color);
            /////////////////////////////////////////////////////////////
        }
    }

    std::clog << "\rDone.                 \n";
}
```

**<p align="center">Listing 6:** [main<span></span>.cc] Final code for the first PPM image</p>

그러면 이전과 동일한 결과 이미지가 출력되어야 합니다.

---

## 출처

**Ray Tracing in One Weekend - Peter Shirley**
https://raytracing.github.io/books/RayTracingInOneWeekend.html#thevec3class
