# Sistema de Predicción de Deserción Estudiantil
 
> **Leyenda de estado** usada en todo el documento:
> - ✅ **Implementado** — declarado en el prompt como parte del notebook `desercion_estudiantil.ipynb`.
> - 🟡 **Supuesto a verificar** — depende del contenido del notebook, que **no estaba disponible** al redactar este documento.
> - 🔵 **Recomendado / por desarrollar** — propuesta nueva; no existe todavía.
 
> ⚠️ **Aviso de alcance.** Este documento se elaboró a partir de la descripción del prompt, sin acceso al archivo `desercion_estudiantil.ipynb`. Los valores concretos (rangos de variables, codificación de `genero` y `trabaja`, reporte de clasificación por clase) deben contrastarse con el notebook antes de implementar.
 
---
 
## 1. Nombre del proyecto
 
**DesertIA** *(nombre sugerido 🔵; puede cambiarse)* — Sistema de predicción de riesgo de deserción universitaria.
 
## 2. Descripción general
 
Sistema para identificar estudiantes universitarios con riesgo de desertar mediante Machine Learning. El modelo actual ✅ es una **regresión logística** que estima, a partir de siete variables, si un estudiante deserta o termina la universidad. Este documento propone la arquitectura para convertir ese notebook en una aplicación completa (API, interfaz, almacenamiento y despliegue).
 
## 3. Problema que busca resolver
 
La deserción universitaria suele detectarse tarde. El sistema busca anticipar el riesgo con variables que la institución ya conoce (edad, asistencia, horas de estudio, cursos desaprobados, situación laboral), para que tutores y bienestar estudiantil puedan intervenir a tiempo.
 
## 4. Objetivo general
 
Diseñar e implementar un sistema que prediga la probabilidad de deserción de un estudiante y la clasifique en niveles de riesgo interpretables, de forma segura y confidencial.
 
## 5. Objetivos específicos
 
1. Reproducir el entrenamiento del notebook en un pipeline versionado y reutilizable.
2. Exponer el modelo mediante una API con validación estricta de entradas.
3. Interpretar correctamente la probabilidad de deserción usando `modelo.classes_`.
4. Ofrecer una interfaz para registrar estudiantes, consultar predicciones y ver estadísticas.
5. Persistir resultados e historial con control de acceso y auditoría.
6. Evaluar y mejorar el modelo (desbalance, umbrales, validación cruzada, otros algoritmos).
---
 
## 6. Tecnologías
 
### 6.1 Ya utilizadas ✅ (según el prompt)
 
| Tecnología | Uso |
|---|---|
| Python | Lenguaje base |
| Pandas, NumPy | Manipulación de datos |
| Matplotlib, Seaborn | Exploración y gráficos |
| Scikit-learn | Modelado y métricas |
| `StandardScaler` | Normalización |
| `LogisticRegression` | Modelo (`class_weight='balanced'`, `max_iter=1000`) |
 
Configuración del experimento ✅: partición 80 % entrenamiento / 20 % prueba, `random_state=42`; evaluación con accuracy, matriz de confusión y `classification_report`.
 
### 6.2 Recomendadas 🔵 (no implementadas)
 
| Tecnología | Propósito |
|---|---|
| FastAPI + Pydantic | Backend y validación de entradas |
| `joblib` | Serializar el pipeline entrenado |
| PostgreSQL + SQLAlchemy + Alembic | Persistencia y migraciones |
| React (o Streamlit como prototipo rápido) | Interfaz |
| Docker / Docker Compose | Empaquetado y entornos |
| pytest, ruff, GitHub Actions | Pruebas, calidad, CI/CD |
| MLflow (opcional) | Seguimiento de experimentos |
 
---
 
## 7. Arquitectura del sistema
 
### 7.1 Capas
 
