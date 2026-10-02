# DiplomadoProject

Project diplomado

# Red de apoyo académico con emparejamiento por IA local

## 1. ¿Qué es la idea de la red de apoyo?

Es una plataforma similar a Brainly o a un grupo de WhatsApp de la universidad, con una diferencia clave: en lugar de publicar la pregunta y esperar a que alguien la vea, el sistema **lee la petición, entiende de qué trata y notifica a las personas que más saben de ese tema**.

### Ejemplo

Un estudiante escribe:

> "No entiendo cómo hacer joins en SQL con tablas que tienen relaciones muchos a muchos".

El sistema detecta que es:

- **Área:** Bases de datos
- **Nivel:** Intermedio
- **Tema:** Joins

Luego busca entre los perfiles a quienes declararon saber SQL o ya ayudaron con temas parecidos, y envía la solicitud a los **3 mejores candidatos**, junto con una frase que explica por qué encajan.

Técnicamente es **búsqueda semántica con embeddings**: cada perfil y cada petición se convierten en un vector y se buscan los más cercanos.

No depende de que las palabras coincidan exactamente; por ejemplo, _"joins"_ y _"consultas entre tablas"_ pueden quedar cerca en el espacio vectorial.

---

## 2. ¿Por qué esta idea y no la plataforma de despliegue?

La alternativa inicial —desplegar proyectos en Kubernetes desde la web— es principalmente **Platform Engineering**, pero presenta dos problemas para este equipo:

1. **La IA quedaría forzada.**  
   El diplomado exige un servicio de IA local con RAG y un componente multimodal (sección 5.7).

2. **Es pesada.**  
   Construir imágenes de repositorios de usuarios dentro del clúster mediante Kaniko o Buildpacks en equipos de 8 GB es difícil.

Con **Minga**, la IA es el núcleo del producto y los requisitos de Kubernetes, GitOps y DevSecOps se cumplen igualmente, porque se evalúa cómo se despliega y opera la aplicación.

Además, responde a una necesidad del Cauca: el nombre hace referencia a la **minga**, entendida como trabajo colectivo y comunitario.

---

## 3. Alcance del producto (MVP)

### Incluido

- Registro y perfil con habilidades en texto libre más etiquetas:
  - Materias
  - Nivel
  - Temas
- Publicar una solicitud de ayuda por:
  - Texto
  - Nota de voz
- Clasificación de la solicitud mediante IA:
  - Área
  - Temas
  - Nivel
- Generación del embedding de la solicitud.
- Emparejamiento con el **Top 3 de ayudantes**.
- Explicación corta generada por el LLM sobre por qué cada ayudante encaja.
- El ayudante acepta o rechaza la solicitud.
- Al aceptar, se comparte el contacto.
- Calificación simple posterior para mejorar el ranking futuro.

### Fuera de alcance

Para mantener el MVP pequeño y reconstruible, quedan fuera:

- Chat en tiempo real.
- Aplicación móvil.
- Pagos.
- Videollamadas.

> Recortar el alcance es parte de la ingeniería: la rúbrica valora una solución pequeña, reconstruible y bien observada.

### Componente multimodal

Se utilizará **whisper.cpp** para transcribir notas de voz.

Se propone utilizar el modelo **base o small**, viable en CPU.

Esto permite cumplir el requisito de procesamiento de audio sin GPU. Generar imágenes o videos no resulta viable con un equipo de 8 GB de RAM.

---

# 4. Arquitectura y distribución del hardware

## Distribución

### Mac M1 — Nodo de IA

El Mac M1 funcionará como nodo dedicado para IA.

Apple Silicon permite ejecutar modelos pequeños aprovechando Metal. Allí correrán:

- Ollama
- whisper.cpp

Estos servicios serán expuestos mediante la red local.

Esta arquitectura replica la **"Ruta C – nodo compartido"** planteada en el diplomado.

### Equipo de 8 GB — Clúster

El equipo de 8 GB ejecutará el clúster **k3d**, encargado de:

- Aplicación
- Base de datos
- Redis
- Argo CD
- Observabilidad

### Degradación elegante

Si el nodo de IA está apagado:

1. Las solicitudes quedan en cola.
2. El sistema continúa funcionando.
3. Se utiliza un emparejamiento básico basado en etiquetas.

Esto permite mantener disponible la funcionalidad principal incluso cuando el servicio de IA no está disponible.

## Diagrama de arquitectura

