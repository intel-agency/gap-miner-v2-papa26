# **Unsloth, Ornith, & Representation Engineering on AMD RDNA2 (RX 6700 XT)**

## **1\. Hardware Sizing & Throughput Strategy (RX 6700 XT 12GB)**

### **Architecture Profile**

* **GPU**: AMD Radeon RX 6700 XT  
* **Architecture**: RDNA 2 / Navi 22 (gfx1031)  
* **VRAM**: 12 GB GDDR6 (192-bit bus, ![][image1] theoretical bandwidth)  
* **Compute Units**: 40 CUs, 2560 stream processors

### **Model Sizing Comparison**

| Model | Variant | Quantization | Size on Disk | Active VRAM (Weights \+ 8K KV) | Expected Throughput | Verdict on RX 6700 XT |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| **Ornith-1.0 / 1.5** | **9B Dense** | UD-Q4\_K\_XL / Q4\_K\_M | ![][image2] | ![][image2] | **75 – 120+ tok/s** | **Optimal Target**. Fully resides in VRAM with room for large contexts. |
| **Ornith-1.0 / 1.5** | **9B Dense** | Q8\_0 | ![][image2] | ![][image3] | **45 – 65 tok/s** | Fits with 4K context; tight headroom for long agent rollouts. |
| **Ornith-1.0 / 1.5** | **35B MoE** | UD-IQ3\_XXS | ![][image3] | ![][image3] | **12 – 22 tok/s** | Exceeds 12GB. Requires PCIe layer offload to system RAM; memory-bus throttled. |
| **Ornith-1.0 / 1.5** | **35B MoE** | UD-Q4\_K\_M | ![][image3] | ![][image3] | **5 – 10 tok/s** | Heavy CPU/RAM offloading; impractical for agentic coding loops. |

**Recommended Target**: **Ornith-1.0-9B or Ornith-1.5-9B in 4-bit quantization** (UD-Q4\_K\_XL or Q4\_K\_M). Keeping the entire model graph and KV cache inside the 12GB GDDR6 frame buffer maximizes prefill and autoregressive token generation.

## **2\. AMD Environment & ROCm Configuration (gfx1031)**

The RX 6700 XT uses the gfx1031 target. Standard ROCm releases primarily package precompiled binaries for gfx1030 (RX 6800/6900). To ensure the runtime dispatches compute kernels rather than falling back to host CPU emulation, export the ROCm ISA override in your environment.

### **Environment Setup (\~/.bashrc or shell profile)**

\# Force HIP/ROCm to target the compatible RDNA2 binary ISA  
export HSA\_OVERRIDE\_GFX\_VERSION=10.3.0

\# Prevent GPU/CPU agent indexing conflicts (use HIP index exclusively)  
export HIP\_VISIBLE\_DEVICES=0  
unset ROCR\_VISIBLE\_DEVICES

\# Memory allocation optimizations  
export PYTORCH\_HIP\_ALLOC\_CONF=garbage\_collection\_threshold:0.8,max\_split\_size\_mb:128

Verify your device visibility:

rocminfo | grep \-E "Name:|gfx"  
python3 \-c "import torch; print('ROCm available:', torch.cuda.is\_available(), '| Device:', torch.cuda.get\_device\_name(0))"

## **3\. Unsloth CLI: Studio & Inference Start**

Unsloth includes a CLI suite for both local browser-based management (unsloth studio) and agent-facing inference endpoints (unsloth start).

### **Installation**

\# Create and activate an isolated environment  
python3 \-m venv \~/unsloth-env  
source \~/unsloth-env/bin/activate

\# Install Unsloth with ROCm/PyTorch dependencies  
pip install \--upgrade pip  
pip install "unsloth\[cu121-ampere-torch240\]" \--extra-index-url https://download.pytorch.org/whl/rocm6.1

*(Alternatively, use the official installer script: curl* \-fsSL *https://unsloth.ai/install.sh | sh)*

### **Running Unsloth Studio**

Unsloth Studio launches a local web UI for model evaluation, dataset management, parameter checking, and quantization export.

\# Run locally on port 8888  
unsloth studio \-H 127.0.0\.1 \-p 8888

\# Launch headless with an environment password  
UNSLOTH\_STUDIO\_PASSWORD="your-strong-password" unsloth studio \-H 0\.0.0.0 \-p 8888

\# Launch with an encrypted Cloudflare tunnel for external access  
unsloth studio \--secure

Navigate to http://127.0.0\.1:8888 in your browser to inspect model weights, quantizations, and system memory allocations.

### **Serving Models via unsloth start**

unsloth start runs an OpenAI- and Anthropic-compatible API server backed by optimized local inference engines (such as llama.cpp HIP kernels), tailored for connecting agentic tools (Claude Code, OpenCode, Promptfoo) directly.

\# Serve Ornith 9B Q4 directly  
unsloth start \\  
  \--model unsloth/Ornith-1.0-9B-GGUF:UD-Q4\_K\_XL \\  
  \--port 8000 \\  
  \--ctx-size 16384 \\  
  \--gpu-layers 99

Test the local endpoint:

curl http://localhost:8000/v1/chat/completions \\  
  \-H "Content-Type: application/json" \\  
  \-d '{  
    "model": "Ornith-1.0-9B",  
    "messages": \[  
      {"role": "user", "content": "Write a Python function to check for palindromes."}  
    \],  
    "temperature": 0.6  
  }'

