# CheckPCBA_IPC610

Inspección visual de soldadura en placas de circuito impreso (PCB) con un
detector **YOLO anchor-free** que corre **íntegramente en el navegador**
(ONNX Runtime Web · WebAssembly). Cada unión se clasifica contra **IPC-A-610G**
y se marca con un color:

* 🟢 **verde** — unión conforme (§5.1).
* 🟠 **naranja** — aceptable con observación: superficie mate/granulosa,
  cobertura mínima. Indicador de proceso (§5.1, Tabla 7-4, §8.3.1).
* 🔴 **rojo** — no conforme: unión fría, soldadura insuficiente, falta de
  humectación, vacíos/poros, puente o ausencia de soldadura (§5.2).

La app abre la cámara o una foto, analiza por **teselas solapadas**, permite
**ampliar el visor** (rueda, botones o pellizco) para revisar cada caja y genera
un **informe PDF** con la evidencia anotada, vistas ampliadas de las zonas
críticas, resumen de hallazgos y recomendaciones citando la norma.

**App publicada:** <https://ojperdomoc.github.io/CheckPCBA_IPC610_UNAL/>

> Ninguna imagen sale del dispositivo: la inferencia y el PDF se generan en el
> navegador. Todas las dependencias (`vendor/`) y el modelo (`modelos/`) están
> versionados en el repositorio; no hay CDN ni backend.

## Estructura

```
index.html              app de inspección (entrada del sitio)
assets/                 app.js, detector.js, ipc.js, report.js, app.css, tarjeta del modelo
modelos/                checkpcba-solder.onnx (1.2 MB) + metadatos y métricas
muestras/               fotos de ejemplo con sus etiquetas (gt/) para validar
vendor/                 ONNX Runtime Web 1.22 (build WASM) + jsPDF
training/               renderizador sintético, dataset, entrenamiento, evaluación y exportación
tools/serve_preview.py  servidor local (redirige la raíz a la app)
Logo_UNAL.png           identidad institucional
```

## Servir en local

```bash
python3 tools/serve_preview.py        # http://localhost:8080/
```

El script escucha en `0.0.0.0`, sirve `.wasm` con el MIME correcto y desactiva
la caché. También sirve cualquier servidor estático apuntando a la raíz.

La cámara del navegador exige **HTTPS** o `localhost`; GitHub Pages ya cumple
esa condición.

## Modelo

| Métrica | Valor |
| --- | --- |
| Arquitectura | CNN separable + FPN ligero, 3 cabezas (strides 8/16/32) |
| Parámetros / tamaño | 0.30 M / 1.2 MB (ONNX fp32) |
| Entrada | 384 × 384 px, RGB, normalización ImageNet |
| Clases | 9 (`ok`, `warn_dull`, `warn_coverage`, `fail_cold`, `fail_insufficient`, `fail_nonwet`, `fail_void`, `fail_bridge`, `fail_missing`) |
| Placas completas (conf. 0.35) | P 0.63 · R 0.93 · **F1 0.75** |
| Recortes de validación | P 0.71 · R 0.72 · F1 0.72 |
| Latencia | ~0.8–1.3 s por foto en escritorio (20–24 teselas) |

Detalle por clase, receta de entrenamiento y limitaciones:
[`assets/checkpcba-solder-card.md`](assets/checkpcba-solder-card.md).

## Reproducir el entrenamiento

```bash
python -m venv .venv && . .venv/bin/activate
pip install torch numpy pillow onnx onnxruntime onnxscript pypdf pymupdf

python training/dataset.py --out training/data --boards 380 --val-boards 40
python training/train.py --data training/data --epochs 16 --size 384 \
    --out modelos/checkpcba-solder.onnx
python training/fix_valfull.py                 # etiquetas de placas completas
python training/eval_onnx.py --model modelos/checkpcba-solder.onnx
python training/make_samples.py                # fotos de ejemplo de la app
```

El dataset se genera con `training/pcb_synth.py` (placas sintéticas con chips,
SOIC/QFN, capacitores electrolíticos, conectores, *through-hole*, serigrafía,
cobre, brillos especulares y efectos de cámara), con etiquetas por unión.

## Limitaciones

* Entrenado con imágenes **sintéticas**: no cubre todas las variantes
  industriales (pastas, acabados, reflujo en vacío, rayos X).
* Clases débiles en precisión: `fail_insufficient` (P 0.32), `fail_void`
  (0.38) y `fail_bridge` (0.41).
* Uniones ocultas (BGA, BTC, QFN con almohadilla térmica) **no** son evaluables
  por inspección visual superficial.
* Es una **ayuda** de inspección: no sustituye la verificación humana ni los
  ensayos eléctricos/funcionales.

## Publicación

`.github/workflows/deploy-pages.yml` publica el sitio en GitHub Pages con el
flujo oficial (`configure-pages`, `upload-pages-artifact`, `deploy-pages`).
El paso de configuración usa `enablement: true`, así que **la primera
ejecución del workflow activa sola GitHub Pages** (Source: GitHub Actions);
no hace falta tocar Settings → Pages. Se dispara en cada push a `main` y
manualmente con `workflow_dispatch`.

URL pública: <https://ojperdomoc.github.io/CheckPCBA_IPC610_UNAL/>
