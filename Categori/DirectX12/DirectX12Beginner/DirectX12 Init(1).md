# DirectX12 초기화(1)
---
Direct3D의 초기화 과정의 첫번째 단계로 윈도우 창 생성 역할을 하는 클래스를 만든다. 

***prac_D3DApp.h***

```c++
#pragma once

#if defined(DEBUG) || defined(_DEBUG)
#define _CRTDBG_MAP_ALLOC	// 메모리 누수 탐지 기능을 사용하기 위해 선언한다.
#include <crtdbg.h>			// 메모리 할당을 추적한다. _CrtDumpMemoryLeaks(), _CrtSetDbgFlag()
#endif

#include "prac_d3dUtil.h"

#pragma comment(lib,"d3dcompiler.lib")
#pragma comment(lib,"D3D12.lib")
#pragma comment(lib, "dxgi.lib")


class D3DApp {
protected:
	D3DApp(HINSTANCE hInstance);
	D3DApp(const D3DApp& rhs) = delete;
	virtual ~D3DApp();
	D3DApp& operator=(const D3DApp& rhs) = delete;

public:
	static D3DApp* GetApp();
	HINSTANCE  AppInst() const;
	HWND	   MainWnd() const;

	int Run();

	virtual bool Initialize();
	virtual LRESULT MsgProc(HWND hWnd, UINT msg, WPARAM wParam, LPARAM lParam);

protected:
	virtual void OnResize();

protected:
	bool InitMainWindow();

protected:

	static D3DApp* mApp;

	HINSTANCE	mhAppInst = nullptr;
	HWND		mhMainWnd = nullptr;

	std::wstring mMainWndCaption = L"d3d App";

	int mClientWidth = 800;
	int mClientHeight = 600;
}; 
```
이 클래스는 단순히 메인이 될 창을 생성하고 창의 핸들을 보관한다.

싱글톤 패턴을 사용하여 언제 어디서든지 객체를 참조할 수 있게 만들어 두었다. 

코드 변경 내용은 다음의 링크에서 확인할 수 있다.

[윈도우 생성 클래스화](https://github.com/IOrez/D3D12Practice/commit/c972dae97dfa4a03f24aeb25af28c88f5b310ab7)


```tip
Direct3D 초기화를 하기위해서는 윈도우 창이 필요하다. 먼저 윈도우 창과 관련된 클래스나 커스텀 함수등을 생성하자.
```
--------------------------------------------------------------
여기서 디버그 관련 헤더(crtdbg.h)를  추가하였다. 이 헤더를 사용한다는 것은 개발자의 의도된 메모리 할당을 디버그에서 추적이 가능하도록 만들겠다는 의미이다. 이 헤더를 사용하기 위해서 _CRTDBG_MAP_ALLOC을 선언한다. 이것을 선언하면 이 헤더에 정의된 _CrtSetDbgFlag를 사용할 수 있다. _CrtSetDbgFlag의 함수 사용은 다음과 같다.

```c++
#if defined(DEBUG) || defined(_DEBUG)
#define _CRTDBG_MAP_ALLOC	
#include <crtdbg.h>			// 메모리 할당을 추적한다. _CrtDumpMemoryLeaks(), _CrtSetDbgFlag()
#endif

#if defined(DEBUG) | defined(_DEBUG)
	_CrtSetDbgFlag(_CRTDBG_ALLOC_MEM_DF | _CRTDBG_LEAK_CHECK_DF);
#endif

```


위와 같이 binary방식의 매개변수를 전달하여 이 함수를 사용할 수 있다.

이 함수는 호출 이후 메모리 누수를 감지를 시작한다.

_CRTDBG_ALLOC_MEM_DF 플래그 값이 디버그모드에서 메모리 할당이 일어날 때 마다 추적한다.

_CRTDBG_LEAK_CHECK_DF 플래그 값이 프로그램이 종료되기 전에

자동으로 _CrtDumpMemoryLeaks() 함수를 호출하여 메모리를 할당한 후 해제 하지 않는 메모리가 있는지 확인한다.

```tip
디버그 관련 내용은 꼭 필요한 것은 아니지만 프로젝트 개발의 밑바탕
이 되는 것이 디버그 관련 클래스, 함수의 제작이라 주위사람들에게 들었다.
이번에는 이 책의 내용 그대로 디버그 기능을 구현할 계획이다.
```