## **4\. Representation Engineering & Abliteration**

### **Is Ornith a Good Candidate for Abliteration?**

**No.** Ornith is fundamentally an **agentic coding model**, post-trained with reinforcement learning specifically to learn scaffold orchestration, multi-file code editing, and execution verification.

1. **Absence of Preachy Conversational Refusal**: Ornith's alignment objectives focus on code correctness, context-window tool-call parsing, and task completion, not conversational safety guardrails.  
2. **High Risk of Capability Degradation**: Abliterating an agentic model often degrades structured syntax tokens, tool-call formatting (\<tool\_call\>), and chain-of-thought scratchpads (\<think\>...\<\\think\>).

### **What Models Are Ideal for Abliteration?**

Abliteration relies on models where **safety alignment is mediated by a distinct, low-rank linear direction in intermediate activation space**. The best candidates are instruction-tuned conversational models:

| Target Model | Parameter Size | Characteristics for Abliteration |
| :---- | :---- | :---- |
| **Meta Llama 3.1 / 3.2 Instruct** | 8B / 3B | **Gold Standard**. Has an exceptionally isolated, single refusal direction located in layers ![][image4] through ![][image5]. Retains near ![][image6] benchmark performance after orthogonal projection. |
| **Qwen 2.5 / 3 Instruct** | 7B / 14B | Very clean latent representation. Exhibits clear separation between harmless and sensitive prompt activations in residual streams. |
| **Mistral 7B Instruct v0.3** | 7B | Compact, well-behaved residual geometry; highly responsive to difference-of-means ablation. |

## **5\. Mathematical Mechanics of Refusal Ablation**

Refusal behavior in aligned language models is typically localized within the residual stream across intermediate layers:

![][image7]Where ![][image8] represents the hidden activation vector at layer ![][image9].

### **Step 1: Activation Collection & Difference-of-Means**

Given a dataset of ![][image10] harmless prompts ![][image11] and ![][image12] harmful/refusal-inducing prompts ![][image13]:

