# GenAI — Session 11: Conversational & Voice-Based AI Agents

---

## 1. The Voice Dimension — Why It Changes Everything

Every session so far has assumed one input/output format: **text**.

Voice breaks every assumption:

```
Text-based agent:
  User types  → tokens → LLM → tokens → text displayed
  Latency:    500ms - 2s feels fine
  Errors:     User re-reads, notices mistakes
  Format:     Structured, can use markdown, lists, headers

Voice-based agent:
  User speaks → audio → STT → tokens → LLM → tokens → TTS → audio played
  Latency:    > 1.5s feels broken, > 3s is unusable
  Errors:     User can't re-listen easily, mistakes feel jarring
  Format:     Must sound natural when spoken aloud
              (no bullet points, no markdown, no "as shown above")
```

> Voice is not just text with audio added. It is a **fundamentally different interaction paradigm** that imposes hard constraints on every layer of your system.

---

## 2. The Full Voice Pipeline Architecture

Before diving into each component, see the complete picture:

```
┌─────────────────────────────────────────────────────────┐
│                 VOICE AGENT PIPELINE                     │
│                                                          │
│  USER SPEAKS                                             │
│      ↓                                                   │
│  [Microphone captures audio waveform]                    │
│      ↓                                                   │
│  ┌─────────────────────────────────────┐                │
│  │   SPEECH-TO-TEXT (STT)              │                │
│  │   Audio → Transcribed Text          │  ~200-400ms    │
│  └─────────────────────────────────────┘                │
│      ↓                                                   │
│  ┌─────────────────────────────────────┐                │
│  │   VOICE ACTIVITY DETECTION (VAD)    │                │
│  │   "Has the user finished speaking?" │  ~50ms         │
│  └─────────────────────────────────────┘                │
│      ↓                                                   │
│  ┌─────────────────────────────────────┐                │
│  │   AGENT (LLM + Tools + Memory)      │                │
│  │   Text → Reasoning → Text Response  │  ~500-1500ms   │
│  └─────────────────────────────────────┘                │
│      ↓                                                   │
│  ┌─────────────────────────────────────┐                │
│  │   TEXT-TO-SPEECH (TTS)              │                │
│  │   Text → Audio waveform             │  ~200-400ms    │
│  └─────────────────────────────────────┘                │
│      ↓                                                   │
│  USER HEARS RESPONSE                                     │
│                                                          │
│  Total pipeline latency: ~1000-2300ms                    │
└─────────────────────────────────────────────────────────┘
```

Every component adds latency. Your job as an engineer is to **minimise total end-to-end latency** while maintaining quality.

---

## 3. Speech-to-Text (STT) Pipelines

STT converts raw audio into text the LLM can process.

### How STT Works (High Level):

```
Raw audio waveform
      ↓
Feature extraction (Mel spectrograms)
      ↓
Acoustic model (what sounds were made?)
      ↓
Language model (what words make sense given context?)
      ↓
Transcribed text
```

### Major STT Providers:

```
┌─────────────────────────────────────────────────────┐
│  Provider          Latency    Accuracy    Best For  │
├─────────────────────────────────────────────────────┤
│  OpenAI Whisper    ~300ms     Very High   General   │
│  (open-source)                            purpose   │
│                                                     │
│  Deepgram Nova-2   ~100ms     High        Real-time │
│                                           streaming  │
│                                                     │
│  Google STT        ~200ms     High        Indian    │
│                                           languages  │
│                                                     │
│  AWS Transcribe    ~250ms     Good        AWS       │
│                                           ecosystem  │
│                                                     │
│  ElevenLabs STT    ~150ms     High        Combined  │
│                                           STT+TTS   │
└─────────────────────────────────────────────────────┘
```

### STT in Code:

