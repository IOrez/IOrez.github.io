# DirectX12 초기화(3)
---
Direct3D의 초기화 과정의 세번째 단계로 

(1) 명령 대기열, 명령 할당자, 명령 목록 생성

(2) SwapChain 생성

(3) 서술자 힙 생성

(4) Render Target View, Depth Stencil View 생성

(추가) 뷰포트, 가위 직사각형 생성

를 수행한다.

---
## 1.명령대기열, 명령할당자, 명령목록 생성

DirectX12는 명령대기열(Command Queue)를 참고하여 물체를 그리는 매커니즘을 가지고 있다. 특히 DirectX12는 멀티쓰레드 기능을 대폭 강화하였는데 하나의 명령 대기열에 각각의 쓰레드가 개별로 가지고 있는 명령 할당자와 명령 목록을 통하여 명령을 집어넣을 수 있다. (이번 클론 코딩을 통해 그런 느낌이 물씬난다. 확실하지는 않으니 흘려 들었으면 하는 바람이다.)

명령대기열, 명령할당자, 명령목록은 다음과 같은 코드로 만들어 진다.

```c++
void D3DApp::CreateCommandObjects()
{
	D3D12_COMMAND_QUEUE_DESC queueDesc = {};
	queueDesc.Type = D3D12_COMMAND_LIST_TYPE_DIRECT;
	queueDesc.Flags = D3D12_COMMAND_QUEUE_FLAG_NONE;
	
	ThrowIfFailed(md3dDevice->CreateCommandQueue(&queueDesc, IID_PPV_ARGS(&mCommandQueue)));

	ThrowIfFailed(md3dDevice->CreateCommandAllocator(
		D3D12_COMMAND_LIST_TYPE_DIRECT,
		IID_PPV_ARGS(mDirectCmdListAlloc.GetAddressOf())
	));

	ThrowIfFailed(md3dDevice->CreateCommandList(
		0,
		D3D12_COMMAND_LIST_TYPE_DIRECT,
		mDirectCmdListAlloc.Get(),
		nullptr,
		IID_PPV_ARGS(mCommandList.GetAddressOf())
	));

	mCommandList->Close();
}

```

여기서 이번 프로젝트는 싱글 쓰레드를 사용하기에 명령할당자와 명령목록을 하나씩만 생성하였다는 것을 눈여겨 볼 수 있다. 명령대기열을 사용하는 중간에 참조하는 자원이 변하여 잘못된 랜더링을 할 수 있기 때문에 앞서 울타리를 생성하였다. Flush와 같이 최근 울타리까지 버퍼를 비우는 함수를 만들어 CPU와 GPU사이의 동기화를 이루도록 만든다.

다음은 명령대기열의 최근 울타리에서 명령수행을 멈추는 함수이다.

```c++
void D3DApp::FlushCommandQueue()
{
	mCurrentFence++;

	ThrowIfFailed(mCommandQueue->Signal(mFence.Get(), mCurrentFence));

	if (mFence->GetCompletedValue() < mCurrentFence)
	{
		HANDLE eventHandle = CreateEventEx(nullptr, false, false, EVENT_ALL_ACCESS);

		ThrowIfFailed(mFence->SetEventOnCompletion(mCurrentFence, eventHandle));
		WaitForSingleObject(eventHandle, INFINITE);
		CloseHandle(eventHandle);
	}
}
```

Signal과 Event를 통해 최근 울타리에 도착하였으면 현재 쓰레드의 대기상태를 해제하는 내용을 담고 있다.

이렇게 명령과 관련된 객체들을 생성하였다면 다음은 SwapChain을 생성해야한다.

---
## 2.SwapChain 생성

더블 버퍼링과 관련된 SwapChain은 후면 버퍼와 명령대기열과 연관이 있다. GPU는 후면버퍼에 명령대기열에 있는 명령을 수행하기 때문이다. 이를 생각한다면 SwapChain 생성에 조금 더 쉽게 다가갈 수 있을 것이다.

다음은 SwapChain 생성과 관련된 함수이다.

