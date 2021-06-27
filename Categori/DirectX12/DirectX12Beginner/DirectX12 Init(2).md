# DirectX12 초기화(2)
---
Direct3D의 초기화 과정의 두번째 단계로 생성된 창에 Direct3D 장치를 부착한다.

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
	bool InitDirect3D();

	void LogAdapters();
	void LogAdapterOutputs(IDXGIAdapter* adapter);
	void LogOutputDisplayModes(IDXGIOutput* output, DXGI_FORMAT format);

protected:

	static D3DApp* mApp;
	
	HINSTANCE	mhAppInst = nullptr;
	HWND		mhMainWnd = nullptr;


	UINT		m4xMsaaQuality = 0;

	Microsoft::WRL::ComPtr<IDXGIFactory4> mdxgiFactory;
	Microsoft::WRL::ComPtr<ID3D12Device> md3dDevice;

	Microsoft::WRL::ComPtr<ID3D12Fence> mFence;



	UINT mRtvDescriptorSize = 0;
	UINT mDsvDescriptorSize = 0;
	UINT mCbvSrvUavDescriptorSize = 0;

	std::wstring mMainWndCaption = L"d3d App";
	DXGI_FORMAT	mBackBufferFormat = DXGI_FORMAT_R8G8B8A8_UNORM;

	int mClientWidth = 800;
	int mClientHeight = 600;
};
```
전체 코드는 다음의 경로에서 확인할 수 있다.

[Direct12 장치 디바이스 생성...](https://github.com/IOrez/D3D12Practice/commit/5d3edbbc0d7a07f5712b663f3247ea46a86a648b#diff-6103f695d77eea2ee27376c3b2b9b2db2d89d677e59648d12368fb2c863a425e)

---
# Direct3D 장치 부착하기 (1) Factory 생성

저번에 클래스화 한 것에 Direct3D 장치를 부착하였다.  눈에 띄는 네임 스페이스가 보인다. Microsoft::WRL은 Windows 런타임 C++ 템플릿 라이브러리를 구성하는 기본 형식을 정의하는 네임스페이스로 Com객체와 연결짓는 인터페이스의 주소를 가리키는 ComPtr을 정의한다. 

장치 생성에 필요한 것은 첫번째로 Factory관련 인터페이스이다. 이 Factory 인터페이스는 지금 현재 컴퓨터에 부착되어 있는 하드웨어 어뎁터(그래픽 카드 연결고리)들을 찾을 수 있다. 여기서 찾은 어뎁터는 Direct3D 장치를 생성하는데 인자로 사용된다. 

어뎁터는 각각 출력포트(출력 장치)의 정보들을 가지고 있다. 만약 출력화면에 문제가 생긴다면 어뎁터를 통하여 출력장치에 문제가 있는지 디버그를 통해 찾아볼 수 있는 기회도 있을 수 있다.

Factory의 생성은 다음과 같다.

```c++
ThrowIfFailed(CreateDXGIFactory1(IID_PPV_ARGS(&mdxgiFactory)));
```

여기서 ThrowIfFailed 메크로 함수는 해당 내용에서 오류가 발생하면 그대로 오류를 던저주는 역할을 한다. 이때 오류는 Direct3D 관련한 오류도 포함된다.

***prac_D3DUtil.h***을 보면 ThrowIfFailed의 정의는 다음과 같다.

```c++
#ifndef ThrowIfFailed
#define ThrowIfFailed(x)										\
{																\
	HRESULT hr__ = (x);										    \
	std::wstring wfn = AnsiToWString(__FILE__);					\
	if(FAILED(hr__)) {throw DxException(hr__,L#x,wfn,__LINE__);}\
}
#endif

```

본론으로 넘어와서 장치를 생성하는 코드는 다음과 같다.

```c++
HRESULT hardwareResult = D3D12CreateDevice(
		nullptr,
		D3D_FEATURE_LEVEL_11_0,
		IID_PPV_ARGS(&md3dDevice));

	if (FAILED(hardwareResult))
	{
		ComPtr<IDXGIAdapter> pWarpAdapter;
		ThrowIfFailed(mdxgiFactory->EnumWarpAdapter(IID_PPV_ARGS(&pWarpAdapter)));

		ThrowIfFailed(D3D12CreateDevice(
			pWarpAdapter.Get(),
			D3D_FEATURE_LEVEL_11_0,
			IID_PPV_ARGS(&md3dDevice)));
	}
```

처음 장치를 생성할 때는 컴퓨터에 설정된 그래픽카드와 연결된 어뎁터를 채용하고 채용할 어뎁터가 없다면 소프트웨어로 구현된 어뎁터로 임시방편으로 채용한 코드이다. 자세한 내용은 이 책의 내용에 서술되어 있다.

--------------------------------------------------------
# Direct3D 장치 부착하기 (2) 울타리 생성, 서술자 크기 확인

CPU와 GPU간의 동기화를 위해 CommandQueue 중간에 울타리(Fence)를 만들 필요가 있다. GPU는 자원을 참조하여 그리는데 그리는 중간에 CPU가 자원의 내용을 덮어쓰면 내용이 변하기 때문에 정확한 출력을 할 수 없다. 그래서 Fence가 필요하다. 

Fence는 특정지점(울타리가 쳐진 곳)까지의 모든 명령을 다 처리할 때까지 CPU를 기다리게 하는 것을 목표로 한다.

다음은 Fence생성에 관한 코드이다.

```c++
ThrowIfFailed(md3dDevice->CreateFence(0, D3D12_FENCE_FLAG_NONE,IID_PPV_ARGS(&mFence))); // CPU. GPU간의 동기화를 위한 Fence 생성
```

Fence생성이 끝났다면 서술자의 크기를 확인해야 한다. 서술자는 GPU마다 다르기 때문에 장치 초기화 단계에서 서술자의 크기를 확인하는 과정을 수행한다.

다음은 서술자의 크기를 확인하는 코드이다.

```c++
// GPU마다 다른 서술자의 크기를 알아낸다.
	mRtvDescriptorSize = md3dDevice->GetDescriptorHandleIncrementSize(D3D12_DESCRIPTOR_HEAP_TYPE_RTV); //RTV 
	mDsvDescriptorSize = md3dDevice->GetDescriptorHandleIncrementSize(D3D12_DESCRIPTOR_HEAP_TYPE_DSV); //DSV
	mCbvSrvUavDescriptorSize = md3dDevice->GetDescriptorHandleIncrementSize(D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV); //CBV_SRV_UAV
```
여기서 RTV, DSV, CBV_SRV_UAV의 서술자는 다음의 역할을 한다.

-RTV : 렌더 대상(Rander Target)관련 자원을 서술
-DSV : 깊이 스텐실(Depth/Stencil)관련 자원을 서술
-CBV_SRV_UAV : 상수 버퍼(Constant Buffer), 셰이더 자원(Shader Resource), 순서없는 접근(Unordered Access View) 관련 서술

각각의 목적에 맞는 서술자를 GetDescriptorHandleIncrementSize 함수를 통하여 크기를 얻을 수 있다.

이 서술자의 크기를 나중 어떻게 사용하는지는 후에 나오는 챕터를 통해 알게 될 것 같다.

```tip
장치를 생성하였다면 울타리 생성과 서술자의 크기를 확인한다.!
```
--------------------------------------------------------------
# Direct3D 장치 부착하기 (3) 4X MSAA 품질 수준 지원 점검

DirectX12는 MSAA(MultiSample Anti-Aliasing)을 지원한다. MSAA는 전방버퍼보다 4배큰 후방버퍼를 사용하여 해상도를 전방버퍼
만큼 줄여서 계단현상을 막는다. 이를 다중표본화 기능이라 말한다.

이 다중표본화 기능의 품질 수준을 장치 초기화 단계에 알아내어 장치가 외곽을 부드럽게 그릴 수 있도록 하여야 한다. 

다음은 해당 그래픽 카드가 지원하는 품질 수준을 알아내는 코드이다.

```c++
D3D12_FEATURE_DATA_MULTISAMPLE_QUALITY_LEVELS msQualityLevels;
	msQualityLevels.Format = mBackBufferFormat;
	msQualityLevels.SampleCount = 4;
	msQualityLevels.Flags = D3D12_MULTISAMPLE_QUALITY_LEVELS_FLAG_NONE;
	msQualityLevels.NumQualityLevels = 0;
	ThrowIfFailed(md3dDevice->CheckFeatureSupport(
		D3D12_FEATURE_MULTISAMPLE_QUALITY_LEVELS,
		&msQualityLevels,
		sizeof(msQualityLevels)));

	m4xMsaaQuality = msQualityLevels.NumQualityLevels;
	assert(m4xMsaaQuality > 0 && "Unexpected MSAA quality level");
```

D3D12_FEATURE_DATA_MULTISAMPLE_QUALITY_LEVELS은 MSAA의 품질 수준을 얻기 위해 제공되어야할 정보를 저장하는 구조체이다. 여기서 후면버퍼의 형식과 몇배의 크기로 그릴 것인지 값을 넣어주고 CheckFeatureSupport함수를 통해 제공한 구조체의 NumQualityLevels의 변수로 품질 수준을 알아낸다.

후면 버퍼의 형식은 DXGI_FORMAT_R8G8B8A8_UNORM 으로 한 픽셀당 32bit로 사용하고 사용목적은 정해지지 않은 형식이다. 

품질 수준을 알아내었으면 다음에 MSAA를 적용하기 위해 보관한다.

```tip
MSAA 품질 수준도 장치 초기화 단계에서 알아내자!
```