---
layout: single
title: "Signals Agent"
date: 2025-05-19
author: Nathan Zhang
categories: [blog]
tags: [signal-processing, FFT, spectrogram, agents, LlamaIndex, OpenAI, Perplexity, numpy, Python]
---

## Project Summary

During the Proving Ground: Agentic Startup RAG-a-Thon, I built a prototype “signals agent” capable of analyzing noisy audio clips by performing multiscale FFTs, conducting spectral analysis, querying a web search API, and returning an explanation of frequency-domain events. You can think of it as an automated lab partner that listens, investigates, and explains what it hears.

Inputs:

M4A/MP3 audio files (recorded live or pre-recorded)

Core Pipeline:

audio2numpy → FFT → time + frequency binning → ReAct Agent (LlamaIndex) + Perplexity API → natural-language report

Outputs:

CSV table of time-frequency energy

Richly annotated spectrogram (in terminal, colorized)

Conversational summary with web-sourced context and citations

By the end of the hackathon, the agent could detect and interpret dominant spectral peaks and trends, link them to plausible sources, and return results in under a minute — all running on local, commodity hardware.

## Data Acquisition and Pre-processing

At the core of our signal analysis workflow is a flexible and adaptive approach to audio ingestion and segmentation. Before any meaningful interpretation can happen, the audio data must be properly captured, cleaned, and structured. This involves several key steps: loading the audio, converting formats, segmenting it across time and frequency, and allowing the agent to choose how to zoom in based on what it finds.

#### Loading the Audio

We support both pre-recorded audio (e.g., .mp3, .m4a) and live-recorded clips using a microphone. Regardless of the source, we standardize everything into WAV format and then convert it to a 1D NumPy array using `audio2numpy` and extract the sampling rate, which allows for easy manipulation in Python:

```python
signal, sample_rate = open_audio("file.m4a")
# If audio is stereo, convert to mono
if signal.ndim == 2:
  signal = np.mean(signal, axis=1)
```

This mono conversion is important because spectral analysis assumes a single channel unless otherwise specified, and collapsing stereo to mono ensures uniformity during analysis. 

#### Spectral Segmentation: Time and Frequency Bins

After loading, the signal is segmented across both time and frequency domains using a sliding window approach. We define a number of `time_bins` to divide the clip into equal-length windows (e.g., 10 bins for a 20-second clip = 2 seconds per slice), and we define `freq_bins` to segment the frequency domain into ranges (e.g., 0–100 Hz, 100–200 Hz, ..., up to 2000 Hz).

```python
# Setup bin edges
bin_edges = np.linspace(cutoff_lo, cutoff_hi, freq_bins + 1)
bin_labels = [f"{int(bin_edges[i])}-{int(bin_edges[i+1])}Hz" for i in range(freq_bins)]
```

Then, for each time slice, we compute an FFT and bin the energy:

```python
fft_result = np.fft.fft(windowed_signal)
frequency = np.fft.fftfreq(len(windowed_signal), d=1/sample_rate)
power = np.abs(fft_result)**2

# Filter to within cutoff range
mask = (frequency >= cutoff_lo) & (frequency <= cutoff_hi)
frequency = frequency[mask]
power = power[mask]

# Bin the power spectrum
indices = np.digitize(frequency, bin_edges) - 1
binned_power = np.zeros(freq_bins)
for i in range(len(frequency)):
    if 0 <= indices[i] < freq_bins:
        binned_power[indices[i]] += power[i]
```

This generates a time-frequency matrix, where each row corresponds to a time slice, and each column corresponds to a frequency range. Each "square" of time and frequency contains a value that indicates the power density. This structured data is then either normalized (if freq_bins > 1) to show energy trends. Spectral binning is a powerful tool. Smaller bins allow the agent to see finer resolution, isolating short-duration spikes or tightly grouped frequencies. Larger bins help identify broader trends and energy distributions. Time bins help localize when something happens (e.g., a sudden burst of energy at 6.5 seconds). Frequency bins help identify what kind of signal it is (e.g., low-frequency mechanical hums vs high-frequency clicks). Adaptive binning allows the agent to first get a coarse overview and then drill into specific segments for further analysis. Rather than hardcoding ranges and bin sizes, we leave this decision to the agent. Based on what it observes in the broad scan, it can decide to zoom in:

Example agent instruction:

```python
query = "Characterize the signals in the audio file located at ./data/audio3.mp3 by using the fft at 20 time bins and 20 frequency bins..."
```

Internally, the tool receives:

```python
fft(file_path="./data/audio3.mp3", cutoff_lo=0, cutoff_hi=2000, time_bins=20, freq_bins=20)
```

If the agent sees interesting peaks in the 0–2000 Hz range between 8–10 seconds, it might issue a follow-up call with:

```python
fft(file_path="./data/audio3.mp3", cutoff_lo=400, cutoff_hi=700, start_sec=8, end_sec=10, time_bins=5, freq_bins=10)
```

