---
layout: post
title: Reducing Reduntant Render State Changes from DirectX 11.0
tags: []
---

I've been experiencing frame rate drops when rendering different meshes using my DirectX 11.0 based framework to debug physics and create testbeds. My GPU is a (very-old but still working) NVIDIA 8400 GS 512 MiB of VRAM. So I've noted that the problem was brute-force setting a bunch of render states before issuing a draw call.

Thankfully in DirectX 11.0 there're functions that allow us set a bunch of resources once if they're stored as resource arrays. Therefore, to tacke this problem, I store a structure called render state structure in my Context_D3D11.cpp file which wraps around a D3D11 (immediate) device context. More specifically, I keep the current and last render state consisting of resource pointer arrays or individual render state variables. Then, just after issuing a drawing call, I can compare if they hold different values. This kind of redundant-state change optimization is also called "low-level state change optimization" because it does not operates in wrapped resources. The example below will hopefully make that clear.

{% highlight cpp %}

struct RenderState
{
	enum Limits
	{
		MaxViewports = D3D11_VIEWPORT_AND_SCISSORRECT_OBJECT_COUNT_PER_PIPELINE,
		MaxRenderTargets = D3D11_SIMULTANEOUS_RENDER_TARGET_COUNT,
		MaxTextures = D3D11_COMMONSHADER_INPUT_RESOURCE_SLOT_COUNT,
		MaxVBuffers = D3D11_IA_VERTEX_INPUT_RESOURCE_SLOT_COUNT,
		MaxCBuffers = D3D11_COMMONSHADER_CONSTANT_BUFFER_API_SLOT_COUNT,
		MaxSamplers = D3D11_COMMONSHADER_SAMPLER_SLOT_COUNT
	};

	// Input Assembler
	ID3D11Buffer* ibuffer;
	UINT ibOffset;
	ID3D11Buffer* vbuffers[MaxVBuffers];
	UINT strides[MaxVBuffers];
	UINT offsets[MaxVBuffers];
	D3D11_PRIMITIVE_TOPOLOGY topology;
	ID3D11InputLayout* inputLayout;

	// Shaders
	ID3D11SamplerState* vsSamplerStates[MaxSamplers];
	ID3D11SamplerState* psSamplerStates[MaxSamplers];
	ID3D11ShaderResourceView* vsShaderViews[MaxTextures];
	ID3D11ShaderResourceView* psShaderViews[MaxTextures];
	ID3D11Buffer* vsCBuffers[MaxCBuffers];
	ID3D11Buffer* psCBuffers[MaxCBuffers];

	// Rasterizer
	D3D11_VIEWPORT viewports[MaxViewports];
	ID3D11RasterizerState* rasterState;

	// Output Merger and Pipeline
	ID3D11PixelShader* ps;
	ID3D11VertexShader* vs;
	ID3D11RenderTargetView* renderTargets[MaxRenderTargets];
	ID3D11DepthStencilView* depthStencilView;
	ID3D11DepthStencilState* depthStencilState;
	UINT stencilRef;
	ID3D11BlendState* blendState;
	FLOAT blendFactors[4];
	UINT blendSampleMask;
};

class Context_D3D11 : public GpuContext
{
public:
	//...
	// The render state below is public and can be acessed everywhere in the graphics engine.
	RenderState m_lastRenderState; 
	RenderState m_renderState;
	//... 
};

{% endhighlight %}

Talking about a particularly example, all the vertex buffers to be rendered are stored in an array into the current render state so I can set a array of vertex buffers at one time using IASetIndexBuffer(...) if the current state is different from the last state. Using the code below, just before rendering a mesh shader part, I can handle the case when a resource is non-sequentially changed.

{% highlight cpp %}