| # | Capa | Responsabilidad | Se comunica con |
|---|---|---|---|
| 1 | **Presentación** 🔵 | Formularios, tablero, historial | Aplicación (HTTP/JSON) |
| 2 | **Aplicación** 🔵 | Endpoints, autenticación, orquestación, reglas de negocio (nivel de riesgo) | Presentación, Datos, ML, Almacenamiento |
| 3 | **Procesamiento de datos** 🟡/🔵 | Carga, validación de esquema y rangos, limpieza, ordenar columnas | Aplicación, ML |
| 4 | **Machine Learning** ✅/🔵 | Escalado, entrenamiento, `predict_proba`, evaluación | Datos, Aplicación |
| 5 | **Almacenamiento** 🔵 | Estudiantes, predicciones, versiones de modelo, auditoría; archivos del modelo | Aplicación |
| 6 | **Resultados** 🔵 | Probabilidad de deserción, nivel de riesgo, gráficos, reportes | Aplicación → Presentación |
 
Principio rector: **la capa de presentación nunca toca el modelo ni la base de datos**; todo pasa por la capa de aplicación.
 
### 7.2 Justificación de decisiones
 
- **Un solo `Pipeline` de scikit-learn** (`StandardScaler` + `LogisticRegression`) serializado junto: evita que en producción se escale distinto que en entrenamiento.
- **Backend separado del entrenamiento**: el entrenamiento es un proceso offline; la API solo carga el artefacto.
- **Base de datos relacional**: los datos son estructurados y se necesita historial, relaciones y auditoría.
- **Nivel de riesgo fuera del modelo**: los umbrales viven en configuración, así se ajustan sin reentrenar.
---
 
## 8. Diagramas Mermaid
 
### 8.1 Arquitectura general
 
```mermaid
flowchart TB
    subgraph P["1. Presentación"]
        UI["Interfaz web"]
    end
    subgraph A["2. Aplicación (API)"]
        API["Endpoints REST"]
        AUTH["Autenticación y roles"]
        RISK["Clasificador de nivel de riesgo"]
    end
    subgraph D["3. Procesamiento de datos"]
        VAL["Validación"]
        CLEAN["Limpieza y orden de columnas"]
    end
    subgraph M["4. Machine Learning"]
        PIPE["Pipeline: StandardScaler + LogisticRegression"]
    end
    subgraph S["5. Almacenamiento"]
        DB[("Base de datos")]
        ART[("Artefactos del modelo")]
    end
    subgraph R["6. Resultados"]
        DASH["Probabilidades, riesgo, gráficos, reportes"]
    end
    UI --> API
    API --> AUTH
    API --> VAL --> CLEAN --> PIPE
    ART --> PIPE
    PIPE --> RISK
    RISK --> DB
    DB --> DASH
    RISK --> DASH
    DASH --> UI
```
 
**Explicación.** El usuario interactúa con la UI; toda petición pasa por la API, que autentica, valida y limpia los datos antes de invocar el pipeline. El pipeline se carga desde artefactos almacenados. La probabilidad de deserción se convierte en nivel de riesgo, se guarda y se devuelve a la UI.
 
### 8.2 Componentes
 
```mermaid
flowchart LR
    FE["Frontend"] -->|"HTTP/JSON"| GW["Routers FastAPI"]
    GW --> SCH["Schemas Pydantic"]
    GW --> SVC["PredictionService"]
    GW --> STS["StatsService"]
    SVC --> PRE["Preprocessing"]
    SVC --> MR["ModelRegistry (carga joblib)"]
    SVC --> RC["RiskClassifier (umbrales en YAML)"]
    SVC --> REPO["Repositories (SQLAlchemy)"]
    STS --> REPO
    REPO --> DB[("PostgreSQL")]
    MR --> FS[("models/*.joblib + metadata.json")]
    TR["Módulo de entrenamiento (offline)"] --> FS
    TR --> DS[("data/")]
```
 
**Explicación.** `PredictionService` coordina todo el caso de uso. `ModelRegistry` carga el modelo una sola vez al iniciar. `RiskClassifier` lee umbrales de configuración. El módulo de entrenamiento es independiente de la API y solo comparte el artefacto.
 
### 8.3 Flujo del procesamiento de datos
 
