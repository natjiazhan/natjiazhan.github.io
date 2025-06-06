---
layout: single
title: "Trying to Get Signals Agent to Work"
date: 2025-06-06
author: Nathan Zhang
categories: [blog]
tags: [audio, agents, signal-processing, trouble-shooting]
---

For the past few weeks, I’ve been building a project called Signals Agent, a tool designed to analyze environmental audio using FFTs and other signal processing techniques, powered by an LLM-based reasoning loop. The goal was simple in theory: create an agent that could make sense of ambient sounds by performing multiscale spectral analysis, identify interesting frequency events, and describe likely real-world sources. As with any real-world agentic system, the reality has been much more complex. This blog post documents my recent efforts to improve the accuracy and reliability of the agent, especially in distinguishing between sound sources with overlapping spectral content. 

## Identifying Interesting Frequencies

Out of the box, the agent was fairly capable at identifying frequency ranges that were "interesting" or worth inspecting. The foundational fft tool was designed to perform frequency-time domain analysis:

```python
csv_str = fft("data/hamilton_ave.m4a", cutoff_lo=0, cutoff_hi=2000, start_sec=0, end_sec=20, time_bins=10, freq_bins=20)
```

The FFT result was then rendered into a Rich terminal heatmap to make peaks more visible. It became clear early on that the agent could reliably detect low-frequency hums, midrange chirps, and high-frequency bursts across a variety of recordings.

(INSERT OUTPUT HERE)

This initial success wasn’t random — it was thanks to an engineered system prompt. The prompt explicitly instructs the agent to begin every analysis with a broad-band FFT sweep (typically 0–2000 Hz) to capture general spectral structure, followed by targeted follow-ups. The agent is told to "drill down" by narrowing frequency ranges and time segments where peaks are observed. This iterative process of zooming in — refining the cutoff_lo, cutoff_hi, and adjusting time_bins and freq_bins — allows the agent to separate persistent tones from transients or short-lived spikes.

For example, if the first FFT shows a wide, stable peak between 90 and 140 Hz across the entire clip, the agent might run a second FFT with:

```python
csv_str = fft(file_path, cutoff_lo=90, cutoff_hi=140, time_bins=20, freq_bins=10)
```

to see how stable that hum is over time. The point is: the agent performs multiple FFTs across resolutions to explore the structure of spectral energy — from broad sweeps to fine slices. This part worked well. But the problem started when I expected the agent to tell me what the source of a frequency actually was. It could identify a 120 Hz drone, but would call everything either electronic hum or HVAC systems.

## The Misclassification Problem

The core issue was that different real-world sources often share the same frequency signature. HVAC systems, AC generators, and even rivers can produce a continuous hum in the 50–200 Hz range. While the agent could detect the presence of energy in that band, it struggled to differentiate why that energy was there. It might classify a 120 Hz hum as an HVAC system, when in reality it came from water flowing in a creek.

This was especially problematic for clips that contained multiple overlapping sound sources. A forest with a stream might have overlapping broadband noise and low-frequency hum, while a construction site might feature machinery sounds and background wind. Despite running multiple FFTs and sometimes invoking additional tools like spectral_flatness or autocorrelation, the agent's predictions often flattened into overly generic labels like "machine" or "ambient."

The problem wasn’t just in classification — it was in the reasoning process. The agent would fixate on dominant frequencies and anchor its conclusions to shallow patterns it had seen before. In some cases, even when told to explore narrower windows and different binning resolutions, it would reach confident but contradictory conclusions based on nearly identical input.

(INSERT THOUGHT HERE)

(INSERT OUTPUT HERE)

I realized that the agent was heavily biased toward frequency-based pattern recognition, but lacked the world knowledge and contextual inference needed to move from "what it hears" to "what it understands."

## Attempt 1: Line of Reasoning

## Attempt 2: Adding More Tools

To give the agent more context beyond just frequency content, I added six new tools to ```functions.py```. Each was chosen to extract a different acoustic property that could help the agent reason about the nature of a sound, especially in edge cases where FFT data alone was ambiguous.

1. ```zero_crossing_rate```

This measures how often the audio waveform crosses the zero amplitude axis. It's commonly used to estimate signal noisiness. High ZCR indicates lots of rapid changes — like wind, static, or splashing water. Lower values suggest more tonal or stable sources, like engines or fans.

2. ```autocorrelation```

Autocorrelation quantifies the degree of periodicity in a signal. If a sound has repeating patterns, like rotating motors, beeps, or speech, this metric highlights it. A periodic signal likely comes from a man-made or biological source. Rivers or crowds tend to be aperiodic, so this helps the agent prioritize hypotheses like machinery, voices, or alarms when periodicity is detected.

3. ```envelope_decay```

This examines how the amplitude envelope of the signal changes over time. Sounds with rapid onsets and decays (e.g. claps or metal hits) have different decay curves compared to continuous or sustained noises. This helps the agent distinguish between transient, impact-like sounds (construction, tools, footsteps) and sustained ones (fans, wind, music).

4. ```spectral_flatness```

Spectral flatness measures how noise-like a sound is across the spectrum. A pure tone has low flatness (energy is concentrated in one frequency), while white noise or turbulent flow has high flatness (energy is spread evenly). Many natural environments — like rivers or wind — produce high flatness values. In contrast, engines and digital tones tend to be flatter in temporal structure but peaky in spectrum.

5. ```fractal_dimension```

This quantifies the complexity or self-similarity of a waveform. A signal with structured, repeated patterns has a lower fractal dimension, while chaotic signals tend to have higher values. This helps distinguish between organic and mechanical processes. It also offers a backup signal complexity metric that isn’t redundant with entropy or ZCR.

6. ```shannon_entropy```

Entropy estimates the unpredictability or randomness of the waveform's amplitude distribution. A high entropy signal is unpredictable and complex, while low entropy suggests repetitive or structured content. Together with flatness and fractal dimension, entropy gives a more complete picture of signal randomness. It also complements autocorrelation — both tools approach structure detection from different angles.

There is some intentional overlap among these tools. For instance, both entropy and fractal dimension relate to signal complexity, while ZCR and spectral flatness speak to noisiness. But each approaches its measurement from a different mathematical lens, which means they occasionally surface contradictions that should help the agent reason through ambiguity. In theory, this meant the agent should be able to tell the difference between a gurgling stream and a humming motor — even if they both produce energy at 100 Hz. One is chaotic, noisy, and organic. The other is mechanical, stable, and tonal. But it didn't.