```python
import openai
import sounddevice as sd
import numpy as np

def capture_audio(duration_seconds: int = 5) -> np.ndarray:
    """Capture audio from microphone."""
    sample_rate = 16000
    audio = sd.rec(
        int(duration_seconds * sample_rate),
        samplerate=sample_rate,
        channels=1,
        dtype='float32'
    )
    sd.wait()  # wait until recording finishes
    return audio

def transcribe_audio(audio: np.ndarray) -> str:
    """Convert audio to text using Whisper."""
    # Save to temp file (Whisper needs file input)
    import tempfile, soundfile as sf
    
    with tempfile.NamedTemporaryFile(suffix=".wav") as f:
        sf.write(f.name, audio, 16000)
        
        with open(f.name, "rb") as audio_file:
            transcript = openai.audio.transcriptions.create(
                model="whisper-1",
                file=audio_file,
                language="en"    # specify for better accuracy
            )
    
    return transcript.text

# Usage
audio = capture_audio(duration_seconds=5)
text = transcribe_audio(audio)
print(f"User said: {text}")
# → "User said: What is the current stock price of Infosys?"
```

---

### Voice Activity Detection (VAD)

A critical but often overlooked component. VAD answers: **"Has the user finished speaking?"**

```
Without VAD:
  → Fixed recording duration (e.g. always 5 seconds)
  → User says "Hello" in 0.5s → waits 4.5s of silence
  → Feels broken and unnatural

With VAD:
  → Recording starts when speech detected
  → Recording stops when silence detected (end of utterance)
  → Feels like a natural conversation
```

```python
import webrtcvad

vad = webrtcvad.Vad(sensitivity=2)  # 0-3, higher = more aggressive

def record_until_silence(silence_threshold_ms: int = 800) -> np.ndarray:
    """
    Record audio until user stops speaking.
    Stops after silence_threshold_ms of silence.
    """
    frames = []
    silent_frames = 0
    
    with sd.InputStream(samplerate=16000, channels=1) as stream:
        while True:
            frame, _ = stream.read(160)  # 10ms frames
            frames.append(frame)
            
            # Check if this frame contains speech
            is_speech = vad.is_speech(frame.tobytes(), 16000)
            
            if not is_speech:
                silent_frames += 1
            else:
                silent_frames = 0
            
            # Stop after 800ms of continuous silence
            if silent_frames > (silence_threshold_ms / 10):
                break
    
    return np.concatenate(frames)
```

---

## 4. Voice-to-LLM-to-Voice Flow

This is the complete integration — connecting STT, the agent, and TTS into one seamless loop.

```python
from agents import Agent, Runner, function_tool

# Define the agent
voice_agent = Agent(
    name="Voice Assistant",
    instructions="""
    You are a helpful voice assistant.
    
    CRITICAL VOICE FORMATTING RULES:
    - Respond in natural spoken English only
    - Never use bullet points, markdown, or headers
    - Never say "asterisk", "hashtag", or formatting symbols
    - Keep responses under 3 sentences for simple questions
    - Use conversational connectors: "So", "Well", "Actually"
    - Spell out numbers under 10: "three" not "3"
    - Read acronyms as words when possible: "RAG" not "R-A-G"
    """,
    tools=[get_stock_price, get_weather, search_web]
)

def voice_to_llm_to_voice(thread=None):
    """Complete voice interaction loop."""
    
    print("🎤 Listening...")
    
    # Step 1: Capture speech
    audio = record_until_silence()
    
    # Step 2: Speech → Text
    user_text = transcribe_audio(audio)
    print(f"You said: {user_text}")
    
    # Step 3: Text → Agent → Text
    result = Runner.run_sync(
        voice_agent,
        user_text,
        thread=thread
    )
    response_text = result.final_output
    print(f"Agent: {response_text}")
    
    # Step 4: Text → Speech
    speak(response_text)
    
    return thread

# Run the conversation loop
thread = Thread()
while True:
    thread = voice_to_llm_to_voice(thread)
```

---

## 5. Text-to-Speech (TTS) — Making the Agent's Voice

TTS converts the agent's text response back to natural-sounding audio.

### Major TTS Providers:

```
┌───────────────────────────────────────────────────────────┐
│  Provider           Latency    Quality     Special Feature │
├───────────────────────────────────────────────────────────┤
│  ElevenLabs         ~300ms     Excellent   Voice cloning  │
│                                            Most natural   │
│                                                           │
│  OpenAI TTS         ~250ms     Very Good   Simple API     │
│  (tts-1, tts-1-hd)                        6 voice options │
│                                                           │
│  Google Cloud TTS   ~200ms     Good        160+ voices    │
│                                            Indian accents │
│                                                           │
│  AWS Polly          ~150ms     Good        Neural voices  │
│                                            AWS ecosystem  │
│                                                           │
│  Deepgram Aura      ~100ms     Good        Fastest        │
│                                            streaming      │
└───────────────────────────────────────────────────────────┘
```

### TTS in Code:

```python
import openai
import pygame
import io

def speak(text: str, voice: str = "nova") -> None:
    """Convert text to speech and play it."""
    
    # Generate speech audio
    response = openai.audio.speech.create(
        model="tts-1",       # tts-1 = faster, tts-1-hd = higher quality
        voice=voice,         # alloy, echo, fable, onyx, nova, shimmer
        input=text,
        speed=1.0            # 0.25 to 4.0
    )
    
    # Play audio directly from bytes
    pygame.mixer.init()
    audio_bytes = io.BytesIO(response.content)
    pygame.mixer.music.load(audio_bytes)
    pygame.mixer.music.play()
    
    # Wait for playback to finish
    while pygame.mixer.music.get_busy():
        pygame.time.Clock().tick(10)
```

---

## 6. Real-Time Conversations

Real-time voice means the agent **starts responding before the user finishes speaking**, or starts speaking before it has finished generating.

### Two Approaches:

**Approach 1 — Turn-Based (Simpler)**
```
User speaks completely
    ↓ [VAD detects end]
STT processes full utterance
    ↓
LLM generates complete response
    ↓
TTS converts complete response
    ↓
Agent speaks completely
    ↓
User speaks again

Latency: Full pipeline latency on every turn (~1-2.5s)
Feel: Slightly robotic but reliable
```

**Approach 2 — Streaming (Natural)**
```
User speaks
    ↓ [VAD detects end]
STT starts transcribing (streaming)
    ↓ [first words transcribed]
LLM starts generating (streaming first tokens)
    ↓ [first sentence generated]
TTS starts speaking first sentence
    ↓ [while speaking sentence 1]
LLM generates sentence 2
TTS queues sentence 2
    ↓
Feels like natural human response speed
```

### Implementing Streaming TTS:

```python
import asyncio

async def stream_voice_response(text_stream):
    """
    Stream TTS as LLM generates tokens.
    Start speaking first sentence while rest is being generated.
    """
    buffer = ""
    sentence_endings = {'.', '!', '?', ':'}
    
    async for token in text_stream:
        buffer += token
        
        # Check if we have a complete sentence
        if buffer[-1] in sentence_endings and len(buffer) > 20:
            # Speak this sentence NOW, don't wait for full response
            await speak_async(buffer.strip())
            buffer = ""
    
    # Speak any remaining text
    if buffer.strip():
        await speak_async(buffer.strip())

# With LangChain streaming
async def voice_response_with_streaming(user_input: str):
    response_stream = voice_agent.astream(user_input)
    await stream_voice_response(response_stream)
```

---

## 7. Streaming & Latency Handling

Latency is the **#1 engineering problem** in voice AI. Every millisecond matters.

### The Latency Budget:

```
Target total latency: < 1500ms (feels natural)
Budget breakdown:

STT:            200-400ms
VAD:             50-100ms  
Network (STT):   50-100ms
LLM (1st token): 300-800ms  ← biggest variable
TTS (1st audio): 150-300ms
Network (TTS):    50-100ms
─────────────────────────
Total:          800-1800ms  ← right at the edge
```

### Latency Reduction Techniques:

**Technique 1 — Parallel Processing**
```python
import asyncio

async def optimised_voice_pipeline(audio: bytes):
    
    # Start STT immediately
    stt_task = asyncio.create_task(transcribe_async(audio))
    
    # While STT is running, warm up LLM connection
    llm_warmup_task = asyncio.create_task(warmup_llm_connection())
    
    # Wait for transcription
    user_text = await stt_task
    
    # LLM connection already warm — start immediately
    await llm_warmup_task
    
    # Stream LLM response directly to TTS
    await stream_voice_response(
        voice_agent.astream(user_text)
    )
```

**Technique 2 — Predictive Prefilling**
```python
# While user is still speaking, start predicting response
# based on partial transcription

async def predictive_pipeline(audio_stream):
    partial_transcript = ""
    
    async for audio_chunk in audio_stream:
        # Continuously update partial transcript
        partial_transcript = await partial_transcribe(audio_chunk)
        
        # If we have enough to predict intent
        if len(partial_transcript.split()) > 5:
            # Start warming up relevant tools
            predicted_tools = predict_needed_tools(partial_transcript)
            await prefetch_tool_data(predicted_tools)
    
    # By the time user finishes, tools already prefetched
```

**Technique 3 — Response Caching**
```python
import hashlib

response_cache = {}

async def cached_voice_response(user_text: str) -> str:
    # Hash the query
    cache_key = hashlib.md5(user_text.lower().strip().encode()).hexdigest()
    
    # Check cache first
    if cache_key in response_cache:
        return response_cache[cache_key]
    
    # Generate fresh response
    response = await voice_agent.arun(user_text)
    
    # Cache for future identical queries
    response_cache[cache_key] = response
    return response

# Great for: "What time is it?", "What's the weather?"
# Common queries with deterministic answers
```

**Technique 4 — Filler Responses**
```python
# While LLM is thinking, play a natural filler
# (like humans say "Hmm, let me think...")

FILLERS = [
    "Let me check that for you.",
    "One moment.",
    "Sure, looking into that now."
]

async def voice_with_filler(user_text: str):
    # Start LLM in background
    llm_task = asyncio.create_task(
        voice_agent.arun(user_text)
    )
    
    # If LLM takes > 600ms, play a filler
    try:
        response = await asyncio.wait_for(
            asyncio.shield(llm_task), 
            timeout=0.6
        )
    except asyncio.TimeoutError:
        # Play filler while waiting
        await speak_async(random.choice(FILLERS))
        response = await llm_task
    
    await speak_async(response)
```

---

## 8. Interruption Handling

Real human conversation involves **interruptions** — the user speaks while the agent is still talking.

```
Without interruption handling:
  Agent: "The stock price of Infosys is currently one thousand—"
  User: "Wait, I meant TCS!"
  Agent: "—eight hundred and forty seven rupees..."  ← ignores user
  
With interruption handling:
  Agent: "The stock price of Infosys is currently one thousand—"
  User: "Wait, I meant TCS!"
  Agent: [stops immediately]
  Agent: "Of course — TCS is trading at three thousand two hundred rupees."
```

```python
import threading

class InterruptibleVoiceAgent:
    
    def __init__(self):
        self.speaking = False
        self.interrupted = False
        self.speech_thread = None
    
    def speak_with_interrupt_detection(self, text: str):
        """Speak text but stop if user starts talking."""
        self.speaking = True
        self.interrupted = False
        
        # Monitor microphone in background
        monitor_thread = threading.Thread(
            target=self._monitor_for_speech
        )
        monitor_thread.start()
        
        # Stream TTS sentence by sentence
        for sentence in split_into_sentences(text):
            if self.interrupted:
                break       # stop mid-response if interrupted
            speak_sentence(sentence)
        
        self.speaking = False
    
    def _monitor_for_speech(self):
        """Detect if user starts speaking while agent talks."""
        while self.speaking:
            audio_chunk = capture_audio_chunk(duration_ms=100)
            if vad.is_speech(audio_chunk):
                self.interrupted = True
                stop_playback()
                break
```

---

## 9. Voice-Specific Prompt Engineering