```mermaid
flowchart TD
    A["Dataset CSV/Excel"] --> B["Carga con Pandas"]
    B --> C{"Esquema correcto? 7 columnas"}
    C -- No --> E1["Error: columnas faltantes"]
    C -- Si --> D["Exploración: tipos, nulos, duplicados, distribución"]
    D --> F["Limpieza: nulos, duplicados, tipos, rangos"]
    F --> G["Variable objetivo: deserta = 1 - termina_universidad"]
    G --> H["X = 6 predictoras, y = deserta"]
    H --> I["train_test_split 80/20, random_state=42"]
```
 
**Explicación.** Cada paso produce una salida verificable. Las reglas exactas de limpieza 🟡 deben tomarse del notebook.
 
### 8.4 Entrenamiento del modelo
 
```mermaid
flowchart TD
    A["X_train, y_train"] --> B["StandardScaler.fit(X_train)"]
    B --> C["transform(X_train)"]
    C --> D["LogisticRegression class_weight=balanced, max_iter=1000"]
    D --> E["fit"]
    E --> F["transform(X_test) con el MISMO scaler"]
    F --> G["Evaluación: accuracy, matriz de confusión, classification_report"]
    G --> H{"Cumple criterios?"}
    H -- Si --> I["Guardar pipeline + metadata (joblib, JSON)"]
    H -- No --> J["Ajustar: umbral, otros modelos, CV"]
```
 
**Explicación.** El scaler se ajusta **solo** con entrenamiento (evita fuga de datos). El artefacto guardado debe incluir versión, fecha, métricas y `classes_`.
 
### 8.5 Secuencia de una predicción
 
```mermaid
sequenceDiagram
    actor U as Usuario
    participant UI as Frontend
    participant API as Backend
    participant V as Validación
    participant M as Modelo
    participant DB as Base de datos
    U->>UI: Ingresa datos del estudiante
    UI->>UI: Validación en cliente
    UI->>API: POST /api/v1/predictions
    API->>V: Validar rangos y tipos
    V-->>API: Datos válidos
    API->>M: predict_proba(X) con columnas ordenadas
    M-->>API: probabilidades
    API->>API: p_deserta = proba[:, index de clase 1 en classes_]
    API->>API: Nivel de riesgo según umbrales
    API->>DB: Guardar predicción
    API-->>UI: JSON con probabilidad y riesgo
    UI-->>U: Resultado visual
```
 
### 8.6 Despliegue
 
```mermaid
flowchart LR
    subgraph Dev["Desarrollo"]
        D1["Notebook + entorno virtual"]
    end
    subgraph Test["Pruebas/Staging"]
        T1["Docker Compose: API + UI + DB de prueba"]
    end
    subgraph Prod["Producción"]
        LB["Proxy inverso HTTPS"] --> BE["Contenedor API"]
        LB --> FE["Contenedor/estático Frontend"]
        BE --> PG[("PostgreSQL gestionado")]
        BE --> VOL[("Volumen/almacén de modelos")]
    end
    Dev -->|"git push + CI"| Test -->|"aprobación"| Prod
```
 
### 8.7 Casos de uso
 
```mermaid
flowchart LR
    T(["Tutor / Bienestar"])
    AD(["Administrador"])
    DS(["Científico de datos"])
    subgraph SYS["Sistema"]
        U1["Registrar estudiante"]
        U2["Predecir riesgo"]
        U3["Consultar historial"]
        U4["Ver estadísticas y reportes"]
        U5["Gestionar usuarios y roles"]
        U6["Reentrenar y publicar modelo"]
        U7["Revisar auditoría"]
    end
    T --> U1 & U2 & U3 & U4
    AD --> U5 & U7 & U4
    DS --> U6
```
 
---
 
## 9. Estructura de carpetas 🔵
 