```text
                    ┌─────────────────────┐
                    │    Vue 3 Frontend   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │     API NestJS      │
                    └──────────┬──────────┘
                               │
                ┌──────────────┴──────────────┐
                │                             │
                ▼                             ▼
     ┌─────────────────────┐       ┌─────────────────────┐
     │ PostgreSQL + pgvector│       │ Redis + BullMQ      │
     └─────────────────────┘       │ Cola de trabajos    │
                                   └──────────┬──────────┘
                                              │
                                              ▼
                                   ┌─────────────────────┐
                                   │    Worker de IA     │
                                   └──────────┬──────────┘
                                              │
                                             LAN
                                              │
                                              ▼
                                   ┌─────────────────────┐
                                   │       Mac M1         │
                                   │                     │
                                   │  Ollama             │
                                   │  whisper.cpp        │
                                   └─────────────────────┘
```

---

# 5. Stack tecnológico

La propuesta utiliza herramientas gratuitas y, en lo posible, de código abierto.

| Capa           | Herramienta                       | Justificación                                            |
| -------------- | --------------------------------- | -------------------------------------------------------- |
| Frontend       | Vue 3 + Vite                      | Tecnología dominada por el equipo                        |
| Backend        | NestJS + TypeScript               | Buen soporte para OpenTelemetry y colas                  |
| Base de datos  | PostgreSQL + pgvector             | Permite almacenar datos y vectores en una sola base      |
| Cola asíncrona | Redis + BullMQ                    | Permite manejar transcripción y emparejamiento como jobs |
| LLM            | Ollama + llama3.2:3b o qwen2.5:3b | Clasifica la petición y explica el emparejamiento        |
| Embeddings     | nomic-embed-text vía Ollama       | Modelo liviano                                           |
| Audio          | whisper.cpp                       | Corre en CPU y puede aprovechar Metal en la M1           |
| Kubernetes     | k3d                               | Distribución liviana de Kubernetes basada en k3s         |
| Empaquetado    | Helm o Kustomize                  | Permite manejar overlays de desarrollo/demo              |
| GitOps         | Argo CD                           | Interfaz visual útil para la demostración                |
| CI/CD          | GitHub Actions                    | Disponible gratuitamente en repositorios públicos        |
| Registro       | GitHub Container Registry         | Registro de contenedores integrado con GitHub            |
| DevSecOps      | Trivy + Syft + Cosign keyless     | Escaneo, SBOM y firma mediante OIDC de GitHub            |
| IaC            | OpenTofu                          | Permite declarar namespaces, releases y configuración    |
| Observabilidad | OpenTelemetry + Grafana Cloud     | Reduce consumo de RAM frente a Prometheus/Loki locales   |
| MLOps          | MLflow                            | Seguimiento de experimentos de emparejamiento            |

## Nota sobre la Mac M1

Se recomienda construir imágenes multi-arquitectura:

- `linux/amd64`
- `linux/arm64`

mediante `docker buildx` en GitHub Actions.

Esto permite utilizar la misma imagen tanto en Windows como en Mac.

En Mac se puede utilizar **Colima** en lugar de Docker Desktop para reducir el consumo de RAM.

## Opción adicional: Oracle Cloud

Como alternativa, Oracle Cloud Always Free ofrece máquinas ARM con recursos suficientes para experimentar con OpenTofu y crear infraestructura como:

- Máquinas virtuales.
- Redes.
- Recursos de infraestructura.

Esta opción requiere tarjeta para verificación y la disponibilidad de recursos puede variar.

Debe tratarse como un **bonus** y verificarse bajo las condiciones actuales del servicio.

---

# 6. Componente MLOps

El componente MLOps se puede desarrollar mediante un dataset sintético y experimentos controlados.

## 6.1 Dataset

Crear un dataset sintético de aproximadamente:

- **50 perfiles ficticios**
- **100 solicitudes**
- Respuesta correcta marcada para cada solicitud

Esto evita utilizar datos personales reales durante la demostración.

## 6.2 Experimentos

Comparar diferentes configuraciones utilizando MLflow:

### Modelos de embeddings

- `nomic-embed-text`
- `all-MiniLM`

### Clasificación

Comparar:

- Emparejamiento directo mediante embeddings.
- Clasificación previa mediante LLM + embeddings.

### Ranking

Experimentar con distintos pesos entre:

- Similitud semántica.
- Calificación del ayudante.
- Otros criterios definidos para el sistema.

## 6.3 Métrica

La métrica principal será:

### Precision@3

Mide si el ayudante correcto aparece dentro de los **3 primeros resultados**.

```text
Precision@3 =
Número de solicitudes donde el ayudante correcto aparece en el Top 3
--------------------------------------------------------------------
Número total de solicitudes evaluadas
```

## 6.4 Despliegue

Una vez comparados los experimentos:

1. Registrar las configuraciones en MLflow.
2. Identificar la configuración seleccionada según las métricas obtenidas.
3. Documentar sus parámetros.
4. Desplegar# Red de apoyo académico con emparejamiento por IA local