void Context_D3D11::BindRenderState()
{
	const RenderState& curState = m_renderState;
	RenderState& lastState = m_lastRenderState;

#define STATE_CHANGED(m) (curState.m != lastState.m)
#define UPDATE_STATE(m) (lastState.m = curState.m)

	//Input Assembler

	//Vertex Input Layout
	if (STATE_CHANGED(inputLayout))
	{
		m_immediateCtx->IASetInputLayout(curState.inputLayout);
		UPDATE_STATE(inputLayout);
	}

	//Vertex Buffers
	{
		UINT total = 0, index = 0;
		UINT max = RenderState::MaxVBuffers;
		for (UINT i = 0; i < max; ++i)
		{
			if (STATE_CHANGED(vbuffers[i]) || STATE_CHANGED(strides[i]) || STATE_CHANGED(offsets[i]))
			{
				++total;
				UPDATE_STATE(vbuffers[i]);
				UPDATE_STATE(strides[i]);
				UPDATE_STATE(offsets[i]);
			}
			else
			{
				if (total)
				{
					m_immediateCtx->IASetVertexBuffers(index, total, &curState.vbuffers[index], &curState.strides[index], &curState.offsets[index]);
				}
				total = 0;
				index = i + 1;
			}
		}

		if (total)
		{
			m_immediateCtx->IASetVertexBuffers(index, total, &curState.vbuffers[index], &curState.strides[index], &curState.offsets[index]);
		}
	}

	//Index Buffer
	if (STATE_CHANGED(ibuffer))
	{
		m_immediateCtx->IASetIndexBuffer(curState.ibuffer, DXGI_FORMAT_R32_UINT, curState.ibOffset);
		UPDATE_STATE(ibuffer);
	}

	//Primitive Topology
	if (STATE_CHANGED(topology))
	{
		m_immediateCtx->IASetPrimitiveTopology(curState.topology);
		UPDATE_STATE(topology);
	}

	// Shaders
	if (STATE_CHANGED(vs))
	{
		m_immediateCtx->VSSetShader(curState.vs, NULL, 0);
		UPDATE_STATE(vs);
	}

	if (STATE_CHANGED(ps))
	{
		m_immediateCtx->PSSetShader(curState.ps, NULL, 0);
		UPDATE_STATE(ps);
	}

	// Vertex Shader Samplers
	{
		UINT total = 0, index = 0;
		UINT max = RenderState::MaxSamplers;
		for (UINT i = 0; i < max; ++i)
		{
			if (STATE_CHANGED(vsSamplerStates[i]))
			{
				++total;
				UPDATE_STATE(vsSamplerStates[i]);
			}
			else
			{
				if (total)
				{
					m_immediateCtx->VSSetSamplers(index, total, &curState.vsSamplerStates[index]);
				}
				index = i + 1;
				total = 0;
			}
		}
		if (total)
		{
			m_immediateCtx->VSSetSamplers(index, total, &curState.vsSamplerStates[index]);
		}
	}

	// Vertex Shader Textures
	{
		UINT total = 0, index = 0;
		UINT max = RenderState::MaxTextures;
		for (UINT i = 0; i < max; ++i)
		{
			if (STATE_CHANGED(vsShaderViews[i]))
			{
				++total;
				UPDATE_STATE(vsShaderViews[i]);
			}
			else
			{
				if (total)
				{
					m_immediateCtx->VSSetShaderResources(index, total, &curState.vsShaderViews[index]);
				}
				index = i + 1;
				total = 0;
			}
		}
		if (total)
		{
			m_immediateCtx->VSSetShaderResources(index, total, &curState.vsShaderViews[index]);
		}
	}

	// Vertex Shader Constant Buffers
	{
		UINT total = 0, index = 0;
		UINT max = RenderState::MaxCBuffers;
		for (UINT i = 0; i < max; ++i)
		{
			if (STATE_CHANGED(vsCBuffers[i]))
			{
				++total;
				UPDATE_STATE(vsCBuffers[i]);
			}
			else
			{
				if (total)
				{
					m_immediateCtx->VSSetConstantBuffers(index, total, &curState.vsCBuffers[index]);
				}
				index = i + 1;
				total = 0;
			}
		}
		if (total)
		{
			m_immediateCtx->VSSetConstantBuffers(index, total, &curState.vsCBuffers[index]);
		}
	}

	// Pixel Shader Samplers
	{
		UINT total = 0, index = 0;
		UINT max = RenderState::MaxSamplers;
		for (UINT i = 0; i < max; ++i)
		{
			if (STATE_CHANGED(psSamplerStates[i]))
			{
				++total;
				UPDATE_STATE(psSamplerStates[i]);
			}
			else
			{
				if (total)
				{
					m_immediateCtx->PSSetSamplers(index, total, &curState.psSamplerStates[index]);
				}
				index = i + 1;
				total = 0;
			}
		}
		if (total)
		{
			m_immediateCtx->PSSetSamplers(index, total, &curState.psSamplerStates[index]);
		}
	}

	// Pixel Shader Textures
	{
		UINT total = 0, index = 0;
		UINT max = RenderState::MaxTextures;
		for (UINT i = 0; i < max; ++i)
		{
			if (STATE_CHANGED(psShaderViews[i]))
			{
				++total;
				UPDATE_STATE(psShaderViews[i]);
			}
			else
			{
				if (total)
				{
					m_immediateCtx->PSSetShaderResources(index, total, &curState.psShaderViews[index]);
				}
				index = i + 1;
				total = 0;
			}
		}
		if (total)
		{
			m_immediateCtx->PSSetShaderResources(index, total, &curState.psShaderViews[index]);
		}
	}

	// Pixel Shader Constant Buffers
	{
		UINT total = 0, index = 0;
		UINT max = RenderState::MaxCBuffers;
		for (UINT i = 0; i < max; ++i)
		{
			if (STATE_CHANGED(psCBuffers[i]))
			{
				++total;
				UPDATE_STATE(psCBuffers[i]);
			}
			else
			{
				if (total)
				{
					m_immediateCtx->PSSetConstantBuffers(index, total, &curState.psCBuffers[index]);
				}
				index = i + 1;
				total = 0;
			}
		}
		if (total)
		{
			m_immediateCtx->PSSetConstantBuffers(index, total, &curState.psCBuffers[index]);
		}
	}

	// Rasterizer

	// Viewports
	{
		UINT total = 0, index = 0;
		UINT max = RenderState::MaxViewports;
		for (UINT i = 0; i < max; ++i)
		{
			if (STATE_CHANGED(viewports[i]))
			{
				++total;
				UPDATE_STATE(viewports[i]);
			}
			else
			{
				if (total)
				{
					m_immediateCtx->RSSetViewports(total, &curState.viewports[index]);
				}
				total = 0;
				index = i + 1;
			}
		}
		if (total)
		{
			m_immediateCtx->RSSetViewports(total, &curState.viewports[index]);
		}
	}

	// Rasterizer State
	if (STATE_CHANGED(rasterState))
	{
		m_immediateCtx->RSSetState(curState.rasterState);
		UPDATE_STATE(rasterState);
	}

	// Output-Merger

	// Render Targets
	{
		UINT total = 0, index = 0;
		UINT max = RenderState::MaxRenderTargets;
		for (UINT i = 0; i < max; ++i)
		{
			if (STATE_CHANGED(renderTargets[i]))
			{
				++total;
				UPDATE_STATE(renderTargets[i]);
			}
			else
			{
				if (total)
				{
					m_immediateCtx->OMSetRenderTargets(total, &curState.renderTargets[index], curState.depthStencilView);
				}
				index = i + 1;
				total = 0;
			}
		}

		if (total)
		{
			m_immediateCtx->OMSetRenderTargets(total, &curState.renderTargets[index], curState.depthStencilView);
		}
	}

	// Depth/Stencil State
	if (STATE_CHANGED(depthStencilState) || STATE_CHANGED(stencilRef))
	{
		m_immediateCtx->OMSetDepthStencilState(curState.depthStencilState, curState.stencilRef);
		UPDATE_STATE(depthStencilState);
		UPDATE_STATE(stencilRef);
	}

	// Blend
	if (STATE_CHANGED(blendState) ||
		STATE_CHANGED(blendFactors[0]) ||
		STATE_CHANGED(blendFactors[1]) ||
		STATE_CHANGED(blendFactors[2]) ||
		STATE_CHANGED(blendFactors[3]) ||
		STATE_CHANGED(blendSampleMask))
	{

		m_immediateCtx->OMSetBlendState(curState.blendState, curState.blendFactors, curState.blendSampleMask);

		UPDATE_STATE(blendState);
		UPDATE_STATE(blendFactors[0]);
		UPDATE_STATE(blendFactors[1]);
		UPDATE_STATE(blendFactors[2]);
		UPDATE_STATE(blendFactors[3]);
		UPDATE_STATE(blendSampleMask);
	}
}

{% endhighlight %}

Particularly, a vertex buffer is implicitly set in its respective wrapper class, at any time in the engine before issuing a high-level draw call, as in the example below. (Note the use of compile-time polymorphism with the PIMPL pattern (the this pointer as the PIMPL here) to avoid implementing virtual interfaces unecessarely.)

{% highlight cpp %}

void GpuContext::BindVertexBufferCommand(const State* s)
{
	Context_D3D11& self = Downcast<Context_D3D11>(this);
	const BindVertexBuffer& cmd = Downcast<BindVertexBuffer>(s);
	const VertexBuffer_D3D11& vb = Downcast<VertexBuffer_D3D11>(cmd.vb);

	self.m_renderState.strides[cmd.slot] = cmd.stride;
	self.m_renderState.offsets[cmd.slot] = cmd.offset;
	self.m_renderState.vbuffers[cmd.slot] = vb.m_buffer;
}

{% endhighlight %}

The good thing about this logic is that it is applied to all render states because of the nature of the D3D11 API. The downside is that it adds ~4 KiB to the immediate context wrapper - the old memory vs. performance scenario. 

That's the end of the short post. Hope it helps somehow.