```text
desercion-estudiantil/
├── README.md                    # Este documento
├── pyproject.toml               # Dependencias y herramientas (ruff, pytest)
├── .env.example                 # Variables de entorno (sin secretos reales)
├── .gitignore                   # Excluye data/, models/, .env
├── configs/
│   ├── config.yaml              # Rutas, semilla (42), proporción de test (0.2)
│   └── risk_thresholds.yaml     # Umbrales de nivel de riesgo
├── data/
│   ├── raw/                     # Dataset original, solo lectura
│   ├── interim/                 # Datos intermedios
│   └── processed/               # train/test ya preparados
├── notebooks/
│   └── desercion_estudiantil.ipynb   # ✅ Notebook original (referencia)
├── src/desercion/               # Paquete reutilizable (entrenamiento y predicción)
│   ├── data/
│   │   ├── loader.py            # Carga del dataset
│   │   ├── validation.py        # Esquema, tipos y rangos
│   │   └── cleaning.py          # Nulos, duplicados, tipos
│   ├── features/
│   │   └── build.py             # Variable objetivo y selección de predictoras
│   ├── models/
│   │   ├── train.py             # Entrenamiento del Pipeline
│   │   ├── evaluate.py          # Métricas, matriz de confusión, CV
│   │   ├── registry.py          # Guardar/cargar artefactos + metadata
│   │   └── predict.py           # predict_proba con classes_ correcto
│   └── config.py                # Lectura de configuración
├── models/                      # Artefactos entrenados
│   ├── pipeline_v1.joblib       # StandardScaler + LogisticRegression
│   └── pipeline_v1.metadata.json # Versión, fecha, métricas, classes_, columnas
├── predictions/                 # Salidas de predicciones por lote (CSV)
├── reports/
│   ├── figures/                 # Gráficos (Matplotlib/Seaborn)
│   └── metrics/                 # Métricas por versión
├── backend/
│   └── app/
│       ├── main.py              # Arranque FastAPI, carga del modelo
│       ├── api/v1/              # students.py, predictions.py, stats.py, auth.py
│       ├── schemas/             # Modelos Pydantic de entrada/salida
│       ├── services/            # prediction_service.py, risk.py, stats_service.py
│       ├── db/                  # models.py, session.py, repositories.py
│       ├── core/                # security.py, logging.py, errors.py
│       └── migrations/          # Alembic
├── frontend/                    # SPA (React) o prototipo Streamlit
│   └── src/ (pages/, components/, services/api.ts)
├── tests/
│   ├── unit/                    # Validación, riesgo, features
│   ├── integration/             # API + DB de prueba
│   └── model/                   # Pruebas de regresión de métricas
├── docker/
│   ├── Dockerfile.backend
│   ├── Dockerfile.frontend
│   └── docker-compose.yml
├── docs/                        # Arquitectura, API (OpenAPI), decisiones
└── .github/workflows/ci.yml     # Lint + pruebas + build
```
 
---
 
## 10. Modelo de Machine Learning
 
Todas las etapas ✅ salvo indicación. Los detalles internos 🟡 deben confirmarse en el notebook.
 
| Etapa | Recibe | Proceso | Produce |
|---|---|---|---|
| 1. Carga | Archivo de datos | Lectura con Pandas | DataFrame con 7 columnas |
| 2. Exploración | DataFrame | Tipos, nulos, estadísticas, gráficos (Matplotlib/Seaborn) | Diagnóstico de calidad y distribución |
| 3. Limpieza 🟡 | DataFrame | Tratar nulos, duplicados, tipos | DataFrame limpio |
| 4. Selección de variables | DataFrame limpio | Elegir `edad, genero, horas_estudio, asistencia, cursos_desaprobados, trabaja` | `X` |
| 5. Variable objetivo | `termina_universidad` | `deserta = 1 - termina_universidad` | `y` (1 = deserta, 0 = termina) |
| 6. División | `X`, `y` | 80/20, `random_state=42` | `X_train, X_test, y_train, y_test` |
| 7. Normalización | `X_train`, `X_test` | `StandardScaler`: `fit` solo en train; `transform` en ambos | Matrices escaladas |
| 8. Entrenamiento | `X_train` escalado, `y_train` | `LogisticRegression(class_weight='balanced', max_iter=1000)` | Modelo ajustado |
| 9. Evaluación | `X_test`, `y_test` | Accuracy, matriz de confusión, `classification_report` | Métricas |
| 10. Predicción | Datos de un estudiante nuevo | Mismas columnas y orden, escalar con el scaler **ya ajustado**, `predict_proba` | Probabilidad y clase |
 
### Regla crítica sobre probabilidades
 
