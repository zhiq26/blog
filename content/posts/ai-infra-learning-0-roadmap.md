---
title: "AI Infra 学习路线图"
date: 2026-10-26T02:00:00+08:00
draft: false
description: "记录 AI Infra 的学习路线。"
slug: "ai-infra-learning-0-roadmap"
tags: ["AI Agent", "AI Infra", "AI Infra Learning", "Roadmap"]
categories: ["AI推理部署学习"]
ShowToc: true
TocOpen: true
---

# AI Infra 学习路线图

## 一、AI Infra 的层级

### 1.1a：高性能推理服务架构（可能是 ToB？）

可以着重看下 **vllm**：[vllm-project/vllm](https://github.com/vllm-project/vllm?utm_source=chatgpt.com)

**相关技术名词：**请求调度、Continuous Batching、Paged KV Cache、Prefix Caching、分布式推理和 GPU Kernel 优化

### 1.1b：模型管理和推理产品工程（可能是 ToC？）

可以着重看下 **ollama**：[ollama/ollama](https://github.com/ollama/ollama?utm_source=chatgpt.com) ；以及 LM Studio

**相关技术名词：**模型下载、生命周期管理、API 封装、配置与用户体验

### 1.2：推理引擎

可以着重看下 **llama.cpp**：[ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp?utm_source=chatgpt.com)

**相关技术名词：**模型张量加载、GGUF、计算图、CPU/GPU Backend、采样器和算子实现

## 二、推理引擎

### 2.1、GGUF 与 Safetensors

GGUF 的设计目标包括单文件分发、快速加载和可扩展的元数据。Safetensors 通常会有多个文件

GGUF 的主要下载渠道有：Hugging Face 和 ModelScope