1. Perform forward passes and extract hidden states at token position ![][image14](usually the final prompt token or first generated token) for intermediate layers![][image15]. 2\. Compute the mean activation centroids: $$\\vec{\\mu}\_{\\This request was blocked by Gemini's filters. They can occasionally trigger by mistake on safe coding, security, or biology-related queries. Please try rephrasing your prompt. You can [send feedback](https://ai.google.dev/gemini-api/docs/troubleshooting#file-bug) or read more about [our policies here](https://policies.google.com/terms/generative-ai/use-policy).

[image1]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAGwAAAAZCAYAAADUicu/AAAFLElEQVR4Xu2YW2hcVRSGp0m94AWDGCNjkj25YDSIqBGKWMULah8UkRqVihfEC1q1NAjeGiqCUgUflCpitT5U64MBtT54qXhDfbC+BKu2VquYWrGWVAwm1FLit+asna6smTkzMUkNeD5YnDn/+vft7LPP2WdyuYyMjIyM/zMdHR0thUJheQhhI8cN+Xz+CNH5fQLaau+fKah7e1tb2yVenzP09PQcwkVY1Nraej3HBT4fER+euxnQAMfHfL4C8/G/5cVqUP9llBsnRvn9HMcniaGWlpZzOb5KbIte9aXFJupYaetPA/9ODnVenxMwkF5iYTyns1fKIP0AuduPZzLvMNI8vRh9RpsEuTOIvZR7z+fS0HpHvS7ohEn+e59TfdzrjOUqzW3p7Ow8zOct5BvxtXl9zkDnviJ2cFEbVJIVIYP73foY9Ev2XMDzNjHk9Qi5b4kxyr7jc5WgHzdJ+5S5weci2r+JFeb0kgkT5KbR/J0+Z6Hde7w2bfSxdG+uyrLFc4rXPAzkfnzP57Su7u7uQ3Vg+4xNVtNe4kKjSf0vE39bLcI7oEvaJwZqXWH4VknbHB/wOQue98MUVpigj1Wpe5XPRcid7MY9PfRiPhs7Rid+5niL90XILfVaNej0pVr/GquHZLWMEg8ZbY9cCOtTZII/kB8cX6t1heHdqm2f7XMW8k+EKUxYU1PTkehDxAjjO93nI+QexzPgdagnt0huJNmM4DmTMfV4UwkYf2xubu60Wnt7+zHa0c/i81kq4/zDXJUVGKEjDZS5OiR3rtyFh3uPEC8IMUb8glTvPQK53bxrOvR3zSss1u/1WjH96zfxNDEsk+H9Hny7ZGNVRh9nPKea833Eg9ZTFumA1wT0Jbradga96MSv3lcJOnMW/o3ECPG1z0fIvaB1F4OLcK33QB368ngS/vsJk/iBWOf9Hvr5lNegnrKDVgjJZ0bqY/tgEnd/m63I+eKgO0JdjT+pT7bARXhcnMb5ngOl/t2ExdXpcn3EOuKZoBNBvQ87T7G81SKyiZFcpb6Qu9lrkVgvZT8JyffffO+pCI/Akyi0i/gzJO+vioVp4Eav1YIfOG22cj5mPY2NjUd5H78HuTDLrC9MbcK2SX3UcbHPyaoNyVNgc2yXWGs9vj+OurS81O21CP15w7Qp8Tn9KXhfCRRciPkP4rdw4NFX8vKNUOftXnPIFn5xzr2LYsfMubwHSgaKtsL5iuVS4jpb3iOPJPExztt8LtLV1XW01lXyORHb8XqkUp4VnUff73WDTPYS+vdFrIN4xZtKwPS61wT0QeIvBtpLpefr3Vj1HVZIlrgs9Tet7gcWkn8WSgYqoA97zUL+m1p3iQLeW7X9FT4n5PP54zS/xed8vy3Ue47mSyZG6ipUeBrJpo6yF1mN83crtTOJlDtPVspdscMaL3qTB89q8co7yenFOuK5vrNKOoh2QTndQn6rDNDraWj78n03z+e4sFdovuYPZ323btf8oz6PNiJbf68LtNcQkr/WJnbcaBvQdhvbwSMkfx/tINYTH+mgHvE+QXOypZdH5JdEf7ltsEBujfptpD4SLUzy5bEcv78LyftrP7FWXw2fRm+ZdnzIO3+9rT+i/49OWkEWmTDyK3WSNhHDIdnS1/TJNCvwbXdiSHaB/XTuPp+P4DuWu/U8fH0M4Bqfn2nk33r6s0z6JVFu9zhdqL83V2YlGyY2dYx5wWz0IaNG9HE3c39FZcwuTNZSVtjHXs+Yo4SUzUbGHCTU8i2VkZGRkTFd/gH8M+Aa3CfGXQAAAABJRU5ErkJggg==>

[image2]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAEoAAAAWCAYAAABnnAr9AAAA8ElEQVR4Xu2WMQrCQBBFk0pBLLRZkCSbhNWAjQewtPUUVpba2FjY2NpYewkbG09hLSh4BCW1/sAG1mmtZP+DDzt/pvrMLhsEhBBCCCHkZ9I0XUBLHEPZc8FMU3peoLU+JEnyiKKojzLEeQWvzLJsJGfRy30O6opQtOuhVvDfCOUY2A1DSFN4N3fOKxDAUHo16HUQzgl6QWtjTEPOEPINtmaCbXlCd+hcXTnoIudq8jwfSM8LbEDbusajblDvoH1RFG13Fm/W3K29AoHMpFehlGph2zZ2wyqV9vvgJ9igrvRcENa4CiiO457sEUIIIX/DB/STKYCe/duJAAAAAElFTkSuQmCC>

[image3]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAFQAAAAWCAYAAABe+7umAAAA9klEQVR4Xu3WoQrCUBQG4C0piEHLQLbd7XJ1YPEBjFafwmTUYjFYrBazL2Gx+BRmQcFHUJb1v7DB5TA0iYb/gwO75570c3TzPCIiIiKiH0uSZIaa49GXdy7M1GWPHEqpXRzHtzAMuzj6eF6gl6dpOpCzuNMM9AOEd0Z4yu3hHKD/RHh7r9hYhDlG7+LOUQUE1Ze9Eu5aCPGAeqCWxpianCH6LmzhCNt3R11RR/tTR53kXElr3ZM9chRBrsszXk4G5w1qm2VZ053Ff+rUPVMFBDeRPSsIgga2d1VsrK28+Kyid7CRbdlzIdShDTKKoo68IyIioj/1AkrKKYBSX0CiAAAAAElFTkSuQmCC>

[image4]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABMAAAAWCAYAAAAinad/AAABJUlEQVR4XmNgGAWDB8jJydXJy8s/BmJvdDkYAMptB+JvQHwdiP8C8UUFBQUNZAWtQPwCiD8D8X8Chl2WlpYWBrGBlmtD1T9AU8bAICsrqwySBCryQZcDAaBcNRD/BuJomBjQVZdAehQVFcWR1RI0DCg+AeqS2TAxIHsRVCwYWS1Bw7ABoNpbID1SUlIiKBLkGAZ11X90cWTDfNHlsAFgOLkB1b8Dhps5uhzcMCDthy6HDoDqJIH4KdAgA3Q5MIAZBlQQgC6HBpiAag4pKSnxwwRUVFREkRXADQPiQBQJNACUnyIuLs6NLAYMGhtkPrJhqNGMBKDyfUBcC8WLgfgYXIGMjIwq0NkOQMEWkGKgLTuBdBIwgPWRzAHZ7go1DAMjqxsFAwQA6w9ZV/nssEsAAAAASUVORK5CYII=>

[image5]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABMAAAAWCAYAAAAinad/AAABN0lEQVR4Xu2SvS5EURSFR0JCqXZ/IyoFyX0BncJPiEQnQqHS6CQSlaEVndB7Bi8gWvMKElEKhUIxvnVz7sm+O8JQKWYlO+eetdZeZ2fPdDpD/B9kWXac5/kjteS1BkVRzAdPF/8Z5wvcYjRIoJ6pN6r/XZh8hKyY+40CradGmqbTCsO87LWAUfRrT8I9eO7HsCRJJtBf0WcNrQf+NJmm0Br67OkS3yTfH5yn3jdomHZUB1JPeDe9p4YJiwv2YKLtMFnPTNnzvhjGueq1BujvNK9VVTXGedgEel8Mk9lrAtoVP8KU5RQKf0fPjuVjGLXeEgLg7z0nsJYttKMWacI2WkIA/C6N51/wF0w2Xl8YfYbLAuSJwmi4VWNZlnOubyQ81m0IvHvirOlX0N54/ICQff3XvD7E4PgEsmldt9S8ycAAAAAASUVORK5CYII=>

[image6]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACsAAAAWCAYAAABZuWWzAAAClUlEQVR4Xu2VT4hNYRjGD0OSZBZ06/4791+uhZWbNElZWijWNmQiRSikoWlYi4UmS2KhsLDCwr9RpBQrTZkaKSKaxSCsdP3e2/td73m7pzk1NRbuU2/nvM/zfN99zne/850o6qOPf49SqbQljuMJ6m69Xi95PQD9tecWHIT4XC6Xr3AdlnuvC2q1WhntfoKsVCojkNNc9yQEA/Qh6j01Ts3IvfcI1PdDfZd1zJD1VKvVHNyj0BO6xW/vhntMtakP1Hnqd3cQhrMQH3XydlpY+HXoX4rF4nLp8/n8avo3PPla7xWfDSI+atZ6CLcd7lbomX8QrmY9cDfwPLBcB/rj8kR7vSZQ7ZflCN6Am+F2SeDYh+vVN+B8besjyA7qZujBAJ5qaHK53ArCHzb6X2QM+9ZyzWZzpfBMutn4TmXxEbQCdy/0uqrdh0G7yGVR6BPIEpb5X/Tiqeumf57i+259ysnYg4VCocj1q+GnCX/EehOYZ9jbpp9M8c1anwDfUbif1DvqkPFORGmrKphn2DuhZ0WmUnxyenR9vaD7dCr03I/p754xtkTY4YSgmCPsNdOnbQNZ2a6vF9DHCfjE9DL3OevpIITFvM9rAh046Wh5g+UhRgLB/dUsPg9ZVTzHLSdj5OW0XAcm7H6vCTRs4ivTaDTWaIhtgdN9OKfPA/0Sl8WhZ8wyGRPO9QRCWCY84DUB2lPRHXdMPpeWUz4RTHx+rIXsU/F4XucZ7BIc4nWIrQinRaSeSWBqkxknA+UT2uaLtUp6PW4+xeYgN16Z56XpxffNeizQHkZmVQ0v82zwfGYw+AQ1Su2KzCHuwbc/Vt9olOJrtVpL0V95PoAF26mBL3htwSH7ki1w0vMW/PMb0176Pv47/AHyOOE77cTtDgAAAABJRU5ErkJggg==>

[image7]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAmwAAABFCAYAAAD3qbryAAAD8ElEQVR4Xu3dTahtYxgH8ONcRFEGTumcffbaO+nUKYUzMdCNoY+ZlBKGN2MMKCMMKLoTA1FupBhRF9ElJXUpE2JgRF3lI4Or22UgHc97reUuT2ufj3323neV36+e9rue913vWnuP/q19PpaWAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADiI4XB4LPcAAOiRqqq2cw8AgB6JwHY89wAA6IkIa7eNx+PDuQ8AAAAAAAAAAEC/jUajG3KPyeLzuir3AABmqqqqU+VPYLRrfX392ryuj5r7jdD0Sbye2KH+fW+l8j77UXV8XnkNAMBcRfi5Nff6anV19er9hKZ4by/udS0AwP9ehKejuTeNCGCf1aHtyTw3Saz9M/faYv6n3AMAuGAinDxSAs/GxsaVeW6WhsPhK3Gdj1v1XV4zrTqwbZcnbnmuS4TFa3Kvrd6vMwA218p9AIC5igByNvfmbbevXuOeXs29nTRB6qC/NBF7vDcYDK4re62srFyR58PFAhsAsHB9DCBxT6/n3k5K0JrF0684/7X6tez1S8f841Hf5D4AwNwMh8NbSjgZDAaXR12/trY2aOa6nlaVp07l6dikyusbcY1T9ev21tbWJbsFq9jrzdzbTZzzTNk33tPneW4v8j3l46ZX1V/nds0DAMxchI53IuB8UB8ux/Edzdw0oalL7HOkGdeBajPq7faabNprx/6nyzV2Co+TxHm3p+N3ox5NvRLSLqrHZzrmAABmqx0yYnyyGUegOhYvy83xLESIeiyu8Wnud5k2sBXlPe31FxAaVf1VaFb22tzcvLQ5jvv6qj3XGt+5NOPPCwDgnBQ6zo0jrN0z6WnRNF+JRv/B2O+lqD9i77tKL8bfp2X/MZo+sC3HuTfn5m7ifn6PeqKjtqO+rZcdivHd9fqnoj6K9/N+s0eM32jGAAAzE6Hj4db4rwgdD9Tj4+dXHUwEqCOx34dL/3zl+kPU83lNNk1gi3Pujb1fzv1Zib2fa43vi/qydfxzMwYAWIjxeHw4wtv9ub8o+w1sEZhujPoi9xclrv1j1NO5DwAwNxE+bsq9Pov7/TX3JinhLvcOauSfwAMATFZN+Hm7SWL9W7kHAMCc7Cesjep//h6vl+U5AADmIMLXyRLA9lt5HwAAAAAAAAAAAJjWaDR6KPfaqqp6wc+uAQD0XAS233IPAIAFiCD2bNTX9fhErta60+fPAgBgoSKMnc29zBM2AIALpDxhi5dDuZ/FujO5BwDAgozH4yr3AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAoHf+BrbO8nU/MVZCAAAAAElFTkSuQmCC>

[image8]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAF4AAAAaCAYAAAA+G+sUAAAEC0lEQVR4Xu2ZW4hNURjHz4xLKMyD69z2PmdGoyG3QYRIiXmSCJEHHggT0gxNcmsiIcyDIooMSVIe3K9PpMitmBdMGbk+SFNeJH7f2WvPLN84cy5zTnOm9r++1rf+6/vWXvu/1/7WPjOhUBy4rrtLcwHSBMSd5TjO/Q6sSecESAMQ3kXcajHcI7LLfRPRsYs6J0AGYcSfpfnuiOLi4ulsoD/YUz0WIMNA9N88gCOaD5BhyI7Pz88fpPkuR1FR0SR2xCLKS54e60qUlJQMYV0RbJpYYWHhCKyAoR469n9A8KpwODw0K8uMOWAvya7QY10NxB7Ful6IzzLXWPxG+K/YN+xqW4YHuHfE14tPW4st0TFZARZ6PRuFR7BhrKtZfMTeosYuFBQUFDJ+jW6OPQb3q7S0dLCJ2y3z2OOpIBKJDJQHrXmB8KxvtebjQkTPhPBSGpj3EbaHcjYDo9Tm99NxsSDikftJfC28fK34PjH7fR+R59GvNt1c/O/+WGfRgfBnOiN8o+Y7C+b8woK2aj5RyIEYS3gEnmL5tZY/n9g54vOgJ5p7Gy8x8DfwF+Afp62kv1N8P5d+Ofw2bFnIOkfgD8Nt94WXMxH/JDbBjJ9ORfiesjgSF9M+xO5gH1jQUh2YDMi/pblkwRx5sYSHr5NWRAi1LzWfsUPmnt5gRw1/njk3mXmjbzjtZiuvQVrGx+G3GH+TNR4VHu6l4z2I19jClIQncaosAmuiNPQ1nPR/6thEwZfEGPI/az5ZxBIebrnw2CHsbluGBylRUtrEZ44+Pk9sA/Ns6ED4cyZHhI+OE7/XGo8KT3vW5wSpCn/F9XZn666B++Ff2OKGC+dfvCMQc8C81rIrYprO0zACtR6u+OulzwYppf9ex8cDuY1yr7QHHe9eZmO3sQVmvI55bwrHgyvx8+BOkTdXcmjrxfAbHHOW0L5yvEM+YUiZaXG8mtYKs6h2hy1cs5PA32+IOWFuoJ3Ytuk8DS28tP71XauupxP2G+LD/BYIu9bXUUVFRS87JikwWU0MgaXm7/gfL6+h5jXIneakodSYw7Xd5yTcPZocuQbrcX2+24CbuayF5z5GGuEj8pTl+9Ufc7xPs1wrPCaIfaW5ZCG/XGMIf591rqVdZx5C94IIrIWn/8QxBw7tR//AlZpHv8aOjQfHOwCj9TMVyKtN/lvx9bUZ2ydtWVlZf/yV9ljWwwj/z98x6D/GKkNe/W+t/firqHWTrdC44GGNJu85VoWN1+Px4Hr/qHlg/JV2XaXfh7dghfiOV3K6j/gIOVZzArlhKTU2l8pXhEKuvDXy3S3zYzN1QAAFdlRY3g5qbhEP4JgeD5Ah8INkAMI/c7xv1YT+JBsgjSgvL++tuQABAgQIkHX4C3O+LJ/OPgUdAAAAAElFTkSuQmCC>

[image9]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAYAAAAcCAYAAABCgc61AAAAh0lEQVR4XmNgoCMwNjZmlZeX9wLiHLignJycK1DgPwwjqYcAkKCCgsIpdHGwBBD3oggCVXJAdfijSAAFa4H4LIogCABV7gI6YgKKoKysrC3UGAEUCaBAJS5nbsMlgdtjQHwayGRSVFRUR5foA7oqFEifQZEACvoC6XfoOq4D8W2Q6+CCwx0AACgkKhEaaBK+AAAAAElFTkSuQmCC>

[image10]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABMAAAAaCAYAAABVX2cEAAABGUlEQVR4XmNgGAWDBygoKGgAsYO8vPx/EAayV6KrAYoHAvEiIL4CxFlAIUZ0NShATk7uJFDhOSD+JSMjI40uLysra4JNHCsAGuItKirKA3Id0OBbDGi2g1yGzMcJgF6TAGIBEBvmXaCBrshqgGJ3kPk4AdCgiTA2UNNuqIHfkeQTgPgSjI8XgMIKxga6KAzmOiT5OUA8BcbHCYA2+gMV3kcWAxpYDzIMGOh+ID6IDQoKZDVYAVBhH1DhQmQxkEag+G8g3gpVcxlZHieA2qqBLg50nTNIDkj3A+XD0eWxAqCGJ+hiUMAIDbuvRHkRCJiACivRBWEAaNAnkIHo4hgAaEgiKEyA+A7QK2no8iAAFJ9OlGGjYBQMJQAA/tBE3OCbgiMAAAAASUVORK5CYII=>

[image11]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAEkAAAAZCAYAAAB9/QMrAAADOElEQVR4Xu2XTUhUURTHh4y+oQ+aNGfGGceJSChKCoqiUgKDCqlAXUiLoI9F0cciKqjog1ok9kUUhO0SXESFJkWLhBbtsqBaGIQUEW2EgpKEqN/Rc8bbTcfVw2reHw7nnP//3vfuO3PufW8ikRAhQoT4SxGPxxemUqmTJSUl3clk8qdaL/YMe4RV+HPyBhTmkhQEv1hyK5Cj15C3GU8x5w3NzhPIg9M9d93cLZLDf1Xttq/91+CBKylQ2uOkaO9czkBX3dNCbfa1vEF5efkELUKLrwngt6h+2dfyBnRQrRQhkUgs8zUB+jrttFpfyxvw8CcowhfC8b4mQN+H3iMd52t5Ac6bIukS/AFfE5SWls5H70+n09Ml1233xwEfNFjfEe3mc74WOLj5brl5JpOJ+poA7TH2zXI+BWJjUSSBFumszwcObvog10OjfaeQ1ZZHo9FpucYHiTErkm4fOY98/gXW5fPFxcVTZA5+NgvexnZc5OpolRR1F7actMB48rWMb8BmEq8RjrlJ4joZy7y5+P10akavU6XjsuekFsndbgXyA2I17nnJuArZnryIlrovI+FkzSO9oEaEFummbDf8Auw09oaL7Yw4D2ngRpN0TpvcEN+H32Q6+Q95C2IPiZ86fJfMSw1+c/Uot12v1SQdrS+IPuwq8XH8c/x15xq/dRL5E+wG1iJjHf484zbie/U5hGtX7pRxo4LBr3WBn7BXWDOT9xYWFk71x7qwIllO3CHXcvLsfz3i9xYzL+XOM+gabjl5R0R/HOI93r2yRaILVxMfcrSBfwT4FVgbt6unK2eVlZXNUU5+oHpshnA2LxAMUyTpqG5HP4y1wjdinx0+V5GyH6nE7Ran9KXiaNkiqXYHO2bmjlO7wg6ZqNwZ5fqNCwz+wZ0cbOOBIhFXYY2O9hFtFX7rKEVqcvL7FucqUiwWixNfM83AnHqLRWfOS+HwR40XzuJAoO3rLrwT+yAxi1ppD8zCqmUcvg5+hxzwOYrU7OSdkaHtdnCYIl108n6uv17jDeLRG+CKNL5A3Coc+lubJ5zFY4UCFrTEJ4OEnD0W60fvOPetKxxjJgvHj5wwPkSIECH+FfwCeiMMzUyXdd4AAAAASUVORK5CYII=>

[image12]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABYAAAAaCAYAAACzdqxAAAABR0lEQVR4XmNgGAXDA0hLS8soKCg4yMvLfwXid0D2IXQ1yACo5g4Q/weqKwTpAwoxoqtBASDFQDwNiJ+jy8EA1BEzQWqNjY1Z0eWxApAGqMv/y8rK6qDLA8WtgHgdEP8GqUGXxwmAiqOBBitAXe6NRf4YyMVQeeIMlpOTi4GxQZqA5nciy8vIyKgAKWaYPFD9DmR5nACoeC4SG+Si1Wjye0G0uLg4N1S+GlkeK4B5H8aHanyHxL8BDAJhKLsLiD8BmSwweZwAaG48UPF1GB9qMLJF0Ujs46QEw330MAZhoIUCQPo2krg3SByYYmxhYngBSDEwcqSR+N+hhs8CWliOJN4N9QnhYNDS0mIDal6BLAZ06UaQAYqKimbI4jCfIIthBUBFJUB8Vh6SlUth4kCLJiAbAMo0sIwDwkB2ABCbw+RHwSgYBUQCAPauYEWX+nAJAAAAAElFTkSuQmCC>

[image13]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAEMAAAAZCAYAAABq35PiAAADGUlEQVR4Xu2YTUhVQRTHL1oRfWLwemTqfb4nSY9qkRRBRWmUtAhK6GMRbbKiog9pU5so6Is+JF1ki1wFQkHUQo0wqF21qCgoyCJctLBNUAtbSNTv9M7YcHo+UlLhcv9wuHPOf87MnP+bmXs1CGLEiBFjlCgrK1ucSqVOV1RU9IZh+FPtC/YC68GW2pzIgSJbpXARQ/3fQnh8A9bt4uXl5Zk/2RGDFMhuuOf7vhhefEDi7J5blosEKK4WIdImJmL0+TEH4l3Kb7Rc5JDNZqdosR2WEyDcVuWvWC5y4Ahsl2K5F5ZZTlBZWblBxWiwXOSAGKco9CvNSZYTwDfBf6ypqZlsuUiBIufJr07BRy0ngFuIDSYSiRnqyw7566IdD1RXV89k7jvYQ+yT5R2o5YSskeN93nIFQeJ+SayqqkpYTgD3CBtwPq/i+RMlBvN2UOADeRYSQ5DJZObS95yNF4QOPmxxIgSC1Ttfdkih/mMJ5h3Ezth4PrDmqSMWQwrDvuWJv8Ke23gymZwuObKTmGwXl+4in4erZSH7uHSX4xa7OLG19N+JldBeA5/EQoljK8ibB3fEfdzh19FeHeg9Jv1lXvq0pdPpBZILtuCvYy2zZBz8Rjef5I34mKgYN6S4MHc/nMXeM9Be6CLbXxTXnE4Rg+d3npscj/8Df1uY+5R/4sVfat5drA97iu2m7zPGbJEdKmLoeG3YSeKv8a9JPv56zf9AvAl/j/rNtLM8e3j2uvmCnBj/tjNIfquDfcbeYO0kHyotLZ1m+/pwYjif9n0Zy/OH/pYJvXNNXsrPc9Bjd9P5Ml6gO4r1HDZzyUV/wfexZuePWozRIo8Ynb4Y8Mex28QuhrnXtYsXEqPV+bS7vPZBK4Zf4ISLYS9Q2t1uEbqVL3tcP7FV6LB5ODH0DhoqKMztDNc+YMWwOwO/xfm+GPKjjbkY8soyC3wc6nFg8pWuMBZTr4vdwbORC29JPjF0vHbny3iBHhPax6wY2CXPf8ec17Vdh/UHeuHyCTAH7qrrO1EoDsf3fyBFvF1mS4PiSywZI0aMGP8TvwBwbfoPsjb7EgAAAABJRU5ErkJggg==>

[image14]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAcAAAAdCAYAAABmH3YuAAAAkUlEQVR4XmNgGNJATk7OB10MDBQUFAqBeBO6OBjIy8uvAeJadHEwAEq8AOq0hwsAOelAwf/oGKZaEqjAAUjvBwmC2CAM1w0CQFc+gutAk4iDGncaXQ5k9DSQJFBRP7ocSPIiVGcwuhxIEqTrFhJ/DYokEC+BsgOB+CaKJFDnCij7HpCtDZcEAkaggA264CgAAgDVGSwMsnxmogAAAABJRU5ErkJggg==>

[image15]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAIkAAAAaCAYAAACD1n8kAAAFB0lEQVR4Xu2aW2gdVRSG0xi14oV6CZFczpmkkWBQUY6CVqXF+tQo9VoFqz4o4oPWPqgYiFcUizWgiNAHQQQVLOLtQVG81KJVqehTFUTwpWDRh4hQxUiJ3+9Zu25WZnLOnJs5Oh8s9p611uzZa83ae+ZM0tNTUFBQUFDw3ydJkrfK5fKCxNsKmifktlQq3eVtualUKkcy2AbkDm9rJ0z+DQplncTbCpon5JY8b/G2XDDIKg1kVTfv7e1EO4nXCYIaYy5TNqf9+G21gE/1vssZxRHldkFxoLu203G0ZCcRFsinXt9OsopEMJcLNCcCfNzbuo1QJF7fKVpSJNyslRbIBm9bisnJyaOYwA7O+yYkIk9CahTJ6xpnYGDgWG/rNiwnHd2lY1pSJAyyniD+7O/vP87bssD/dGQvN/phzr9oZGRkcHh4+BjvtxQ1imSu3mJb5vQqDnL0sTd0ipYUCUF8gDzp9VmMjY2VKIpzvT4vNYqk7h1pOUMMd1uRrPe2TtF0kWj1E8Tv3LArvC2DFfh/6JWNUEeRHPT6bkMxKo7x8fGjva1TNF0kBHB/nhXLBW/Cf9brGyGrSBh/ja2+B72tm+hUHOTxeltUL3mbaLpIuMB7eYoE3+3I21ZcWTLjz0sjq0jQ32fJXbRFa0Vie9nrYzjvUq/LSyvGWCoOYXH0en0j2LeuF71eNF0kCgI55PVZEPg25Gmvb4S0IrFgf8W229sEti+RW7w+JuvcPLRiDMWh/Hp9oFYceSm3ayexIvmMbh/vJ6d5u4cX1ovx/9HrGyGtSNCt1ZxKGd9HsM1jOzEc4z+N7gZ0kzqmvVHnJ9WPb2vN7Qj6t2O7s8etXPnpETo0NHSy/MnBSRlj5MZym1okXGNTHIfAdwrdvT02x6TKOv1IYH7D2LbQPyM6pQ/7RvTX2A7b1p3kKi62E9nm7WmUqj95z/P6vCQpRcJc3tecJiYmjo/1+J7Pdb9DNtvx2fgdCHb6V6tVsfgbw/E7YQHYzd8abJG/CukV+rNpY8Tg95DlbcrbYsxn0fcRxeHHL0c/Buj/wTWm1R8cHDyF4/2hoOjvi/y+D58dKJL+cjt3Ega5jHaOC457exb4H0jq/0WUSlwkzKGSVD9hz2tO6kvQ38bxjCV8gWScYP4X2tw3qaC0A5h+0Q0eHR0dCH3dAOTZcKzi8f5pY8SUq58MNJ/XvE3Y3MPL5A8W15XITGLvgMhvsX98PfofMYd3zbaK4+eCDf0Xapn3EPpngl6U27WTQC9JPMsr64EAViJvlquJV+CHxfumkaTsJHlJqqv6q8RWXnyDadfQ9Cnp2Dea7iA+O8L5Whh+viljpIJtr9c1AuPc6ucQ0AdObI+FY/p71DLHy+k/+o/n37b27CQtolfJ1iNIq0LvLd4hjWaKhMA3k5R7wjH9XWop+InoButD1iXxDaB/SEWS2LtG2k7ix4htMYzzlNc1AnNJNC+nu85a7SRxkXyuVo8ZbC8EvdnatpP8azRTJJx7swoDmS1Xd4rDf1lVUknM8xLznUZ2on9Vfdo5PevN9xcVBLKHxJ+ZNYYH/QM0K7y+Ueyj5rxyQrs9qf49TTvMzza/T5Bvrb+P61doH8FvN+0Tif3cRr72Y/+fi0T/4qBEnqPns7ezm62Oj5VU/zJcCz9GgGuOIj95fSuIC7Ue9I4W4lI+vF10fZFY9ac+jwuaI+S2q4ukoKCgoKCgoAD+AsK/j1IIKg/GAAAAAElFTkSuQmCC>