`predict_proba` devuelve columnas en el orden de `modelo.classes_`. Para obtener la probabilidad de **deserción**:
 
```python
idx = list(modelo.classes_).index(1)        # 1 = deserta
p_deserta = modelo.predict_proba(X_nuevo)[:, idx]
```
 
No asumir `[:, 1]` ni confundirla con la probabilidad de terminar (`1 - p_deserta`). Si el notebook entrenó con otra codificación, ajustar aquí y en toda la API.
 
---
 
## 11. Flujo de funcionamiento paso a paso 🔵
 
1. **Ingreso**: el usuario completa el formulario.
2. **Validación en cliente**: campos obligatorios, rangos.
3. **Envío**: `POST /api/v1/predictions` con JSON (HTTPS + token).
4. **Validación en servidor** (Pydantic): tipos, rangos, categorías permitidas; se rechazan valores fuera del rango observado en entrenamiento, o se marcan con advertencia.
5. **Procesamiento**: ordenar columnas exactamente como en entrenamiento.
6. **Carga del modelo**: ya cargado en memoria al iniciar; se valida la versión.
7. **Predicción**: el pipeline escala y ejecuta `predict_proba`.
8. **Probabilidad de deserción**: usando `classes_` (sección 10).
9. **Nivel de riesgo**: comparación con umbrales de `risk_thresholds.yaml`.
10. **Resultado**: respuesta JSON y visualización.
11. **Almacenamiento**: se guarda la predicción con versión del modelo y usuario.
Niveles de riesgo **propuestos** (valores de ejemplo, deben calibrarse con datos reales y criterio institucional):
 
| Nivel | p_deserta |
|---|---|
| Bajo | < 0.40 |
| Medio | 0.40 – 0.70 |
| Alto | ≥ 0.70 |
 
---
 
## 12. Diseño de la interfaz 🔵
 
| Pantalla | Componentes |
|---|---|
| **Inicio / Tablero** | Tarjetas (total evaluados, % riesgo alto), gráfico de distribución de riesgo, tendencia temporal |
| **Registrar estudiante** | Código seudonimizado, formulario con las 6 variables, botón "Predecir" |
| **Resultado** | Medidor de probabilidad de deserción, etiqueta de nivel de riesgo, factores principales (coeficientes), advertencias de datos atípicos |
| **Historial** | Tabla filtrable (fecha, riesgo, versión del modelo), exportar CSV |
| **Estadísticas y reportes** | Gráficos por asistencia, horas de estudio, trabajo; reporte descargable |
| **Administración** | Usuarios, roles, auditoría |
 
Criterios: diseño responsivo, accesibilidad (contraste, etiquetas), y mostrar siempre la **probabilidad como apoyo a la decisión**, no como veredicto.
 
---
 
## 13. API 🔵
 
Base: `/api/v1`. Autenticación: Bearer token (JWT).
 
| Método | Ruta | Descripción |
|---|---|---|
| POST | `/students` | Registrar estudiante |
| POST | `/predictions` | Realizar predicción |
| GET | `/predictions/{id}` | Consultar un resultado |
| GET | `/students/{id}/predictions` | Historial de un estudiante |
| GET | `/stats/summary` | Estadísticas generales |
| GET | `/health` | Estado del servicio y versión del modelo |
 
### Esquema de entrada (rangos de ejemplo 🟡 — confirmar con el dataset)
 
```python
from typing import Literal
from pydantic import BaseModel, Field
 
class PredictionRequest(BaseModel):
    student_code: str = Field(min_length=3, max_length=40)
    edad: int = Field(ge=16, le=70)                  # 🟡 ajustar al rango observado
    genero: Literal["F", "M"]                        # 🟡 codificación real a verificar
    horas_estudio: float = Field(ge=0, le=80)        # 🟡
    asistencia: float = Field(ge=0, le=100)          # 🟡 ¿porcentaje o proporción?
    cursos_desaprobados: int = Field(ge=0, le=30)    # 🟡
    trabaja: Literal[0, 1]                           # 🟡
```
 
### Ejemplo de solicitud
 
