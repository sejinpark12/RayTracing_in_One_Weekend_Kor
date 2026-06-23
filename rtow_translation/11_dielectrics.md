# 11. Dielectrics
물, 유리, 다이아몬드와 같은 투명한 머티리얼은 유전체(dielectrics)입니다. 광선이 유전체와 충돌하면 광선은 반사 광선과 굴절(투과) 광선으로 나뉩니다. 하지만 여기서는 한 번 충돌할 때마다 산란 광선을 하나만 생성하도록, 반사와 굴절 중 하나를 랜덤하게 선택하는 방식으로 처리할 것입니다.

용어를 잠깐 복습하면, _반사_ 광선은 광선이 표면에 충돌하여 새로운 방향으로 "반사"되어 나가는 광선을 의미합니다.

_굴절_ 광선은 한 머티리얼에서 다른 머티리얼(유리 또는 물)로 들어갈 때 방향이 꺾입니다. 이런 이유로 연필을 물에 일부만 담갔을 때 물속에 있는 부분이 꺾여보이는 것입니다.

굴절 광선이 얼마나 꺾이는지는 머티리얼의 _굴절률_ (_refractive index_)에 의해 결정됩니다. 일반적으로 굴절률은 빛이 진공으로부터 머티리얼로 들어갈 때 얼마나 꺾이는지를 나타낸 값입니다. 유리의 굴절률은 1.5 ~ 1.7 정도이고 다이아몬드의 굴절률은 대략 2.4입니다. 그리고 공기도 1.000293이라는 매우 작은 굴절률을 가지고 있습니다.

투명한 머티리얼이 또 다른 투명한 머티리얼 안에 있는 경우에는, 상대 굴절률로 굴절을 나타낼 수 있습니다. 상대 굴절률은 안쪽 머티리얼의 굴절률을 둘러싸고 있는 머티리얼의 굴절률로 나눈 값입니다. 예를 들어, 물 속에 있는 유리공을 렌더링하고 싶은 경우에는 그 유리공의 상대 굴절률은 1.125가 됩니다. 유리의 굴절률 1.5를 물의 굴절률 1.333으로 나누면 이 값을 구할 수 있습니다.

대부분의 일반적인 머티리얼 굴절률은 인터넷에서 검색하면 쉽게 알 수 있습니다.

## 11.1 Refraction
디버깅하기 가장 어려운 부분은 굴절광입니다. 굴절광이 조금이라도 존재한다면 일단 전부 굴절되도록 처리해 보겠습니다. 여기서는 씬에 두 개의 유리공을 추가했고, 다음과 같은 결과를 확인할 수 있습니다. 아직 이것이 올바른 구현인지 틀린 구현인지는 설명하지 않았지만, 곧 설명할 것입니다!

<p align="center"><img src="https://raytracing.github.io/images/img-1.15-glass-first.png"></p>

**<p align="center">Image 15:** _Glass first</p>_

위의 결과가 맞을까요? 현실에서 유리공은 시각적으로 좀 독특하게 보입니다. 유리공 내부 굴절 때문에 뒤쪽 배경이 상하 반전되어 보여야 하고, 이상한 검은 부분도 없어야 합니다. 하지만 위의 결과는 그렇지 않으므로 틀렸습니다. 이미지 정중앙을 통과하는 광선 하나를 출력해 봤는데, 계산이 명백히 잘못되어 있었습니다. 그런 식으로 대표 광선 하나만 확인해 봐도 디버깅이 되는 경우가 많습니다.

---

## 11.2 Snell's Law
굴절은 스넬의 법칙으로 설명합니다.

$$ \eta \cdot \sin\theta = \eta' \cdot \sin\theta' $$

$\theta$ 와 $\theta'$ 은 광선과 법선 벡터 사이의 각도이고 $\eta$ 와 $\eta'$ 은 굴절률입니다. 그림은 다음과 같습니다.

<p align="center"><img src="https://raytracing.github.io/images/fig-1.17-refraction.jpg"></p>

