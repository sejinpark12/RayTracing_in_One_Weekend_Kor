# 14. Where Next?
## 14.1 A Final Render
이제 이 책의 표지 이미지를 만들어 보겠습니다. 이 이미지에는 수많은 랜덤 구들이 그려집니다.

```cpp
int main() {
  hittable_list world;

///////////////////////// 수정 //////////////////////////////////////////////////////
  auto ground_material = make_shared<lambertian>(color(0.5, 0.5, 0.5));           //
  world.add(make_shared<sphere>(point3(0, -1000, 0), 1000, ground_material));     //
                                                                                  //
  for (int a = -11; a < 11; a++) {                                                //
    for (int b = -11; b < 11; b++) {                                              //
      auto choose_mat = random_double();                                          //
      point3 center(a + 0.9 * random_double(), 0.2, b + 0.9 * random_double());   //
                                                                                  //
      if ((center - point3(4, 0.2, 0)).length() > 0.9) {                          //
        shared_ptr<material> sphere_material;                                     //
                                                                                  //
        if (choose_mat < 0.8) {                                                   //
          // diffuse                                                              //
          auto albedo = color::random() * color::random();                        //
          sphere_material = make_shared<lambertian>(albedo);                      //
          world.add(make_shared<sphere>(center, 0.2, sphere_material));           //
        } else if (choose_mat < 0.95) {                                           //
          // metal                                                                //
          auto albedo = color::random(0.5, 1);                                    //
          auto fuzz = random_double(0, 0.5);                                      //
          sphere_material = make_shared<metal>(albedo, fuzz);                     //
          world.add(make_shared<sphere>(center, 0.2, sphere_material));           //
        } else {                                                                  //
          // glass                                                                //
          sphere_material = make_shared<dielectric>(1.5);                         //
          world.add(make_shared<sphere>(center, 0.2, sphere_material));           //
        }                                                                         //
      }                                                                           //
    }                                                                             //
  }                                                                               //
                                                                                  //
  auto material1 = make_shared<dielectric>(1.5);                                  //
  world.add(make_shared<sphere>(point3(0, 1, 0), 1.0, material1));                //
                                                                                  //
  auto material2 = make_shared<lambertian>(color(0.4, 0.2, 0.1));                 //
  world.add(make_shared<sphere>(point3(-4, 1, 0), 1.0, material2));               //
                                                                                  //
  auto material3 = make_shared<metal>(color(0.7, 0.6, 0.5), 0.0);                 //
  world.add(make_shared<sphere>(point3(4, 1, 0), 1.0, material3));                //
////////////////////////////////////////////////////////////////////////////////////

  camera cam;

///////////////////////// 수정 ////////////////////
  cam.aspect_ratio      = 16.0 / 9.0;           //
  cam.image_width       = 1200;                 //
  cam.samples_per_pixel = 500;                  //
  cam.max_depth         = 50;                   //
                                                //
  cam.vfov     = 20;                            //
  cam.lookfrom = point3(13, 2, 3);              //
  cam.lookat   = point3(0, 0, 0);               //
  cam.vup      = vec3(0, 1, 0);                 //
                                                //
  cam.defocus_angle = 0.6;                      //
  cam.focus_dist    = 10.0;                     //
  ////////////////////////////////////////////////

  cam.render(world);
}
```

**<p align="center">Listing 88:** [<span>main.</span>cc] _Final scene</p>_

(위의 코드는 프로젝트 샘플 코드와 약간 다르다는 점에 주의하세요. 고품질 이미지를 위해서 위의 코드는  `samples_per_pixel` 을 500으로 설정했습니다. 그러므로 이미지를 렌더링하는 데 꽤 많은 시간이 걸릴 것입니다. 하지만 프로젝트 샘플 코드는 개발과 검증 과정에서 렌더링 시간이 너무 길어지지 않도록 `samples_per_pixel` 을 10으로 설정했습니다.)

렌더링 결과는 다음과 같습니다.

<p align="center"><img src="https://raytracing.github.io/images/img-1.23-book1-final.jpg"></p>

**<p align="center">Image 23:** _Final scene</p>_