## 1. ¿Qué es la idea de la red de apoyo?

Es una plataforma similar a Brainly o a un grupo de WhatsApp de la universidad, con una diferencia clave: en lugar de publicar la pregunta y esperar a que alguien la vea, el sistema **lee la petición, entiende de qué trata y notifica a las personas que más saben de ese tema**.

### Ejemplo

Un estudiante escribe:

> "No entiendo cómo hacer joins en SQL con tablas que tienen relaciones muchos a muchos".

El sistema detecta que es:

- **Área:** Bases de datos
- **Nivel:** Intermedio
- **Tema:** Joins

Luego busca entre los perfiles a quienes declararon saber SQL o ya ayudaron con temas parecidos, y envía la solicitud a los **3 mejores candidatos**, junto con una frase que explica por qué encajan.

Técnicamente es **búsqueda semántica con embeddings**: cada perfil y cada petición se convierten en un vector y se buscan los más cercanos.

No depende de que las palabras coincidan exactamente; por ejemplo, _"joins"_ y _"consultas entre tablas"_ pueden quedar cerca en el espacio vectorial.

---

## 2. ¿Por qué esta idea y no la plataforma de despliegue?

La alternativa inicial —desplegar proyectos en Kubernetes desde la web— es principalmente **Platform Engineering**, pero presenta dos problemas para este equipo:

1. **La IA quedaría forzada.**  
   El diplomado exige un servicio de IA local con RAG y un componente multimodal (sección 5.7).

2. **Es pesada.**  
   Construir imágenes de repositorios de usuarios dentro del clúster mediante Kaniko o Buildpacks en equipos de 8 GB es difícil.

Con **Minga**, la IA es el núcleo del producto y los requisitos de Kubernetes, GitOps y DevSecOps se cumplen igualmente, porque se evalúa cómo se despliega y opera la aplicación.

Además, responde a una necesidad del Cauca: el nombre hace referencia a la **minga**, entendida como trabajo colectivo y comunitario.

---

## 3. Alcance del producto (MVP)

### Incluido

- Registro y perfil con habilidades en texto libre más etiquetas:
  - Materias
  - Nivel
  - Temas
- Publicar una solicitud de ayuda por:
  - Texto
  - Nota de voz
- Clasificación de la solicitud mediante IA:
  - Área
  - Temas
  - Nivel
- Generación del embedding de la solicitud.
- Emparejamiento con el **Top 3 de ayudantes**.
- Explicación corta generada por el LLM sobre por qué cada ayudante encaja.
- El ayudante acepta o rechaza la solicitud.
- Al aceptar, se comparte el contacto.
- Calificación simple posterior para mejorar el ranking futuro.

### Fuera de alcance

Para mantener el MVP pequeño y reconstruible, quedan fuera:

- Chat en tiempo real.
- Aplicación móvil.
- Pagos.
- Videollamadas.

> Recortar el alcance es parte de la ingeniería: la rúbrica valora una solución pequeña, reconstruible y bien observada.

### Componente multimodal

Se utilizará **whisper.cpp** para transcribir notas de voz.

Se propone utilizar el modelo **base o small**, viable en CPU.

Esto permite cumplir el requisito de procesamiento de audio sin GPU. Generar imágenes o videos no resulta viable con un equipo de 8 GB de RAM.

---

# 4. Arquitectura y distribución del hardware

## Distribución

### Mac M1 — Nodo de IA

El Mac M1 funcionará como nodo dedicado para IA.

Apple Silicon permite ejecutar modelos pequeños aprovechando Metal. Allí correrán:

- Ollama
- whisper.cpp

Estos servicios serán expuestos mediante la red local.

Esta arquitectura replica la **"Ruta C – nodo compartido"** planteada en el diplomado.

### Equipo de 8 GB — Clúster

El equipo de 8 GB ejecutará el clúster **k3d**, encargado de:

- Aplicación
- Base de datos
- Redis
- Argo CD
- Observabilidad

### Degradación elegante

Si el nodo de IA está apagado:

1. Las solicitudes quedan en cola.
2. El sistema continúa funcionando.
3. Se utiliza un emparejamiento básico basado en etiquetas.

Esto permite mantener disponible la funcionalidad principal incluso cuando el servicio de IA no está disponible.

## Diagrama de arquitectura

