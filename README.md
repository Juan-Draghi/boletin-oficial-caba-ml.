# Aplicación de Machine Learning para la identificación de normativa relevante en el Boletín Oficial de la Ciudad de Buenos Aires

Este repositorio contiene el código y parte del contenido generado para el trabajo final de la **Diplomatura en Inteligencia Artificial Aplicada a Entornos Digitales de Gestión (FCE–UBA, 2025)**.

El proyecto aborda un problema real de trabajo: **detectar automáticamente normativa relevante para el ejercicio profesional de la arquitectura y el urbanismo en el Boletín Oficial de la Ciudad de Buenos Aires (BOCBA)**, reduciendo la carga de revisión manual diaria.

---

## 1. Objetivo del proyecto

Desarrollar un **clasificador supervisado** que, dado un fragmento de texto extraído del BOCBA, indique si la norma es **pertinente** o **no pertinente** para el ámbito profesional de la arquitectura y el urbanismo.

En particular:

- Disminuir el tiempo de lectura y selección manual de normas.
- Reducir el riesgo de pasar por alto normativa relevante.
- Contar con un pipeline actualizable, que pueda seguir aprendiendo a partir de nuevo feedback.

---

## 2. Datos y construcción del dataset

### Fuente

- Boletín Oficial de la Ciudad de Buenos Aires (PDFs descargados desde el sitio oficial).
- Período analizado:
  - **Train**: 2018 – 1er semestre de 2024
  - **Validación (val)**: 2º semestre de 2024
  - **Test**: 2025 (ampliado en meses para aumentar la cantidad de positivos)

Se incluyen los datasets completos.

Cada archivo contiene, entre otras, las columnas:

- `contexto`: fragmento de texto extraído del BO.
- `label`: 1 si el fragmento se considera pertinente, 0 en caso contrario.
- `origen_pdf`: nombre del archivo del boletín.
- `pagina`: número de página en el boletín.

### Extracción de candidatos

La detección de fragmentos candidatos se realiza combinando:

- **VERBOS_ACCION**: verbos típicos de acción normativa (aprueba, deroga, modifica, etc.).
- **KEYWORDS**: términos clave vinculados al ejercicio profesional (Código Urbanístico, Registro de Profesionales, etc.).
- **PATRONES_NORMAR**: expresiones regulares que capturan referencias a normas específicas (leyes, decretos, resoluciones, etc.).

En la versión de producción, se consideran candidatos los fragmentos que cumplen:

- `VERBO AND KEYWORD AND NORMAR`, o
- `VERBO AND (KEYWORD OR NORMAR)`

El texto se segmenta en oraciones y se arma una **ventana de contexto** alrededor de la oración que dispara la coincidencia.

### Etiquetado y curación

- Se desarrolló una interfaz con **Gradio** para:
  - Visualizar el fragmento de contexto.
  - Editarlo para dejar solo la parte relevante (sumario y/o articulado).
  - Asignar etiqueta `pertinente` / `no pertinente`.
- Posteriormente se hizo una **revisión manual** para asegurar:
  - Que todas las normas relevantes del período estuvieran presentes.
  - Que las leyes “ómnibus” se descompusieran en registros separados por artículo.

El resultado es un dataset con proporciones de positivos realistas (alrededor del 30% en train y valores más bajos en val/test, que reflejan la escasez real de normativa relevante).

---

## 3. Modelado: baseline clásico

La primera etapa de modelado se realizó con **Machine Learning clásico**, utilizando:

- Representación de texto: `TF-IDF` (unigramas y bigramas).
- Modelos probados:
  - Regresión Logística
  - Naive Bayes
  - Random Forest
  - **Máquinas de Vectores de Soporte (SVM)**

Dado que el objetivo principal es **no perder normas relevantes**, se priorizó:

- **Recall (sensibilidad)** sobre precisión.
- Optimización con la métrica **F2** (que pondera más el recall).
- Ajuste de umbral de decisión a partir de curvas de precisión–recall.

