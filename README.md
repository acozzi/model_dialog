# model_dialog — Clasificacion de logs de WAF con LLMs chicos

Este repositorio compara dos enfoques para clasificar lineas de log de un
**Web Application Firewall** (WAF) usando modelos de lenguaje pequenos
(TinyLlama-1.1B y Qwen3-0.6B), entre si y contra un ground truth:

1. **Prompting + salida estructurada por JSON Schema** sobre los modelos base
   servidos por Ollama.
2. **Fine-tuning con LoRA** de los mismos modelos base sobre un dataset
   sintetico de logs con y sin ataque.

Los dos notebooks estan pensados para correr en Google Colab con GPU T4.

## Contenido

| Notebook | Que hace |
|---|---|
| [`waf_finetune_tinyllama_qwen3.ipynb`](waf_finetune_tinyllama_qwen3.ipynb) | Fine-tunea TinyLlama-1.1B y Qwen3-0.6B con LoRA sobre un dataset sintetico de logs HTTP/WAF. Publica los adaptadores en HuggingFace. |
| [`waf_clasificador_tinyllama_qwen.ipynb`](waf_clasificador_tinyllama_qwen.ipynb) | Clasifica un set de logs de test con los **4 modelos** (los 2 base servidos por Ollama y los 2 fine-tuneados) y compara accuracy, tiempos y consistencia. |

## Categorias de clasificacion

Ambos notebooks etiquetan cada log como `normal` o `attack`; en el segundo
caso identifican el tipo de ataque. El notebook de fine-tuning maneja el
vocabulario completo (`sql_injection`, `xss`, `path_traversal`,
`command_injection`, `ssrf`, `xxe`) en espanol; el clasificador comun usa
un schema en ingles (`sqli`, `xss`, `path_traversal`, `command_injection`,
`other`) y traduce los tipos del fine-tuning al schema comun para poder
cruzar resultados.

## Flujo recomendado

### 1) Entrenar los modelos

Abrir [`waf_finetune_tinyllama_qwen3.ipynb`](waf_finetune_tinyllama_qwen3.ipynb)
en Colab (T4) y correr las celdas en orden. Pasos clave:

1. **Setup** — instala `unsloth`, `bitsandbytes`, `transformers`, `trl` y
   `torchao` con versiones pineadas para Python 3.13. Si es una VM nueva
   pide reiniciar la sesion.
2. **Dataset sintetico** — genera ~1.900 lineas de log estilo Apache/Nginx
   mezclando payloads canonicos de OWASP (por categoria) con trafico
   benigno. Split 85/15 estratificado por tipo.
3. **Entrenamiento** — `entrenar_y_evaluar(cfg)` corre el ciclo completo
   (carga en 4bit -> LoRA en todas las lineales -> tokenizacion con mascara
   `-100` sobre el prompt -> SFTTrainer -> evaluacion -> guardado del
   adaptador) para cada modelo. Los dos modelos entrenan uno tras otro en
   la misma sesion.
4. **Comparacion** — tabla y grafico de accuracy binaria (attack/normal) vs.
   accuracy del tipo de ataque, con detalle de errores.
5. **Publicacion** — sube cada adaptador LoRA a `alcozzi/waf-classifier-<key>`
   en HuggingFace (requiere `HF_TOKEN`).

### 2) Clasificar y comparar

Abrir [`waf_clasificador_tinyllama_qwen.ipynb`](waf_clasificador_tinyllama_qwen.ipynb)
en Colab y correr las celdas en orden. Pasos clave:

1. **Setup de Ollama** — instala Ollama con soporte GPU (`zstd` + `pciutils`),
   lo levanta como servidor local y hace pull de `tinyllama` y `qwen3:0.6b`.
2. **Setup de Unsloth** — instala Unsloth para poder cargar los adaptadores
   LoRA fine-tuneados desde el Hub de HuggingFace.
3. **Schema + few-shot** — define el JSON Schema que Ollama impone al
   output, el system prompt del analista SOC y 10 ejemplos few-shot (uno
   normal + dos por cada tipo de ataque).
4. **Guardarrail** — `_fix_inconsistencies` corrige de manera deterministica
   inconsistencias del LLM (por ejemplo, "attack" sin `attack_type`, o
   "normal" con un `attack_type` no nulo).
5. **Comparacion** — corre un dataset de test de 12 logs contra los 4
   modelos y compila una tabla comparativa con accuracy binaria, accuracy
   de tipo, tiempo promedio, confianza promedio y cantidad de correcciones
   aplicadas.

## Requisitos

- Google Colab (GPU T4) o equivalente con:
  - `torch` con CUDA
  - `bitsandbytes` (para carga en 4 bits)
- Cuenta de HuggingFace con `HF_TOKEN` (necesario si los adaptadores son
  privados).
- Ollama instalado en el runtime (el notebook lo hace).

Las versiones de Python/pip estan pineadas en las celdas de instalacion de
cada notebook.

## Decisiones de diseno

- **Salida estructurada.** En el notebook de clasificacion se usa
  `format=` de Ollama con un JSON Schema para forzar output valido; en el
  notebook de fine-tuning el modelo *aprende* a emitir el JSON directamente
  (el schema queda "grabado" en los pesos).
- **Few-shot vs. fine-tuning.** El clasificador base recibe 10 ejemplos
  few-shot en cada llamada (como turnos previos, no como texto en el
  prompt); los fine-tuneados no reciben ejemplos porque ya los
  interiorizaron durante el entrenamiento.
- **LoRA sobre todas las lineales.** A diferencia del cuaderno de
  `gpt-oss-20b` que inspiro este trabajo, aca no hay MoE ni riesgo de que
  el LoRA explote en parametros, asi que se entrenan atencion + MLP
  (`q_proj, k_proj, v_proj, o_proj, gate_proj, up_proj, down_proj`).
- **Mascara del prompt.** La loss se calcula **solo** sobre la respuesta
  del asistente (etiqueta -100 en todo system+user). La mascara se arma a
  mano en `preparar_dataset` en vez de usar `train_on_responses_only` de
  Unsloth para evitar errores conocidos en Python 3.13.
- **Compatibilidad de TRL.** Las versiones nuevas renombraron
  `max_seq_length` a `max_length` en `SFTConfig` y `tokenizer` a
  `processing_class` en `SFTTrainer`. `entrenar_y_evaluar` lo detecta por
  introspeccion en vez de hardcodear.

## Estructura del repositorio

```
model_dialog/
├── README.md                                  <- este archivo
├── waf_finetune_tinyllama_qwen3.ipynb         <- fine-tuning con Unsloth+LoRA
└── waf_clasificador_tinyllama_qwen.ipynb      <- comparacion prompting vs fine-tuning
```