```text
                    ┌─────────────────────┐
                    │    Vue 3 Frontend   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │     API NestJS      │
                    └──────────┬──────────┘
                               │
                ┌──────────────┴──────────────┐
                │                             │
                ▼                             ▼
     ┌─────────────────────┐       ┌─────────────────────┐
     │ PostgreSQL + pgvector│       │ Redis + BullMQ      │
     └─────────────────────┘       │ Cola de trabajos    │
                                   └──────────┬──────────┘
                                              │
                                              ▼
                                   ┌─────────────────────┐
                                   │    Worker de IA     │
                                   └──────────┬──────────┘
                                              │
                                             LAN
                                              │
                                              ▼
                                   ┌─────────────────────┐
                                   │       Mac M1         │
                                   │                     │
                                   │  Ollama             │
                                   │  whisper.cpp        │
                                   └─────────────────────┘
```

---

# 5. Stack tecnológico

La propuesta utiliza herramientas gratuitas y, en lo posible, de código abierto.

| Capa           | Herramienta                       | Justificación                                            |
| -------------- | --------------------------------- | -------------------------------------------------------- |
| Frontend       | Vue 3 + Vite                      | Tecnología dominada por el equipo                        |
| Backend        | NestJS + TypeScript               | Buen soporte para OpenTelemetry y colas                  |
| Base de datos  | PostgreSQL + pgvector             | Permite almacenar datos y vectores en una sola base      |
| Cola asíncrona | Redis + BullMQ                    | Permite manejar transcripción y emparejamiento como jobs |
| LLM            | Ollama + llama3.2:3b o qwen2.5:3b | Clasifica la petición y explica el emparejamiento        |
| Embeddings     | nomic-embed-text vía Ollama       | Modelo liviano                                           |
| Audio          | whisper.cpp                       | Corre en CPU y puede aprovechar Metal en la M1           |
| Kubernetes     | k3d                               | Distribución liviana de Kubernetes basada en k3s         |
| Empaquetado    | Helm o Kustomize                  | Permite manejar overlays de desarrollo/demo              |
| GitOps         | Argo CD                           | Interfaz visual útil para la demostración                |
| CI/CD          | GitHub Actions                    | Disponible gratuitamente en repositorios públicos        |
| Registro       | GitHub Container Registry         | Registro de contenedores integrado con GitHub            |
| DevSecOps      | Trivy + Syft + Cosign keyless     | Escaneo, SBOM y firma mediante OIDC de GitHub            |
| IaC            | OpenTofu                          | Permite declarar namespaces, releases y configuración    |
| Observabilidad | OpenTelemetry + Grafana Cloud     | Reduce consumo de RAM frente a Prometheus/Loki locales   |
| MLOps          | MLflow                            | Seguimiento de experimentos de emparejamiento            |

## Nota sobre la Mac M1

Se recomienda construir imágenes multi-arquitectura:

- `linux/amd64`
- `linux/arm64`

mediante `docker buildx` en GitHub Actions.

Esto permite utilizar la misma imagen tanto en Windows como en Mac.

En Mac se puede utilizar **Colima** en lugar de Docker Desktop para reducir el consumo de RAM.

## Opción adicional: Oracle Cloud

Como alternativa, Oracle Cloud Always Free ofrece máquinas ARM con recursos suficientes para experimentar con OpenTofu y crear infraestructura como:

- Máquinas virtuales.
- Redes.
- Recursos de infraestructura.

Esta opción requiere tarjeta para verificación y la disponibilidad de recursos puede variar.

Debe tratarse como un **bonus** y verificarse bajo las condiciones actuales del servicio.

---

# 6. Componente MLOps

El componente MLOps se puede desarrollar mediante un dataset sintético y experimentos controlados.

## 6.1 Dataset

Crear un dataset sintético de aproximadamente:

- **50 perfiles ficticios**
- **100 solicitudes**
- Respuesta correcta marcada para cada solicitud

Esto evita utilizar datos personales reales durante la demostración.

## 6.2 Experimentos

Comparar diferentes configuraciones utilizando MLflow:

### Modelos de embeddings

- `nomic-embed-text`
- `all-MiniLM`

### Clasificación

Comparar:

- Emparejamiento directo mediante embeddings.
- Clasificación previa mediante LLM + embeddings.

### Ranking

Experimentar con distintos pesos entre:

- Similitud semántica.
- Calificación del ayudante.
- Otros criterios definidos para el sistema.

## 6.3 Métrica

La métrica principal será:

### Precision@3

Mide si el ayudante correcto aparece dentro de los **3 primeros resultados**.

```text
Precision@3 =
Número de solicitudes donde el ayudante correcto aparece en el Top 3
--------------------------------------------------------------------
Número total de solicitudes evaluadas
```

## 6.4 Despliegue

Una vez comparados los experimentos:

1. Registrar las configuraciones en MLflow.
2. Identificar la configuración seleccionada según las métricas obtenidas.
3. Documentar sus parámetros.
4. Desplegar