Voice agents need **different prompt design** from text agents:

```python
voice_agent = Agent(
    instructions="""
    You are a voice assistant. You communicate through speech only.
    
    LANGUAGE RULES:
    ✅ Use: "For example", "In other words", "To summarise"
    ❌ Avoid: bullet points, numbered lists, headers, bold, italics
    
    RESPONSE LENGTH:
    ✅ Simple questions: 1-2 sentences
    ✅ Complex questions: 3-4 sentences maximum
    ❌ Never: paragraphs of continuous explanation
    
    NUMBER FORMATTING:
    ✅ Say: "one thousand eight hundred rupees"
    ❌ Say: "₹1,800" (unreadable by TTS)
    
    CONFIRMATION PATTERN:
    ✅ Always confirm what you understood before answering
    "You're asking about Infosys stock — here's what I found..."
    
    UNCERTAINTY:
    ✅ Say: "I'm not entirely sure, but..."
    ❌ Never: silently give wrong information
    
    HANDOFF:
    ✅ Say: "I've sent that to your email" or "I've set that reminder"
    → Always confirm actions taken out loud
    """
)
```

---

## The Complete Voice Agent System

```
┌─────────────────────────────────────────────────────────┐
│              PRODUCTION VOICE AGENT SYSTEM               │
│                                                          │
│  INPUT LAYER                                             │
│  Microphone → VAD → STT → Text                          │
│                                                          │
│  INTELLIGENCE LAYER                                      │
│  Text → Agent (LLM + Tools + Memory + Thread) → Text    │
│                                                          │
│  OUTPUT LAYER                                            │
│  Text → TTS → Speaker                                    │
│                                                          │
│  REAL-TIME FEATURES                                      │
│  → Streaming (first sentence plays while rest generates) │
│  → Interruption detection (user can cut agent off)       │
│  → Filler responses (agent buys time naturally)          │
│  → Response caching (instant for common queries)         │
│                                                          │
│  LATENCY TARGETS                                         │
│  → First audio out: < 1000ms                            │
│  → Full response: < 2500ms                              │
│  → Interruption response: < 300ms                       │
└─────────────────────────────────────────────────────────┘
```

---

## How Sessions 10 & 11 Connect

```
Session 10 — Graph Databases:
  How agents store and traverse RELATIONSHIPS
  between pieces of knowledge

Session 11 — Voice Agents:
  How agents operate in REAL-TIME
  under hard latency constraints
  
  Voice agents still use everything from before:
  → Threads (Session 8) — conversation memory
  → Memory systems (Session 9) — user preferences
  → Graph DB (Session 10) — if agent needs 
    relationship-aware knowledge retrieval
  → RAG (Sessions 4-5) — if agent needs documents
  
  Voice just adds a new I/O layer on top.
  The intelligence layer stays the same.
```

---

## Quick Recap — Session 11

| Concept | One-liner |
|---|---|
| Voice vs text | Voice imposes hard latency constraints and different formatting rules |
| STT | Audio → text, ~200-400ms, Whisper/Deepgram most common |
| VAD | Detects when user stops speaking so agent knows when to respond |
| TTS | Text → audio, ~150-300ms, ElevenLabs/OpenAI TTS most common |
| Turn-based | Complete utterances before responding — simpler but slower feel |
| Streaming | Speak first sentence while generating rest — natural feel |
| Latency budget | Target < 1500ms total; LLM first token is the biggest variable |
| Parallel processing | Run STT and LLM warmup simultaneously to save time |
| Filler responses | Buy time naturally while LLM thinks — "Let me check that" |
| Interruption handling | Detect user speech during agent playback and stop immediately |
| Voice prompt rules | No markdown, short sentences, spell out numbers, confirm actions |

---

Session 12 is **Model Context Protocol (MCP) & Agent Communication** — the emerging standard for how agents talk to tools and to each other at a systems level. It's what makes Claude able to connect to Google Drive, GitHub, Slack and dozens of other services seamlessly. Ready?