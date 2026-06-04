# 1. Overview
저는 여러 해 동안 많은 그래픽스 강의를 해왔습니다. 제 강의에서는 종종 레이 트레이싱을 다룹니다. 모든 코드를 직접 작성해야 하긴 하지만 별도의 API 없이 멋진 이미지를 얻을 수 있기 때문입니다. 멋진 프로그램을 가능한 한 빠르게 만들 수 있도록 제 강의 노트를 다시 정리하기로 했습니다. 여기에서 모든 기능을 갖춘 레이 트레이서를 구현하진 않을 것입니다. 하지만 레이 트레이싱을 영화의 핵심 요소로 만든 간접 조명(Indirect lighting) 기능은 확실히 포함할 것입니다. 다음 과정을 따라가세요. 더 깊이 파고들고 싶을 만큼 흥미를 느꼈다면, 여기서 만든 레이 트레이서 구조는 더 큰 규모의 레이 트레이서로 확장하는데 도움이 될 것입니다.
</br>

**레이 트레이싱**(Ray Tracing)은 여러 가지 의미를 가질 수 있습니다. 이 책에서 설명할 것은 엄밀히 말하면 **패스 트레이서**(Path Tracer)이며, 꽤 범용적인 형태의 패스 트레이서입니다. 코드는 매우 단순하지만(복잡한 계산은 컴퓨터에게 맡기세요!) 여러분이 만들 이미지를 본다면 매우 만족스러울 것입니다.
</br>

레이 트레이서를 만드는 과정을 단계적으로 설명할 것입니다. 그리고 몇 가지 디버깅 팁도 함께 설명하겠습니다. 이 과정을 끝까지 진행한다면 멋진 이미지를 만들어 내는 레이 트레이서를 완성할 수 있습니다. 주말 안에 끝낼 수 있겠지만, 더 오래 걸린다고 해도 걱정하지 마세요. C++를 사용하여 코드를 작성할 것이지만, 반드시 따라야 할 필요는 없습니다. 그러나 C++는 빠르고 포팅이 쉬우며 대부분의 영화와 비디오 게임 렌더러가 C++로 작성되었기 때문에 C++로 코드를 작성하는 것을 추천합니다. 대부분의 C++ 최신 기능을 사용하진 않을 것이지만, 상속(Inheritance)과 연산자 오버로딩(Operator Overloading)은 레이 트레이서를 구현하는데 매우 유용합니다.
</br>

> 이 책의 전체 소스코드는 온라인으로 제공되지 않습니다. 하지만 이 코드들은 실제로 동작하는 코드이며 `vec3` 클래스의 몇 가지 간단한 연산자를 제외하고 모두 이 과정을 따라가면서 확인할 수 있습니다. 직접 코드를 타이핑하는 것이 학습을 하는데 중요하다고 생각합니다. 하지만 코드가 제공되는 경우에는 저도 직접 타이핑하지 않고 사용합니다. 코드가 제공되지 않을 때만 제가 위에서 말한 것을 실천하는 셈입니다. 그러니 코드를 달라고 하지 마세요!
</br>

위의 문단을 그대로 남겨 둔 이유는, 제 생각이 180도 완전히 바뀌었다는 점이 재밌기 때문입니다. 코드를 직접 타이핑한 몇몇 독자들은 눈에 잘 띄지 않는 오류를 겪었는데, 결국 제 코드와 비교해 보면서 그 오류를 찾을 수 있었습니다. 그러므로 먼저 코드를 직접 타이핑해 보세요. 그리고 만약 여러분의 코드에 문제가 생긴다면 [RayTracing project](https://github.com/RayTracing/raytracing.github.io/) GitHub 레포지토리에 제공된 각 과정의 전체 코드를 참고하세요.
</br>

구현 코드에 대한 참고 사항 - 다음 목표를 중점으로 코드가 구현됩니다.
- 개념 구현에 초점을 맞춥니다.
- 가능한 한 단순하게 C++를 사용합니다. 프로그래밍 스타일은 C 스타일에 매우 가깝지만, 코드 사용성과 이해를 위해 최신 기능을 활용합니다.
- 초기 원본 책의 코딩 스타일을 가능한 한 지켜서 코드의 일관성을 유지합니다.
- 코드 한 줄의 길이를 최대 96자로 제한하여, 코드 베이스와 책 코드 예시의 한 줄의 길이가 동일하도록 합니다.
</br>

따라서 코드는 기본 구현이며, 독자들이 직접 고도화할 수 있도록 많은 부분들을 남겨두었습니다. 코드를 최적화하고 최신으로 개선하는 방법은 아주 다양하지만 여기서는 단순한 해결책을 우선으로 합니다.
</br>

이 책에서는 여러분이 어느 정도 벡터를 다루는데(내적과 벡터합 같은) 익숙하다고 가정합니다. 만약 벡터에 대해 잘 모른다면 약간의 복습이 필요합니다. 복습이 필요하거나 처음 배운다면 아래의 자료를 참고하세요.
</br>

- [The Graphics Codex](http://graphicscodex.com/)
- [Fundamentals of Computer Graphics, Fourth Edition](https://www.amazon.com/gp/product/1482229390/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=1482229390&linkCode=as2&tag=inonwe09-20&linkId=FYWORJLWAJOLB3L5)
- [Computer Graphics: Principles and Practice (3rd Edition)](https://www.amazon.com/gp/product/0321399528/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=0321399528&linkCode=as2&tag=inonwe09-20&linkId=HQRNNI5TVG2IVRMT)

</br>

이 프로젝트의 정보, Github 레포지토리, 디렉토리 구조, 빌드/실행 방법, 컨트리뷰션 방법에 대해서는 프로젝트의 [README](https://github.com/RayTracing/raytracing.github.io/blob/release/README.md) 파일을 참고하세요.
</br>

추가적인 프로젝트 관련 자료는 [Further Reading](https://github.com/RayTracing/raytracing.github.io/wiki/Further-Readings) 위키 페이지를 참고하세요.
</br>

이 책은 브라우저에서 바로 프린트하더라도 읽기 좋도록 구성되어 있습니다. 또한 [Assets](https://github.com/RayTracing/raytracing.github.io/releases/) 색션에서 각 release마다 PDF 파일도 제공하고 있습니다.
</br>

저희와 연락을 원하신다면 부담없이 아래의 주소로 이메일을 보내 주세요.
- Peter Shirley, ptrshrl@gmail.com
- Steve Hollasch, steve@hollasch.net
- Trevor David Black, trevordblack@trevord.black
</br>

마지막으로, 구현에 문제가 발생했거나, 질문이 있거나, 여러분의 아이디어나 작업물을 공유하고 싶다면 [GitHub Discussions](https://github.com/RayTracing/raytracing.github.io/discussions/) 포럼을 참고하세요.
</br>

이 프로젝트에 도움을 주신 모든 분들에게 감사드립니다. [acknowledgments](https://raytracing.github.io/books/RayTracingInOneWeekend.html#acknowledgments)에서 그분들의 이름을 찾을 수 있습니다.

이제 시작합니다!
</br>

---

## 출처

**Ray Tracing in One Weekend - Peter Shirley**
https://raytracing.github.io/books/RayTracingInOneWeekend.html#overview