여기서 눈여겨볼 흥미로운 점은 유리 구 아래에 그림자가 없어서 공중에 떠 있는 것처럼 보인다는 것입니다. 이건 버그가 아닙니다. 실제로 유리 구를 자주 볼 일이 없어서 그렇지만, 실제로도 보면 좀 이상하게 보입니다. 그리고 흐린 날에는 정말로 떠있는 것처럼 보입니다. 유리 구 아래 바닥 부분은 여전히 많은 빛이 도달합니다. 유리 구는 하늘로부터 오는 빛을 차단하는 것이 아니라 굴절시켜 빛의 방향을 바꾸기 때문입니다.

---

## 14.2 Next Steps
멋진 레이 트레이서를 완성했습니다! 이제 무엇을 해야 할까요?

### 14.2.1 Book 2: Ray Tracing: The Next Week
이 시리즈의 두 번째 책은 지금까지 만든 레이 트레이서를 기반으로 합니다. 다음과 같은 새로운 기능들이 추가됩니다.

- Motion blur - 움직이는 오브젝트를 사실적으로 렌더링하는 기능 
- Bounding volume hierarchies - 복잡한 씬의 렌더링 속도를 향상시키는 기법
- Texture maps - 오브젝트 표면에 이미지를 입히는 기능
- Perlin noise - 여러 기법에서 매우 유용하게 사용하는 랜덤 노이즈 생성기
- Quadrilaterals - 구 이외의 도형 렌더링! 이것은 원반, 삼각형, 링 또는 다른 어떤 2D 프리미티브를 구현하는 데 기초가 됩니다.
- Lights - 씬에 광원을 추가하는 기능
- Transforms - 오브젝트 배치, 회전에 유용한 기능
- Volumetric rendering - 연기, 구름, 다른 기체 형태 볼륨을 렌더링하는 기능

### 14.2.2 Book 3: Ray Tracing: The Rest of Your Life
세 번째 책은 두 번째 책의 내용에서 다시 한 번 확장됩니다. 이 책의 많은 부분은 렌더링 이미지 품질과 렌더링 성능 둘 다를 개선하는 데에 관한 내용입니다. 그리고 올바른 광선을 생성하고 적절하게 누적하는 데 초점을 맞춥니다.

이 책은 전문적인 수준의 레이 트레이서를 만드는 데 진지한 관심이 있거나, subsurface scattering 또는 nested dielectrics와 같은 고급 효과 구현의 기초에 관심이 있는 독자들을 위한 책입니다.

### 14.2.3 Other Directions
이 시리즈에서 아직 다루지 않은 기법들을 포함하여, 현재 단계에서의 추가적인 진행 방향들이 많이 있습니다. 예룰 들면 다음과 같은 것들이 있습니다.

**Triangles** — 대부분의 멋진 3D 모델은 삼각형 메쉬 형태로 이루어져 있습니다. 그런데 이런 모델 파일의 입출력은 굉장히 까다로워서, 거의 모두가 기존에 만들어진 라이브러리를 활용합니다. 또한 삼각형이 아주 많은 대규모 _메쉬_ 들을 효과적으로 처리하는 것도 또 다른 어려움입니다.

**Parallelism** — 서로 다른 랜덤 시드를 사용해서 $N$ 개의 코어에서 $N$ 개의 코드 복사본을 실행합니다. 그리고 그 $N$ 개의 실행 결과 평균을 구합니다. 이 평균 계산을 계층적으로 처리할 수도 있습니다. 한 번에 전부 평균을 내는 대신, 둘씩 묶어 평균내서 중간 결과를 만들고, 그 중간 결과들을 다시 둘씩 평균내는 방식입니다. 이 병렬화 방식은 아주 적은 코드만을 추가하여, 수천 개의 코어에서도 잘 확장될 수 있습니다. 

**Shadow Rays** — 광원을 향해 광선을 발사하면, 특정 지점에서 정확히 어떻게 그림자가 지는지 계산할 수 있습니다. 이 기법을 사용하여 날카로운 그림자와 부드러운 그림자를 렌더링할 수 있고, 씬의 사실감을 높일 수 있습니다.

재미있게 즐겨보세요. 그리고 멋진 이미지를 만들면 꼭 보내 주세요!

---

## 출처

**Ray Tracing in One Weekend - Peter Shirley**
https://raytracing.github.io/books/RayTracingInOneWeekend.html#wherenext?