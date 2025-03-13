# Документация системы синхронного перевода

В этом документе представлены детальные схемы и описания работы четырех ключевых сервисов системы синхронного перевода речи.

## Содержание
1. [Общая архитектура](#общая-архитектура)
2. [synchro-python (Клиент)](#synchro-python-клиент)
3. [streaming-proxy (Диспетчер)](#streaming-proxy-диспетчер)
4. [whisper_grpc_api (Распознавание речи)](#whisper_grpc_api-распознавание-речи)
5. [tts_grpc_api (Синтез речи)](#tts_grpc_api-синтез-речи)

## Общая архитектура

Система синхронного перевода состоит из четырех основных компонентов, взаимодействующих для обеспечения потокового перевода речи в реальном времени.

```mermaid
graph TB
    User["Пользователь"] -->|"Говорит на исходном языке"| SynchroClient
    
    subgraph "Общая архитектура"
        SynchroClient["synchro-python<br>Клиент"] <-->|"Двунаправленный<br>аудиопоток"| StreamingProxy
        
        StreamingProxy["streaming-proxy<br>Диспетчер"] -->|"Передача<br>аудио"| WhisperAPI
        WhisperAPI["whisper_grpc_api<br>ASR"] -->|"Распознанный<br>текст"| StreamingProxy
        
        StreamingProxy -->|"Запрос<br>перевода"| LLM["LLM сервис"]
        LLM -->|"Переведенный<br>текст"| StreamingProxy
        
        StreamingProxy -->|"Текст для<br>синтеза"| TTS
        TTS["tts_grpc_api<br>TTS"] -->|"Синтезированная<br>речь"| StreamingProxy
    end
    
    StreamingProxy -->|"Переведенная речь"| SynchroClient
    SynchroClient -->|"Воспроизведение<br>перевода"| User
    
    classDef main fill:#f9f,stroke:#333,stroke-width:2px;
    classDef secondary fill:#bbf,stroke:#333,stroke-width:1px;
    classDef external fill:#bfb,stroke:#333,stroke-width:1px;
    
    class SynchroClient,StreamingProxy,WhisperAPI,TTS main;
    class LLM secondary;
    class User external;
```

## synchro-python (Клиент)

synchro-python - это клиентское приложение, которое взаимодействует с пользователем, обрабатывает аудиовход и аудиовыход.

```mermaid
flowchart TB
    subgraph "synchro-python"
        direction TB
        
        subgraph "Аудиовходы"
            InputChannel["Микрофон<br>InputChannelNode"] 
            InputFile["Аудиофайл<br>InputFileNode"]
        end
        
        subgraph "Процессоры"
            VADNode["Детектор голоса<br>VADNode"]
            DenoiserNode["Шумоподавление<br>DenoiserNode"]
            NormalizerNode["Нормализация<br>NormalizerNode"]
            ResamplerNode["Ресемплер<br>ResamplerNode"]
            MixerNode["Микшер<br>MixerNode"]
        end
        
        InputChannel -->|"Аудиопоток"| MixerNode
        InputFile -->|"Аудиопоток"| MixerNode
        
        MixerNode -->|"Объединенный<br>поток"| VADNode
        VADNode -->|"Аудиопоток<br>с голосом"| DenoiserNode
        DenoiserNode -->|"Очищенный<br>аудиопоток"| NormalizerNode
        NormalizerNode -->|"Нормализованный<br>аудиопоток"| ResamplerNode
        
        ResamplerNode -->|"Подготовленный<br>аудиопоток"| ConnectorNode["Сетевой коннектор<br>SeamlessConnectorNode"]
        
        ConnectorNode <-->|"Socket.IO<br>WebSocket"| StreamingProxy["streaming-proxy"]
        
        ConnectorNode -->|"Переведенный<br>аудиопоток"| OutputProcessor["Процессор вывода"]
        
        OutputProcessor -->|"Аудиопоток"| OutputChannel["Динамики<br>OutputChannelNode"]
        OutputProcessor -->|"Аудиопоток"| OutputFile["Запись в файл<br>OutputFileNode"]
        
        GraphManager["Менеджер графа<br>GraphManager"] <-->|"Управление<br>компонентами"| ConnectorNode
        GraphManager <-->|"Управление<br>компонентами"| MixerNode
        GraphManager <-->|"Управление<br>компонентами"| InputChannel
        GraphManager <-->|"Управление<br>компонентами"| OutputChannel
        
        ConfigManager["Конфигурация<br>Настройки языков"] -->|"Параметры"| GraphManager
        UserInterface["CLI-интерфейс<br>пользователя"] -->|"Команды"| GraphManager
    end
    
    classDef core fill:#f9f,stroke:#333,stroke-width:2px;
    classDef processor fill:#bfb,stroke:#333,stroke-width:1px;
    classDef io fill:#bbf,stroke:#333,stroke-width:1px;
    classDef manager fill:#ddd,stroke:#333,stroke-width:1px;
    classDef external fill:#fbb,stroke:#333,stroke-width:1px;
    
    class ConnectorNode,GraphManager core;
    class VADNode,DenoiserNode,NormalizerNode,ResamplerNode,MixerNode,OutputProcessor processor;
    class InputChannel,InputFile,OutputChannel,OutputFile io;
    class ConfigManager,UserInterface manager;
    class StreamingProxy external;
```

### Основные процессы в synchro-python:

1. **Захват аудио**:
   - Получение аудиопотока с микрофона пользователя
   - Обработка аудио (нормализация, подавление шума)
   - Разделение аудио на фрагменты для передачи

2. **Передача аудио**:
   - Установление и поддержание WebSocket соединения с streaming-proxy
   - Потоковая передача аудиофрагментов
   - Обработка сетевых ошибок и повторные подключения

3. **Получение переведенного аудио**:
   - Прием аудиопотока с переведенной речью
   - Буферизация для плавного воспроизведения
   - Контроль задержки и синхронизации

4. **Пользовательский интерфейс**:
   - Выбор исходного и целевого языков
   - Управление режимами перевода (последовательный/синхронный)
   - Индикация процессов и состояния системы

### Технические детали:

- **Язык программирования**: Python
- **Архитектура**: Графовая система с узлами и рёбрами для обработки аудиопотоков
- **Протоколы связи**: Socket.IO через WebSocket для аудиопотоков
- **Формат аудио**: 16-битный PCM, частота дискретизации от 16 кГц до 48 кГц, моно
- **Конфигурация**: Hydra для управления конфигурацией и параметрами
- **Клиентские библиотеки**: PyAudio для захвата и воспроизведения аудио, SimpleClient для Socket.IO
- **Обработка аудио**: Модули VAD (Voice Activity Detection), шумоподавление, нормализация, ресемплинг
- **Форматы ввода/вывода**: Микрофон, аудиофайлы, динамики, запись в файл
- **Менеджер графа**: Управление жизненным циклом узлов обработки, координация потоков данных

## streaming-proxy (Диспетчер)

streaming-proxy - центральный компонент системы, который координирует процесс перевода, маршрутизирует потоки данных между различными сервисами и обрабатывает аудио- и текстовые потоки.

```mermaid
flowchart TB
    subgraph "streaming-proxy"
        direction TB
        
        SocketIOServer["Socket.IO Server<br>AsyncServer"] -->|"Обработка<br>событий"| SessionManager["Менеджер сессий<br>SessionManager"]
        
        SessionManager -->|"Создание и<br>управление"| PipelineManager["Менеджер пайплайнов<br>PipelineManager"]
        
        subgraph "Пайплайн (PipelineCore)"
            SttLayer["Слой распознавания речи<br>SttLayer"] -->|"Распознанный<br>текст"| TranslationLayer["Слой перевода<br>TranslationLayer"]
            TranslationLayer -->|"Переведенный<br>текст"| TtsLayer["Слой синтеза речи<br>TtsLayer"]
        end
        
        PipelineManager -->|"Создание и<br>управление"| SttLayer
        TtsLayer -->|"Аудио<br>результат"| PipelineCallback["Callback<br>функция"]
        PipelineCallback -->|"Отправка<br>аудио"| SocketIOServer
        
        subgraph "Коннекторы к внешним сервисам"
            WhisperConnector["Коннектор ASR<br>WhisperSttConnector"] <-->|"gRPC"| WhisperAPI["whisper_grpc_api"]
            LlmConnector["Коннектор LLM<br>LlamaCppServerConnector"] <-->|"HTTP API"| LLM["LLM сервис"]
            TtsConnector["Коннектор TTS<br>PiperVoskConnector"] <-->|"gRPC"| TTS["tts_grpc_api"]
        end
        
        SttLayer <-->|"Запросы и<br>ответы"| WhisperConnector
        TranslationLayer <-->|"Запросы и<br>ответы"| LlmConnector
        TtsLayer <-->|"Запросы и<br>ответы"| TtsConnector
        
        RemoteLogger["Логгер событий<br>RemoteLogger"] <-->|"Логирование"| SttLayer
        RemoteLogger <-->|"Логирование"| TranslationLayer
        RemoteLogger <-->|"Логирование"| TtsLayer
        RemoteLogger -->|"Отправка<br>логов"| SocketIOServer
        
        LlmSteps["Шаги LLM<br>BaseStep"] <-->|"Обработка<br>запросов"| TranslationLayer
    end
    
    Client["synchro-python"] <-->|"Socket.IO<br>WebSocket"| SocketIOServer
    
    classDef core fill:#f9f,stroke:#333,stroke-width:2px;
    classDef layer fill:#bfb,stroke:#333,stroke-width:1px;
    classDef connector fill:#bbf,stroke:#333,stroke-width:1px;
    classDef external fill:#fbb,stroke:#333,stroke-width:1px;
    classDef manager fill:#ddd,stroke:#333,stroke-width:1px;
    
    class SocketIOServer,PipelineManager,SessionManager core;
    class SttLayer,TranslationLayer,TtsLayer,LlmSteps layer;
    class WhisperConnector,LlmConnector,TtsConnector,PipelineCallback connector;
    class WhisperAPI,LLM,TTS,Client external;
    class RemoteLogger manager;
```

### Основные процессы в streaming-proxy:

1. **Управление сессиями и соединениями**:
   - Прием входящих Socket.IO соединений от клиентов
   - Настройка потоков перевода с выбором языков источника и назначения
   - Создание и управление пайплайнами обработки для каждой сессии
   - Обработка событий подключения/отключения клиентов

2. **Организация пайплайна обработки**:
   - Создание трехслойной архитектуры (распознавание, перевод, синтез)
   - Асинхронная обработка потоков данных между слоями
   - Параллельное выполнение циклов обработки для каждого слоя
   - Управление жизненным циклом пайплайнов

3. **Распознавание речи (SttLayer)**:
   - Отправка аудиоданных через gRPC в Whisper API
   - Потоковое получение распознанного текста
   - Фильтрация стоп-слов и буферизация результатов
   - Сегментация текста для обработки

4. **Перевод текста (TranslationLayer)**:
   - Пошаговая обработка текста через различные этапы LLM
   - Гейтинг контента для фильтрации ненужных фрагментов
   - Формирование запросов к LLM API с использованием шаблонов
   - Обработка ответов и поддержание контекста перевода

5. **Синтез речи (TtsLayer)**:
   - Отправка переведенного текста в TTS сервис через gRPC
   - Получение фрагментов синтезированной речи
   - Добавление интервалов тишины между фрагментами
   - Передача аудиорезультатов обратно клиенту

6. **Логирование и аналитика**:
   - Сбор и отправка метрик и логов клиенту
   - Измерение времени выполнения операций
   - Отслеживание производительности каждого этапа обработки
   - Структурированное логирование событий с контекстом

7. **Возврат аудиопотока клиенту**:
   - Формирование потока с переведенной речью
   - Отправка через Socket.IO клиенту
   - Управление качеством и параметрами синтеза (голос, темп)

### Технические детали:

- **Язык программирования**: Python
- **Серверный фреймворк**: Socket.IO AsyncServer с ASGI-совместимостью
- **Асинхронная обработка**: asyncio для параллельного выполнения слоев пайплайна
- **Коннекторы внешних сервисов**:
  - WhisperSttConnector: gRPC клиент для сервиса распознавания речи
  - LlamaCppServerConnector: HTTP клиент для LLM сервиса
  - PiperVoskConnector: gRPC клиент для сервиса синтеза речи
- **Буферизация**: TimeoutTextBuffer для обработки и сегментации текста
- **Протоколы связи**: gRPC для ASR и TTS, HTTP для LLM
- **Модульная архитектура**: трехслойная структура с разделением ответственности
- **Логирование**: RemoteLogger для отправки событий клиенту
- **Шаги обработки LLM**: Настраиваемые шаги обработки (гейтинг, перевод, коррекция)
- **Управление конфигурацией**: Pydantic-схемы для валидации настроек

## whisper_grpc_api (Распознавание речи)

whisper_grpc_api - сервис автоматического распознавания речи (ASR), основанный на модели Whisper, предоставляющий gRPC API для преобразования аудио в текст.

```mermaid
flowchart TB
    subgraph "whisper_grpc_api"
        direction TB
        
        gRPCServer["gRPC<br>сервер"] -->|"Аудиоданные"| InputProcessor["Процессор<br>входных данных"]
        
        InputProcessor -->|"Подготовленное<br>аудио"| WhisperModel["Модель<br>Whisper"]
        
        WhisperModel -->|"Распознанный<br>текст"| PostProcessor["Пост-процессор<br>текста"]
        
        PostProcessor -->|"Форматированный<br>текст"| ResponseFormatter["Форматировщик<br>ответа"]
        
        ResponseFormatter -->|"gRPC<br>ответ"| gRPCServer
        
        ModelManager["Менеджер<br>моделей"] <-->|"Загрузка<br>и обновление"| WhisperModel
        
        LanguageDetector["Детектор<br>языка"] <-->|"Определение<br>языка"| WhisperModel
        
        SegmentManager["Менеджер<br>сегментов речи"] <-->|"Разделение<br>на сегменты"| InputProcessor
        SegmentManager <-->|"Объединение<br>результатов"| PostProcessor
    end
    
    StreamingProxy["streaming-proxy"] <-->|"gRPC"| gRPCServer
    
    classDef core fill:#f9f,stroke:#333,stroke-width:2px;
    classDef processor fill:#bfb,stroke:#333,stroke-width:1px;
    classDef manager fill:#bbf,stroke:#333,stroke-width:1px;
    classDef external fill:#fbb,stroke:#333,stroke-width:1px;
    
    class gRPCServer,WhisperModel core;
    class InputProcessor,PostProcessor,ResponseFormatter processor;
    class ModelManager,LanguageDetector,SegmentManager manager;
    class StreamingProxy external;
```

### Основные процессы в whisper_grpc_api:

1. **Прием аудиоданных**:
   - Получение аудиофрагментов через gRPC
   - Валидация входных данных
   - Преобразование в формат, необходимый для модели

2. **Предобработка аудио**:
   - Конвертация форматов (при необходимости)
   - Сегментация аудио на части
   - Нормализация и фильтрация шума (если требуется)

3. **Распознавание речи**:
   - Загрузка соответствующей языковой модели
   - Выполнение инференса модели Whisper
   - Обработка вероятностей и предсказаний

4. **Постобработка текста**:
   - Пунктуация и форматирование
   - Фильтрация заполнителей ("эм", "хм" и т.д.)
   - Объединение фрагментов в полные предложения

5. **Возврат результатов**:
   - Формирование gRPC ответа
   - Отправка распознанного текста
   - Включение метаданных (уверенность, альтернативы)

### Технические детали:

- **Модель**: OpenAI Whisper или ее варианты (small, medium, large)
- **Языки программирования**: Python с PyTorch/TensorFlow
- **gRPC и Protocol Buffers** для определения API
- **CUDA/GPU** для ускорения инференса
- **Опционально**: интеграция с системами кэширования для часто встречающихся фраз

## tts_grpc_api (Синтез речи)

tts_grpc_api - сервис синтеза речи (TTS), преобразующий текст в естественно звучащую речь с поддержкой множества языков и голосов, основанный на двух ключевых системах: Piper и Vosk TTS.

```mermaid
flowchart TB
    subgraph "tts_grpc_api"
        direction TB
        
        GRPCServer["gRPC<br>SynthesizerService"] -->|"Запрос<br>синтеза"| ServiceRouter["Маршрутизатор<br>сервисов"]
        
        subgraph "Piper TTS Engine"
            PiperService["Piper<br>Service"] <-->|"Управление<br>моделями"| VoiceManager["Менеджер<br>Голосов"]
            PiperService -->|"Текст для<br>синтеза"| Phonemizer["Фонемизатор"]
            Phonemizer -->|"Последовательность<br>фонем"| PiperModel["ONNX<br>Модель"]
            PiperModel -->|"Аудио<br>данные"| AudioProcessor["Аудио<br>Процессор"]
        end
        
        subgraph "Vosk TTS Engine"
            VoskService["Vosk<br>Service"] -->|"Текст для<br>синтеза"| G2P["Конвертер<br>Графема-Фонема"]
            G2P -->|"Фонетическое<br>представление"| VoskModel["ONNX<br>Модель"]
            VoskModel -->|"Аудио<br>данные"| AudioFader["Контроллер<br>Громкости"]
        end
        
        ServiceRouter -->|"Piper<br>запрос"| PiperService
        ServiceRouter -->|"Vosk<br>запрос"| VoskService
        
        AudioProcessor -->|"Обработанное<br>аудио"| AudioChunker["Разделитель<br>на чанки"]
        AudioFader -->|"Обработанное<br>аудио"| AudioChunker
        
        AudioChunker -->|"Аудио<br>поток"| GRPCServer
        
        ModelInitializer["Инициализатор<br>Моделей"] -.->|"Загрузка<br>моделей"| PiperModel
        ModelInitializer -.->|"Загрузка<br>моделей"| VoskModel
        
    end
    
    StreamingProxy["streaming-proxy"] <-->|"gRPC<br>запросы/ответы"| GRPCServer
    
    classDef core fill:#f9f,stroke:#333,stroke-width:2px;
    classDef engine fill:#bfb,stroke:#333,stroke-width:1px;
    classDef processor fill:#bbf,stroke:#333,stroke-width:1px;
    classDef external fill:#fbb,stroke:#333,stroke-width:1px;
    
    class GRPCServer,ServiceRouter core;
    class PiperService,VoskService,PiperModel,VoskModel engine;
    class Phonemizer,G2P,AudioProcessor,AudioFader,AudioChunker,VoiceManager,ModelInitializer processor;
    class StreamingProxy external;
```

### Основные процессы в tts_grpc_api:

1. **Маршрутизация запросов синтеза речи**:
   - Получение текста и параметров синтеза через gRPC API
   - Определение запрашиваемого TTS сервиса (Piper или Vosk)
   - Перенаправление запроса соответствующему обработчику

2. **Работа с Piper TTS**:
   - Загрузка соответствующей модели для запрошенного голоса
   - Фонемизация текста с использованием espeak или графемно-фонемных конвертеров
   - Преобразование фонем в идентификаторы для модели
   - Синтез речи с использованием ONNX моделей
   - Потоковая обработка и возврат аудио

3. **Работа с Vosk TTS**:
   - Загрузка модели для нужного языка
   - Графемно-фонемное преобразование текста
   - Синтез речи с применением ONNX модели
   - Настройка параметров синтеза (скорость речи, выбор дикторов)
   - Обработка громкости аудио

4. **Управление моделями**:
   - Инициализация и загрузка ONNX моделей в память
   - Кэширование моделей для повторного использования
   - Управление несколькими голосами для разных языков
   - Выбор подходящего провайдера выполнения (CPU/CUDA)

5. **Потоковая передача**:
   - Разделение синтезированного аудио на чанки
   - Потоковая отправка аудио через gRPC
   - Управление двунаправленным потоковым синтезом для интерактивного взаимодействия

### Технические детали:

- **Модели синтеза**: 
  - **Piper**: ONNX модели на основе VITS (Variational Inference with adversarial learning for end-to-end TTS)
  - **Vosk TTS**: Модели, основанные на нейросетевых архитектурах для синтеза речи
- **Языки программирования**: Python с ONNX Runtime для инференса
- **Фонемизация**: Библиотеки piper_phonemize и espeak для преобразования текста в фонемы
- **Обработка аудио**: Управление громкостью, нормализация и потоковая передача
- **gRPC API**: Унифицированный интерфейс для различных TTS бэкендов
- **Инференс моделей**: Поддержка как CPU, так и GPU (CUDA) для оптимизации производительности

## Взаимодействие компонентов и жизненный цикл запроса

```mermaid
sequenceDiagram
    participant User as Пользователь
    participant SC as synchro-python (Клиент)
    participant SP as streaming-proxy (Диспетчер)
    participant ASR as whisper_grpc_api (ASR)
    participant LLM as LLM сервис
    participant TTS as tts_grpc_api (TTS)
    
    User->>SC: Говорит на исходном языке
    SC->>SC: Записывает и обрабатывает аудио
    
    SC->>+SP: Отправляет аудиопоток (WebSocket)
    
    SP->>+ASR: Отправляет аудио для распознавания (gRPC)
    ASR-->>-SP: Возвращает распознанный текст
    
    SP->>SP: Проверяет качество распознанного текста
    
    SP->>+LLM: Отправляет текст для перевода (API)
    LLM-->>-SP: Возвращает переведенный текст
    
    SP->>+TTS: Отправляет текст для синтеза (gRPC)
    TTS-->>-SP: Возвращает синтезированную речь
    
    SP-->>-SC: Отправляет переведенную речь (WebSocket)
    
    SC->>User: Воспроизводит переведенную речь
    
    Note over User,TTS: Полный процесс от исходной речи до перевода обычно занимает менее 1-2 секунд
```

## Конфигурация и развертывание

Все компоненты системы могут быть развернуты как в отдельных контейнерах Docker, так и на физических/виртуальных машинах. Для обеспечения масштабируемости и отказоустойчивости рекомендуется использовать оркестрацию контейнеров (например, Kubernetes).

### Пример конфигурации контейнеров:

```mermaid
graph LR
    subgraph "Инфраструктура"
        subgraph "Кластер Kubernetes"
            PC["synchro-python<br>Клиентские контейнеры"]
            
            subgraph "Сервисный слой"
                SPD["streaming-proxy<br>Deployment"]
                SPD -->|"Масштабирование"| SPD1["Pod 1"]
                SPD -->|"Масштабирование"| SPD2["Pod 2"]
                SPD -->|"Масштабирование"| SPD3["Pod n"]
                
                ASRD["whisper_grpc_api<br>Deployment"]
                ASRD -->|"Масштабирование"| ASRD1["Pod 1"]
                ASRD -->|"Масштабирование"| ASRD2["Pod 2"]
                
                TTSD["tts_grpc_api<br>Deployment"]
                TTSD -->|"Масштабирование"| TTSD1["Pod 1"]
                TTSD -->|"Масштабирование"| TTSD2["Pod 2"]
            end
            
            SPS["streaming-proxy<br>Service"]
            ASRS["whisper_grpc_api<br>Service"]
            TTSS["tts_grpc_api<br>Service"]
            
            SPD1 & SPD2 & SPD3 --- SPS
            ASRD1 & ASRD2 --- ASRS
            TTSD1 & TTSD2 --- TTSS
        end
        
        LB["Load Balancer"]
        
        LLM["Внешний LLM<br>сервис"]
    end
    
    PC <-->|"WebSocket"| LB
    LB <-->|"Маршрутизация"| SPS
    SPS <-->|"gRPC"| ASRS
    SPS <-->|"gRPC"| TTSS
    SPS <-->|"API"| LLM
    
    classDef client fill:#bfb,stroke:#333,stroke-width:1px;
    classDef service fill:#bbf,stroke:#333,stroke-width:1px;
    classDef pod fill:#f9f,stroke:#333,stroke-width:1px;
    classDef balancer fill:#fbb,stroke:#333,stroke-width:1px;
    classDef external fill:#ddd,stroke:#333,stroke-width:1px;
    
    class PC client;
    class SPS,ASRS,TTSS service;
    class SPD1,SPD2,SPD3,ASRD1,ASRD2,TTSD1,TTSD2 pod;
    class LB balancer;
    class LLM external;
```

## Заключение

Система синхронного перевода представляет собой сложное распределенное приложение, состоящее из четырех основных компонентов:

1. **synchro-python** - клиентское приложение, обеспечивающее взаимодействие с пользователем
2. **streaming-proxy** - центральный компонент, координирующий все процессы перевода
3. **whisper_grpc_api** - сервис распознавания речи, преобразующий аудио в текст
4. **tts_grpc_api** - сервис синтеза речи, преобразующий переведенный текст в речь

Взаимодействие этих компонентов позволяет реализовать синхронный перевод речи с минимальной задержкой, что может быть применено в различных областях: от международных конференций и деловых переговоров до образовательных программ и туристических приложений.

Масштабируемая архитектура системы обеспечивает возможность обслуживания большого числа пользователей одновременно, а модульная структура позволяет легко обновлять отдельные компоненты без необходимости изменения всей системы.
