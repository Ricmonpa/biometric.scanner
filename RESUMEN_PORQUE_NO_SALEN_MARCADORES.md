# Por qué no salen los puntitos / marcadores faciales — Resumen para analizar

## Cuándo se dibujan los puntos

En el código, los 68 puntos (marcadores) **solo** se pintan cuando:

```js
if (results.length > 0) {
  drawFaceMeshCyan(ctx, results, w, h);
  // ...
}
```

Es decir: **si `faceapi.detectAllFaces(...).withFaceLandmarks()` devuelve al menos una cara**. Si devuelve 0 caras, no se llama a `drawFaceMeshCyan` y no se ve ningún punto.

---

## Causas posibles por las que NO se ven los puntos

### 1. La detección devuelve 0 caras (`results.length === 0`)

- **Qué pasa:** El overlay muestra "CARAS: 0" / "Sin cara detectada". No se ejecuta `drawFaceMeshCyan`.
- **Por qué podría pasar:**
  - **Modelos no cargan:** Si `tinyFaceDetector` o `face_landmark_68` fallan (404, CORS, ruta `./weights` incorrecta), `faceMeshReady` puede no ponerse en `true` o la detección falla. Revisar consola por errores de red o de face-api.
  - **Entrada al detector:** Ahora se usa `faceapi.detectAllFaces(video, ...)`. En algunos navegadores/móviles el elemento `<video>` no entrega un frame válido a TensorFlow (por seguridad, rendimiento o implementación). Resultado: siempre 0 caras.
  - **TinyFaceDetector muy estricto:** Con `scoreThreshold: 0.1` e `inputSize: 416`, en mala luz, cara pequeña o ángulo raro puede no superar el umbral y devolver 0 caras.
  - **Backend TensorFlow:** Se fuerza CPU en Chrome/Android. Si hay error al usar el backend, la detección puede fallar en silencio o devolver [].

### 2. La detección devuelve caras pero los puntos se pintan fuera de vista

- **Qué pasa:** `results.length > 0` pero no se ven puntos en pantalla.
- **Por qué podría pasar:**
  - **Coordenadas mal interpretadas:** En `drawFaceMeshCyan` se convierten las posiciones a píxeles. Si face-api devuelve coordenadas normalizadas (0–1) y el código las trata como absolutas (o al revés), los puntos pueden quedar en (0,0), fuera del canvas o en una esquina. Lo mismo si se usa mal el `box` (box-relative).
  - **Canvas con tamaño incorrecto:** Si `faceMeshCanvas.width/height` no coinciden con el video (p. ej. 300x150 por defecto vs video 640x480), las coordenadas calculadas no coinciden con lo que se muestra y los puntos pueden dibujarse fuera del área visible.

### 3. Capas / CSS tapan el canvas

- **Qué pasa:** Se dibuja en `faceMeshCanvas` pero algo lo tapa.
- **En el código:** El video tiene `z-index: 1`, el canvas `z-index: 15`, el overlay `z-index: 10`. El canvas está por encima del overlay. Si algún hijo del overlay tiene `z-index` alto o está posicionado encima, podría tapar los puntos. Revisar que ningún elemento cubra el área central donde deberían verse los puntos.

### 4. Canvas solo con puntos (sin rellenar el fondo)

- **Qué pasa:** El canvas tiene fondo transparente y solo se dibujan los 68 puntos. El video (con filtro) está debajo. Si el canvas no tiene tamaño interno correcto (`width`/`height`) o está en 0x0, no se verá nada.
- **En el código:** Cada frame se hace `faceMeshCanvas.width = w; faceMeshCanvas.height = h` (con `w,h` del video). Si `w` o `h` son 0 o no se ejecuta ese bloque (p. ej. early return por dimensiones inválidas), el canvas puede quedar vacío o mal dimensionado.

---

## Qué comprobar en el navegador (para Claude / debugging)

1. **Consola (F12):**
   - ¿Aparece "FACE MESH: listo" o algún error al cargar modelos?
   - ¿Hay errores de red (404, CORS) al cargar `./weights/*`?
   - ¿Qué sale en el log de dimensiones? (`videoW`, `videoH`, `canvasW`, `canvasH`, `match`).

2. **Overlay de debug (si sigue visible):**
   - Si dice "CARAS: 0" → el problema es **detección** (nunca se llega a dibujar).
   - Si dice "CARAS: 1" (o más) y aun así no se ven puntos → el problema es **dibujo o coordenadas** (o algo tapa el canvas).

3. **Un solo frame de prueba (si puedes añadir un `console.log` una vez cuando `results.length > 0`):**
   - Imprimir `results[0].landmarks.positions[0]` y `results[0].landmarks.positions[67]` (y si hay `box`, el `box`).
   - Así se ve si las coordenadas son normalizadas (0–1), en píxeles o relativas al box.

4. **Detección con canvas en vez de video:**
   - En algunos entornos es más fiable pasar un canvas al que se acaba de dibujar el frame del video (mismo tamaño que el video) que pasar el elemento `<video>`. Si al cambiar a `detectAllFaces(canvasConFrameDelVideo, ...)` empiezan a aparecer caras, el fallo estaba en leer desde `video`.

---

## Resumen en una frase

**No se ven los puntitos porque o bien la detección devuelve 0 caras (y entonces no se dibuja nada), o bien se dibujan pero en coordenadas incorrectas o tapadas; el código actual solo pinta puntos cuando `results.length > 0`.**
