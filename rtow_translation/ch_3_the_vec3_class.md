> **이 글은 Peter Shirley의 [Ray Tracing in One Weekend](https://raytracing.github.io/books/RayTracingInOneWeekend.html)를 번역한 것입니다.
> Ray Tracing in One Weekend를 공부하면서 다시 한번 복습하는 느낌으로 번역을 해보려고 합니다. 영어가 서툴러 번역이 잘못되었을 수도 있으므로 잘못된 부분을 발견하신다면 지적해 주시면 감사하겠습니다.**

거의 모든 그래픽스 프로그램은 기하학적 벡터와 색상을 저장하는 클래스를 가지고 있습니다. 많은 시스템에서 이 벡터들은 4차원(xyz + 동차좌표 또는 RGB + 투명도)입니다. 우리가 만들려고 하는 것은 3차원 좌표계로 충분합니다. `vec3` 클래스를 사용하여 색상, 위치, 방향, 오프셋 그리고 그 이외의 모든 것을 나타냅니다. 위치에 색상을 더하는 바보같은 동작을 방지할 수 없다는 이유로 이 방식을 싫어하는 사람들도 있습니다. 타당한 지적이지만, 명백히 틀린 경우가 아니라면 이 강좌에서는 항상 "더 적은 코드"를 지향할 것이므로 `vec3` 클래스의 타입 별칭(type alias)을 `point3`과 `color`로 선언해 줍니다. 두 타입은 단지 `vec3` 클래스의 타입 별칭이기 때문에, `point3`을 받는 함수에 `color`를 입력한다고 해도 경고가 발생하지 않습니다. 의도와 사용을 분명히 하기 위해서만 두 타입 별칭을 사용합니다.

## 3.1 Variables and Methods

---

`vec3` 클래스의 상단입니다.

```cpp
#ifndef VEC3_H
#define VEC3_H

#include <cmath>
#include <iostream>

using std::sqrt;

class vec3 {
    public:
        vec3() : e{0, 0, 0} {}
        vec3(double e0, double e1, double e2) : e{e0, e1, e2} {}

        double x() const { return e[0]; }
        double y() const { return e[1]; }
        double z() const { return e[2]; }

        vec3 operator-() const { return vec3(-e[0], -e[1], -e[2]); }
        double operator[](int i) const { return e[i]; }
        double& operator[](int i) { return e[i]; }

        vec3& operator+=(const vec3 &v) {
            e[0] += v.e[0];
            e[1] += v.e[1];
            e[2] += v.e[2];
            return *this;
        }

        vec3& operator*=(const double t) {
            e[0] *= t;
            e[1] *= t;
            e[2] *= t;
            return *this;
        }

        vec3& operator/=(const double t) {
            return *this *= 1 / t;
        }

        double length() const {
            return sqrt(length_squared());
        }

        double length_squared() const {
            return e[0] * e[0] + e[1] * e[1] + e[2] * e[2];
        }

    public:
        double e[3];
};

// Type aliases for vec3
using point3 = vec3;    // 3D point
using color = vec3;     // RGB color

#endif
```

**<p align="center">Listing 4:** [vec3<span></span>.h] vec3 class</p>

여기서는 `double`형을 사용했지만, 몇몇 레이 트레이서는 `float`형을 사용합니다. 어느 쪽이든 괜찮습니다. - 여러분의 취향대로 하십시오.

---

## 3.2 vec3 Utility Functions

---

vec3.h 헤더 파일의 두 번째 부분에는 벡터 유틸리티 함수들이 포함되어 있습니다.

```cpp
// vec3 Uitility Functions

inline std::ostream& operator<<(std::ostream &out, const vec3 &v) {
    return out << v.e[0] << ' ' << v.e[1] << ' ' << v.e[2];
}

inline vec3 operator+(const vec3 &u, const vec3 &v) {
    return vec3(u.e[0] + v.e[0], u.e[1] + v.e[1], u.e[2] + v.e[2]);
}

inline vec3 operator-(const vec3 &u, const vec3 &v) {
    return vec3(u.e[0] - v.e[0], u.e[1] - v.e[1], u.e[2] - v.e[2]);
}

inline vec3 operator*(const vec3 &u, const vec3 &v) {
    return vec3(u.e[0] * v.e[0], u.e[1] * v.e[1], u.e[2] * v.e[2]);
}

inline vec3 operator*(double t, const vec3 &v) {
    return vec3(t * v.e[0], t * v.e[1], t * v.e[2]);
}

inline vec3 operator*(const vec3 &v, double t) {
    return t * v;
}

inline vec3 operator/(vec3 v, double t) {
    return (1 / t) * v;
}

inline double dot(const vec3 &u, const vec3 &v) {
    return u.e[0] * v.e[0]
        + u.e[1] * v.e[1]
        + u.e[2] * v.e[2];
}

inline vec3 cross(const vec3 &u, const vec3 &v) {
    return vec3(u.e[1] * v.e[2] - u.e[2] * v.e[1],
                u.e[2] * v.e[0] - u.e[0] * v.e[2],
                u.e[0] * v.e[1] - u.e[1] * v.e[0]);
}

inline vec3 unit_vector(vec3 v) {
    return v / v.length();
}
```

**<p align="center">Listing 5:** [vec3<span></span>.h] vec3 utility functions</p>

---

## 3.3 Color Utility Functions

---

새로 만든 `vec3` 클래스를 사용하여, 한 픽셀의 색상을 표준 출력 스트림으로 출력하는 유틸리티 함수를 만들 것입니다.

```cpp
#ifndef COLOR_H
#define COLOR_H

#include "vec3.h"

#include <iostream>

void write_color(std::ostream &out, color pixel_color) {
    // Write the translated [0, 255] value of each color component.
    out << static_cast<int>(255.999 * pixel_color.x()) << ' '
        << static_cast<int>(255.999 * pixel_color.y()) << ' '
        << static_cast<int>(255.999 * pixel_color.z()) << '\n';
}

#endif
```

**<p align="center">Listing 6:** [color<span></span>.h] color utility functions</p>

이제 위의 헤더를 추가하여 main을 수정합니다.

```cpp
#include "color.h"  // 추가
#include "vec3.h"   // 추가

#include <iostream>

int main() {

    // Image

    const int image_width = 256;
    const int image_height = 256;

    // Render

    std::cout << "P3\n" << image_width << ' ' << image_height << "\n255\n";

    for (int j = image_height - 1; j >= 0; --j) {
        std::cerr << "\rScanlines remaining: " << j << ' ' << std::flush;
        for (int i = 0; i < image_width; ++i) {
            color pixel_color(double(i) / (image_width - 1), double(j) / (image_height - 1), 0.25); // 추가
            write_color(std::cout, pixel_color);    // 추가
        }
    }

    std::cerr << "\nDone.\n";
}
```

**<p align="center">Listing 7:** [main<span></span>.cc] Final code for the first PPM image</p>

---

## 출처

**Ray Tracing in One Weekend - Peter Shirley**
https://raytracing.github.io/books/RayTracingInOneWeekend.html#thevec3class