```c++
void D3DApp::CreateSwapChain()
{
	mSwapChain.Reset();

	DXGI_SWAP_CHAIN_DESC sd;
	sd.BufferDesc.Width = mClientWidth;
	sd.BufferDesc.Height = mClientHeight;
	sd.BufferDesc.RefreshRate.Numerator = 60;
	sd.BufferDesc.RefreshRate.Denominator = 1;
	sd.BufferDesc.Format = mBackBufferFormat;
	sd.BufferDesc.ScanlineOrdering = DXGI_MODE_SCANLINE_ORDER_UNSPECIFIED;
	sd.BufferDesc.Scaling = DXGI_MODE_SCALING_UNSPECIFIED;
	sd.SampleDesc.Count = m4xMsaaQuality ? 4 : 1;
	sd.SampleDesc.Quality = m4xMsaaQuality ? (m4xMsaaQuality - 1) : 0;
	sd.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT;
	sd.BufferCount = SwapChainBufferCount;
	sd.OutputWindow = mhMainWnd;
	sd.Windowed = true;
	sd.SwapEffect = DXGI_SWAP_EFFECT_FLIP_DISCARD;
	sd.Flags = DXGI_SWAP_CHAIN_FLAG_ALLOW_MODE_SWITCH;

	ThrowIfFailed(mdxgiFactory->CreateSwapChain(
		mCommandQueue.Get(),
		&sd,
		mSwapChain.GetAddressOf()
	));

}
```

DirectX12에서 어떤 객체를 생성할때 대부분은 생성정보를 넘겨줘야 한다. SwapChain을 생성할때는 후면버퍼의 정보와 SwapChain의 속성을 알려주면 된다. 이 책에서는 Msaa 기능을 사용하지 않는다. 따라서 샘플링개수는 1, 품질은 0이 된다. 

버퍼 개수는 전방버퍼, 후면버퍼 2개이고 랜더링할 창의 핸들은 앞에서 생성한 윈도우창을 사용할 것이므로 이 내용을 SwapChain의 속성으로 전달한다.

끝에 Factory에서 SwapChain을 생성할 때 명령대기열(CommandQueue)의 주소를 필요로 하는 것을 알 수 있다. 

---
## 3.서술자 힙 생성

랜더링 파이프라인에서 자원을 바인딩할 때 자원 그자체를 묶을 수는 없고 반드시 서술자를 통해서 참조되도록 하여야 한다. 서술자는 자원의 정보를 간략하게 표현하는 자기소개서와 같은 역할이다. GPU는 서술자를 통해 자원을 해석하고 사용한다.

지난번에는 서술자의 크기만 계산하였으나 이번에는 서술자의 크기만큼 메모리를 할당하는 것을 목표로 한다. 서술자 힙은 종류별로 1개씩, 서술자는 자원마다 1개씩 만들면 된다. 여기서는 Rtv(Render Target View)와 Dsv(Depth Stencil View) 총 2종류의 서술자 힙이 필요하고 Rtv에서는 전방 버퍼, 후면 버퍼의 역할을 수행할 2개의 버퍼가 있으므로 2개의 서술자가 깊이-스텐실 버퍼는 1개의 버퍼가 있으므로 1개의 서술자가 필요하다.

다음은 서술자 힙 생성에 관한 함수이다.

```c++
void D3DApp::CreateRtvAndDsvDescriptorHeaps()
{
    D3D12_DESCRIPTOR_HEAP_DESC rtvHeapDesc;
    rtvHeapDesc.NumDescriptors = SwapChainBufferCount;
    rtvHeapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_RTV;
    rtvHeapDesc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_NONE;
	rtvHeapDesc.NodeMask = 0;
    ThrowIfFailed(md3dDevice->CreateDescriptorHeap(
        &rtvHeapDesc, IID_PPV_ARGS(mRtvHeap.GetAddressOf())));


    D3D12_DESCRIPTOR_HEAP_DESC dsvHeapDesc;
    dsvHeapDesc.NumDescriptors = 1;
    dsvHeapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_DSV;
    dsvHeapDesc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_NONE;
	dsvHeapDesc.NodeMask = 0;
    ThrowIfFailed(md3dDevice->CreateDescriptorHeap(
        &dsvHeapDesc, IID_PPV_ARGS(mDsvHeap.GetAddressOf())));
}
```

힙을 생성하였다면 힙에 접근할 수 있는 시작 핸들을 가져올 함수 역시 필요하다. 이 책에서는 다음과 같이 함수를 만들어 시작 핸들을 들고오도록 하였다.

