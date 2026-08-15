# Qwen3.8-27B on Ryzen AI Max+ 395: Vulkan, MTP and OpenCode real-world tests

**English** | [Deutsch](README.de.md)

Test date: August 15, 2026

> This report may be shared in full or in excerpts. It contains no usernames, private directories or credentials.

## TL;DR

I set up and systematically tested Qwen3.8-27B as a local coding and general-purpose agent on a GMKtec EVO-X2 with Ryzen AI Max+ 395, Radeon 8060S and 128 GB unified memory.

The best stable setup was:

- Unsloth UD-Q5_K_XL
- llama.cpp b10436 with Vulkan and full GPU offload
- 128K context
- embedded MTP with four draft tokens
- q8_0 K/V cache
- OpenCode using the native Qwen3.8 Jinja template

The reproducible 512-token sustained benchmark at 128K context reached **16.45 generation tok/s**, **31.44 prompt tok/s** and **3.06 seconds TTFT**. At 64K it reached **16.68 generation tok/s**. During continued agent steps with KV/LCP reuse, new prompt portions often processed above 200 tok/s. One agent step averaged 23.04 generation tok/s, with short windows around 24–26 tok/s.

The desired sustained average of 20 tok/s was not reached. The configuration is still practical for real agent work. Its biggest downside is the cold start of a new OpenCode session: processing the very large tool prompt can take roughly two to three minutes.

## Test system

- Mini PC: GMKtec EVO-X2
- APU: AMD Ryzen AI Max+ 395
- GPU: Radeon 8060S, `gfx1151`
- Memory: 128 GB unified memory
- Operating system: Windows 11
- AMD Software: 26.7.1 WHQL
- Display driver: 32.0.31035.1003

The installed AMD version was already the current official WHQL release at test time. I therefore did not replace the driver or force a system-wide ROCm installation.

## Model and runtime

- Model: Qwen3.8-27B
- Base model: `Qwen/Qwen3.8-27B`
- GGUF architecture: `qwen35`
- Parameters: 27,320,697,856
- Quantization: Unsloth UD-Q5_K_XL dynamic mixed quantization
- Model file: `Qwen3.8-27B-UD-Q5_K_XL.gguf`
- Model size: 18.83 GiB
- Vision projector: `mmproj-BF16.gguf`, 0.87 GiB
- Native context length: 262,144 tokens
- Configured context length: 131,072 tokens
- Runtime: llama.cpp b10436, commit `6fed9f6ff`
- Backend: Vulkan with full GPU offload
- Embedded MTP predictor: one next-N-prediction layer

The GGUF contains several tensor quantization types. The generic `general.file_type=14` field therefore does not describe the complete Unsloth quant by itself. LM Studio identifies the concrete variant as `Q5_K_XL`.

## MTP tuning at Q5 and 64K context

Every variant used the same 512-token reasoning benchmark.

| MTP draft tokens | Generation | Prompt | TTFT | Process working set | Draft acceptance |
|---:|---:|---:|---:|---:|---:|
| 2 | 15.54 tok/s | 17.02 tok/s | 5.64 s | 41.07 GiB | 87.77% |
| 3 | 15.77 tok/s | 31.52 tok/s | 3.06 s | 41.15 GiB | 80.13% |
| **4** | **16.68 tok/s** | **33.87 tok/s** | **2.85 s** | **41.31 GiB** | **83.80%** |
| 5 | 15.28 tok/s | 32.17 tok/s | 3.00 s | 41.46 GiB | 82.07% |

Four draft tokens were the fastest stable value on this system. More draft tokens were not automatically better.

## Quantization and context comparison

| Quantization | Context | Generation | Prompt | TTFT | Process working set |
|---|---:|---:|---:|---:|---:|
| UD-Q5_K_XL, MTP 4 | 64K | 16.68 tok/s | 33.87 tok/s | 2.85 s | 41.31 GiB |
| Q6_K, MTP 4 | 64K | 14.24 tok/s | 28.93 tok/s | 3.33 s | 46.10 GiB |
| Q8_0, MTP 4 | 64K | 12.74 tok/s | 31.44 tok/s | 3.06 s | 56.96 GiB |
| **UD-Q5_K_XL, MTP 4** | **128K** | **16.45 tok/s** | **31.44 tok/s** | **3.06 s** | **44.16 GiB** |

Q6_K and Q8_0 were larger and slower on this hardware. Moving Q5 from 64K to 128K cost only about 1.4% generation speed and 2.85 GiB additional working set, so 128K remained the practical default.

## Vulkan versus ROCm

The identical API test with Q5, MTP 4 and 64K context produced:

- Direct llama.cpp b10436 with Vulkan: **16.68 tok/s**
- LM Studio 0.4.21 with ROCm runtime 2.28.2: **12.04 tok/s**

The direct Vulkan server was about **37% faster** than LM Studio/ROCm in this test.

The official direct llama.cpp ROCm b10436 binary did not enumerate the Radeon 8060S in this Windows configuration, while LM Studio could use its ROCm runtime. This does not prove a general ROCm or driver defect; it only documents the behavior of these particular builds on this machine.

## Actual server parameters

The relevant configuration was:

