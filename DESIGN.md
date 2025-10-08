# Flow Dynamics Explorer - Design Document

## Project Goals and Philosophy

### The Problem

Standard CPAP analysis tools like OSCAR focus primarily on Apnea-Hypopnea Index (AHI) and traditional sleep apnea metrics. These metrics are inadequate for:

- **UARS (Upper Airway Resistance Syndrome)** presentations where AHI appears normal but sleep quality is severely impacted
- **High loop gain** breathing patterns characterized by periodic breathing and ventilatory instability
- **Flow limitation** that doesn't meet hypopnea criteria but still causes arousals
- Understanding the **relationship between breathing patterns and how users actually feel**

A user can have an AHI of 2 (considered "normal") and still feel completely wrecked due to respiratory effort-related arousals (RERAs), flow limitation, and periodic breathing patterns that never appear in their "official" metrics.

### The Solution

Flow Dynamics Explorer takes a different approach:

1. **Prioritize what correlates with actual sleep quality** over what insurance companies care about
2. **Quantify breathing pattern regularity and periodicity** - paradoxically, higher chaos (less regularity) in minute ventilation often indicates better sleep
3. **Detect flow limitation with much higher sensitivity** than standard scoring rules
4. **Estimate arousal burden** from breathing pattern changes rather than relying on EEG
5. **Make advanced analysis accessible** to users, not just sleep physicians

### Design Principles

- **Transparency**: Users should understand what's being measured and why
- **Flexibility**: Allow deep exploration for power users while providing sensible defaults
- **Evidence over authority**: Show the data, explain the logic, let users draw conclusions
- **Practicality**: Focus on metrics that can inform treatment decisions
- **Open**: No gatekeeping behind medical professionals - users own their data

## Core Metrics

### 1. Periodicity Analysis

**What**: Identifies rhythmic oscillations in minute ventilation (Cheyne-Stokes respiration, periodic breathing)

**Why**: Periodic breathing indicates ventilatory instability and loop gain issues. It's correlated with poor sleep quality even when AHI is low.

**How**:
- FFT analysis on minute ventilation time series
- Identify dominant frequencies in the 0.01-0.05 Hz range (20-100 second cycles)
- Calculate power spectral density at these frequencies
- Classify periodicity severity

**Output**: Periodicity score (0-100), dominant cycle length, visual spectrogram

### 2. Regularity Analysis

**What**: Measures chaos/entropy in breathing patterns

**Why**: Counter-intuitively, healthy breathing is somewhat chaotic. High regularity (low entropy) indicates rigid ventilatory control often seen with high loop gain.

**How**:
- Sample entropy calculation on minute ventilation
- ApEn (Approximate Entropy) or SampEn (Sample Entropy)
- Invert the entropy value to create regularity score
- Higher score = more regular = worse (rigid ventilatory control)
- Lower score = more chaotic = better (healthy variability)

**Output**: Regularity score (0-100, higher is worse), interpretation guidance

### 3. Enhanced Flow Limitation Detection

**What**: Identifies inspiratory flow limitation with higher sensitivity than standard AASM criteria

**Why**: Traditional flow limitation scoring misses subtle patterns that still cause arousals and sleep fragmentation.

**How**:
- Analyze inspiratory flow shape using multiple methods:
  - Flattening index (ratio of actual to expected flow curve)
  - M-shape detection in flow waveform
  - Breath-by-breath flow shape classification
  - Deviation from ideal flow contour
- Flag breaths with even minor flow limitation
- Calculate percentage of breaths with FL across different severity thresholds

**Output**:
- FL percentage (at multiple sensitivity thresholds)
- FL severity distribution
- Visual flow shape analysis

### 4. Estimated Arousal Index

**What**: Infers likely respiratory arousals from breathing pattern changes

**Why**: Respiratory arousals are the missing piece for UARS patients - they don't show up in AHI but wreck sleep quality. This metric is more sensitive than clinical RERA scoring and provides a broader view of arousal burden.

**How**:
- Calculate breath-by-breath respiratory rate and tidal volume
- Establish rolling baseline (120-second window)
- Flag breaths with significant increases vs baseline:
  - Respiratory rate increase >20%
  - Tidal volume increase >30%
- Apply 15-second refractory period to avoid duplicate detections
- Sum events and calculate index per hour

**Output**:
- Estimated arousal index (events per hour)
- Individual event markers with rate/volume increases
- Timeline visualization

**Note**: Rule-based approach for transparency and tunability. ML classification may be explored in future versions.

### 5. Minute Ventilation Dynamics

**What**: Tracks minute ventilation over time with high temporal resolution

**Why**: MV patterns reveal loop gain behavior, treatment effectiveness, and overall ventilatory stability

