# AudioSphere Listener

**AudioSphere Listener** es un componente personalizado para Gradio (versión 5.x) que provee una visualización 3D en tiempo real de la forma de onda de audio mediante una esfera de partículas. Permite integración plug-and-play con cualquier modelo ASR (reconocimiento automático de voz), ofrece detección inteligente de silencio y es completamente responsive (desktop y mobile).  

---

## Tabla de Contenidos

1. [Visión General](#visión-general)  
2. [Características Principales](#características-principales)  
3. [Arquitectura y Estructura de Archivos](#arquitectura-y-estructura-de-archivos)  
4. [Tecnologías Utilizadas](#tecnologías-utilizadas)  
5. [Instalación y Publicación](#instalación-y-publicación)  
6. [Uso en Gradio (Ejemplos)](#uso-en-gradio-ejemplos)  
   1. [Configuración Básica](#configuración-básica)  
   2. [Integración con ASR (Whisper, Google STT, etc.)](#integración-con-asr-whisper-google-stt-etc)  
   3. [Personalización Avanzada (Props y Callbacks)](#personalización-avanzada-props-y-callbacks)  
7. [Descripción Técnica Detallada](#descripción-técnica-detallada)  
   1. [Inicialización y Pipeline de Audio](#inicialización-y-pipeline-de-audio)  
   2. [Motor 3D (Three.js) y Fallback (PixiJS)](#motor-3d-threejs-y-fallback-pixijs)  
   3. [Detección de Silencio (RMS) y AGC/Noise Gate](#detección-de-silencio-rms-y-agcnoise-gate)  
   4. [Sistema de Temas Dinámicos](#sistema-de-temas-dinámicos)  
   5. [Arquitectura de Plugins Visuales](#arquitectura-de-plugins-visuales)  
   6. [Accesibilidad y Atajos de Teclado](#accesibilidad-y-atajos-de-teclado)  
8. [Props y Currículum de API](#props-y-currículum-de-api)  
9. [Roadmap y Futuras Mejoras](#roadmap-y-futuras-mejoras)  
10. [Contacto y Soporte](#contacto-y-soporte)  

---

## Visión General

AudioSphere Listener redefine la experiencia de monitoreo de audio en aplicaciones Gradio al sustituir la forma de onda tradicional por una **esfera tridimensional de partículas** que reacciona dinámicamente a la señal de audio entrante. Cada partícula varía su tamaño, opacidad y color según la energía de las bandas de frecuencia, mientras que toda la esfera rota suavemente en los ejes X, Y y Z para potenciar la sensación de inmersión.  

El componente incluye:
- **Detección inteligente de silencio** basada en RMS (Root Mean Square).  
- **Integración plug-and-play con modelos ASR** (Whisper, Google Speech-to-Text, DeepSpeech, etc.) a través de callbacks en Python.  
- **Tematización dinámica** (Dark/Light, esquemas personalizados) con transiciones suaves.  
- **Fallback automático** a PixiJS o Canvas 2D en navegadores sin WebGL o con bajo rendimiento.  
- **Responsive Design**: adaptativo a pantallas de escritorio, tablets y móviles.  
- **Arquitectura de plugins visuales** que permite añadir nuevos efectos (ondas concéntricas, humo, etc.) sin modificar el núcleo.  
- **Accesibilidad (ARIA)** y **atajos de teclado** para que la interacción sea eficiente en cualquier contexto.  

---

## Características Principales

1. **Esfera de Partículas 3D**  
   - Visualización en tiempo real de la forma de onda de audio mediante una malla de partículas renderizada con Three.js.  
   - Cada partícula se posiciona en la superficie de una esfera de “radio virtual” y se anima según datos de FFT (Fast Fourier Transform).  
   - Animaciones de expansión/contracción, variación de color (fucsia neón, azul eléctrico, verde ácido, etc.) y rotación constante.  

2. **Detección Automática de Silencio (RMS)**  
   - Procesamiento de fragmentos de audio (~250 ms) con `AudioContext.decodeAudioData`.  
   - Cálculo de RMS:  
     \[
       RMS = \sqrt{\frac{1}{N} \sum_{i=1}^{N} x_i^2}
     \]  
     donde \(x_i\) son las muestras PCM normalizadas.  
   - Ventana deslizante (buffer de los últimos _n_ fragmentos) para evitar falsas detecciones.  
   - Si RMS < `silenceThreshold` durante `requiredSilentChunks` consecutivos (por defecto 8 × 250 ms = 2 s), se dispara el callback `on_silence`.  

3. **Integración Plug-and-Play con Modelos ASR**  
   - El callback `on_audio_chunk(blob)` entrega cada fragmento (~250 ms) al backend Python.  
   - El desarrollador puede enviar ese `blob` a cualquier servicio ASR (Whisper local, Google Cloud STT, etc.) y recibir transcripciones parciales o finales.  
   - El callback `on_transcription(text)` se invoca para actualizar la interfaz con la transcripción recibida.  
   - Opcionalmente, el backend puede devolver un código de idioma, enviado al callback para mostrar etiquetas de idioma (`ES – Español`, `EN – English`).  

4. **Fallback a PixiJS o Canvas 2D**  
   - Si el navegador no soporta WebGL (o si se fuerza `fallback_to_pixi=True`), el componente utiliza PixiJS para emular la esfera de partículas en 2D.  
   - En hardware limitado, se reduce dinámicamente el número de partículas para mantener 30 fps (adaptive particle count).  

5. **Responsive Design (Mobile y Desktop)**  
   - El `<canvas>` ocupa el 100 % del ancho del contenedor padre y escala su altura al 60 % del contenedor (o conforme a `canvas_width`/`canvas_height`).  
   - En pantallas pequeñas (< 400 px de ancho), el conteo de partículas se reduce automáticamente (por ejemplo, de 12 000 → 6 000).  
   - La rotación manual con arrastre del mouse se desactiva en dispositivos táctiles para evitar interferencias con el scroll.  

6. **Tematización Dinámica**  
   - El dropdown de temas (`show_theme_switcher=True`) permite alternar entre varios esquemas predefinidos (Cyberpunk, Neón Verde, Púrpura Galáctico, etc.) y cargar themes personalizados desde `themes.json`.  
   - Las transiciones de color y fondo se animan en 0.3 s para no romper la experiencia.  
   - Soporta modo oscuro/ligero automático (`prefers-color-scheme`), y el atributo `data-theme="light"` en el contenedor ajusta botones, sliders y cajas de transcripción.  

7. **Arquitectura de Plugins Visuales**  
   - Se expone la interfaz TypeScript `VisualPlugin` que puede implementarse externamente.  
   - Los plugins se colocan en la carpeta `/plugins` y se cargan dinámicamente, sin necesidad de recompilar el componente.  
   - Ejemplos de plugins:  
     - `WaveFloorPlugin`: Piso ondulante bajo la esfera que vibra con la señal de baja frecuencia.  
     - `HaloGlowPlugin`: Halo luminoso que se expande y contrae detrás de la esfera.  
     - `LaserBeamsPlugin`: Rayos láser 3D que se disparan en función de picos de alta frecuencia.  

8. **Accesibilidad y Atajos de Teclado**  
   - Botones con `role="button"`, `aria-label="Iniciar grabación"` y `aria-live="polite"` para notificaciones de estado.  
   - Teclas rápidas:  
     - **Barra espaciadora** → Inicia/Detiene la grabación.  
     - **Flecha arriba/abajo** → Ajusta `silence_threshold` en incrementos de 0.005.  
   - Mensajes textuales breves (“Silencio detectado: deteniendo escucha”) se emiten en la caja de transcripción con énfasis visual, facilitando a usuarios con pérdida auditiva.  

---

## Arquitectura y Estructura de Archivos

audio\_sphere\_listener/
├── component.ts                 # Lógica TypeScript principal (compilado a JS)
├── component.js                 # Versión compilada y minificada (para publicación en Gradio CC)
├── component.css                # Estilos CSS puros para layout, responsividad y temas
├── component.toml               # Metadatos para publicar en Gradio CC
├── component.py                 # Wrapper Python para integración con Gradio
├── themes.json                  # Esquemas de color (temas) cargados dinámicamente
├── plugins/                     # Carpeta para plugins de visualización (VisualPlugin)
│   ├── WaveFloorPlugin.ts       # Ejemplo de plugin: piso ondulante
│   ├── HaloGlowPlugin.ts        # Ejemplo de plugin: halo luminoso
│   └── LaserBeamsPlugin.ts      # Ejemplo de plugin: rayos láser 3D
└── README.md                    # Este documento

````

- **component.ts / component.js**: contiene toda la lógica de inicialización, renderizado (Three.js o PixiJS), detección de silencio, callbacks JS ↔ Python, tematización y carga de plugins.  
- **component.css**: define estilos globales (contenedor, canvas, botones, sliders, dropdowns, caja de transcripción), media queries para responsive y reglas para modo claro/oscuro.  
- **component.toml**: metadatos para la publicación en Gradio CC (nombre, versión, descripción, autor, licencia).  
- **component.py**: clase `AudioSphereListener` en Python que extiende `gradio.components.Component`, inyecta scripts/CSS, expone props y callbacks para el desarrollador en `app.py`.  
- **themes.json**: archivo JSON que lista temas predefinidos; puede modificarse o ampliarse sin recompilar.  
- **plugins/**: directorio dedicado a plugins visuales que implementan la interfaz `VisualPlugin`. Al arrancar, el componente detecta archivos `.ts` o `.js` en `/plugins` y los inicializa automáticamente.  

````
---

## Tecnologías Utilizadas

- **Gradio 5.x**   
  Framework para crear interfaces web interactivas en Python. `AudioSphereListener` se publica como componente personalizado y se usa directamente en un `gr.Blocks()`.

- **TypeScript / JavaScript**  
  - **Three.js** v0.128.0 (o superior)  
    Motor 3D WebGL para renderizar la esfera de partículas en alta calidad.  
  - **PixiJS** v6.5.8  
    Fallback 2D en caso de que WebGL no esté disponible o en hardware limitado.  
  - **Web Audio API**  
    - `navigator.mediaDevices.getUserMedia` → captura del micrófono.  
    - `AudioContext`, `AnalyserNode` → procesamiento de señal en tiempo real (FFT y datos de dominio de tiempo).  
    - `MediaRecorder` → fragmentación de audio en `Blob`s de ~250 ms.  
  - **AudioWorklet (opcional)**  
    Para implementar Noise Gate y Control Automático de Ganancia (AGC) en el frontend, de modo que el audio enviado al ASR tenga mejor calidad.

- **CSS Puro**  
  - **Responsive Design**: media queries, flexbox, medidas relativas (%, rem).  
  - **Tematización Dinámica**: variables CSS o cambios en `data-theme` para aplicar esquemas claro/oscuro y otros temas.  
  - **Accesibilidad**: roles ARIA, estilos de foco, alto contraste en botones y textos.

- **Python**  
  - **Wrapper de Componente** (`component.py`) extiende `gradio.components.Component`, inyecta HTML, CSS y JS necesarios al cargar la página.  
  - **Callbacks ASR**: permite usar modelos locales (Whisper) o externos (Google Cloud, AWS Transcribe, Azure Speech) para transcripción en tiempo real.

- **JSON**  
  - **themes.json**: define esquemas de color para tematización dinámica sin recompilación.  
  - Plugins visuales (`VisualPlugin`) pueden exponer su propia configuración en JSON, si es necesario.

- **Herramientas Adicionales**  
  - **Rollup / Webpack / esbuild** (o similar) para compilar `component.ts` a `component.js`, optimización y minificación.  
  - **tsconfig.json** y **package.json** (no incluidos en esta carpeta) para gestionar dependencias y compilación de TypeScript.  

---

## Instalación y Publicación

Para que otros desarrolladores puedan instalar y usar **AudioSphere Listener** en sus proyectos Gradio, siga estos pasos:

1. **Clonar o descargar el repositorio**  
   ```bash
   git clone https://github.com/AitriaAI/AudioSphere-Listener.git
   cd audio_sphere_listener

2. **Instalar dependencias frontend (Solo si desea compilar localmente)**

   ```bash
   npm install three pixi.js typescript
   ```

   * Configure `tsconfig.json` según convenga (target ES6, módulo ESNext).
   * Compilar TypeScript:

     ```bash
     npx tsc
     ```
   * Esto generará `component.js` a partir de `component.ts`.

3. **Verificar que estén presentes**

   * `component.js` (minificado y listo para producción).
   * `component.css`
   * `component.toml`
   * `component.py`
   * `themes.json`
   * Carpeta `plugins/` (con plugins de ejemplo).

4. **Publicar en Gradio CC**

   ```bash
   cd audio_sphere_listener
   gradio cc publish .
   ```

   * Esto publicará el componente en el registro de Gradio CC.
   * Una vez publicado, recibirá un identificador, por ejemplo:

     ```
     audio_sphere_listener@0.1.0
     ```
   * Tome nota de este identificador para que otros puedan instalarlo.

5. **Instalación en otro proyecto Gradio**
   En el proyecto donde desea usarlo, ejecute:

   ```bash
   gradio cc install audio_sphere_listener@0.1.0
   ```

   * Esto descargará `component.py`, `component.js`, `component.css` y demás archivos necesarios.

6. **Verificación de Archivos Instalados**

   * En la carpeta de componentes de Gradio (por defecto `~/.gradio/cc_components/audio_sphere_listener@0.1.0/`), debe ver:

     ```
     component.py
     component.js
     component.css
     component.toml
     themes.json
     plugins/
     ```

---

## Uso en Gradio (Ejemplos)

A continuación se muestran varios escenarios de implementación en un archivo `app.py` de Gradio. Se asume que ya se instaló el componente mediante `gradio cc install audio_sphere_listener@0.1.0`.

### Configuración Básica

```python
import gradio as gr
from audio_sphere_listener import AudioSphereListener

def on_start():
    print("🔊  Grabación iniciada.")
    return "Grabando..."

def on_audio_chunk(blob):
    # Este callback recibe cada fragmento (~250 ms) como 'blob'.
    # Puede procesarlo o enviarlo a un ASR remoto/local.
    print("📦  Nuevo chunk de audio recibido.")
    return None  # No retornamos nada aquí; la transcripción se maneja en on_transcription

def on_silence():
    print("🤫  Silencio detectado. Deteniendo escucha.")
    return "Silencio detectado."

def on_stop():
    print("⏹️  Grabación detenida.")
    return "Grabación finalizada."

with gr.Blocks() as demo:
    gr.Markdown("## AudioSphere Listener: Ejemplo Básico")

    # Caja de texto para mostrar mensajes de estado (grabando, silencio, detenido)
    status_box = gr.Textbox(label="Estado", interactive=False)

    # Instanciar el componente con callbacks mínimos
    audio_sphere = AudioSphereListener(
        on_start=lambda: status_box.update(on_start()),
        on_audio_chunk=on_audio_chunk,
        on_silence=lambda: status_box.update(on_silence()),
        on_stop=lambda: status_box.update(on_stop()),
        canvas_width=400,
        canvas_height=400,
        allow_sensitivity_adjustment=True,  # Muestra slider para ajustar umbral de silencio
    )

demo.launch()
```

**Explicación**

* **`AudioSphereListener(...)`**: se crean instancias de los callbacks `on_start`, `on_audio_chunk`, `on_silence` y `on_stop`.
* **`status_box.update(...)`**: se usa para actualizar dinámicamente la caja de estado.
* `allow_sensitivity_adjustment=True` habilita el slider para ajustar en tiempo real el umbral de silencio (`silence_threshold`).

---

### Integración con ASR (Whisper, Google STT, etc.)

```python
import gradio as gr
from audio_sphere_listener import AudioSphereListener

# Ejemplo con Whisper local
import whisper
model = whisper.load_model("base")

LANGUAGE_LABELS = {
    "es": "ES – Español",
    "en": "EN – English",
    "fr": "FR – Français",
    # ...
}

def transcribe_blob(blob):
    """
    - Decodifica el blob (~250 ms) con Whisper.
    - Retorna la transcripción parcial y el idioma detectado.
    """
    result = model.transcribe(blob, language=None, task="transcribe")
    text = result["text"]
    lang = result.get("language", "es")
    return text, LANGUAGE_LABELS.get(lang, lang.upper())

def on_audio_chunk(blob):
    # Llamada al ASR en cada fragmento
    partial_text, language_label = transcribe_blob(blob)
    return partial_text, language_label

with gr.Blocks() as demo:
    gr.Markdown("## AudioSphere Listener + Whisper ASR")

    transcription_box = gr.Textbox(label="Transcripción en Vivo", interactive=False)
    language_label = gr.Label(label="Idioma Detectado")

    audio_sphere = AudioSphereListener(
        on_audio_chunk=lambda b: (
            transcription_box.update(transcribe_blob(b)[0]),
            language_label.update(transcribe_blob(b)[1])
        ),
        on_start=lambda: print("Grabación iniciada..."),
        on_silence=lambda: transcription_box.update("Silencio detectado"),
        on_stop=lambda: transcription_box.update("Transcripción completada"),
        detect_language=True,              # El componente sabe que se muestra idioma
        allow_sensitivity_adjustment=True,
        show_theme_switcher=True,
        canvas_width=500,
        canvas_height=500
    )

demo.launch()
```

**Puntos clave**

* **`detect_language=True`** indica al componente que muestre el idioma detectado en la UI.
* El callback `on_audio_chunk` retorna una tupla `(texto, etiqueta_idioma)`, que se asigna a los componentes `transcription_box` y `language_label`.
* La transcripción se actualiza en tiempo real, fragmento a fragmento.

---

### Personalización Avanzada (Props y Callbacks)

```python
import gradio as gr
from audio_sphere_listener import AudioSphereListener

# Simulador de ASR remoto (pseudo-código)
def asr_remote_service(audio_blob):
    # Ejemplo: enviar por HTTP a un endpoint de transcripción
    import requests
    files = {"file": ("chunk.webm", audio_blob, "audio/webm")}
    resp = requests.post("https://api-mi-asr.com/transcribe", files=files)
    data = resp.json()
    return data["text"], data.get("language", "en")

def on_start():
    # Se ejecuta cuando el usuario presiona “🎤 Comenzar”
    return "▶️  Grabando..."

def on_silence():
    # Se ejecuta al detectar silencio (2 s por debajo del umbral)
    return "⏸️  Silencio detectado"

def on_stop():
    # Se ejecuta al detener manual o por silencio
    return "⏹️  Grabación detenida"

with gr.Blocks() as demo:
    gr.Markdown("## AudioSphere Listener: Personalización Completa")

    transcription_box = gr.Textbox(label="Transcripción (en Vivo)", interactive=False)
    language_label = gr.Label(label="Idioma Detectado")

    audio_sphere = AudioSphereListener(
        on_start=lambda: transcription_box.update(on_start()),
        on_audio_chunk=lambda b: transcription_box.update(asr_remote_service(b)[0]),
        on_transcription=lambda txt: transcription_box.update(txt),
        on_silence=lambda: transcription_box.update(on_silence()),
        on_stop=lambda: transcription_box.update(on_stop()),

        canvas_width=600,
        canvas_height=400,

        silence_threshold=0.015,       # Umbral de RMS más estricto
        required_silent_chunks=6,      # 6 * 250ms = 1.5s para silencio
        particle_count=15000,          # Mayor detalle de partículas
        particle_size_range=(0.015, 0.07),
        color_theme={
            "background_gradient": ["#001F00", "#003300"],
            "particle_colors": ["#00FF00", "#AAFF00"],
            "rim_color": "#55FF55"
        },
        detect_language=True,
        allow_sensitivity_adjustment=True,
        fallback_to_pixi=False,
        show_theme_switcher=True
    )

demo.launch()
```

**Aspectos de esta configuración**

* Se modifica el `silence_threshold` y `required_silent_chunks` para detectar silencio más rápidamente.
* Se aumenta `particle_count` a 15 000 para una esfera más detallada, y se establecen rangos de tamaño más amplios (`0.015` – `0.07`).
* `color_theme` define un esquema propio (fondo verde oscuro y partículas verde neón).
* Los callbacks `on_start`, `on_audio_chunk`, `on_silence` y `on_stop` actualizan la caja de transcripción con mensajes específicos.

---

## Descripción Técnica Detallada

### Inicialización y Pipeline de Audio

1. **Captura de Micrófono**

   * Se llama a

     ```ts
     navigator.mediaDevices.getUserMedia({ audio: true });
     ```

     solicitando permiso al usuario.
   * Si el permiso es rechazado, se notifica en consola y el componente permanece inactivo.

2. **Creación de `AudioContext` y `AnalyserNode`**

   * `AudioContext` es la instancia principal:

     ```ts
     this.audioCtx = new AudioContext();
     const source = this.audioCtx.createMediaStreamSource(this.mediaStream);
     this.analyser = this.audioCtx.createAnalyser();
     this.analyser.fftSize = 2048;
     source.connect(this.analyser);
     ```
   * `AnalyserNode` proporciona dos salidas:

     * `getByteTimeDomainData(Uint8Array)`: datos de forma de onda (0–255).
     * `getByteFrequencyData(Uint8Array)`: datos de frecuencia (FFT, 0–255 por banda).

3. **Fragmentación de Audio con `MediaRecorder`**

   * Se inicializa:

     ```ts
     this.recorder = new MediaRecorder(this.mediaStream, { mimeType: "audio/webm" });
     this.recorder.ondataavailable = (e: BlobEvent) => { … };
     this.recorder.start(250);
     ```
   * Cada 250 ms, `ondataavailable` recibe un `Blob` que contiene aproximadamente 250 ms de audio comprimido en `audio/webm`.
   * Dentro de este callback se desencadenan dos acciones principales:

     1. **Envío al ASR**: si `on_audio_chunk` está definido, se invoca con el `Blob`.
     2. **Cálculo de RMS**:

        * El `Blob` se lee como `ArrayBuffer` con `FileReader`.
        * `AudioContext.decodeAudioData(arrayBuffer)` → `AudioBuffer a partir de datos PCM float32`.
        * Se extraen muestras: `audioBuffer.getChannelData(0)` → `Float32Array`.
        * Se calcula RMS:

          ```ts
          let sumSquares = 0;
          for (let i = 0; i < pcmData.length; i++) {
            sumSquares += pcmData[i] * pcmData[i];
          }
          const rms = Math.sqrt(sumSquares / pcmData.length);
          ```
        * Se agrega `rms` al `this.rmsHistory` (capacidad = `requiredSilentChunks`).
        * Si todos los valores en `rmsHistory` están por debajo de `silenceThreshold`, se invoca `triggerSilence()`.

4. **Detección de Silencio y AGC/Noise Gate**

   * **RMS** en cada fragmento mide la energía global de la señal.
   * **Ventana deslizante** (últimos *n* fragmentos); si *n* valores consecutivos < `silenceThreshold`, se considera silencio.
   * **Noise Gate y AGC (opcional, AudioWorklet)**:

     * Antes de conectar el `AnalyserNode`, se inserta un `AudioWorkletNode` personalizado (`noise-gate-processor.js`), que filtra automáticamente sonidos por debajo de cierto umbral y aplica ganancia automática para normalizar.
     * Este módulo mejora la calidad de audio enviado al ASR y reduce falsas detecciones de silencio.

---

### Motor 3D (Three.js) y Fallback (PixiJS)

#### Three.js

1. **Inicialización**

   * Se crea:

     ```ts
     this.threeRenderer = new THREE.WebGLRenderer({
       canvas: this.canvas,
       antialias: true,
       alpha: true
     });
     this.threeRenderer.setPixelRatio(window.devicePixelRatio);
     this.threeRenderer.setSize(width, height);
     ```
   * **Escena**: `new THREE.Scene()`
   * **Cámara**: `new THREE.PerspectiveCamera(45, width/height, 0.1, 1000)` posicionada en `(0, 0, 5)`.
   * **Luces**:

     * `AmbientLight(0xffffff, 0.5)`
     * `DirectionalLight(0xffffff, 0.7)` ubicado en `(5, 5, 5)`.
   * **Grupo de partículas**: `this.sphereGroup = new THREE.Group(); this.threeScene.add(this.sphereGroup);`

2. **Geometría y Material de Partículas**

   * **BufferGeometry** con atributos `position` (Float32Array × 3 × *N*) y `color` (Float32Array × 3 × *N*).
   * Distribución de puntos en esfera (algoritmo de esfera de Fibonacci):

     ```ts
     const phi = Math.acos(1 - 2 * (i + 0.5) / particleCount);
     const theta = Math.PI * (1 + Math.sqrt(5)) * (i + 0.5);
     const x = Math.sin(phi) * Math.cos(theta);
     const y = Math.sin(phi) * Math.sin(theta);
     const z = Math.cos(phi);
     positions[3*i] = x;
     positions[3*i+1] = y;
     positions[3*i+2] = z;
     ```
   * **Vertex Colors**: interpolación lineal entre `colorStart` y `colorEnd` para cada partícula:

     ```ts
     colorStart.lerp(colorEnd, i / particleCount).toArray(colors, i * 3);
     ```
   * **PointsMaterial**:

     ```ts
     const material = new THREE.PointsMaterial({
       size: randomBetween(particleSizeRange[0], particleSizeRange[1]),
       vertexColors: true,
       transparent: true,
       opacity: 0.8
     });
     const particles = new THREE.Points(geometry, material);
     this.sphereGroup.add(particles);
     this.particleSystem = particles;
     ```

3. **Render Loop y Animación**

   * Se crea una función `animateThree()` que:

     1. Llama a `requestAnimationFrame(animateThree)`.
     2. Rota suavemente la esfera:

        ```ts
        this.sphereGroup.rotation.y += 0.002;
        this.sphereGroup.rotation.x += 0.001;
        ```
     3. Ajusta tamaño del renderer si cambió el contenedor (responsive).
     4. Si `this.listening === true`:

        * Lee datos de frecuencia:

          ```ts
          const freqData = new Uint8Array(this.analyser.frequencyBinCount);
          this.analyser.getByteFrequencyData(freqData);
          const avgFreq = freqData.reduce((sum, v) => sum + v, 0) / freqData.length / 255;
          ```
        * Modifica `PointsMaterial.size = scaleFactor * maxSize` y `opacity = 0.5 + avgFreq × 0.5`.
     5. Llama a `this.threeRenderer.render(this.threeScene, this.threeCamera)`.

#### PixiJS (Fallback)

1. **Inicialización**

   * Si WebGL no está disponible o `fallback_to_pixi=True`, se crea:

     ```ts
     this.pixiApp = new PIXI.Application({
       view: this.canvas,
       backgroundAlpha: 0,
       antialias: true,
       resolution: window.devicePixelRatio,
       width: width,
       height: height
     });
     this.pixiContainer = new PIXI.Container();
     this.pixiApp.stage.addChild(this.pixiContainer);
     ```

2. **Generación de Partículas en 2D**

   * Se crea un sprite circular para cada partícula:

     ```ts
     const gfx = new PIXI.Graphics();
     const size = randomBetween(particleSizeRange[0] * 100, particleSizeRange[1] * 100);
     const colorHex = PIXI.utils.string2hex(randomBetweenColor(colorStart, colorEnd));
     gfx.beginFill(colorHex);
     gfx.drawCircle(0, 0, size);
     gfx.endFill();
     const texture = this.pixiApp.renderer.generateTexture(gfx);
     const sprite = new PIXI.Sprite(texture);
     // Posición aleatoria proyectada en 2D como si fuera superficie de esfera
     const angle = Math.random() * Math.PI * 2;
     const radius = Math.random() * Math.min(width, height) * 0.4;
     sprite.x = width/2 + radius * Math.cos(angle);
     sprite.y = height/2 + radius * Math.sin(angle);
     sprite.alpha = 0.8;
     this.pixiContainer.addChild(sprite);
     ```
   * Cada tick (`app.ticker.add(this.animatePixi.bind(this))`) rota el contenedor y, si `this.listening === true`, ajusta `sprite.scale.set(1 + avgFreq × 0.5)` y `sprite.alpha = 0.5 + avgFreq × 0.5`.

---

### Detección de Silencio (RMS) y AGC/Noise Gate

1. **Buffer Deslizante de RMS**

   * `this.rmsHistory` almacena hasta `requiredSilentChunks` valores de RMS.
   * Cada vez que se recibe un fragmento de audio (Blob), se calcula RMS:

     ```ts
     function calculateRMS(buffer: Float32Array): number {
       let sumSquares = 0;
       for (let i = 0; i < buffer.length; i++) {
         sumSquares += buffer[i] * buffer[i];
       }
       return Math.sqrt(sumSquares / buffer.length);
     }
     ```
   * Se agrega el valor a `rmsHistory`; si su longitud supera `requiredSilentChunks`, se elimina el elemento más antiguo.
   * Si `rmsHistory.length === requiredSilentChunks` y **todos** los valores < `silenceThreshold`, se invoca `triggerSilence()`.

2. **Noise Gate y AGC (AudioWorklet)**

   * Antes de conectar el `AnalyserNode`, opcionalmente se registra y utiliza un `AudioWorkletNode` que filtra ruidos por debajo de un umbral y aplica ganancia automática:

     ```ts
     await this.audioCtx.audioWorklet.addModule("noise-gate-processor.js");
     const noiseGateNode = new AudioWorkletNode(this.audioCtx, "noise-gate-processor", {
       processorOptions: { threshold: this.silenceThreshold }
     });
     source.connect(noiseGateNode).connect(this.analyser);
     ```
   * El procesador `noise-gate-processor.js` (implementado en Web Audio Worklet) analiza muestras en tiempo real y suprime audio cuando está por debajo de `threshold`, además de normalizar niveles altos.

---

### Sistema de Temas Dinámicos

1. **Archivo `themes.json`**
   Define esquemas de color para el fondo y partículas.

   ```json
   {
     "cyberpunk": {
       "background_gradient": ["#1E1B2D", "#0F0C1B"],
       "particle_colors": ["#FF00FF", "#00FFFF"],
       "rim_color": "#FF0099"
     },
     "neon-green": {
       "background_gradient": ["#001F00", "#003300"],
       "particle_colors": ["#00FF00", "#AAFF00"],
       "rim_color": "#55FF55"
     },
     "galactic-purple": {
       "background_gradient": ["#2B0035", "#1A001D"],
       "particle_colors": ["#AA00FF", "#FF00AA"],
       "rim_color": "#D58AFF"
     }
   }
   ```

2. **Selector de Tema (Dropdown)**

   * Si `show_theme_switcher=True`, en `initControls()` se agrega un `<select>` con las claves de `themes.json` (Cyberpunk, Neon Green, Galactic Purple, etc.).
   * Al cambiar la selección, `applyTheme(themeKey)` actualiza:

     * **Fondo** del contenedor:

       ```ts
       container.style.background = `linear-gradient(135deg, ${background_gradient[0]}, ${background_gradient[1]})`;
       ```
     * **Colores de Partículas**: recalcula el atributo `color` de `BufferGeometry` en Three.js, o regenera texturas de PixiJS si corresponde.
   * Transición suave (0.3 s) mediante CSS (`transition: background 0.3s ease`).

---

### Arquitectura de Plugins Visuales

1. **Interfaz TypeScript `VisualPlugin`**

   ```ts
   export interface VisualPlugin {
     /** Nombre único del plugin */
     name: string;

     /**
      * Inicializa el plugin.
      * - `sceneOrContainer` : instancia de THREE.Scene o PIXI.Container
      * - `props` : configuración del componente (AudioSphereListenerProps)
      */
     initialize(
       sceneOrContainer: THREE.Scene | PIXI.Container,
       props: AudioSphereListenerProps
     ): void;

     /**
      * Actualiza el plugin en cada frame de animación.
      * - `timestamp` : tiempo actual (ms)
      * - `analyserData` : Uint8Array con datos de frecuencia (FFT)
      */
     update(timestamp: number, analyserData: Uint8Array): void;

     /** Limpia recursos y listeners cuando el componente se destruye. */
     dispose(): void;
   }
   ```

2. **Carga Dinámica**

   * En el constructor de `AudioSphereListener`, después de inicializar Three.js/PixiJS, se recorre la carpeta `plugins/` y se importa cada módulo que cumpla con la interfaz `VisualPlugin`.
   * Cada plugin recibe la instancia de la escena (`Three.Scene` o `Pixi.Container`) y `props` para configurarse.
   * En cada iteración del render loop (`animateThree` o `animatePixi`), se llama a `plugin.update(timestamp, analyserData)`.

3. **Ejemplo de Plugin (WaveFloorPlugin.ts)**

   ```ts
   import * as THREE from "three";
   import { VisualPlugin, AudioSphereListenerProps } from "../component";

   export default class WaveFloorPlugin implements VisualPlugin {
     name = "WaveFloorPlugin";
     private floorMesh?: THREE.Mesh;
     private analyser?: AnalyserNode;

     initialize(scene: THREE.Scene, props: AudioSphereListenerProps) {
       this.analyser = props.analyserNode; // Asumiendo que el plugin recibe referencia
       const geometry = new THREE.PlaneGeometry(5, 5, 64, 64);
       const material = new THREE.MeshPhongMaterial({
         color: 0x2222ff,
         side: THREE.DoubleSide,
         flatShading: true
       });
       this.floorMesh = new THREE.Mesh(geometry, material);
       this.floorMesh.rotation.x = -Math.PI / 2;
       this.floorMesh.position.y = -1;
       scene.add(this.floorMesh);
     }

     update(timestamp: number, freqData: Uint8Array) {
       if (!this.floorMesh || !this.analyser) return;
       this.analyser.getByteFrequencyData(freqData);
       const avgLowFreq = freqData.slice(0, freqData.length / 4).reduce((a, b) => a + b, 0) / (freqData.length / 4);
       const displacement = avgLowFreq / 255;
       const geometry = this.floorMesh.geometry as THREE.PlaneGeometry;
       for (let i = 0; i < geometry.attributes.position.count; i++) {
         const y = Math.sin(i + timestamp * 0.001) * displacement * 0.5;
         (geometry.attributes.position as THREE.BufferAttribute).setY(i, y);
       }
       (geometry.attributes.position as THREE.BufferAttribute).needsUpdate = true;
     }

     dispose() {
       if (this.floorMesh) {
         this.floorMesh.geometry.dispose();
         (this.floorMesh.material as THREE.Material).dispose();
         this.floorMesh.parent?.remove(this.floorMesh);
       }
     }
   }
   ```

   * Este plugin crea un “suelo” ondulante que vibra con la energía de las frecuencias bajas.
   * Se suscribe al `AnalyserNode` para extraer datos de frecuencia en `update()`.

---

### Accesibilidad y Atajos de Teclado

1. **Atributos ARIA**

   * Botones:

     ```html
     <button 
       role="button" 
       aria-label="Iniciar grabación" 
       class="audio-sphere-btn start-btn">
       🎤 Comenzar
     </button>
     ```
   * Caja de transcripción:

     ```html
     <div 
       role="region" 
       aria-live="polite" 
       aria-label="Transcripción en vivo" 
       class="audio-sphere-transcription">
       <!-- Texto de transcripción -->
     </div>
     ```

2. **Atajos de Teclado**

   * Dentro del constructor de `AudioSphereListener`, se agrega un listener global:

     ```ts
     document.addEventListener("keydown", (event) => {
       if (event.code === "Space") {
         event.preventDefault();
         this.listening ? this.stopListening() : this.startListening();
       } else if (event.code === "ArrowUp") {
         this.silenceThreshold = Math.max(0, this.silenceThreshold - 0.005);
       } else if (event.code === "ArrowDown") {
         this.silenceThreshold += 0.005;
       }
     });
     ```
   * Se actualiza visualmente el slider de silencio cuando cambia via flechas.
   * Notificaciones de estado (“Silencio detectado”, “Grabación iniciada”) se emiten en la caja de transcripción con `aria-live="polite"` para lectores de pantalla.

---

## Props y Currículum de API

A continuación se detalla cada prop que el desarrollador puede configurar al instanciar `AudioSphereListener` en Gradio:

| Prop                     | Tipo           | Descripción                                                                                                                                                              | Default        |
| ------------------------ | -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------- |
| `on_start`               | `callable`     | Callback en Python que se ejecuta cuando el usuario presiona “🎤 Comenzar”. Retorna texto o None.                                                                        | `None`         |
| `on_audio_chunk`         | `callable`     | Callback que recibe cada `Blob` de audio (\~250 ms). Ideal para enviar al ASR.                                                                                           | `None`         |
| `on_silence`             | `callable`     | Callback que se dispara cuando se detecta silencio constante (RMS < `silence_threshold` por `required_silent_chunks`).                                                   | `None`         |
| `on_stop`                | `callable`     | Callback en Python que se invoca cuando la grabación se detiene manualmente o por detección de silencio.                                                                 | `None`         |
| `on_transcription`       | `callable`     | Callback que recibe transcripciones parciales o finales desde el backend ASR.                                                                                            | `None`         |
| `canvas_width`           | `int`          | Ancho mínimo en píxeles del `<canvas>`. El componente se escala responsivamente si el contenedor es mayor.                                                               | `300`          |
| `canvas_height`          | `int`          | Alto mínimo en píxeles del `<canvas>`. El componente se escala responsivamente según altura disponible en el contenedor.                                                 | `300`          |
| `silence_threshold`      | `float`        | Nivel mínimo de RMS (0–1) para considerar “actividad de voz”. Si el RMS cae por debajo de este umbral en `required_silent_chunks` consecutivos, se dispara `on_silence`. | `0.01`         |
| `required_silent_chunks` | `int`          | Cantidad de fragmentos (cada uno de 250 ms) consecutivos con RMS < `silence_threshold` para disparar `on_silence`.                                                       | `8`            |
| `particle_count`         | `int`          | Número de partículas que forman la esfera. A mayor número, más detalle, pero puede impactar el rendimiento.                                                              | `10000`        |
| `particle_size_range`    | `tuple[float]` | Rango `[minSize, maxSize]` (Three.js units o píxeles en PixiJS) para definir el tamaño inicial de cada partícula.                                                        | `(0.02, 0.05)` |
| `color_theme`            | `dict`         | Esquema de color para fondo y partículas. Estructura:                                                                                                                    |                |

````json
{
  "background_gradient": ["#1E1B2D", "#0F0C1B"],
  "particle_colors": ["#FF00FF", "#00FFFF"],
  "rim_color": "#FF0099"
}
````
                                                                                             | Esquema Cyberpunk por defecto |
| `detect_language`            | `bool`           | Si `true`, se espera que el backend ASR retorne un código de idioma (ISO 639-1). Se muestra en la etiqueta `language_label`.                                              | `False`      |
| `allow_sensitivity_adjustment` | `bool`         | Si `true`, exhibe un slider para ajustar en tiempo real el `silence_threshold`.                                                                                            | `False`      |
| `fallback_to_pixi`           | `bool`           | Si `true`, fuerza el uso de PixiJS en lugar de Three.js (útil para navegadores sin WebGL o baja capacidad gráfica).                                                        | `False`      |
| `show_theme_switcher`        | `bool`           | Si `true`, habilita un dropdown para cambiar entre temas cargados desde `themes.json`.                                                                                    | `False`      |
| `elem_id`                    | `str`            | Identificador HTML único para el contenedor del componente. Si no se provee, se genera automáticamente.                                                                    | Generado     |

---

## Roadmap y Futuras Mejoras

1. **Optimización mediante Web Workers**  
   - Externalizar el cálculo de RMS y FFT a un Web Worker, liberando el hilo principal para mantener 60 fps estables, aun en hardware de gama baja.

2. **Grabación Completa y Exportación**  
   - Funcionalidad para concatenar todos los fragments de `MediaRecorder` en un único `Blob` final y permitir la descarga en formatos `webm`, `ogg` o `mp3`.  
   - Prop `enable_record_download: bool` que, cuando se activa, muestra un botón “Descargar grabación”.

3. **Soporte Multicanal (Estéreo)**  
   - Adaptar el pipeline de audio para procesar y visualizar de forma separada los canales izquierdo y derecho, ya sea mediante dos esferas independientes o un diseño de doble anillo de partículas.

4. **Visualización de Texto en 3D (Three.js TextGeometry)**  
   - Integrar el texto de transcripción como `TextGeometry` que orbita alrededor de la esfera, girando en sincronía con la animación principal y cambien la escala según la claridad de la transcripción.

5. **Módulo de Autenticación y Colaboración en Tiempo Real**  
   - Incorporar WebRTC (TURN/STUN) para permitir que múltiples usuarios compartan en tiempo real su esfera de visualización en conferencias remotas, cada quien con su propia instancia de AudioSphere.  
   - Prop `session_id: str` que identifica la sala colaborativa y sincroniza audio/visual a través de WebSockets.

---

## Contacto y Soporte

- **Repositorio Oficial**:  
  https://github.com/tu-usuario/audio_sphere_listener  

- **Documentación Adicional y Ejemplos**:  
  - Carpeta `/examples/`: proyectos de muestra que muestran diferentes escenarios de uso.  
  - `/docs/`: guías detalladas para desarrollo de plugins y temas personalizados.

- **Preguntas y Reporte de Errores**:  
  - Cree un **Issue** en GitHub describiendo el problema con detalle (versión de navegador, logs de consola, sistema operativo).  
  - Para solicitudes de nuevas características, agregue un **Feature Request** describiendo el caso de uso y posibles soluciones.

- **Licencia**:  
  Apache 2.0 License. Puedes revisar el archivo [APACHE-2.0](https://www.apache.org/dev/) en el repositorio.

---

**AudioSphere Listener** está diseñado para elevar la experiencia de interacción por voz en aplicaciones Gradio. Su combinación de renderizado 3D avanzado, detección de silencio inteligente, integración con ASR y arquitectura extensible lo convierte en la solución ideal para cualquier proyecto que demandé un feedback visual y auditivo de vanguardia.