```json
{
  "student_code": "A-00123",
  "edad": 21,
  "genero": "F",
  "horas_estudio": 12.5,
  "asistencia": 82,
  "cursos_desaprobados": 2,
  "trabaja": 1
}
```
 
### Ejemplo de respuesta
 
```json
{
  "prediction_id": 981,
  "student_code": "A-00123",
  "probabilidad_desercion": 0.73,
  "nivel_riesgo": "alto",
  "clase_predicha": 1,
  "modelo_version": "v1",
  "advertencias": [],
  "fecha": "2026-11-02T00:40:00-05:00"
}
```
 
### Servicio de predicción (ejemplo)
 
```python
import joblib, pandas as pd
 
COLUMNAS = ["edad", "genero", "horas_estudio", "asistencia",
            "cursos_desaprobados", "trabaja"]
 
class PredictionService:
    def __init__(self, path: str, thresholds: dict):
        self.pipeline = joblib.load(path)            # StandardScaler + LogisticRegression
        self.idx = list(self.pipeline.classes_).index(1)   # clase "deserta"
        self.thresholds = thresholds
 
    def predict(self, data: dict) -> dict:
        X = pd.DataFrame([data])[COLUMNAS]           # orden fijo
        p = float(self.pipeline.predict_proba(X)[0, self.idx])
        nivel = ("alto" if p >= self.thresholds["alto"]
                 else "medio" if p >= self.thresholds["medio"] else "bajo")
        return {"probabilidad_desercion": round(p, 4), "nivel_riesgo": nivel}
```
 
> Nota: `genero` debe llegar numérico al scaler. Si el notebook lo codifica manualmente 🟡, replicar esa codificación dentro del pipeline (por ejemplo con `ColumnTransformer`) 🔵.
 
Errores: `422` validación, `401/403` acceso, `404` no encontrado, `503` modelo no disponible. Formato de error uniforme `{ "error": {"codigo": "...", "mensaje": "..."} }` sin exponer detalles internos.
 
---
 
## 14. Base de datos 🔵
 
**¿Se necesita?** Para el notebook actual, no. Para un sistema con historial, estadísticas y control de acceso, **sí se recomienda**. Hoy **no existe** una base de datos.
 
```mermaid
erDiagram
    USER ||--o{ PREDICTION : "realiza"
    USER ||--o{ AUDIT_LOG : "genera"
    STUDENT ||--o{ PREDICTION : "tiene"
    MODEL_VERSION ||--o{ PREDICTION : "produce"
    USER {
        int id PK
        string email UK
        string password_hash
        string rol
        bool activo
    }
    STUDENT {
        int id PK
        string student_code UK
        int edad
        string genero
        float horas_estudio
        float asistencia
        int cursos_desaprobados
        bool trabaja
        datetime created_at
    }
    MODEL_VERSION {
        int id PK
        string version UK
        datetime trained_at
        float accuracy
        json metricas
        json classes
        bool activa
    }
    PREDICTION {
        int id PK
        int student_id FK
        int model_version_id FK
        int user_id FK
        float prob_desercion
        string nivel_riesgo
        json input_snapshot
        datetime created_at
    }
    AUDIT_LOG {
        int id PK
        int user_id FK
        string accion
        string recurso
        string ip
        datetime created_at
    }
```
 
`input_snapshot` guarda los valores usados en cada predicción, para poder reproducir el resultado aunque los datos del estudiante cambien.
 
---
 
## 15. Seguridad y calidad 🔵
 
| Tema | Recomendación |
|---|---|
| Validación | Pydantic en servidor; rangos basados en el entrenamiento; advertencia o rechazo de valores atípicos |
| Confidencialidad | Usar códigos seudonimizados en lugar de nombres/documentos; minimizar datos; cifrado en tránsito (HTTPS) y en reposo; cumplir la normativa local de protección de datos personales |
| Control de acceso | Roles (tutor, administrador, científico de datos); mínimo privilegio; cada tutor ve solo su población |
| Errores | Manejadores globales; mensajes genéricos al cliente, detalle en logs |
| Registro | Auditoría de quién consultó/predijo qué estudiante, sin guardar datos sensibles en logs |
| API | JWT con expiración, rate limiting, CORS restringido, secretos en variables de entorno |
| Pruebas unitarias | Validación, clasificación de riesgo, índice de `classes_` |
| Integración | API + base de datos de prueba, flujo completo de predicción |
| Mantenimiento del modelo | Versionado, monitoreo de deriva de datos, reentrenamiento periódico, pruebas de regresión de métricas |
| Ética | El modelo apoya decisiones de acompañamiento; no debe usarse para sancionar ni excluir estudiantes. Revisar sesgo por `genero` |
 
