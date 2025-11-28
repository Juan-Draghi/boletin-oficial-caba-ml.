# Detector de Normativa Legal para Arquitectos (CPAU)

> **Automatización del relevamiento normativo mediante Machine Learning: Reduciendo la carga manual y minimizando el riesgo legal.**

Este proyecto implementa un pipeline de NLP supervisado para detectar automáticamente normativa relevante para el ejercicio de la arquitectura y el urbanismo en el **Boletín Oficial de la Ciudad de Buenos Aires (BOGCBA)**.

Desarrollado como trabajo final para la Diplomatura en IA Aplicada (FCE–UBA, 2025) y actualmente en producción para uso interno.

---

## 🎯 El Problema de Negocio
Los bibliotecarios y referencistas del Consejo Profesional (CPAU) deben revisar diariamente cientos de páginas de documentos legales para encontrar normas críticas (códigos de edificación, habilitaciones, etc.).
* **El Costo:** Horas de lectura manual propensas a la fatiga.
* **El Riesgo:** Pasar por alto una modificación normativa ("Falso Negativo") puede tener consecuencias legales graves para los matriculados.

## 💡 La Solución
Un clasificador binario que procesa los PDFs del Boletín Oficial, extrae fragmentos candidatos mediante heurísticas y utiliza un modelo de Machine Learning para determinar su pertinencia con un **Recall (Sensibilidad) superior al 85%**.

---

## 🚀 Hallazgos Técnicos Clave: SVM vs. LLMs
Uno de los puntos más interesantes de este proyecto fue la comparativa de costo-efectividad entre métodos clásicos y Deep Learning.

| Enfoque | Modelo | Resultado | Conclusión |
| :--- | :--- | :--- | :--- |
| **ML Clásico** | **TF-IDF + SVM** | 🏆 **Ganador** | Mejor manejo de pocos datos, más rápido, F1-Score superior (0.75). |
| **LLM Fine-Tuning** | **RoBERTalex** | 📉 Inferior | Sufrió de *overfitting* por el tamaño del dataset y desajuste de dominio (España vs. Argentina). |

**Decisión de Arquitectura:** Se implementó **SVM** en producción. Esto demuestra que, para tareas de clasificación de texto con dominios muy específicos y datasets limitados (<3000 ejemplos), un modelo clásico bien calibrado suele superar a los Transformers, siendo infinitamente más barato de mantener.

---

## 📊 Metodología y Métricas

### 1. Enfoque "Recall-First"
En el ámbito legal, un Falso Positivo (leer una norma irrelevante) es una molestia menor, pero un **Falso Negativo (perderse una ley) es inaceptable**.
* Se optimizó el modelo priorizando la métrica **F2-Score** y el **Recall**.
* El umbral de decisión no es el estándar (0.5), sino uno calibrado específicamente para capturar la mayor cantidad de positivos posibles.

### 2. Construcción del Dataset (Human-in-the-loop)
* **Fuente:** PDFs del BOGCBA (2018–2025).
* **Curación:** Etiquetado manual asistido por una interfaz en **Gradio**.
* **Split Temporal:** Train (2018-2024) / Test (2025). Se valida con "el futuro" para simular el escenario real de producción.

---

## 🛠️ Stack Tecnológico y Pipeline

El sistema funciona con un flujo de 3 etapas modularizadas:

1.  **Ingesta y Filtrado (Reglas):**
    * Extracción de texto de PDF.
    * Triangulación de candidatos usando Regex: `VERBOS_ACCION` + (`KEYWORDS` o `PATRONES_NORMATIVOS`).
2.  **Clasificación (ML):**
    * Vectorización TF-IDF (1-2 n-grams).
    * Inferencia con Support Vector Machine (SVM).
3.  **Feedback Loop (Mejora Continua):**
    * Interfaz visual (Gradio) para que el experto humano valide las predicciones diarias.
    * Los errores del modelo se reinyectan en el dataset de entrenamiento (`08_BO_SVM_retrain.ipynb`).

---

## 📂 Estructura del Repositorio

```text
├── notebooks/
│   ├── 01-03_Dataset_Builder/   # Extracción, limpieza y etiquetado (Gradio)
│   ├── 04_Baseline_ML/          # Comparativa: Regresión Logística, Naive Bayes, RF, SVM
│   ├── 05_FineTuning_LLM/       # Experimentos con RoBERTalex (Hugging Face)
│   └── 06-08_Production/        # Pipeline diario: Inferencia -> Feedback -> Reentrenamiento
├── data/                        # Datasets (train/val/test) anonimizados
├── models/                      # Serializados (.pkl) del modelo ganador
└── docs/                        # Informe técnico detallado
```

## 🔄 Evolución del proyecto

Este trabajo es la cuarta iteración de una serie de prototipos para automatizar el relevamiento normativo en el Boletín Oficial de CABA:

1. **v1 – Búsqueda por palabras clave**  
   Script en Python que recorre PDFs y exporta coincidencias a Excel.

2. **v2 – Palabras clave + verbos de acción normativa**  
   Reducción de falsos positivos incorporando verbos como "aprueba", "deroga", etc.

3. **v3 – LLM vía API**  
   Uso de un modelo de lenguaje para evaluar la relevancia de los fragmentos candidatos.

4. **v4 – Clasificación supervisada (repo actual)**  
   Construcción de un dataset etiquetado, entrenamiento de modelos clásicos de ML, 
   exploración de fine-tuning y despliegue de un pipeline SVM + TF-IDF.


## ✒️ Autor

Juan Draghi – Bibliotecario, Consejo Profesional de Arquitectura y Urbanismo (CPAU).
Este proyecto fue desarrollado como trabajo final de la Diplomatura en IA Aplicada a Entornos Digitales de Gestión (FCE–UBA, 2025)

## Uso de herramientas de IA

Parte del diseño metodológico, del código en Python y de la documentación de este proyecto fue asistida mediante el uso de modelos de lenguaje (ChatGPT, OpenAI), utilizados como apoyo durante la cursada de la Diplomatura en Inteligencia Artificial.

Las decisiones de diseño, la definición del problema, la curaduría del dataset y la validación de resultados fueron realizadas por el autor.

