# QBullVME - Semantic, Spatial, Temporal, Multimedia, Multiview Information System

**Versão:** 0.2.7 Alpha  
**Stack:** Qt6 / C++17 / OpenCV / FFmpeg / ONNX Runtime / OpenShot

QBullVME é uma plataforma mission-critical de informação geoespacial multimídia com arquitetura DataSourceLayer→SymbologyView, combinando análise semântica, edição espacial avançada, processamento temporal, e multiview isolado para operações governamentais. O sistema integra processamento de vídeo baseado em nós (node-based), aceleração por IA (ONNX, ComfyUI, Leejet), pipeline de transformações lazy com zero-copy, e sistema de múltiplas views com garantias de isolamento entre saídas críticas.

---

## Índice

1. [Visão Geral](#1-visão-geral)
2. [Arquitetura Geral](#2-arquitetura-geral)
3. [VME-Data 4.0 (Camada de Dados)](#3-vme-data-40-camada-de-dados)
4. [VME-Engine v2.4 (Camada de Processamento)](#4-vme-engine-v24-camada-de-processamento)
5. [Sistema de Nós (Node System)](#5-sistema-de-nós-node-system)
6. [Sistema Semântico](#6-sistema-semântico)
7. [Sistema de Mapeamento Temporal](#7-sistema-de-mapeamento-temporal)
8. [Playback System](#8-playback-system)
9. [Integração com IA](#9-integração-com-ia)
10. [Principais Classes e Hierarquias](#10-principais-classes-e-hierarquias)
11. [Design Patterns](#11-design-patterns)
12. [Dependências Externas](#12-dependências-externas)
13. [Estrutura de Diretórios](#13-estrutura-de-diretórios)
14. [Como Compilar](#14-como-compilar)

---

## 1. Visão Geral

QBullVME é uma ferramenta de edição e análise de vídeo com arquitetura orientada a dados, projetada para workflows que envolvem:

- **Edição não-linear** com pipeline baseado em nós visuais
- **Análise semântica** com attach de metadados a frames individuais
- **Processamento por IA** (detecção de objetos, classificação de cena, texto-para-imagem/vídeo)
- **Transformações temporais** (speed, reverse, cut, chain) com avaliação lazy
- **Gerenciamento inteligente de memória** com frame paging e cache LRU
- **Isolamento de dependências** (OpenShot isolado via Bridge Pattern)

### Inovações Técnicas Principais

- **Qt6-First**: Uso de `QVideoFrame`/`QAudioBuffer` como tipos fundamentais
- **Zero-Copy Frame Processing**: Mínimo overhead de memória
- **Canonical Frame Type**: `VMEFrame` como tipo único de frame em todo o sistema
- **Semantic-Aware Media**: Anotações integradas ao dado de mídia
- **Transform Composition**: Encadeamento lazy de transformações temporais
- **Node-Based Processing**: Programação visual para pipelines de vídeo
- **Bridge Pattern**: OpenShot completamente isolado da lógica de domínio
- **RAII extensivo**: Gerenciamento automático de recursos via handles

## 2. Arquitetura Geral

### 2.1 Camadas

```
┌──────────────────────────────────────────────────────────┐
│                   Application Layer                       │
│   (MainWindow, Nodes UI, Widgets, TimelineEditor)         │
└──────────────────────────────────────────────────────────┘
                            ↓
┌──────────────────────────────────────────────────────────┐
│                  VME-Engine v2.4                          │
│   (Node System, Processing Pipeline, NodeRuntime)         │
└──────────────────────────────────────────────────────────┘
                            ↓
┌──────────────────────────────────────────────────────────┐
│                   VME-Data 4.0                            │
│   (Media, Semantic, Storage, TemporalMapping)             │
└──────────────────────────────────────────────────────────┘
                            ↓
┌──────────────────────────────────────────────────────────┐
│                   Qt6 Foundation                          │
│   (QVideoFrame, QAudioBuffer, QImage, QtNodes)            │
└──────────────────────────────────────────────────────────┘
```

### 2.2 Princípios Arquiteturais

- **Separation of Concerns**: Limites claros entre camadas (Data ↔ Engine ↔ UI)
- **Dependency Inversion**: Dependências em abstrações, não implementações concretas
- **Single Responsibility**: Cada classe tem uma única razão para mudar
- **Open/Closed**: Aberto para extensão, fechado para modificação
- **Interface Segregation**: Interfaces coesas e específicas
- **Lazy Transform**: Transformações temporais só calculam quando o frame é solicitado

## 3. VME-Data 4.0 (Camada de Dados)

É a biblioteca fundamental (`vme-data` / `VME::Data`) que fornece os tipos de mídia, semântica e armazenamento.

### 3.1 VMEFrame — Tipo Canônico de Frame

```cpp
class VMEFrame {
    QVideoFrame m_video;                    // Frame de vídeo Qt6 nativo
    std::optional<QAudioBuffer> m_audio;    // Áudio opcional associado
    qint64 m_pts;                           // Presentation timestamp
    qint64 m_frameNumber;                   // Índice do frame
    QVariantMap m_semantics;               // Metadados semânticos
};
```

Características: thread-safe (Qt COW), GPU-capable, serializável, zero-copy para conversões.

### 3.2 ClipContext — Gerenciador de Recursos (RAII)

````cpp
class ClipContext {
    ContextId m_contextId;                  // ID único do contexto
    SourceId m_sourceId;                    // Fonte original
    GenerationId m_generation;              // Profundidade de transformação
    std::shared_ptr<MaterializedMediaSource> m_source;
    std::shared_ptr<FramePagingSystem> m_paging;
    std::shared_ptr<ISemanticStorage> m_semantics;
};
````

- Owns todos os recursos via RAII
- Gerencia o ciclo de vida dos frames
- Rastreia linhagem de transformações
- Factory methods: `ClipContext::fromFile()`, `ClipContext::derive()`

### 3.3 MaterializedMediaSource — Provedor de Frames

Interface abstrata para obtenção de frames:

```cpp
class MaterializedMediaSource {
    virtual QImage getFrame(qint64 frameNumber) = 0;
    virtual qint64 frameCount() const = 0;
};
```

Implementações:
- `FileBasedSource` — Leitura de arquivo de vídeo
- `PagedMemorySource` — Frames cacheados em RAM
- `TransformedMediaSource` — Aplica transformações lazy

### 3.4 TransformedMediaSource — Transformação Lazy

```cpp
class TransformedMediaSource : public MaterializedMediaSource {
    std::shared_ptr<MaterializedMediaSource> m_source;
    std::unique_ptr<IFrameTransform> m_transform;
    
    QImage getFrame(qint64 frameNumber) override {
        qint64 sourceFrame = m_transform->mapOutputFrameToSource(frameNumber);
        return m_source->getFrame(sourceFrame);
    }
};
```

O frame só é decodificado e transformado quando solicitado (lazy evaluation).

### 3.5 VideoClip - API de Alto Nível

```cpp
class VideoClip : public BaseMediaClip {
    std::shared_ptr<ClipContext> m_context;
    std::unique_ptr<OpenShotBridge> m_bridge;
    std::unique_ptr<FramePagingSystemVME> m_paging;
    
    VMEFramePtr getFrame(qint64 frameNum);
    std::shared_ptr<VideoClip> createSubclip(qint64 start, qint64 end);
    // Templates para processBatch e processFrame com callbacks
};
```

### 3.6 AudioClip - Suporte a Áudio

```cpp
class AudioClip : public BaseMediaClip {
    std::shared_ptr<AudioClipContext> m_context;
    
    QAudioBuffer getBuffer(qint64 startSample, qint64 sampleCount);
    std::shared_ptr<AudioClip> withSpeed(double speed);
    std::shared_ptr<AudioClip> withVolume(double gain);
};
```

### 3.7 FramePagingSystemVME — Cache Inteligente

Sistema de paginação com:
- LRU eviction para frames não utilizados
- Swap automático para disco quando memória é insuficiente
- Prefetch preditivo (prefetchRange)
- Memory limits configuráveis

### 3.8 Storage Layer

- **ISemanticStorage**: Interface para armazenamento semântico
- **InMemorySemanticStorage**: Rápido, volátil
- **DatabaseSemanticStorage**: SQLite persistente
- **TimelineSemanticStorage**: Otimizado para queries de timeline
- **GlobalFramePagePool**: Pool global de frames com deduplicação automática

---

## 4. VME-Engine v2.4 (Camada de Processamento)

### 4.1 Node System Architecture

O VME-Engine implementa o padrão **Engine-Delegator-Widget** (arquitetura em 3 camadas para cada nó):

```
┌───────────────────────────────────────────────────────┐
│  BaseNodeDataDelegator (NodeDelegateModel)             │
│  ├── Gerencia entradas/saídas                          │
│  ├── Conecta engine ↔ widget                           │
│  └── Implementa NodeRuntime                            │
├───────────────────────────────────────────────────────┤
│  BaseNodeEngine (QObject)                              │
│  ├── Lógica de processamento (onStart)                 │
│  ├── State machine: Idle→Ready→Processing→Finished     │
│  └── Error recovery com retry                          │
├───────────────────────────────────────────────────────┤
│  BaseNodeWidget (QWidget)                              │
│  └── Interface visual do nó                            │
└───────────────────────────────────────────────────────┘
```

### 4.2 BaseNodeEngine — Núcleo de Processamento

```cpp
class BaseNodeEngine : public QObject {
    enum class State { Idle, Ready, Processing, Finished, Error, Canceled };
    enum class ProcessingMode { Auto, Manual };
    
    virtual void onStart() = 0;   // Hook de processamento
    virtual void onStop();         // Hook de parada
    virtual void onRetry();        // Tentativa de recuperação
    
    void configure(const QVariantMap &parameters);  // Config parametrizada
    
    // Sistema de retry com limite configurável
    void setMaxRetries(int count);
    int maxRetries() const;
    
signals:
    void progressChanged(int percent);
    void dataAvailable(NodeDataPtr data);
    void stateChanged(State state, const QString &details);
    void errorOccurred(const ErrorContext &context);
};
```

### 4.3 BaseNodeDataDelegator — Wrapper QtNodes

Herda de `QtNodes::NodeDelegateModel` e implementa `NodeRuntime`, fornecendo:

```cpp
class BaseNodeDataDelegator : public QtNodes::NodeDelegateModel, public NodeRuntime {
    void setEngine(std::unique_ptr<BaseNodeEngine> engine);
    void setWidget(std::unique_ptr<BaseNodeWidget> widget);
    
    ProcessingMode processingMode() const; // AutoOnly, ManualOnly, Both
    
    // NodeRuntime interface (controle global de playback)
    void runtimeStartProcessing();
    void runtimePauseProcessing();
    void runtimeStopProcessing();
    void runtimeResetNodeState();
    
    // Bypass: permite pular o processamento do nó
    void setBypass(bool enabled);
    bool supportsBypass() const;
};
```

### 4.4 NodeRuntime — Interface de Controle Global

```cpp
class NodeRuntime {
    virtual void runtimeStartProcessing() = 0;
    virtual void runtimePauseProcessing() = 0;
    virtual void runtimeStopProcessing() = 0;
    virtual void runtimeResetNodeState() = 0;
    virtual void runtimeEnablePlayProcessing(bool enable) = 0;
};
```

Permite que o `NodeFlowEngine` controle todos os nós de forma uniforme durante uma execução global (playback).

### 4.5 NodeFlowEngine — Orquestrador de Pipeline

Gerencia o fluxo de dados entre nós conectados, acionando processamento em cascata:
- Quando dados chegam a um nó de entrada (ex: VideoInput), o engine propaga o processamento
- Cada nó downstream é automaticamente acionado quando seus dados de entrada ficam prontos
- Suporta modos Auto e Manual de processamento por nó

---

## 5. Sistema de Nós (Node System)

### 5.1 Tipos de Nós

**Input Nodes:**
| Nó | Descrição |
|---|---|
| `VideoInputNode` | Carrega arquivos de vídeo |
| `ImageLoaderNode` | Carrega imagens estáticas |
| `TextListNode` | Gera dados de texto em lista |

**Processing Nodes:**
| Nó | Descrição |
|---|---|
| `VideoMultiplySpeedNode` | Transformação de velocidade (ex: 4x) |
| `ONNXDetectionNode` | Detecção de objetos com ONNX |
| `ONNXClassificationModel` | Classificação de frames |
| `VideoFadeInModel` | Efeito de fade in |
| `PitchShiftEffectModel` | Alteração de pitch de áudio (RubberBand) |
| `ImageEditorModel` | Edição de imagem integrada |
| `ClipCombineModel` | Combinação de clipes |
| `ClipSplitterModel` | Divisão de clipes |
| `AudioVideoCombineNode` | Combinação de streams de áudio/vídeo |

**Analysis Nodes:**
| Nó | Descrição |
|---|---|
| `SemanticAnalysisModel` | Análise semântica de cena |
| `SemanticFilterModel` | Filtragem por dados semânticos |
| `SmartSemanticFilterModel` | Filtro semântico inteligente |
| `ObjectAnalysisModel` | Análise de objetos detectados |
| `TimelineFilterModel` | Filtro de timeline baseado em IA |
| `Places365SceneDetectionModel` | Detecção de cena (Places365) |

**AI Generation Nodes:**
| Nó | Descrição |
|---|---|
| `ComfyTextToVideoModel` | Texto-para-vídeo via ComfyUI |
| `LeejetTextToImageModel` | Texto-para-imagem via Leejet/Stable Diffusion |

**Output Nodes:**
| Nó | Descrição |
|---|---|
| `VideoClipShowNode` | Preview e playback do clipe |
| `ImageShowModel` | Exibição de imagem |
| `DataListDispatcherNode` | Dispatcher de listas de dados |
| `TimelineSegmentEditorNode` | Editor de segmentos na timeline |

### 5.2 Node Data Types (Sistema de Tipos)

```cpp
// Tipos base de dados que trafegam entre nós
class VideoClipData : public NodeData {
    std::shared_ptr<VideoClip> clip;
};

class SemanticMediaClipData : public NodeData {
    std::shared_ptr<SemanticMediaClip> clip;
};

class DetectionResultData : public NodeData {
    std::vector<Detection> detections;
};

// Hierarquia VME:
// VMENodeData (base) → VideoClipData, AudioClipData, ImageClipData,
//                      SemanticMediaClipData, FrameData, DetectionData,
//                      SemanticEventData, TextListData, AIAVData
```

### 5.3 Fluxo de Processamento Típico

```
VideoInputNode → VideoMultiplySpeedNode → ONNXDetectionNode → VideoClipShowNode
     │                    │                       │                  │
     │  loadClip()        │  createSpeedTimeline() │  detect()       │  display()
     │  emit VideoClip    │  emit VideoClip        │  emit Detections │  render()
     ▼                    ▼                       ▼                  ▼
  Arquivo MP4          Speed 4x                 Objetos            Preview
```

O fluxo de processamento global é disparado por:
1. `MainWindow::onPlayClicked()`
2. `NodeFlowWidget::startGlobalPlayback()`
3. `NodeFlowEngine::start()` que itera sobre todos os nós
4. Cada nó chama `runtimeStartProcessing()` em ordem topológica

---

## 6. Sistema Semântico

### 6.1 SemanticMediaClip — Mídia com Anotações

```cpp
class SemanticMediaClip {
    std::shared_ptr<VideoClip> m_baseClip;
    std::shared_ptr<ISemanticStorage> m_storage;
    
    std::shared_ptr<SemanticLayer> createLayer(const QString& name);
    std::vector<std::shared_ptr<SemanticEntity>> getEntitiesAtTime(qint64 timeMs);
};
```

### 6.2 SemanticEntity — Entidade Semântica

```cpp
struct SemanticEntity {
    QString id;
    QString label;
    TimeRange timeRange;
    std::shared_ptr<Geometry> geometry;  // BBox, Polygon, Keypoints, Mask
    QVariantMap attributes;
    double confidence;
};
```

### 6.3 Camadas Semânticas (EventType)

| Tipo | Descrição |
|---|---|
| `DETECTION` | Resultados de detecção de objetos |
| `CLASSIFICATION` | Classificação de frames |
| `SEGMENTATION` | Segmentação pixel-wise |
| `TRANSCRIPTION` | Fala para texto |
| `USER_ANNOTATION` | Anotações manuais do usuário |

### 6.4 Tipos de Geometria

- `BBoxGeometry` — Bounding box (x, y, width, height)
- `PolygonGeometry` — Polígono arbitrário
- `KeypointsGeometry` — Keypoints (pose estimation)
- `MaskGeometry` — Máscara binária pixel-wise

---

## 7. Sistema de Mapeamento Temporal

### 7.1 TemporalMapping — Interface Base

```cpp
class TemporalMapping {
    virtual Time map(Time t) const = 0;                    // t_out → t_in
    virtual Duration transformDuration(Duration d) const = 0;
    virtual std::unique_ptr<TemporalMapping> compose(
        const TemporalMapping& other) const = 0;           // Compõe mappings
    virtual bool isIdentity() const = 0;
};
```

### 7.2 Hierarquia de Mappings

```
TemporalMapping (interface base abstrata)
 ├── IdentityMapping    — Nenhuma transformação (map(t) = t)
 ├── ScaleMapping       — Speed/time stretch (map(t) = t * factor)
 ├── ReverseMapping     — Reverso (map(t) = duration - t)
 ├── CutMapping         — Corte/trim temporal (janela fixa)
 └── ChainedMapping     — Composição genérica de múltiplos mappings
```

### 7.3 Composição de Mappings

```cpp
auto scale = std::make_unique<ScaleMapping>(2.0);   // 2x speed
auto cut = std::make_unique<CutMapping>(5s, 15s);   // Janela 5-15s
auto composed = scale->compose(*cut);               // composição lazy

// Otimizações automáticas:
// Scale ∘ Scale = Scale(a * b)
// Identity ∘ X = X
// X ∘ Identity = X
```

---

## 8. Playback System

### 8.1 MediaPlaybackEngine

Engine de playback baseado em `QVideoFrame`, com suporte a seek frame-exato:

```cpp
class MediaPlaybackEngine : public QObject {
    void setSource(std::shared_ptr<MediaSource> source);
    void play();
    void pause();
    void seek(FrameTime time);
    
signals:
    void frameReady(VMEFramePtr frame);
    void stateChanged(PlaybackState state);
};
```

### 8.2 SemanticClipMediaSource

Bridge entre o playback e os dados semânticos:

```cpp
class SemanticClipMediaSource : public MediaSource {
    VMEFramePtr getFrame(qint64 frameNumber) override;
    bool hasSemanticDataAt(qint64 frameNumber) const override;
    std::vector<OverlayObject> collectLayerData(qint64 frameNumber);
};
```

Renderiza overlays semânticos durante o playback (bounding boxes, labels, etc).

### 8.3 Audio Engine

Engine de áudio com threading dedicado:
- `AudioClock` — Clock síncrono para alinhamento áudio/vídeo
- `AudioDecoderThread` — Decodificação em thread separada
- `AudioOutputDevice` — Saída de áudio Qt6
- `LockFreeRingBuffer` — Buffer lock-free para comunicação entre threads
- `RubberBandPitchShift` — Correção de pitch em speed change usando RubberBand

---

## 9. Integração com IA

### 9.1 ONNX Runtime

O projeto integra modelos de IA via ONNX Runtime para inferência local:

- **ONNXDetectionModel** — Detecção de objetos (YOLO, SSD, etc.) com saída de bounding boxes, confiança e classes
- **ONNXClassificationModel** — Classificação de frames individuais
- **TimelineFilterModel** — Filtro inteligente de timeline baseado em predições de IA

```cpp
class ONNXDetectionModel : public CustomBaseNodeDataDelegator {
    // Carrega modelo .onnx
    // Processa frames e emite DetectionResultData
    // Overlay de bounding boxes no preview
};
```

### 9.2 ComfyUI (Texto-para-Vídeo)

- **ComfyTextToVideoModel** — Geração de vídeo a partir de descrição textual via backend ComfyUI

### 9.3 Leejet / Stable Diffusion (Texto-para-Imagem)

- **LeejetTextToImageModel** — Geração de imagens via Stable Diffusion (Leejet fork)
- Suporte a CUDA 12 com dependências (cublas, cudart)

### 9.4 Análise Semântica

- **SemanticAnalysisModel** — Análise semântica de cenas
- **ObjectAnalysisModel** — Pós-processamento de detecções
- **Places365SceneDetectionModel** — Classificação de cena usando modelo Places365
- **SmartSemanticFilterModel** — Filtro semântico adaptativo que combina múltiplos critérios

---

## 10. Principais Classes e Hierarquias

### 10.1 Media Classes

```
BaseMediaClip (abstract)
 ├── VideoClip
 │    ├── context(): ClipContext
 │    ├── getFrame(qint64) → VMEFramePtr
 │    ├── createSubclip(start, end) → VideoClip
 │    ├── processFrame/processBatch (templates)
 │    └── renderToFile() / applyEffect()
 ├── AudioClip
 │    ├── context(): AudioClipContext
 │    ├── getBuffer() → QAudioBuffer
 │    ├── withSpeed(speed) → AudioClip
 │    └── withVolume(gain) → AudioClip
 └── ImageClip
```

### 10.2 Source & Storage Classes

```
MaterializedMediaSource (abstract)
 ├── FileBasedSource (arquivo de vídeo)
 ├── PagedMemorySource (cache em RAM)
 └── TransformedMediaSource (transformação lazy)

FramePagingSystem (cache)
 └── FramePagingSystemVME (LRU + swap + prefetch)

ClipContext (RAII resource owner)
 ├── ContextId, SourceId, GenerationId
 ├── TransformSignature
 └── Factory: fromFile(), fromSource(), derive()
```

### 10.3 Temporal Mapping Classes

```
TemporalMapping (interface)
 ├── IdentityMapping: map(t) = t
 ├── ScaleMapping:    map(t) = t * factor
 ├── ReverseMapping:  map(t) = duration - t
 ├── CutMapping:      map(t) = start + t (janela fixa)
 └── ChainedMapping:  composição genérica

Time / Duration (value objects type-safe)
```

### 10.4 Engine & Node Classes

```
BaseNodeEngine (QObject)
 ├── State machine: Idle → Ready → Processing → Finished | Error
 ├── Error recovery: retry, maxRetries, ErrorContext
 └── ProcessingMode: Auto / Manual

BaseNodeDataDelegator : NodeDelegateModel, NodeRuntime
 ├── setEngine(), setWidget()
 ├── ProcessingMode (AutoOnly, ManualOnly, Both)
 ├── Bypass support
 └── Instance identification (auto-generated name)

NodeRuntime (interface)
 └── start/pause/stop/reset processing

NodeFlowEngine (orquestrador)
 └── startGlobalPlayback() em ordem topológica
```

### 10.5 Semantic Classes

```
SemanticMediaClip
 ├── VideoClip m_baseClip
 └── ISemanticStorage m_storage

SemanticLayer
 └── List<SemanticEntity>

SemanticEntity
 ├── id, label, timeRange, geometry, attributes, confidence
 └── Geometry (BBox, Polygon, Keypoints, Mask)

ISemanticStorage
 ├── InMemorySemanticStorage
 ├── DatabaseSemanticStorage (SQLite)
 └── TimelineSemanticStorage

AnalysisPipeline
 └── PipelineState, IAnalysisProcessor, ModelInfo
```

### 10.6 Audio Engine Classes

```
AudioEngine (thread dedicada)
 ├── AudioClock
 ├── AudioDecoderThread
 ├── AudioOutputDevice
 └── LockFreeRingBuffer

RubberBandPitchShift
 └── Correção de pitch usando RubberBand library
```

---

## 11. Design Patterns

| Padrão | Onde é Usado | Descrição |
|---|---|---|
| **Strategy** | `IFrameTransform`, `TemporalMapping`, `IAudioTransform` | Algoritmos de transformação intercambiáveis |
| **Bridge** | `OpenShotBridge`, `AudioClipContext` | Isola dependências externas (OpenShot, FFmpeg) do código de domínio |
| **Factory Method** | `ClipContext::fromFile()`, `MaterializedSourceFactory` | Encapsula criação complexa de objetos |
| **Observer** | Qt Signals/Slots em todo o sistema | Desacopla componentes via notificação de eventos |
| **RAII** | `ClipContext`, `FrameHandle`, `PixelMapper` | Gerenciamento automático de recursos |
| **Lazy Evaluation** | `TransformedMediaSource`, `TransformedAudioSource` | Computação adiada até o momento do acesso |
| **Template Method** | `BaseNodeEngine::onStart()`, `BaseNodeDataDelegator` | Hooks virtuais para personalização |
| **Chain of Responsibility** | Pipeline de nós conectados | Dados fluem entre nós em cadeia |
| **MVC** | `BaseNodeDataDelegator` (Controller) + `BaseNodeEngine` (Model) + `BaseNodeWidget` (View) | Separação entre dados, lógica e apresentação |
| **State** | `BaseNodeEngine::State` (Idle, Processing, Error, etc.) | Máquina de estados para processamento |
| **Singleton** | `GlobalFramePagePool::instance()` | Pool global de frames com deduplicação |
| **Value Object** | `Time`, `Duration`, `TimeRange`, `ContextId` | Objetos imutáveis type-safe |
| **Composite** | `ChainedMapping` | Compõe múltiplos TemporalMapping em um só |
| **Adapter** | `ShotAVDataAdapter`, `TimelineAggregatorAdapter` | Adapta dados legados para nova arquitetura |

---

## 12. Dependências Externas

| Dependência | Versão | Uso |
|---|---|---|
| **Qt6** | 6.x | UI, Multimedia, Widgets, Charts, OpenGL, Network, SQL, Concurrent |
| **QtNodes** | modificado | Framework de editor de nós visuais |
| **OpenShot** | custom (libopenshot-qt6) | Decodificação/encoding de vídeo (isolado via Bridge) |
| **OpenShot Audio** | custom | Engine de áudio do OpenShot |
| **OpenCV** | 4.x | Processamento de imagem, visão computacional |
| **FFmpeg** | via vcpkg | Codecs de áudio/vídeo (libavcodec, libavformat, swscale, swresample) |
| **ONNX Runtime** | via vcpkg | Inferência de modelos de IA (detecção, classificação) |
| **RubberBand** | via vcpkg | Correção de pitch em tempo real (stretch de áudio) |
| **SQLite** | via vcpkg | Armazenamento persistente de dados semânticos |
| **JsonCpp** | via vcpkg | Serialização JSON para projetos e configuração |
| **CUDA 12** | opcional | Aceleração GPU para inferência (Leejet, ONNX) |

## 13. Estrutura de Diretórios

```
QBullVME/
├── src/                          # Código fonte principal
│   ├── main.cpp                  # Entry point
│   ├── mainwindow.h/.cpp        # Janela principal
│   ├── CMakeLists.txt            # Build principal
│   │
│   ├── core/                     # Core library (engine interno)
│   │   └── temporal/             # TemporalMapping system (VME 5.0)
│   │       ├── TemporalMapping.h # Interface base
│   │       ├── IdentityMapping   # Sem transformação
│   │       ├── ScaleMapping      # Speed
│   │       ├── ReverseMapping    # Reverso
│   │       ├── CutMapping        # Cut/trim
│   │       └── ChainedMapping    # Composição
│   │
│   ├── vme-data/                 # VME-Data library (lib separada: VME::Data)
│   │   ├── CMakeLists.txt
│   │   ├── core/                 # TimeRange, AttributeBundle, Metadata, SourceContext
│   │   ├── media/                # VideoClip, AudioClip, ImageClip, VMEFrame, BaseMediaClip
│   │   ├── storage/              # ClipContext, FramePaging, MaterializedSource, ISemanticStorage
│   │   ├── semantic/             # SemanticEntity, SemanticLayer
│   │   ├── geometry/             # BBox, Polygon, Keypoints, Mask
│   │   ├── timeline/             # SemanticTrack, SemanticEvent, indices (Temporal, Spatial, Class)
│   │   ├── analyzable/           # SemanticMediaClip, MetadataManager
│   │   ├── pipeline/             # AnalysisPipeline, ModelInfo
│   │   ├── project/              # VideoProject
│   │   ├── migration/            # OpenShotBridge, MediaClipFactory, ShotAVDataAdapter
│   │   └── processing/           # BufferHandles, ClipContextCache, TimeTransform
│   │
│   ├── engine/                   # VME-Engine v2.4
│   │   ├── BaseNodeEngine.h/.cpp # Engine base com state machine
│   │   ├── BaseNodeDataDelegator # Wrapper QtNodes + NodeRuntime
│   │   ├── DefaultNodeEngine     # Engine padrão
│   │   ├── NodeFlowEngine        # Orquestrador de pipeline
│   │   ├── NodeRuntime.h         # Interface de controle global
│   │   └── NodeEmbeddedWidget    # Widget container para nós
│   │
│   ├── model/                    # Implementações de nós
│   │   ├── CustomBaseNodeDataDelegator
│   │   ├── VideoMultiplySpeedModel
│   │   ├── VideoFadeInModel
│   │   ├── ClipCombineModel / ClipSplitterModel
│   │   ├── PitchShiftEffectModel
│   │   ├── ImageEditorModel
│   │   ├── VMENodePainterDelegate / VMENodeGeometry
│   │   ├── ai/                   # ONNXDetection, ONNXClassification, TimelineFilter
│   │   ├── semantic/             # SemanticAnalysis, SemanticFilter, SmartSemanticFilter, ObjectAnalysis
│   │   ├── comfy/                # ComfyTextToVideoModel
│   │   └── leejet/               # LeejetTextToImageModel
│   │
│   ├── nodes/                    # Nós individuais (engine + widget + node)
│   │   ├── videoInput/
│   │   ├── VideoClipShow/
│   │   ├── ONNXDetection/
│   │   ├── VideoMultiplySpeed/
│   │   ├── ImageEditor/
│   │   ├── ImageLoader/
│   │   ├── LeejetTextToImage/
│   │   ├── ComfyImageToVideo/
│   │   ├── SemanticMediaFilter/
│   │   ├── ClipCombine/
│   │   ├── Dispatcher/
│   │   ├── TextList/
│   │   ├── TimelineSegmentEditor/
│   │   └── PitchShiftEffect/
│   │
│   ├── nodeData/                 # NodeData types (dados entre nós)
│   │   ├── QVideoClip.h
│   │   ├── ShotNodeData.h
│   │   ├── SemanticDataContainer
│   │   ├── core/BaseNodeData.h
│   │   └── vme/
│   │       ├── base/VMENodeData.h
│   │       ├── media/            # VideoClipData, AudioClipData, FrameData, etc.
│   │       ├── semantic/         # DetectionData, SemanticEventData
│   │       ├── timeline/         # SemanticTimelineData, SemanticTrackData
│   │       ├── generic/          # TextListData, ListData, TextData
│   │       └── ai/               # AIAVData, AIDataBase, ModelMetadata
│   │
│   ├── widget/                   # Widgets reutilizáveis
│   │   ├── VideoPlayer, VideoQtPlayer
│   │   ├── ShotVideoPlayer
│   │   ├── VideoDisplayParameterWidget
│   │   ├── TransformerTreeWidget
│   │   └── NavigatorDock
│   │
│   ├── MediaPlayer/              # Sistema de playback
│   │   ├── MediaPlaybackEngine
│   │   ├── MediaPlayerWidget
│   │   ├── SemanticDataPanel, SemanticOverlaySystem
│   │   ├── VMEFrameUtils
│   │   ├── Playbackclock
│   │   └── audio-engine/         # AudioEngine, AudioClock, AudioDecoderThread, etc.
│   │
│   ├── TimeLine/                 # Timeline UI e regras
│   │   ├── TemporalOperators
│   │   ├── TimelineRuleEngine
│   │   ├── TimelineVisualizerWidget
│   │   └── TimelineChartsWidget
│   │
│   ├── Effects/                  # Efeitos de áudio
│   │   └── RubberBandPitchShift
│   │
│   └── audio/                    # Biblioteca de processamento de áudio
│       └── CMakeLists.txt
│
├── thirdparty/                   # Dependências de terceiros
│   └── src/nodeeditor/           # QtNodes modificado
│
├── docs/                         # Documentação técnica detalhada
│   ├── ARQUITETURA.md
│   ├── VME-Architecture-Documentation.md
│   ├── MediaDB_v0.5_Architecture.md
│   ├── VME_5.0_Technical_Documentation_01.md
│   ├── TimelineEditor_*.md       # Documentação do Timeline Editor
│   ├── Pipeline_MediaDB_v08_Analysis.md
│   └── vme-*.plantuml            # Diagramas UML/PlantUML da arquitetura
│
├── ui/                           # Arquivos .ui (Qt Designer)
│
├── build/                        # Artefatos de build
│
└── python_c++/                   # Bridge Python/C++ para prototipação
    ├── py_video_backend.py
    ├── PyVideoClip.cpp/.h
```

## 14. Como Compilar

### Pré-requisitos

- **CMake** ≥ 3.16
- **Qt6** (Core, Gui, Widgets, Multimedia, MultimediaWidgets, OpenGL, Charts, Network, SQL, Concurrent)
- **vcpkg** com pacotes: OpenCV, FFmpeg, ONNX Runtime, RubberBand, SQLite, JsonCpp
- **libopenshot-qt6** (build customizado)
- **libopenshot-audio** (build customizado)
- **QtNodes** (fork modificado em `thirdparty/src/nodeeditor`)

### Passos

```bash
# 1. Configure com CMake
cd build
cmake .. -G "Visual Studio 17 2022" ^
    -DCMAKE_BUILD_TYPE=Release ^
    -DCMAKE_TOOLCHAIN_FILE="[vcpkg-root]/scripts/buildsystems/vcpkg.cmake" ^
    -DVCPKG_TARGET_TRIPLET=x64-windows

# 2. Compile
cmake --build . --config Release

# 3. Execute
./bin/QBullVME.exe
```

### Estrutura de Build

O build gera duas bibliotecas principais e o executável:
- **vme_data.dll** — VME-Data 4.0 library (VME::Data)
- **audio_processing.dll** — Audio Processing library (Audio::Processing)
- **QBullVME.exe** — Aplicação principal

---

*Documentação gerada em Junho 2026 para QBullVME v0.2.7 Alpha*