---
 
## 16. Resultados, evaluación y mejoras
 
### 16.1 Resultados reportados en el prompt ✅
 
- **Exactitud**: 91,53 %
- **Matriz de confusión**: `[[71, 4], [80, 837]]`
Verificación aritmética: 71 + 4 + 80 + 837 = **992** estudiantes de prueba; (71 + 837) / 992 = **91,53 %** ✔. Con partición 80/20 implica un dataset de ≈ 4.960 registros (derivado).
 
### 16.2 Lectura de la matriz
 
Con la convención de scikit-learn (filas = real, columnas = predicho, orden de `classes_` = [0, 1]):
 
| | Pred. 0 (termina) | Pred. 1 (deserta) |
|---|---|---|
| **Real 0** | 71 | 4 |
| **Real 1** | 80 | 837 |
 
Métricas **derivadas de la matriz** (no del reporte original, que no se proporcionó):
 
| Clase | Soporte | Precision | Recall | F1 |
|---|---|---|---|---|
| 0 (termina) | 75 | 71/151 = 47,0 % | 71/75 = 94,7 % | ≈ 62,8 % |
| 1 (deserta) | 917 | 837/841 = 99,5 % | 837/917 = 91,3 % | ≈ 95,2 % |
 
> 🟡 **Reemplazar por los valores del `classification_report` real** y confirmar la orientación de la matriz.
 
### 16.3 Hallazgos a revisar
 
1. **Posible inconsistencia de interpretación.** Si 1 = deserta, la matriz implica que ~92 % de los estudiantes de prueba (917/992) desertan. Es una tasa de deserción muy alta; debe verificarse si el dataset realmente la refleja o si la codificación de `termina_universidad` está invertida. **Esto cambia por completo cómo se interpreta la probabilidad.**
2. **Desbalance**: el accuracy es engañoso. Una clase constante "deserta" ya lograría ≈ 92,4 % (917/992). El modelo apenas supera esa línea base (91,53 % es incluso menor).
3. **Baja precision en la clase 0**: de 151 estudiantes predichos como "termina", 80 en realidad desertan. Si el objetivo es detectar **desertores**, esos 80 son falsos negativos de riesgo: casos que el sistema no alertaría.
4. **Sensibilidad según el objetivo**: si la clase de interés (alerta) es deserción, el recall de clase 1 (91,3 %) significa que ~8,7 % de los desertores no se detectan.
### 16.4 Limitaciones
 
- Solo 6 predictoras; sin factores económicos, académicos históricos o institucionales.
- Un solo modelo lineal y una sola partición (sin validación cruzada).
- Umbral por defecto 0,5 sin justificar.
- Riesgo de sesgo por `genero` y `trabaja`.
- Sin monitoreo ni calibración de probabilidades.
### 16.5 Mejoras propuestas 🔵
 
1. **Línea base** (`DummyClassifier`) para comparar contra el accuracy.
2. **Métricas por clase** como criterio principal (recall de la clase de alerta, PR-AUC, F1).
3. **Validación cruzada estratificada** (k = 5 o 10) y reporte de media ± desviación.
4. **Ajuste del umbral** con curva precision-recall según el costo de no detectar un desertor.
5. **Comparar algoritmos**: Random Forest, Gradient Boosting, SVM, árbol de decisión.
6. **Calibración** de probabilidades (`CalibratedClassifierCV`) para que los niveles de riesgo sean confiables.
7. **Validación de datos nuevos**: rechazar o advertir valores fuera del rango observado en entrenamiento; no extrapolar sin justificación documentada.
8. **Pipeline único** (`Pipeline` + `ColumnTransformer`) para evitar discrepancias de preprocesamiento.
9. **Análisis de equidad** por género y situación laboral.
---
 