**<p align="center">Figure 17:** _Ray refraction</p>_

굴절광의 방향을 결정하기 위해서는 $\sin\theta'$ 를 계산해야 합니다.

$$ \sin\theta' = \frac{\eta}{\eta'} \cdot \sin\theta $$

굴절된 표면 안쪽에는 굴절광 $\mathbf{R'}$ 과 법선 벡터 $\mathbf{n'}$ 이 있고, 이 둘 사이의 각은 $\theta'$ 입니다. 여기서 $\mathbf{R'}$ 을 $\mathbf{n'}$ 에 수직인 벡터와 평행인 벡터로 나눌 수 있습니다.

$$ \mathbf{R'} = \mathbf{R'}_{\bot} + \mathbf{R'}_{\parallel} $$

$\mathbf{R'}_{\bot}$ 와 $\mathbf{R'}_{\parallel}$ 에 대해서 정리하면 다음과 같습니다.

$$ \mathbf{R'}_{\bot} = \frac{\eta}{\eta'} (\mathbf{R} + |\mathbf{R}| \cos(\theta) \mathbf{n}) $$
$$ \mathbf{R'}_{\parallel} = -\sqrt{1 - |\mathbf{R'}_{\bot}|^2} \mathbf{n} $$

여러분이 원한다면 위 수식을 직접 증명해보는 것도 좋습니다. 그러나 여기서는 수식이 맞다고 가정하고 계속 진행할 것입니다. 이후 내용을 이해하는 데 이 증명을 알아야 할 필요는 없기 때문입니다.

$\cos\theta$ 를 제외한 우변의 모든 항의 값을 알고 있습니다. 두 벡터의 내적은 두 벡터 사이의 코사인으로 나타낼 수 있다고 알려져 있습니다.

$$ \mathbf{a} \cdot \mathbf{b} = |\mathbf{a}| |\mathbf{b}| \cos\theta $$

$\mathbf{a}$ 와 $\mathbf{b}$ 가 단위 벡터라고 가정한다면 다음과 같습니다.

$$ \mathbf{a} \cdot \mathbf{b} = \cos\theta $$

알고 있는 값들로 $\mathbf{R'}_{\bot}$ 를 다시 정리할 수 있습니다.

$$ \mathbf{R'}_{\bot} =
     \frac{\eta}{\eta'} (\mathbf{R} + (\mathbf{-R} \cdot \mathbf{n}) \mathbf{n}) $$
     
위의 식은
$$\mathbf{R'}_{\bot} = \frac{\eta}{\eta'} \left(\mathbf{R} + |\mathbf{R}| \cos(\theta)\mathbf{n}\right)$$
에서 $|\mathbf{R}|\cos\theta$ 를 $(-\mathbf{R})\cdot \mathbf{n}$ 로 바꾼 것뿐입니다.

$\mathbf{R}$ 는 표면 안쪽으로 향하고 있고, 법선 $\mathbf{n}$은 바깥쪽을 향하므로, 입사각은 보통 $\mathbf{R}$ 자체가 아니라 $-\mathbf{R}$ 와 $\mathbf{n}$ 사이 각도로 정의합니다. 따라서 $\theta$ 는 $-\mathbf{R}$ 과 $\mathbf{n}$ 사이의 각도입니다. 
그래서
$$(-\mathbf{R})\cdot \mathbf{n} = |-\mathbf{R}|\,|\mathbf{n}|\,\cos\theta$$ 입니다.

$|-\mathbf{R}| = |\mathbf{R}|$ 이고 $\mathbf{n}$ 은 단위벡터이므로
$$(-\mathbf{R})\cdot \mathbf{n} = |\mathbf{R}|\cos\theta$$
가 됩니다.

두 수식을 다시 합치면 $\mathbf{R'}$ 를 계산하는 함수를 작성할 수 있습니다.

```cpp
...

inline vec3 reflect(const vec3& v, const vec3& n) {
     return v - 2 * dot(v, n) * n;
}

///////////////////////// 추가 ////////////////////////////////////////////////////////////
inline vec3 refract(const vec3& uv, const vec3& n, double etai_over_etat) {             //
  auto cos_theta = std::fmin(dot(-uv, n), 1.0);                                         //
  vec3 r_out_perp = etai_over_etat * (uv + cos_theta * n);                              //
  vec3 r_out_parallel = -std::sqrt(std::fabs(1.0 - r_out_perp.length_squared())) * n;   //
  return r_out_perp + r_out_parallel;                                                   //
//////////////////////////////////////////////////////////////////////////////////////////
}

```

**<p align="center">Listing 71:** [vec3.h] _Refraction function</p>_

그리고 항상 굴절만 하는 유전체 머티리얼은 다음과 같습니다.

```cpp
...

class metal : public material {
  ...
};

///////////////////////// 추가 ////////////////////////////////////////////////////////////////////
class dielectric : public material {                                                            //
  public:                                                                                       //
    dielectric(double refraction_index) : refraction_index(refraction_index) {}                 //
                                                                                                //
    bool scatter(const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered)    //
    const override {                                                                            //
      attenuation = color(1.0, 1.0, 1.0);                                                       //
      double ri = rec.front_face ? (1.0 / refraction_index) : refraction_index;                 //
                                                                                                //
      vec3 unit_direction = unit_vector(r_in.direction());                                      //
      vec3 refracted = refract(unit_direction, rec.normal, ri);                                 //
                                                                                                //
      scattered = ray(rec.p, refracted);                                                        //
      return true;                                                                              //
    }                                                                                           //
                                                                                                //
  private:                                                                                      //
    // Refractive index in vacuum or air, or the ratio of the material's refractive index over  //
    // the refractive index of the enclosing media                                              //
    double refraction_index;                                                                    //
};                                                                                              //
//////////////////////////////////////////////////////////////////////////////////////////////////
```

**<p align="center">Listing 72:** [material.h] _Dielectric material class that always refracts</p>_

이제 왼쪽 구를 굴절률이 대략 1.5인 유리로 변경하여 굴절을 표현하겠습니다.

```cpp
auto material_ground = make_shared<lambertian>(color(0.8, 0.8, 0.0));
auto material_center = make_shared<lambertian>(color(0.1, 0.2, 0.5));
///////////////////////// 수정 //////////////////////////////////////////
auto material_left = make_shared<dielectric>(1.50);                   //
////////////////////////////////////////////////////////////////////////
auto material_right = make_shared<metal>(color(0.8, 0.6, 0.2), 1.0);
```

**<p align="center">Listing 73:** [main<span></span>.cc] _Changing the left sphere to glass</p>_

다음과 같은 결과를 얻을 수 있습니다.

<p align="center"><img src="https://raytracing.github.io/images/img-1.16-glass-always-refract.png"></p>

**<p align="center">Image 16:** _Glass sphere that always refracts</p>_

---

## 11.3 Total Internal Reflection
굴절에서 실제로 문제가 되는 점 하나는, 어떤 입사각에서는 스넬의 법칙을 만족하는 굴절 방향이 존재하지 않는다는 것입니다. 광선이 높은 굴절률의 매질에서 더 낮은 굴절률의 매질로, 표면을 거의 스치듯한 각도로 들어오는 경우에는 굴절각이 90&deg; 보다 크게 굴절할 수 있습니다. 스넬의 법칙과 $\sin\theta'$ 유도 과정을 다시 보면

$$ \sin\theta' = \frac{\eta}{\eta'} \cdot \sin\theta $$

광선은 유리 안쪽에 있고 바깥쪽이 공기라면 ($\eta = 1.5$ , $\eta' = 1.0$)

$$ \sin\theta' = \frac{1.5}{1.0} \cdot \sin\theta $$

$\sin\theta'$ 의 값은 1보다 클 수 없습니다. 따라서

$$ \frac{1.5}{1.0} \cdot \sin\theta > 1.0 $$

위 식에서는 등식이 더 이상 성립하지 않으므로, 해가 존재할 수 없습니다. 해가 존재하지 않는다면 유리에서 굴절은 불가능하고 그러므로 광선은 반드시 반사만 해야 합니다.

```cpp
if (ri * sin_theta > 1.0) {
  // Must Reflect
  ...
} else {
  // Can Refract
  ...
}
```

**<p align="center">Listing 74:** [material.h] _Determining if the ray can refract</p>_

여기서 모든 빛은 반사되고 실제로 이런 현상은 주로 고체 오브젝트 내부에서 일어나므로 _전반사_ (_total internal reflection_) 라고 부릅니다. 이것이 물속에서 물밖을 바라볼 때, 물과 공기의 경계면이 때때로 완벽한 거울처럼 보이는 이유입니다. 물속에서 수직으로 위를 올려다보면 물밖의 것들이 보이지만, 수면 가까이에서 수면을 비스듬히 바라보면 수면이 거울처럼 보입니다.

삼각함수를 사용하여 `sin_theta` 를 다음과 같이 계산할 수 있습니다.

$$ \sin\theta  = \sqrt{1 - \cos^2\theta} $$ 

$$ \cos\theta = \mathbf{R} \cdot \mathbf{n} $$

```cpp
double cos_theta = std::fmin(dot(-unit_direction, rec.normal), 1.0);
double sin_theta = std::sqrt(1.0 - cos_theta * cos_theta);

if (ri * sin_theta > 1.0) {
  // Must Reflect
  ...
} else {
  // Can Refract
  ...
}
```

**<p align="center">Listing 75:** [material.h] _Determining if the ray can refract</p>_

그리고 굴절이 가능한 경우에는 항상 굴절하는 유전체 머티리얼은 다음과 같습니다.

```cpp
class dielectric : public material {
  public:
    dielectric(double refraction_index) : refraction_index(refraction_index) {}

    bool scatter(const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered)
    const override {
      attenuation = color(1.0, 1.0, 1.0);
      double ri = rec.front_face ? (1.0 / refraction_index) : refraction_index;

      vec3 unit_direction = unit_vector(r_in.direction());
///////////////////////// 수정 ////////////////////////////////////////////////
      double cos_theta = std::fmin(dot(-unit_direction, rec.normal), 1.0);  //
      double sin_theta = std::sqrt(1.0 - cos_theta * cos_theta);            //
                                                                            //
      bool cannot_refract = ri * sin_theta > 1.0;                           //
      vec3 direction;                                                       //
                                                                            //
      if (cannot_refract)                                                   //
        direction = reflect(unit_direction, rec.normal);                    //
      else                                                                  //
        direction = refract(unit_direction, rec.normal, ri);                //
                                                                            //
      scattered = ray(rec.p, direction);                                    //
//////////////////////////////////////////////////////////////////////////////
      return true;
    }

  private:
    // Refractive index in vacuum or air, or the ratio of the material's refractive index over
    // the refractive index of the enclosing media
    double refraction_index;
};
```

**<p align="center">Listing 76:** [material.h] _Dielectric material class with reflection</p>_

감쇠는 항상 1입니다. 즉, 유리 표면은 빛을 전혀 흡수하지 않습니다.

새로 수정한 `dielectric::scatter()` 함수로 이전 씬을 다시 렌더링해 보면 변화가 … 없습니다. 어라?

공기보다 굴절률이 큰 머티리얼의 구에서는 전반사가 발생하는 입사각이 존재하지 않습니다. 광선이 구에 들어갈 때와 구를 빠져나올 때 모두 그렇습니다. 그 이유는 구의 기하형태 때문에 그렇습니다. 구의 표면에 비스듬히 들어오는 광선은 항상 더 작은 각도로 굴절됩니다. 그리고 구 내부에서 공기로 나갈 때는 다시 원래 각도로 굴절됩니다.

어떻게 하면 구에서 전반사를 확인할 수 있을까요? 만약 구의 굴절률이 그 구를 둘러싼 주변 매질의 굴절률보다 더 _작다면_ , 구의 표면에 비스듬한 각도로 광선을 입사했을 때 전반사가 발생합니다. 이렇게 한다면 충분히 전반사를 관찰할 수 있습니다.

월드 전체를 물로 채우고 구의 머티리얼을 공기로 바꿉니다. 물의 굴절률은 대략 1.33이고 공기의 굴절률은 1.00입니다. 즉, 구는 공기 방울이 됩니다! 이를 위해 왼쪽 구 머티리얼의 굴절률을 다음과 같이 바꿔줍니다.

$$\frac{\text{index of refraction of air}}{\text{index of refraction of water}}$$

```cpp
auto material_ground = make_shared<lambertian>(color(0.8, 0.8, 0.0));
auto material_center = make_shared<lambertian>(color(0.1, 0.2, 0.5));
///////////////////////// 수정 //////////////////////////////////////
auto material_left   = make_shared<dielectric>(1.00 / 1.33);      //
////////////////////////////////////////////////////////////////////
auto material_right  = make_shared<metal>(color(0.8, 0.6, 0.2), 1.0);
```

**<p align="center">Listing 77:** [main<span></span>.cc] _Left sphere is an air bubble in water</p>_

그러면 다음과 같이 바뀐 결과를 얻을 수 있습니다.

<p align="center"><img src="https://raytracing.github.io/images/img-1.17-air-bubble-total-reflection.png"></p>

**<p align="center">Image 17:** _Air bubble sometimes refracts, sometimes reflects</p>_

정면에 가깝게 들어오는 광선은 굴절하는 반면, 비스듬히 들어오는 광선은 반사되는 것을 볼 수 있습니다. 왼쪽 구의 오른쪽 부분에 가운데 파란색 구가 반사됩니다.

---

## 11.4 Schlick Approximation
실제 유리는 각도에 따라 반사도가 달라집니다(Fresnel 반사). 창문을 정면이 아니라 비스듬한 각도에서 보면 거울처럼 보이는 현상을 생각해 보세요. 이것을 계산하는 복잡한 식이 있긴 하지만, 거의 대부분의 사람들은 Christophe Schlick이 만든 Schlick Approximation을 사용합니다. 이 근사식은 계산 비용이 적고 놀라울 정도로 정확합니다. 이렇게 해서 완전한 유리 머티리얼이 완성되었습니다.

```cpp
class dielectric : public material {
  public:
    dielectric(double refraction_index) : refraction_index(refraction_index) {}

    bool scatter(const ray& r_in, const hit_record& rec, color& attenuation, ray& scattered)
    const override {
      attenuation = color(1.0, 1.0, 1.0);
      double ri = rec.front_face ? (1.0 / refraction_index) : refraction_index;

      vec3 unit_direction = unit_vector(r_in.direction());
      double cos_theta = std::fmin(dot(-unit_direction, rec.normal), 1.0);
      double sin_theta = std::sqrt(1.0 - cos_theta * cos_theta);

      bool cannot_refract = ri * sin_theta > 1.0;
      vec3 direction;

///////////////////////// 수정 ////////////////////////////////////////////////
      if (cannot_refract || reflectance(cos_theta, ri) > random_double())   //
//////////////////////////////////////////////////////////////////////////////
        direction = reflect(unit_direction, rec.normal);
      else
        direction = refract(unit_direction, rec.normal, ri);

      scattered = ray(rec.p, direction);
      return true;
    }

  private:
    // Refractive index in vacuum or air, or the ratio of the material's refractive index over
    // the refractive index of the enclosing media
    double refraction_index;

///////////////////////// 추가 ////////////////////////////////////////////////
    static double reflectance(double cosine, double refraction_index) {     //
      // Use Schlick's approximation for reflectance.                       //
      auto r0 = (1 - refraction_index) / (1 + refraction_index);            //
      r0 = r0 * r0;                                                         //
      return r0 + (1 - r0) * std::pow((1 - cosine), 5);                     //
    }                                                                       //
//////////////////////////////////////////////////////////////////////////////
};
```

**<p align="center">Listing 78:** [material.h] _Full glass material</p>_

---

## 11.5 Modeling a Hollow Glass Sphere
속이 빈 유리 구를 모델링해 보겠습니다. 이 구는 어느 정도의 두께를 가지고 내부에는 공기로 된 또 다른 구가 있는 형태입니다. 오브젝트를 통과하는 광선의 경로를 생각해 보면, 처음에 바깥 유리 구에 충돌하여 굴절되고, 그다음 안쪽 공기 구에 충돌하여 두 번째 굴절된 다음, 내부 공기층으로 들어가게 됩니다. 그리고 광선이 계속 진행하여 안쪽 공기 구의 안쪽 표면에서 충돌하여 다시 굴절되고, 바깥 유리 구의 안쪽 표면에서 충돌하여 마지막으로 굴절된 다음, 씬의 대기 중으로 빠져나오게 됩니다.

바깥 유리구는 굴절률이 약 1.50인 유리 구로 모델링합니다. 바깥 공기층에서 유리로 들어가는 굴절을 모델링하는 것입니다. 안쪽 공기 구는 약간 다릅니다. _공기 구_ 의 굴절률은 바깥 유리 구 머티리얼로 둘러싸인 상대 굴절률이기 때문입니다. 그러므로 유리에서 내부 공기로 들어가는 굴절을 모델링 합니다.

이것을 지정하는 것은 실제로 간단합니다. 유전체 머티리얼의 `refraction_index` 파라미터를 오브젝트의 굴절률을 그 오브젝트를 감싸고 있는 매질의 굴절률로 나눈 _비율_ 로 생각하면 됩니다. 이 경우에는, 안쪽 구의 공기 굴절률을 둘러싸고 있는 바깥 쪽 구의 유리 굴절률로 나눈 값이 됩니다. 즉, $1.00/1.50 = 0.67$ 입니다.

코드는 다음과 같습니다.

```cpp
...
auto material_ground = make_shared<lambertian>(color(0.8, 0.8, 0.0));
auto material_center = make_shared<lambertian>(color(0.1, 0.2, 0.5));
///////////////////////// 수정 ////////////////////////////////////////////
auto material_left   = make_shared<dielectric>(1.50);                   //
auto material_bubble = make_shared<dielectric>(1.00 / 1.50);            //
//////////////////////////////////////////////////////////////////////////
auto material_right  = make_shared<metal>(color(0.8, 0.6, 0.2), 0.0);

world.add(make_shared<sphere>(point3( 0.0, -100.5, -1.0), 100.0, material_ground));
world.add(make_shared<sphere>(point3( 0.0,    0.0, -1.2),   0.5, material_center));
world.add(make_shared<sphere>(point3(-1.0,    0.0, -1.0),   0.5, material_left));
///////////////////////// 추가 //////////////////////////////////////////////////////////
world.add(make_shared<sphere>(point3(-1.0,    0.0, -1.0),   0.4, material_bubble));   //
////////////////////////////////////////////////////////////////////////////////////////
world.add(make_shared<sphere>(point3( 1.0,    0.0, -1.0),   0.5, material_right));
```

**<p align="center">Listing 79:** [main<span></span>.cc] _Scene with hollow glass sphere</p>_

결과는 다음과 같습니다.

<p align="center"><img src="https://raytracing.github.io/images/img-1.18-glass-hollow.png"></p>

**<p align="center">Image 18:** _A hollow glass sphere</p>_

---

## 출처

**Ray Tracing in One Weekend - Peter Shirley**
https://raytracing.github.io/books/RayTracingInOneWeekend.html#dielectrics