This recursive refinement mimics how a person might explore a spectrogram—start big, then zoom in where things look interesting. 


This adaptive segmentation pipeline is essential for the agent’s success. By giving it control over resolution in both time and frequency, we enable a kind of multiscale thinking: a high-level scan followed by focused inspection. This not only makes the system more flexible, but also more intelligent—capable of narrowing down complex, noisy data to the most relevant features, just like an expert human would. In addition, this flexibility allows the agent to do whatever it thinks is best, creating some interesting out-of-the-box outcomes. 

## The Back End: The Agent & Its Tools

Now, let's get into the meat of our system. At the core, we have an intelligent agent powered by the [LlamaIndex ReAct framework](https://docs.llamaindex.ai/en/stable/examples/agent/react_agent/). The agent’s job is to repeatedly reason about the audio it’s given: what sounds are present, where they occur in time, and what might be causing them. It doesn’t try to guess everything at once; instead, it searches, thinks, acts, observes, and repeats. Given a query (e.g., “Tell me what’s happening in this audio clip”), the agent starts with a broad FFT scan to get a general idea of the frequency activity across the full clip. Based on where energy spikes or trends appear, it drills down into specific time windows or narrower frequency bands. If a particular signal catches its attention, say, a strong 650 Hz tone between 8–10 seconds, it might use Perplexity to search for likely sources of that frequency. This decision-making is powered by a tool-based architecture. Tools are defined independently in `functions.py`, and the agent selects which ones to call, what parameters to use, and when to follow up.

### Tool Registry

Below is a summary of the tools available to the agent:

| Tool Name           | Purpose                                           | Key Parameters                                                                          |
| ------------------- | ------------------------------------------------- | --------------------------------------------------------------------------------------- |
| `fft`               | Compute spectral energy in time-frequency bins    | `file_path`, `cutoff_lo`, `cutoff_hi`, `start_sec`, `end_sec`, `time_bins`, `freq_bins` |
| `search_perplexity` | Look up likely causes of a frequency range        | `query` (natural language), `model` (e.g., ``)                                 |
| `file_meta_data`    | Extract duration, bitrate, and file size metadata | `file_path`                                                                             |
| `record_audio`      | Record new audio clips using a microphone         | `duration`, `sample_rate`, `channels`                                                   |

The agent chooses among these tools dynamically, depending on its internal reasoning and user instructions. Let’s take a closer look at what each tool does and how it helps the agent accomplish its mission.

### The FFT Tool

What is FFT? Prior to this project I was familiar with a Fourier Transform but not a *Fast* Fourier Transform, FFT (Fast Fourier Transform) is a mathematical algorithm that converts a time-domain signal (audio waveform) into its frequency-domain representation. In other words, it answers the question: *"Which frequencies are present at any given time?"* Real-world signals often overlap in time—multiple events happening simultaneously. FFT helps isolate *what* is happening *when* by breaking the signal into windows and computing a spectrum for each one. This creates a spectrogram, a 2D matrix of energy values: time vs. frequency.

The agent begins by calling the `fft()` tool with a wide frequency range and a modest number of time and frequency bins:

```python
fft(file_path="data/audio3.mp3", cutoff_lo=0, cutoff_hi=2000, time_bins=20, freq_bins=20)
```

This returns a CSV-formatted spectrogram as a `str`:

```
Time,0-100Hz,100-200Hz,...,1900-2000Hz
0.0-1.0sec,0.01,0.03,...,0.00
1.0-2.0sec,0.00,0.05,...,0.01
...
```

This structured output allows the agent to spot peaks (high energy) and issue refined follow-ups like:

```python
fft(file_path="data/audio3.mp3", cutoff_lo=600, cutoff_hi=700, start_sec=8, end_sec=10, time_bins=5, freq_bins=5)
```

That zoom-in behavior lets the agent focus on narrow, interesting regions just like a human would when inspecting a spectrogram by eye.

### The Perplexity Tool

Once the agent identifies suspicious or dominant frequencies, it can ask: *“What could be making this sound?”*