```c++
D3D12_CPU_DESCRIPTOR_HANDLE D3DApp::CurrentBackBufferView()const
{
	return CD3DX12_CPU_DESCRIPTOR_HANDLE(
		mRtvHeap->GetCPUDescriptorHandleForHeapStart(),
		mCurrBackBuffer,
		mRtvDescriptorSize);
}

D3D12_CPU_DESCRIPTOR_HANDLE D3DApp::DepthStencilView()const
{
	return mDsvHeap->GetCPUDescriptorHandleForHeapStart();
}
```

---
## 4.Render Target View, Depth Stencil View 생성

GPU는 자원에 직접 그릴수 없다. 여기서 자원이라면 Buffer 그 자체를 의미한다. 그래서 간접적으로 Buffer의 형태를 본떠 만든 도화지인 View를 만든다. GPU는 View를 통해서 Buffer에 정보를 조작한다. 

이를 위해 View를 생성해야하는데 앞서 말했듯이 View를 만들기 위해서는 View의 본판이 될 Buffer의 정보가 필요하다. 우리는 물체를 그릴 버퍼(2개)와 깊이-스텐실 버퍼 (1개)를 만들었으므로 View 역시 같은 개수로 만들어 준다.

다음은 Render Target View를 만드는 내용이다.

```c++
CD3DX12_CPU_DESCRIPTOR_HANDLE rtvHeapHandle(mRtvHeap->GetCPUDescriptorHandleForHeapStart());
	for (UINT i = 0; i < SwapChainBufferCount; i++)
	{
		ThrowIfFailed(mSwapChain->GetBuffer(i, IID_PPV_ARGS(&mSwapChainBuffer[i])));
		md3dDevice->CreateRenderTargetView(mSwapChainBuffer[i].Get(), nullptr, rtvHeapHandle);
		rtvHeapHandle.Offset(1, mRtvDescriptorSize);
	}
```

눈 여겨 봐야할 점은 CreateRenderTargetView에서 View와 서술자 핸들과 연결시켜 준 부분이다. View는 서술자를 통해 버퍼의 정보를 파악한다.

Depth Stencil View는 Render Target View와는 다르게 접근해야 한다. 깊이-스텐실은 GPU에서만 사용하기 때문에 View를 GPU의 기본메모리에 할당해주는 것이 중요하다. GPU에 자원을 할당하는 방법으로는 CreateComittedResource가 있다. 이 함수를 활용하여 Depth Stencil View를 만든다.

```tip
Depth Stencil View는 최적의 성능을 위해서 기본 힙에 넣어야 한다.!
```

다음은 Depth Stencil View를 만드는 내용이다.

```c++
ThrowIfFailed(md3dDevice->CreateCommittedResource(
        &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT),
		D3D12_HEAP_FLAG_NONE,
        &depthStencilDesc,
		D3D12_RESOURCE_STATE_COMMON,
        &optClear,
        IID_PPV_ARGS(mDepthStencilBuffer.GetAddressOf())));
```

이와 관련된 자세한 내용은 책을 통해 참고하길 바란다.

---
## (추가) 뷰포트, 가위 직사각형 생성

보통 랜더링을 하면 윈도우창의 클라이언트 영역 전체를 사용하는 경우가 대부분이지만 화면 분할과 같은 특수 효과를 주기 위하여 뷰포트의 개념을 한번쯤은 살펴볼 필요가 있다.

뷰포트는 3차원의 장면을 후면 버퍼의 일부를 차지하는 직사각형 영역에 그리는 것을 가능하게 해준 개념이다.

다음의 코드로 뷰포트를 설정할 수 있다.

```c++
	mScreenViewport.TopLeftX = 0;
	mScreenViewport.TopLeftY = 0;
	mScreenViewport.Width    = static_cast<float>(mClientWidth);
	mScreenViewport.Height   = static_cast<float>(mClientHeight);
	mScreenViewport.MinDepth = 0.0f;
	mScreenViewport.MaxDepth = 1.0f;

    mCommandList->RSSetViewports(1, &mScreenViewport);
```

가위 직사각형은 필터와 같은 개념으로 특정 직사각형 영역을 벗어나서 물체를 그리게 된다면 그 벗어난 부분은 픽셀 계산을 수행하지 않도록 만드는 개념이다. 

다음의 코드로 가위 직사각형을 설정할 수 있다.

```c++
    mScissorRect = { 0, 0, mClientWidth/2, mClientHeight/2 };
    mCommandList->RSSetScissorRects(1,&mScissorRect);
```

명령목록을 통해 뷰포트, 가위 직사각형을 설정한 것을 다시한번 보자.! 
