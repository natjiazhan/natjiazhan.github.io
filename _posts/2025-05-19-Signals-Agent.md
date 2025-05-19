---
layout: single
title: "Signals Agent"
date: 2025-05-19
author: Nathan Zhang
categories: [blog]
tags: [signal-processing, FFT, spectrogram, agents, LlamaIndex, OpenAI, Perplexity, numpy, Python]
---

Project Summary
During the Proving Ground: Agentic Startup RAG-a-Thon I built a prototype “signals agent” that can take a noisy audio clip as an input, perform multiple FFTs at varying resolution, perform spectral analysis, consult an LLM-backed web-search tool, and return an explanation that highlights where and why interesting frequency-domain events occur. You can think of it as an automated lab partner that listens, investigates, and narrates what it finds.

Inputs: M4A/MP3 files (Can be recorded live or pre-recorded)

Core pipeline: audio2numpy → FFT → time + energy binning → ReAct agent (LlamaIndex) + Perplexity API → natural-language report

Outputs:

CSV table of time-frequency energy

Annotated spectrogram PNG

A conversational summary that cites web evidence

The agent finished the hackathon able to flag dominant spectral peaks and trends and related those to possible sources-all in under a minute on commodity hardware.
