# System Architecture

## Overview

The Adaptive Safety Oracle uses a 3-layer architecture for real-time learning from edge cases.

## Architecture Diagram
```
┌─────────────────────────────────────────────────────────┐
│                     INPUT IMAGE                         │
│                   (Driving Scene)                       │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│                  DETECTION LAYER                        │
│  ┌──────────────┐         ┌──────────────┐              │
│  │   YOLOv8n    │         │   Weather    │              │
│  │  Detection   │         │   Analyzer   │              │
│  └──────────────┘         └──────────────┘              │
│        ↓                         ↓                      │
│    Objects              Visibility Conditions           │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│            SCENE CLASSIFICATION                         │
│     Generate Scene Signature                            │
│     e.g., "multiple pedestrians"                        │
└─────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│               RULE MATCHING                             │
│                                                         │
│   Rule Found? ─┬─→ YES → Apply Rule ⚡ (~0.2s)          │
│                │                                        │
│                └─→ NO → Generate Rule 🧠 (~300s)        │
│                         ↓                               │
│                   Cosmos Reason2-2B                     │
│                         ↓                               │
│                  Store in Rulebook                      │
└─────────────────────────────────────────────────────────┘
```

## Components

### 1. Detection Layer

**YOLOv8n Object Detector**
- Detects: persons, vehicles, bicycles, animals
- Zone-based filtering: road vs background
- Confidence thresholding

**Weather Analyzer**
- Brightness analysis (darkness detection)
- Contrast analysis (fog detection)
- Blur detection (rain/motion blur)

**Output**: Scene objects + environmental conditions

### 2. Scene Signature Generator

Classifies detected scene into categories:
- `"multiple pedestrians"` - 2+ persons, few/no vehicles
- `"single pedestrian"` - 1 person, low vehicle count
- `"bicycle on road"` - Bicycle detected
- `"animal on road"` - Animal detected
- `"normal traffic"` - Standard driving conditions

### 3. Cosmos Rule Generator

**Model**: nvidia/Cosmos-Reason2-2B

**Input**:
- Image
- Scene signature
- Detected objects
- Uncertainty score

**Process**:
- Generates natural language safety rule
- Format: TRIGGER → ACTION → CONFIDENCE

**Output Example**:
```
TRIGGER: Multiple pedestrians on road without vehicles
ACTION: Reduce speed to 10 mph, increase following distance to 5 seconds
CONFIDENCE: 0.90
```

**Time**: ~300 seconds

### 4. Safety Rulebook

**Storage**: JSON file
**Structure**:
```json
{
  "trigger_condition": "multiple pedestrians",
  "action": "Reduce speed to 10 mph...",
  "confidence": 0.90,
  "times_applied": 3
}
```

**Lookup**: Fast string matching on scene signature  
**Performance**: <0.2 seconds

## Data Flow Example

### First Encounter (Learning)
```
Input: Image with 5 pedestrians, 0 vehicles
  ↓
Detection: 5 persons detected
  ↓
Signature: "multiple pedestrians"
  ↓
Rulebook: No match found
  ↓
Generate: Cosmos creates rule (~300s)
  ↓
Store: Add to rulebook
  ↓
Apply: "Reduce speed to 10 mph"
```

### Second Encounter (Instant Application)
```
Input: Similar image with pedestrians
  ↓
Detection: 5 persons detected
  ↓
Signature: "multiple pedestrians"
  ↓
Rulebook: MATCH FOUND! ✅
  ↓
Apply: Existing rule (~0.2s)
  ↓
Result: 1500x FASTER
```

## Performance Characteristics

| Metric | First Encounter | Future Encounters |
|--------|----------------|-------------------|
| Time | ~300 seconds | ~0.2 seconds |
| Speedup | Baseline | 1500x |
| Cosmos calls | 1 | 0 |
| Rulebook size | +1 rule | No change |

## Key Innovation

**Positive Feedback Loop:**
```
More scenarios → More rules → Faster system
```

As the rulebook grows, the percentage of scenarios requiring slow Cosmos generation decreases, making the overall system faster over time.
