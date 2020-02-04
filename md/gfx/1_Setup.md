[https://github.com/gfx-rs/gfx](https://github.com/gfx-rs/gfx)

gfx를 이용해서 rust로 게임 엔진을 만들어보려고 하는데, 내용이 상당히 방대하고 어려워서 정리가 좀 필요하겠다 싶어 간단하게 공부한 것을 써 본다.

gfx는 Vulkan 스타일의 API를 제공하는 러스트의 저수준 그래픽스 라이브러리다. 내부적으로는 DirectX 11 & 12 / Metal / OpenGL / Vulkan 등의 다양한 백엔드를 제공하며 중간에 추상화 레이어를 하나 둬서 어떤 백엔드를 쓰든 상관없이 Vulkan 스타일의 API를 이용해 그래픽스 프로그래밍을 할 수 있게 해준다.

 gfx 자체가 고성능 + 저수준에서의 자유로운 커스터마이즈를 목적으로 하기 때문에 사용하기가 쉽지 않고, 튜토리얼 등의 문서가 딱히 제대로 되어 있지 않아 처음 진입 장벽이 상당히 높다. 다만 구조 자체가 Vulkan과 굉장히 유사하게 되어 있어 Vulkan에 대한 지식이 있다면 상대적으로 쉽게 시작할 수 있다. vulkan의 경우 [https://vulkan-tutorial.com](https://vulkan-tutorial.com/) 이 사이트가 튜토리얼이 굉장히 친절하게 잘 되어있다.

gfx 사용법은 문서로는 설명되어 있지 않지만 예시 코드가 몇 개 있기 때문에 vulkan 튜토리얼을 참조하면서, gfx 상에서 각 코드가 vulkan의 무엇에 대응되는지 비교하면서 공부하면 그나마 좀 쉽게 배울 수 있다.

## Instance

instance는 gfx 라이브러리와 어플리케이션을 연결하는 장치 정도로 생각하면 된다.

![img](https://github.com/gfx-rs/gfx/raw/master/info/hal.svg?sanitize=true)

gfx는 위 이미지의 Hardware Abstraction Layer(HAL)을 제공함으로써 뒤에서 무슨 그래픽스 라이브러리를 쓰든 상관없이 어플리케이션이 일관된 동작을 할 수 있게 만든다. Backend trait이 이런 추상화된 인터페이스를 가리킨다. gfx-backend-dx11, gfx-backend-dx-12, gfx-backend-vulkan, gfx-backend-metal, gfax-backend-gl 등의 모듈이 이 Backend trait을 구현한 실제 그래픽스 백엔드를 제공한다.

그래서 gfx-hal을 사용할 경우 아래와 같이 적절한 조건부 컴파일 조건들을 설정해두면 백엔드와 무관하게 동작하는 프로그램을 손쉽게 작성할 수 있다.

```rust
#[cfg(feature = "dx11")]
extern crate gfx_backend_dx11 as back;
#[cfg(feature = "dx12")]
extern crate gfx_backend_dx12 as back;
#[cfg(any(feature = "gl", feature = "wgl"))]
extern crate gfx_backend_gl as back;
#[cfg(feature = "metal")]
extern crate gfx_backend_metal as back;
#[cfg(feature = "vulkan")]
extern crate gfx_backend_vulkan as back;

let instance = back::Instance::create("example", 1)
				.expect("Failed to create an instance!");
```

이렇게 할 경우 `cargo run --features=dx11` 등과 같은 커맨드로 사용할 백엔드를 지정해서 컴파일할 수 있다.

그리고 코드에서 보이는 것처럼 인스턴스는 `Instance::create` 함수를 이용해서 생성하는데, 인자로 name 문자열과 version 정수값을 받는다. 이 값들은 vulkan을 backend로 사용할 때 외에는 아무런 역할이 없다. vulkan의 경우 instance를 만들 때 name, version 값을 넘겨줘야해서 이걸 인자로 받는 듯 한데 vulkan에서도 이게 무슨 역할인지 잘 모르겠다. 대충 구분용인 것 같고 적당히 적절한 값 넣어주면 크게 이상은 없어 보인다.



### Rendering Step

인스턴스를 가져오고 나면 vulkan의 경우 대충 아래와 같은 과정을 거쳐 렌더링을 하게 된다.

1. 인스턴스로부터 Physical Device 가져오기. Physical Device는 간단하게 말해 컴퓨터에 실제로 장치된 GPU에 대응된다고 생각하면 된다. 컴퓨터에 장치된 Physical Device들 중에서 해당 Device가 내가 원하는 조건들을 만족하는지 확인해본 후 선택할 수 있다.
2. Physical Device를 선택하고 나면 이 Physical Device로부터 Logical Device를 생성한다. Logical Device는 이름대로 어플리케이션 입장에서 다루게 되는 논리적인 Device다. 실제로 Device와 관련된 모든 작업은 Logical Device를 통해서 이루어진다고 생각하면 된다.
3. Pysical Device로부터 해당 Device에서 사용 가능한 Queue Family 목록을 가져올 수 있다. vulkan은 모든 그리기, 메모리 작업을 비동기적으로 수행하기 때문에 이러한 동작을 집어넣을 Queue가 필요하다. 이런 Command를 넣기 위한 Queue를 Queue Family로부터 할당받을 수 있다. 각각의 Queue Family는 특정한 연산 집합을 제공하는 큐를 할당받을 수 있게 해주기 때문에 목적에 따라 다른 Queue Family를 사용할 수 있다(Graphics, Compute, Memory Transfer 등). 
4. 실제로 렌더링을 하기 위한 Window Surface가 필요하다. vulkan의 경우 GLFW, SDL 등의 다양한 도구를 이용할 수 있다. Window Surface는 실제로 렌더링을 하기 위한 윈도우의 네이티브 핸들에 대한 추상화라고 생각하면 된다. 그리고 추가적으로 Swap Chain이 필요하다. Swap Chain은 Render Target의 집합으로 볼 수 있다. 더블 버퍼링등을 하기 위해(화면에 그려지는 과정이 보이지 않게 하기 위해) 사용한다고 생각하면 된다.
5. Swap Chain에서 가져온 Image에 뭔가 그리기 위해서는 이걸 ImageView와 FrameBuffer로 감싸야 할 필요가 있다. Image View는 Image에서 실제로 사용할 부분(어느 부분에 그리기 작업을 진행할 건지)이 어디인지를 가리키고, frame buffer는 해당 Image View에서 color, depth, stencil 등의 그리기용 버퍼를 가리킨다. Swap Chain은 여러 개의 이미지를 사용하기 때문에 그리기 작업을 할 때 정확히 어떤 이미지에 뭘 그릴 건지를 잘 지정해주어야 한다.
6. vulkan에서 Render Pass는 렌더링 과정에서 어떤 이미지가 이용될지, 어떻게 사용될지, 거기 담긴 내용물이 어떻게 다뤄질지 등을 기술한다. Render Pass는 이미지의 타입이 무엇인지만 기술하며 Frame Buffer가 Render Pass에 실제 이미지를 바인딩한다.
7. vulkan에서 Graphics Pipeline은 Pipeline 오브젝트를 만들어서 설정할 수 있다. 이 오브젝트는 뷰포트 사이즈 같은 설정 가능한 상태를 나타낸다. 어떤 render target을 사용할지 Pipeline에 지정해주어야 하는데, 이때 Render Pass가 이용된다. 
8. vulkan에서 사용하는 대부분의 연산은 queue를 거쳐야 한다. 이러한 연산은 CommandBuffer에 먼저 기록되며, Command Buffer는 Command Pool로부터 할당받을 수 있다. Command Pool은 다시 특정한 Queue Family와 연관되어 있다. 
9. 이제 Command Buffer에 기록된 연산을 Queue에 전달하면 이 연산들이 비동기적으로 그리기 작업을 진행한다. 따라서 이런 연산이 완료될때까지 기다리려면 세마포어 등의 도구를 사용해야 한다.

이제 vulkan에서의 이러한 과정이 gfx 상에서는 어떻게 이루어지는지 살펴보자.

### Adapter & Queue Family

gfx에서는 instance의 `enumerate_adapters`를 이용해서 Adapter 목록을 가져올 수 있다. Adapter는 위에서 언급한 Physical Device + GPU에 대한 메타데이터를 담고 있는 구조체라고 생각하면 된다.

```rust
let instance = back::Instance::create("example", 1)
				.expect("Failed to create an instance!");
let adapters = instance.enumerate_adapters();
```

이제 이렇게 가져온 adapter 목록에서 내가 원하는 조건을 만족하는 adapter를 골라서 선택하면 된다. adapter.info에 여러가지 정보들이 들어가 있는데,  예제 코드에서는 그냥 맨 첫번째 adapter를 골라서 쓴다. 필요할 때 조건을 잘 찾아보고 필터링해서 쓰면 될 것 같다. 어떤 필드들이 있는지는 공식 문서에 잘 정리되어 있으니 그걸 참조하면 된다.

그리고 Adapter 내부의 `queue_families ` 필드를 통해 해당 Adapter에서 사용 가능한 Queue Family 목록을 가져올 수 있다.

이 Queue Family 목록에서 실제로 내가 사용할 목적에 맞는 Queue Family를 가져와서 사용해야 하는데, 그리기를 하려고 하는 경우 이걸 판별하기 위해 Surface를 먼저 만들어 둘 필요가 있다.

```rust
let surface = unsafe {
    instance.create_surface(&window).expect("Failed to create a surface!")
}
```

window의 경우 rust의 cross platform 윈도우 생성 및 관리 라이브러리인 `winit`를 사용하는게 편하다. 해당 라이브러리를 통해 생성한 윈도우를 create_surface의 인자로 넘겨주면 surface 구조체를 얻을 수 있다.

```rust
let adapter = adapters.remove(0);
let family = adapter
            .queue_families
            .iter()
            .find(|family| {
                surface.supports_queue_family(family) &&
                family.queue_type().supports_graphics()
            })
            .unwrap();
```

이렇게 surface 구조체를 얻어오면 위와 같은 과정을 통해 적당한 QueueFamily를 가져올 수 있다. 코드를 보면 이게 무슨 의미인지 명확하다. 사용하고 싶은 QueueFamily는 1. 우리가 그림을 그릴 surface에서 해당 queue family를 지원해야 하며, 2. 그래픽스를 지원하는 타입의 queue family여야 한다. 

### Gpu, Device, Queue

이렇게 사용할 family를 찾고 나면,  physical device로부터 `Gpu` 구조체를 생성할 수 있다. Gpu 구조체는 Logical Device와 Queue Group(하나의 Queue Family로부터 만들어진 모든 큐 목록을 나타냄)을 포함하는 구조체다. physical device 구조체의 `open` 함수를 이용해 Gpu를 만들 수 있다.

```rust
let mut gpu = unsafe {
    adapter
    	.physical_device
    	.open(&[(family, &[1.0])], hal::Features::empty())
    	.unwrap()
};
let queue_group = gpu.queue_groups.pop().unwrap();
let device = gpu.device;
```

`open` 함수의 자세한 인터페이스는 [공식 문서](https://docs.rs/gfx-hal/0.4.1/gfx_hal/adapter/trait.PhysicalDevice.html#tymethod.open) 를 참조. open의 인자로 사용할 queue_family와 각각의 queue_family에 대한 우선순위 값을 지정해줄 수 있다. 우선순위는 0(low) ~ 1(high)사이의 floating 값으로 표현되며 커맨드 버퍼 실행 스케쥴링에 각 큐가 끼치는 영향을 뜻한다.

코드를 보면 queue_group도 그냥 gpu의 queue_groups 목록에서 하나 꺼내서 사용하게 되는데 어차피 사용한 Queue Family가 하나라 여기서 생성된 큐의 모음인 QueueGroup도 하나만 생성되었을 것이며 이걸 그냥 쓰면 되기 때문이다.

```rust
let mut command_pool = unsafe {
    device.create_command_pool(queue_group.family, pool::CommandPoolCreateFlags::empty())
}
```

이제 이 queue group으로부터 command pool을 생성할 수 있다.

이렇게 구성을 하고나면 렌더링을 위한 가장 기초적인 setup은 끝이다. 이제 다음 글에서는 실제로 렌더링을 하기 위해 Queue, Command Buffer, Swap Chain, Image View 등을 사용하는 방법을 살펴 보자.