**How**:
- Calculate rolling minute ventilation from flow data (25 Hz)
- Smooth with appropriate window (default 60 seconds)
- Identify baseline, variability, and excursions
- Correlate with other metrics

**Output**:
- MV time series plot
- Statistical summary (mean, SD, CV)
- Stability score

## Architecture

### Tech Stack

**Platform**: .NET 8+ Desktop Application

**Backend/Analysis Engine**:
- C# for all signal processing and analysis logic
- Math.NET Numerics for FFT, signal processing, and statistics
- Custom EDF parser for reading OSCAR files
- LINQ for data manipulation

**Frontend/UI**:
- WPF (Windows) or Avalonia UI (cross-platform)
- LiveCharts or OxyPlot for interactive plotting
- Modern MVVM architecture

**Data Storage**:
- SQLite (via Entity Framework Core or Dapper) for session database
- Original EDF files remain untouched
- Export to JSON/CSV for interoperability

**Deployment**:
- Self-contained single executable (no external dependencies)
- PublishSingleFile + ReadyToRun for optimal startup
- No installation required - just download and run

### Core Components

1. **EDF Parser**: Read OSCAR BRP and other EDF files
2. **Signal Processor**: Calculate derived signals (MV, etc.)
3. **Metric Calculators**: Modular analysis modules for each metric
4. **Visualization Engine**: Generate plots and interactive views
5. **Report Generator**: Produce human-readable summaries
6. **Settings Manager**: User preferences, thresholds, algorithms

### Data Flow

```
EDF Files → Parser → Raw Signals (flow, pressure, etc.)
                ↓
          Signal Processor → Derived Signals (MV, tidal volume, etc.)
                ↓
          Metric Calculators → Analysis Results
                ↓
          Visualization + Report Generator → User Interface
```

## UI/UX Approach

### Key Screens

1. **Import**: Load OSCAR data or individual EDF files
2. **Dashboard**: Overview of key metrics for a session
3. **Deep Dive**: Detailed view with zoomable waveforms and metric overlays
4. **Trends**: Multi-session comparison and progress tracking
5. **Settings**: Configure thresholds, algorithms, preferences

### Visualization Priorities

- **Interactive waveforms**: Pan, zoom, select regions
- **Overlay metrics**: Show FL, estimated arousals, etc. on flow data
- **Spectrograms**: Visualize periodicity
- **Distribution plots**: Histogram of breath characteristics
- **Timeline view**: Night at a glance with color-coded segments

### User Experience Goals

- **Not overwhelming**: Progressive disclosure - show simple summary first, details on demand
- **Educational**: Explain what each metric means and why it matters
- **Customizable**: Power users can tweak everything
- **Fast**: Responsive even with full-night high-resolution data

## Implementation Phases

### Phase 1: Core Desktop MVP
- Port existing JavaScript algorithms to C#
- EDF file parsing for OSCAR BRP files
- Calculate all 5 core metrics
- Basic WPF/Avalonia GUI with:
  - File import
  - Metric dashboard view
  - Simple plots for each metric
- Export results to JSON/CSV

### Phase 2: Enhanced Visualization
- Interactive waveform viewer with pan/zoom
- Overlay metrics on flow data timeline
- Spectrograms for periodicity analysis
- Distribution plots and histograms
- Night-at-a-glance summary view

### Phase 3: Analysis Refinement
- Settings/preferences UI for tuning thresholds
- Multiple sensitivity levels for FL detection
- Detailed breath-by-breath analysis view
- Annotation and note-taking capabilities

### Phase 4: Advanced Features
- Multi-session tracking and trends
- Treatment comparison (before/after pressure changes, device changes, etc.)
- Data export for sharing with doctors
- Community metric contributions framework

## Development Decisions

- **Clinical validation**: Pursue validation with sleep medicine experts and existing datasets once core functionality is stable
- **Community contributions**: Keep closed-source during initial development; open to PRs and plugin architecture once stable
- **OSCAR integration**: Standalone application that imports OSCAR's EDF files; potential future integration TBD
- **Cloud features**: Everything stays local forever - no cloud sync, no data upload. Reduces liability, privacy concerns, and HIPAA complications.

## License

MIT License - prioritizing maximum reach and helping as many people as possible over preventing proprietary derivatives.

## Success Criteria

This project succeeds if:

1. Users with low AHI but poor sleep quality can identify and quantify their issues
2. Treatment changes (pressure adjustments, ASV, etc.) can be evaluated based on meaningful metrics
3. The tool helps users have productive conversations with their doctors
4. Power users can experiment with custom analysis
5. At least one person says "holy shit, this finally shows what I've been feeling"

---

*This is a living document. Expect changes as development progresses and users provide feedback.*