El mejor rendimiento se obtuvo con **TF-IDF + SVM**, que mostró:

- Alto **recall** en el conjunto de test.
- Tasas de falsos negativos muy bajas (pocos casos relevantes sin detectar).
- AUC-ROC y AUC-PR elevadas, dadas las proporciones de clase del problema.

---

## 4. Fine-tuning de modelo de lenguaje

Se exploró el **fine-tuning** de un modelo de lenguaje especializado en dominio legal:

- Modelo utilizado: [`BSC-LT/RoBERTalex`](https://huggingface.co/BSC-LT/RoBERTalex)
- Tarea: clasificación binaria de pertinencia sobre los mismos fragmentos de texto.

En este caso, el modelo de lenguaje:

- No superó el rendimiento de **TF-IDF + SVM** en el conjunto de test.
- Posibles razones:
  - Tamaño relativamente reducido del dataset supervisado.
  - Desajuste entre el dominio legal-administrativo en el que fue pre-entrenado el modelo (España) y la normativa específica de CABA.
  - El baseline clásico ya captura bien patrones muy claros en sumarios y articulados.

Este resultado refuerza la importancia de:

- Ajustar expectativas frente a modelos grandes.
- Evaluar siempre contra un baseline clásico bien calibrado.

---

## 5. Pipeline operativo

El proyecto se concretó en un **pipeline utilizable en el trabajo diario**, basado en tres notebooks:

1. **`06_BO_SVM.ipynb`**
   - Carga un PDF reciente del Boletín Oficial.
   - Extrae candidatos según las reglas (`VERBO + KEYWORD/NORMAR`).
   - Aplica el modelo **TF-IDF + SVM**, usando un umbral ajustado por F2.
   - Genera un CSV con fragmentos clasificados como pertinentes / no pertinentes.

2. **`07_BO_SVM_feedback.ipynb`**
   - Interfaz gráfica con **Gradio** para:
     - Leer `contexto`, `origen_pdf`, `score_svm`, `pred_pertinente`.
     - Editar el texto del fragmento.
     - Confirmar o corregir la etiqueta.
   - Las correcciones se acumulan en un archivo de feedback (`train_feedback_master.csv`).

3. **`08_BO_SVM_retrain.ipynb`**
   - Reentrena periódicamente el modelo:
     - Usa `train` original + feedback acumulado.
     - Recalcula el umbral óptimo t_F2 usando el conjunto de validación.
   - Evalúa en test y guarda:
     - El nuevo pipeline TF-IDF + SVM.
     - El nuevo umbral de decisión.

De esta manera, el modelo puede **seguir aprendiendo con el uso cotidiano**, incorporando nueva normativa y mejorando progresivamente.

---

## 6. Estructura del repositorio

```text
boletin-oficial-caba-ml/
├── README.md
├── requirements.txt
├── .gitignore
├── notebooks/
│   ├── 01_dataset_builder.ipynb
│   ├── 02_triage_etiquetado.ipynb
│   ├── 03_baseline_tfidf_4modelos.ipynb
│   ├── 04_finetuning_robertalex.ipynb
│   ├── 05_bo_svm_inferencia.ipynb
│   ├── 06_bo_svm_feedback.ipynb
│   └── 07_bo_svm_retrain.ipynb
├── data/
│   ├── README_data.md
│   └── samples/
│       ├── sample_train.csv
│       ├── sample_val.csv
│       └── sample_test.csv
├── models/
│   └── (no se versiona: modelos entrenados)
└── docs/
    └── informe_trabajo_final.pdf

```

## 7. Autor

Juan Draghi – Bibliotecario, Consejo Profesional de Arquitectura y Urbanismo (CPAU).
Este proyecto fue desarrollado como trabajo final de la Diplomatura en IA Aplicada a Entornos Digitales de Gestión (FCE–UBA, 2025)
