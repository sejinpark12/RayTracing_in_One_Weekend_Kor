> **이 글은 Peter Shirley의 [Ray Tracing in One Weekend](https://raytracing.github.io/books/RayTracingInOneWeekend.html)를 번역한 것입니다.
> Ray Tracing in One Weekend를 공부하면서 다시 한번 복습하는 느낌으로 번역을 해보려고 합니다. 영어가 서툴러 번역이 잘못되었을 수도 있으므로 잘못된 부분을 발견하신다면 지적해 주시면 감사하겠습니다.**

저는 수년 동안 많은 그래픽스 강의를 해왔습니다. 모든 코드를 작성해야 하지만 API 없이 멋진 이미지를 얻을 수 있기 때문에 제 강의에서는 종종 레이 트레이싱을 다룹니다. 가능한 한 빨리 멋진 프로그램을 만들 수 있도록 이 책에서 저의 강의 노트를 정리할 생각입니다. 여기에서 완전한 레이 트레이서를 구현하지는 않을 것입니다. 하지만 우리가 구현할 레이 트레이서는 레이 트레이싱 기술을 영화의 주요 요소로 만든 간접 조명(Indirect lighting) 처리 기능을 가지고 있습니다. 다음 과정을 따라가십시오. 만약 흥미가 생기고 계속하고 싶다면, 여러분이 만든 레이 트레이서의 구조는 더 큰 규모의 레이 트레이서로 확장하기에 좋을 것입니다.
</br>

**레이 트레이싱**(Ray Tracing)은 많은 것을 의미할 수 있습니다. 제가 설명할 것은 엄밀히 말하면 **패스 트레이서**(Path Tracer)이며 상당히 일반적인 것입니다. 코드는 매우 간단하지만(컴퓨터가 일하게 하세요!) 여러분이 만들 수 있는 이미지를 본다면 매우 기뻐할 것입니다.
</br>

저는 몇 가지 디버깅 팁과 함께 레이 트레이서 작성에 대해 설명할 것입니다. 마지막에 여러분은 멋진 이미지를 생성하는 레이 트레이서를 갖게 될 것입니다. 주말 안에 이것들을 끝낼 수 있어야 합니다. 더 오래 걸린다고 해도 걱정하지 마십시오. 저는 C++로 코드를 작성할 것이지만, 반드시 따라야 할 필요는 없습니다. 그러나 C++는 빠르고 포팅이 가능하고 대부분의 영화와 비디오 게임 렌더러가 C++로 작성되었기 때문에 C++로 작성하는 것을 추천합니다. 대부분의 C++ 최신 기능을 사용하지 않을 것이지만, 상속(Inharitance)과 연산자 오버로딩(Operator Overloading)은 레이 트레이서를 구현하는데 매우 유용합니다. 온라인 상에서 코드를 제공하지 않습니다. 하지만 코드는 진짜이며 vec3 클레스의 몇 가지 간단한 연산자를 제외하고 모두 볼 수 있습니다. 저는 학습을 위해 직접 코드를 타이핑하는 것을 믿습니다. 하지만 코드를 이용할 수 있을 경우에 이 방법을 사용합니다. 그래서 코드를 이용할 수 없는 경우에는 제가 설명한 것을 연습합니다. 묻지 마세요!
</br>

제가 했던 것과 다른 것도 재밌기 때문에 마지막 부분을 남겨두었습니다. 몇몇 독자들은 저희가 코드를 비교했을 때 도움이 되는 미묘한 에러들을 발견했습니다. 그러므로 코드를 타이핑하십시오. 하지만 제 코드를 확인하고 싶다면 여기에 있습니다.

- https://github.com/RayTracing/raytracing.github.io/
  </br>

어느 정도 벡터를 다루는데 익숙(내적과 벡터합 같은)하다고 가정합니다. 벡터에 대해 잘 모른다면 약간의 복습이 필요합니다. 복습이 필요하거나 처음 배운다면 아래를 확인하십시오.

- [Fundamentals of Computer Graphics, Fourth Edition](https://www.amazon.com/gp/product/1482229390/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=1482229390&linkCode=as2&tag=inonwe09-20&linkId=FYWORJLWAJOLB3L5)
- [Computer Graphics: Principles and Practice (3rd Edition)](https://www.amazon.com/gp/product/0321399528/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=0321399528&linkCode=as2&tag=inonwe09-20&linkId=HQRNNI5TVG2IVRMT)
- [The Graphics Codex](http://graphicscodex.com/)
  </br>

문제가 발생하거나 누군가에게 보여주고 싶은 멋진 일을 했다면 다음 주소로 이메일을 보내주십시오.

- ptrshrl@gmail.com
  </br>

추가 자료 및 [블로그](https://in1weekend.blogspot.com/)를 포함하여 이 책과 관련된 사이트를 유지 관리할 것입니다.
</br>

이 프로젝트를 위해 도와주신 모든분들에게 감사드립니다. [감사의 말](https://raytracing.github.io/books/RayTracingInOneWeekend.html#acknowledgments)에서 그분들을 찾을 수 있습니다.

시작합시다!
</br>

--

## 출처

**Ray Tracing in One Weekend - Peter Shirley**
https://raytracing.github.io/books/RayTracingInOneWeekend.html#overview