This is where the `search_perplexity()` tool comes in. It connects to the [Perplexity.ai](https://www.perplexity.ai/) API and lets the agent submit targeted queries like:

```python
query = "Identify both man-made and natural sources of sound in the range of 600-700 Hz"
```

Perplexity returns a short, LLM-generated response with inline citations, helping the agent ground its answers in real-world knowledge. This is especially valuable for non-obvious signals (e.g., industrial hums, HVAC-systems, wind rustling tree leaves, etc.). 

### The Metadata Tool

Before diving into FFT, the agent often wants context: *How long is this clip? What is its bitrate?*

The `file_meta_data()` tool runs `ffprobe` under the hood to extract key metadata:

```python
file_meta_data("data/audio3.mp3")
```

This returns:

```
Property,Value
Duration (seconds),10.23
Bit Rate (kbps),192000
Size (bytes),160000
```

This helps the agent plan how much to analyze and how to divide the signal over time. Even if the agent wasn't prompted to run FFT analysis after, this information could still be valuable to the user. 

### The Record Audio Tool

The `record_audio()` tool allows live data capture during runtime:

```python
record_audio(duration=10)
```

This uses `pyaudio` to stream audio from a microphone and saves it as a WAV file, enabling the agent to react to real-time input—not just preloaded data. In the future, I hope to incorporate an ambient running mode in line with this tool. I plan to have the agent start running its FFT analysis after recording a duration of audio after a certain threshold frequency is picked up or a significant shift in frequency range is received. By combining these tools, the agent becomes more than just a signal visualizer—it becomes a spectral reasoning system, capable of:

* Listening
* Investigating patterns
* Calling external knowledge
* Explaining what’s going on

All without any human intervention or hardcoded thresholds.

## The Front End: User Interface and Output Formatting

While the intelligence of Signals Agent lives in its tools and reasoning loops, the way users interact with the system is still important. We designed a simple, elegant command-line interface (CLI) using Python’s `rich` library to guide users through the process, provide clear feedback, and display results in an engaging way. This front-end is implemented in two main modules: `app.py` and `format.py`.

### The Audio Analysis Terminal

![The Welcome Interface]('/images/agent_welcome_screen.png')

The `app.py` module is the user-facing entry point to the agent. When you run it, you're greeted with an interactive terminal interface that guides you through every step—from selecting or recording audio to entering your analysis instructions. Here’s how the interaction works:

1. **Welcome Screen:**
   Users are greeted with a title screen and two options:

   * Use an existing file from the `/data` directory
   * Record new audio using a microphone

2. **File Selection or Recording:**
   If you choose to use an existing file, it displays a numbered list of `.mp3` and `.m4a` files for selection.
   If you opt to record, it asks how long you'd like to record and streams microphone input to a WAV file.

   ```python
   selected_file = record_audio(duration=10)  # Records and saves as WAV
   ```

3. **Analysis Instructions:**
   You’re prompted to enter a natural-language task (e.g., "Analyze the humming sound and identify likely sources"). A default query is provided if you just hit Enter (it auto-fills).

4. **Running the Agent:**
   The app calls `run_agent()` asynchronously with your query and displays the agent’s thoughts, tool usage, and results in real time. This is helpful for seeing where the agent draws its    information, and it allows the user to see the spectrogram at each resolution the agent chooses to analyze at. 

5. **Loop or Exit:**
   Once the analysis is complete, it asks if you want to run another analysis or exit.

### Spectrogram Visualization in the Terminal

Raw CSVs of frequency data aren’t easy to interpret on their own, especially in a terminal. That’s where `format.py` comes in. It converts the FFT tool’s CSV output into a colorized Rich Table, mimicking the look and feel of a spectrogram.

#### Key features of `format_fft_output_as_rich_table()`:

* **CSV Parsing:**
  Reads the `Time`, `Frequency Bin`, and energy values from the CSV string returned by the FFT tool.

* **Color Mapping:**
  Maps each energy value to a hex color using the Jet colormap (`matplotlib.cm.jet`), where:

  * Blue = low energy
  * Yellow/Red = high energy

  ```python
  def _get_color_for_value(value, v_min, v_max):
      normalized = (value - v_min) / (v_max - v_min)
      r, g, b, _ = matplotlib.cm.jet(normalized)
      return f"#{int(r*255):02x}{int(g*255):02x}{int(b*255):02x}"
  ```

* **Rich Table Output:**
  Each time slice becomes a row. Each frequency bin becomes a column. The color-coding turns this into a kind of **ASCII spectrogram**, rendered right in your terminal window.

  ```python
  table.add_column("Time")
  table.add_column("0–100Hz")
  table.add_column("100–200Hz")
  ...
  ```

This feature is especially helpful for understanding how energy varies over time. The colorization makes important patterns—like pulses, peaks, or drops—visually obvious, even without switching to an external plotting tool, kind of like a heat map. 

### Why a Terminal UI?

Apart from the fact that we had limited time and wanted to focus on the subtsance of our project rather than a flashy shell, there were some technical reasons to keep it in the terminal. Not having a front-end reduces the total number of components involved in the process. This limits the number of connections where a fault or error could appear last minute. In a competition based event, this could be the difference between having something to present vs. not. 

The terminal UI helps:

* Keep dependencies minimal
* Enable fast iteration and debugging
* Preserve compatibility with remote or low-resource systems
* Focus on substance over style while still delivering visual clarity through `rich`

That said, the architecture cleanly separates input/output from the core logic, so wrapping this in a GUI (or even a web interface) would be straightforward in the future which is something I definitely plan on doing. 