## 17. Instalación y configuración 🔵
 
```bash
git clone <url-del-repositorio>
cd desercion-estudiantil
python -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\activate
pip install -e ".[dev]"
cp .env.example .env                 # completar valores
```
 
`.env.example` (ejemplo):
 
```env
DATABASE_URL=postgresql://user:password@localhost:5432/desercion
MODEL_PATH=models/pipeline_v1.joblib
JWT_SECRET=cambiar_en_produccion
ENV=development
```
 
Dependencias base: `pandas`, `numpy`, `scikit-learn`, `joblib`, `matplotlib`, `seaborn` ✅ (ya usadas); `fastapi`, `uvicorn`, `pydantic`, `sqlalchemy`, `alembic`, `pytest` 🔵.
 
### Entornos 🔵
 
| Entorno | Descripción |
|---|---|
| Desarrollo | Máquina local, `venv`, SQLite o PostgreSQL local, notebook |
| Pruebas/Staging | Docker Compose, datos sintéticos o anonimizados, CI automático |
| Producción | Contenedores tras proxy HTTPS, PostgreSQL gestionado, secretos en gestor seguro, copias de seguridad |
 
## 18. Ejecución 🔵
 
```bash
# 1. Entrenar y guardar el modelo
python -m desercion.models.train --config configs/config.yaml
 
# 2. Iniciar el backend
uvicorn backend.app.main:app --reload
 
# 3. Iniciar el frontend
cd frontend && npm install && npm run dev
 
# 4. Todo junto con contenedores
docker compose -f docker/docker-compose.yml up --build
```
 
Documentación interactiva de la API (FastAPI): `http://localhost:8000/docs`.
 
La carga del modelo ocurre una vez al arrancar la API (`ModelRegistry`), verificando que `classes_` y la lista de columnas coincidan con `metadata.json`.
 
## 19. Ejemplos de uso 🔵
 
```bash
curl -X POST http://localhost:8000/api/v1/predictions \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"student_code":"A-00123","edad":21,"genero":"F","horas_estudio":12.5,
       "asistencia":82,"cursos_desaprobados":2,"trabaja":1}'
```
 
---
 
## 20. Estado de implementación
 
| Componente | Estado |
|---|---|
| Carga, exploración, limpieza del dataset | ✅ (detalles 🟡) |
| Variable objetivo, división 80/20, `StandardScaler` | ✅ |
| Regresión logística balanceada y evaluación | ✅ |
| Pipeline serializado (`joblib`) y metadata | 🔵 |
| Módulo de predicción con `classes_` | 🔵 |
| Backend/API, validaciones, autenticación | 🔵 |
| Base de datos y migraciones | 🔵 |
| Frontend | 🔵 |
| Pruebas, CI/CD, Docker | 🔵 |
| Monitoreo, calibración, validación cruzada | 🔵 |
 
## 21. Hoja de ruta sugerida
 
1. **Fase 0**: verificar notebook (codificación de la variable objetivo, rangos, reporte por clase).
2. **Fase 1**: refactorizar a `src/desercion/` + pipeline serializado + pruebas.
3. **Fase 2**: backend con `/predictions` y validaciones.
4. **Fase 3**: base de datos, historial, autenticación y auditoría.
5. **Fase 4**: frontend y reportes.
6. **Fase 5**: mejoras del modelo (CV, umbrales, comparación, calibración), despliegue y monitoreo.
---
 
## 22. Pendientes de verificación (requieren el notebook)
 
- [ ] Confirmar que `1 = deserta` y la orientación real de la matriz de confusión.
- [ ] Obtener el `classification_report` completo.
- [ ] Rangos reales y unidades de `edad`, `horas_estudio`, `asistencia`, `cursos_desaprobados`.
- [ ] Codificación de `genero` y `trabaja`.
- [ ] Reglas de limpieza aplicadas.
- [ ] Orden exacto de las columnas pasadas al `StandardScaler`.