```text
--ctx-size 131072
--parallel 1
--gpu-layers all
--spec-draft-ngl all
--flash-attn on
--cache-type-k q8_0
--cache-type-v q8_0
--batch-size 2048
--ubatch-size 512
--threads 16
--threads-batch 32
--load-mode mmap
--fit off
--jinja
--reasoning-format deepseek
--reasoning-effort high
--reasoning-preserve
--spec-type draft-mtp
--spec-draft-n-max 4
--spec-draft-p-min 0.75
--metrics
--slots
--cache-ram 4096
```

## Chat template and OpenCode encoding

The GGUF-embedded Qwen3.8/Unsloth Jinja template was used with:

- ChatML role markers `<|im_start|>` and `<|im_end|>`
- `<think>` reasoning with separate `reasoning_content`
- native XML tool calling through `<tool_call>`, `<function=...>` and `<parameter=...>`
- `<tool_response>` for tool results
- parallel tool calls and nested object arguments
- vision markers `<|vision_start|><|image_pad|><|vision_end|>`
- preserved thinking across multi-step tool loops

A local capture endpoint recorded the parameters OpenCode actually sent:

- `reasoning_effort: "high"`; the template maps this internally to `xhigh`
- `temperature: 1.0`
- `top_p: 0.95`
- `top_k: 20`
- `min_p: 0.0`
- `presence_penalty: 0.0`
- `max_tokens: 32000`
- `tool_choice: "auto"`
- 41 tools in the tested main request

## Agent performance in practice

A cold OpenCode main request contained approximately 13,264 prompt tokens and all 41 tool definitions. According to the server log, prompt evaluation took about 154.5 seconds, or 85.83 tok/s. Together with an additional title request, one very simple exact-answer interaction took approximately 189 seconds in the cold session.

This is the largest practical bottleneck: not generation speed, but the initial processing of OpenCode's large tool prompt. Prompt caching and longest-common-prefix/KV reuse help substantially during an active session. New prompt portions repeatedly processed above 200 tok/s in continued agent steps.

The 23–26 tok/s values observed during individual agent phases are real short-window or per-step measurements, but they should not be confused with the reproducible sustained 512-token average of 16.45–16.68 tok/s.

## Functional and stability tests passed

- Coding agent: read a temporary multi-file project, ran four intentionally failing tests, diagnosed the issue, added a validation file and modified two production files. Afterwards 4/4 tests passed, and the test files remained untouched.
- Tool calling: two parallel calls, nested JSON objects and arrays, and continuation after an intentionally failed tool call worked.
- Web: web search and WebFetch were present in the real OpenCode request, and current information was retrieved from the official Python website.
- Browser/MCP: Playwright MCP 0.0.79 navigated to a local test page, read computed DOM styles and captured a screenshot.
- Vision: the model correctly identified the heading and invisible status text in the screenshot. The problematic foreground and background colors were both `#18D38A`.
- Long agent loop: no repetition loops or obvious token artifacts were observed.

## What could still be improved or investigated

Reproducible comparisons would be especially useful for these questions:

1. Why does the official direct llama.cpp ROCm binary fail to detect the `gfx1151` GPU under Windows while LM Studio's ROCm runtime works?
2. Do newer llama.cpp builds improve Vulkan shaders, MTP support or ROCm device detection on Ryzen AI Max+?
3. Are other `--spec-draft-p-min` values faster or more stable with MTP 4?
4. How much can the cold OpenCode start be reduced through smaller tool schemas, a separate title agent or consistent prompt caching?
5. Do other dynamic Q5 GGUFs offer a better quality/speed tradeoff on the same hardware?
6. How does the setup behave with genuinely long prompts near 128K? This test configured 128K but did not fill the context window completely.
7. Does omitting the vision projector provide measurable text-only startup-time or memory improvements?

Feedback with exact build numbers, start parameters and reproducible measurement conditions would be especially valuable. These measurements come from one Windows machine and should not be generalized to Linux, other driver versions or other Ryzen AI Max devices without independent reproduction.

## Conclusion

Qwen3.8-27B UD-Q5_K_XL is stable and useful as a local general-purpose OpenCode agent on Ryzen AI Max+ 395. Vulkan was clearly faster than the tested LM Studio ROCm runtime in this configuration. MTP 4 was the best value, 128K context caused only a small performance penalty, and the agent, tool, browser and vision tests worked.

The honest limitation is that **16.45 tok/s in the sustained benchmark is not a sustained 20 tok/s**. Cache reuse makes an ongoing agent session feel considerably faster, while the cold OpenCode tool prompt can still cost two to three minutes.

## Primary sources

- [Official Qwen3.8-27B model](https://huggingface.co/Qwen/Qwen3.8-27B)
- [Unsloth Qwen3.8-27B GGUF](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF)
- [llama.cpp b10436](https://github.com/ggml-org/llama.cpp/releases/tag/b10436)
- [LM Studio changelog](https://lmstudio.ai/changelog/lmstudio-v0.4.14)
- [LM Studio CLI documentation](https://lmstudio.ai/docs/cli)
- [OpenCode providers](https://opencode.ai/docs/providers)
- [OpenCode tools](https://opencode.ai/docs/tools)
- [AMD Software 26.7.1 release notes](https://www.amd.com/de/resources/support-articles/release-notes/RN-RAD-WIN-26-7-1.html)
- [AMD ROCm compatibility on Windows](https://rocm.docs.amd.com/projects/radeon-ryzen/en/latest/docs/compatibility/compatibilityryz/windows/windows_compatibility.html)
