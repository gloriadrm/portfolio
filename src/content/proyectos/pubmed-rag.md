---
title: "RAG biomédico sobre PubMed y PubMed Central"
problem: "¿Cómo responder preguntas biomédicas en lenguaje natural con evidencia verificable, cuando un LLM generalista puede generar respuestas plausibles pero sin garantía de estar fundamentadas en literatura científica real?"
description: "Sistema RAG que clasifica la intención de una consulta biomédica en lenguaje natural, recupera evidencia de un corpus de PubMed y PubMed Central con la estrategia adaptada a esa intención, y genera una respuesta fundamentada citando cada fuente hasta el fragmento exacto. Combina Qdrant con filtrado nativo por metadatos, embeddings bge-m3 ejecutados en local y una arquitectura multiproveedor de LLM (OpenAI, Gemini, Ollama), expuesto mediante una API en FastAPI con interfaz web y actualización incremental del corpus, todo contenerizado con Docker."
stack: ["FastAPI", "LangChain", "Qdrant", "bge-m3", "OpenAI / Gemini / Ollama", "Docker"]
competencias: ["Diseño de retrieval condicionado por la intención de la consulta, no por una estrategia de búsqueda fija", "Evaluación comparativa de bases vectoriales y modelos de embeddings con alternativas descartadas y argumentadas", "Migración de una integración externa deprecada a la infraestructura vigente sin interrumpir el servicio", "Arquitectura multiproveedor de LLM mediante patrón de fábrica, desacoplando la lógica de negocio del proveedor de inferencia", "Procesamiento asíncrono de trabajo CPU-bound dentro de una API síncrona, con persistencia de estado y recuperación ante fallos", "Diseño de prompts con grounding explícito para mitigar alucinaciones en un dominio de alta exigencia factual"]
order: 2
github: "https://github.com/gloriadrm/Pubmed-RAG-system"
memoria: "/proyectos/pubmed-rag/memoria.pdf"
queDescarte:
  - "Embeddings biomédicos especializados (pubmedbert-base-embeddings, S-PubMedBert-MedQuAD): se evaluaron por su especialización en literatura médica. Aunque ofrecían buen rendimiento en consultas puntuales, mostraban mayor variabilidad entre búsquedas y menor estabilidad en la recuperación global del corpus. Se priorizó la consistencia del retrieval frente al mejor resultado puntual."
  - "Weaviate, Pinecone, Milvus, FAISS y pgvector como base de datos vectorial: se descartaron por requerir un modelo SaaS, carecer de persistencia y API propias, exigir una infraestructura desproporcionada para el tamaño del corpus, o no permitir un filtrado por metadatos tan eficiente durante el recorrido del índice. Qdrant ofrecía el mejor equilibrio entre simplicidad operativa, filtrado nativo e integración con LangChain."
  - "Cola distribuida (Celery o RQ) para la actualización asíncrona del corpus: se descartó introducir infraestructura distribuida para un único proceso CPU-bound. Un ProcessPoolExecutor de un solo worker resolvía el problema con mucha menor complejidad operativa."
aprendizajes:
  - "En un sistema RAG, el rendimiento depende mucho más del pipeline de recuperación que del LLM utilizado: ningún modelo generativo puede fundamentar una respuesta en un fragmento que el retrieval no recuperó."
  - "La calidad de los metadatos condiciona el retrieval tanto como la calidad del embedding: sin ellos no es posible filtrar por fuente, citar el origen de una respuesta ni verificar la licencia de reutilización."
  - "En un dominio biomédico, la trazabilidad de la evidencia es un requisito del sistema, no una funcionalidad adicional."
mejorasFuturas:
  - "Evaluación cuantitativa del retrieval con métricas como Recall@K, Precision@K, MRR y nDCG sobre un conjunto de preguntas de referencia."
  - "Reranking de segunda etapa para mejorar la precisión de los documentos entregados al modelo generativo, especialmente en consultas transversales entre varios papers."
  - "Recuperación híbrida combinando búsqueda vectorial con búsqueda léxica (BM25)."
  - "Llevar la ingesta bajo demanda (/ingest) al mismo patrón asíncrono ya aplicado a la actualización del corpus."
  - "Evaluación automática de respuestas mediante conjuntos de preguntas de referencia y métricas específicas para sistemas RAG (RAGAS o similares), para medir de forma continua la calidad del sistema tras cada actualización del corpus."
---

## Dataset

El corpus combina dos fuentes de NCBI con responsabilidades distintas: PubMed aporta metadatos bibliográficos — título, resumen, autores, fecha — como registro base de cada artículo, y PubMed Central (PMC) enriquece ese registro con el texto completo estructurado por secciones cuando el artículo está en acceso abierto y su licencia permite reutilización. Solo se indexa texto completo bajo licencias que permiten obras derivadas (CC BY, CC BY-SA, CC BY-NC, CC BY-NC-SA, CC0); las licencias ND quedan excluidas porque una respuesta RAG combina fragmentos de múltiples documentos.

## Arquitectura

![Arquitectura general: interfaz web y API bifurcadas en flujo de consulta y gestión del corpus, convergiendo en Qdrant y el proveedor LLM](../../assets/proyectos/pubmed-rag/arquitectura-general.png)

*Interfaz web y API como entrada única, bifurcada en flujo de consulta y gestión del corpus, convergiendo en Qdrant y en el proveedor LLM activo.*

El flujo de consulta clasifica primero la pregunta del usuario — resumen temático, artículo concreto, comparación transversal o fuera de dominio — y esa clasificación decide tanto el filtro sobre Qdrant como la técnica de recuperación: abstracts de PubMed para una visión general, texto completo de PubMed Central con MMR (Maximum Marginal Relevance) para maximizar diversidad entre papers, o resolución directa del artículo cuando la pregunta apunta a uno concreto. El contexto recuperado se numera y se pasa a un LLM con un prompt que impone grounding explícito: ninguna afirmación puede aparecer sin una fuente numerada que la respalde.

## Casos de uso

El sistema adapta automáticamente la estrategia de recuperación al tipo de consulta detectado — cada pestaña corresponde a un modo de funcionamiento distinto.

<div class="proyecto-tabs tabs tabs-lift">

<input type="radio" name="casos-de-uso" class="tab" aria-label="Resumen temático" checked="checked" />

<div class="tab-content border-base-300 bg-base-100 p-6">

Para preguntas generales, recupera los abstracts más relevantes de PubMed y construye una síntesis respaldada por varias publicaciones.

![Interfaz de consulta: resumen temático con clasificación, retrieval y fuentes citadas](../../assets/proyectos/pubmed-rag/consulta-resumen-tematico.png)

</div>

<input type="radio" name="casos-de-uso" class="tab" aria-label="Artículo concreto" />

<div class="tab-content border-base-300 bg-base-100 p-6">

Cuando la consulta identifica un artículo mediante su título, autor o PMCID, el sistema restringe la búsqueda al texto completo de ese trabajo en PubMed Central.

![Pregunta sobre un artículo concreto y respuesta generada a partir de su texto completo](../../assets/proyectos/pubmed-rag/consulta-articulo-1.png)

![Fuentes utilizadas: fragmentos del mismo artículo citados por sección](../../assets/proyectos/pubmed-rag/consulta-articulo-2.png)

</div>

<input type="radio" name="casos-de-uso" class="tab" aria-label="Consulta transversal" />

<div class="tab-content border-base-300 bg-base-100 p-6">

Para preguntas que requieren combinar evidencia de distintos trabajos, el sistema busca sobre los textos completos disponibles y utiliza MMR para aumentar la diversidad de los fragmentos recuperados.

![Pregunta transversal y mecanismos identificados citando varias fuentes por afirmación](../../assets/proyectos/pubmed-rag/consulta-transversal-1.png)

![Fuentes utilizadas: seis artículos distintos, no fragmentos repetidos de uno solo](../../assets/proyectos/pubmed-rag/consulta-transversal-2.png)

</div>

<input type="radio" name="casos-de-uso" class="tab" aria-label="Fuera de dominio" />

<div class="tab-content border-base-300 bg-base-100 p-6">

Las preguntas no relacionadas con el ámbito biomédico se detectan antes del retrieval, evitando generar respuestas aparentemente válidas sin evidencia en el corpus.

![Consulta fuera de dominio: el sistema responde sin ejecutar retrieval](../../assets/proyectos/pubmed-rag/consulta-fuera-dominio.png)

</div>

</div>

## Trazabilidad hasta el fragmento original

La respuesta no se limita a enumerar las publicaciones recuperadas. La interfaz permite desplegar los fragmentos exactos utilizados como contexto por el LLM, manteniendo la trazabilidad desde cada cita hasta la evidencia original.

![Fuentes de una consulta transversal con fragmentos expandidos, mostrando el texto exacto usado como evidencia](../../assets/proyectos/pubmed-rag/trazabilidad-fragmentos.png)

## Mantenimiento del corpus

A diferencia de la mayoría de demostraciones de sistemas RAG, el proyecto no se limita a consultar un corpus estático: incorpora un proceso de actualización incremental que sincroniza la colección con los cambios publicados por NCBI — artículos nuevos, revisiones, retractaciones y cambios de licencia — sin interrumpir el servicio de consultas mientras se ejecuta.

![Resultado de una actualización incremental del corpus: artículos revisados, reindexados y sin errores](../../assets/proyectos/pubmed-rag/actualizacion-corpus.png)

*El proceso reporta qué cambió en cada ejecución — no solo si terminó, sino qué se revisó, qué se reindexó y si hubo errores.*

## Decisiones de diseño

Cada decisión responde a una pregunta que cualquier sistema RAG tiene que resolver, en el orden en que atraviesan el pipeline.

<div class="decision-card">

<div class="decision-card__title"><span class="decision-card__number">①</span> ¿Cómo decidir qué buscar?</div>

<div class="decision-card__step decision-card__step--problema">
<div class="decision-card__step-label">Problema</div>

Aplicar siempre la misma búsqueda semántica sobre toda la colección y filtrar los resultados a posteriori trata igual a una pregunta que necesita seis abstracts diversos y a una que necesita los diez fragmentos de un único artículo.

</div>

<div class="decision-card__step decision-card__step--decision">
<div class="decision-card__step-label">Decisión</div>

Un LLM clasifica la intención de la consulta antes de tocar la base vectorial, y esa clasificación fija el número de documentos, el filtro y la técnica de búsqueda desde el primer paso.

</div>

<div class="decision-card__step decision-card__step--resultado">
<div class="decision-card__step-label">Resultado</div>

Cada tipo de pregunta recibe la estrategia de retrieval que necesita, en lugar de un resultado genérico corregido después de una recuperación amplia.

</div>

</div>

<div class="decision-card">

<div class="decision-card__title"><span class="decision-card__number">②</span> ¿Cómo recuperar la evidencia?</div>

<div class="decision-card__step decision-card__step--problema">
<div class="decision-card__step-label">Problema</div>

Un único método de recuperación no sirve igual para una visión general de un tema, el detalle de un artículo concreto o una comparación entre varios trabajos.

</div>

<div class="decision-card__step decision-card__step--decision">
<div class="decision-card__step-label">Decisión</div>

El retrieval no sigue una única estrategia: según el tipo de consulta, el sistema recupera abstracts, restringe la búsqueda al texto completo de un artículo concreto o combina evidencia de varios documentos mediante MMR para maximizar su diversidad.

</div>

<div class="decision-card__step decision-card__step--resultado">
<div class="decision-card__step-label">Resultado</div>

La técnica de recuperación se ajusta a lo que necesita cada pregunta, no al revés — cada estrategia entrega exactamente el tipo de evidencia que ese caso requiere.

</div>

</div>

<div class="decision-card">

<div class="decision-card__title"><span class="decision-card__number">③</span> ¿Cómo representar los documentos?</div>

<div class="decision-card__step decision-card__step--problema">
<div class="decision-card__step-label">Problema</div>

Los modelos de embeddings especializados en biomedicina parecían la opción más lógica, pero era necesario validar si realmente mejoraban la recuperación sobre este corpus.

</div>

<div class="decision-card__step decision-card__step--decision">
<div class="decision-card__step-label">Decisión</div>

Se comparó bge-m3 con varios modelos biomédicos mediante un benchmark sobre el corpus del proyecto. Aunque algunos modelos especializados obtenían mejores resultados en casos concretos, bge-m3 ofreció un comportamiento más estable y consistente entre consultas.

</div>

<div class="decision-card__step decision-card__step--resultado">
<div class="decision-card__step-label">Resultado</div>

Se priorizó la consistencia global del retrieval frente al mejor resultado puntual — cualquier estrategia de recuperación opera sobre un espacio semántico estable.

<details>
<summary>Ver benchmark comparativo de embeddings</summary>

![Score de similitud del primer resultado recuperado, por modelo de embeddings y consulta de prueba](../../assets/proyectos/pubmed-rag/comparacion-embeddings-score.png)

*bge-m3 mantiene un score más estable entre consultas; los modelos biomédicos especializados alcanzan picos más altos pero con mayor dispersión — ver "Qué descarté" para el detalle completo.*

</details>

</div>

</div>

<div class="decision-card">

<div class="decision-card__title"><span class="decision-card__number">④</span> ¿Cómo generar respuestas fiables?</div>

<div class="decision-card__step decision-card__step--problema">
<div class="decision-card__step-label">Problema</div>

Un modelo de lenguaje puede completar una respuesta con conocimiento propio aunque se le entregue contexto recuperado, mezclando evidencia verificada con contenido no verificable.

</div>

<div class="decision-card__step decision-card__step--decision">
<div class="decision-card__step-label">Decisión</div>

El modelo nunca responde directamente: toda respuesta se genera exclusivamente a partir del contexto recuperado, citando cada afirmación por su fuente y declarando cuándo la evidencia disponible es insuficiente.

</div>

<div class="decision-card__step decision-card__step--resultado">
<div class="decision-card__step-label">Resultado</div>

La trazabilidad llega hasta el fragmento exacto utilizado como evidencia, no solo hasta el título del documento citado.

</div>

</div>

<div class="decision-card">

<div class="decision-card__title"><span class="decision-card__number">⑤</span> ¿Cómo desacoplar el LLM?</div>

<div class="decision-card__step decision-card__step--problema">
<div class="decision-card__step-label">Problema</div>

Depender de un único proveedor de LLM expone el sistema a límites de cuota, cambios de precio o interrupciones del servicio sin ninguna alternativa disponible.

</div>

<div class="decision-card__step decision-card__step--decision">
<div class="decision-card__step-label">Decisión</div>

Se diseñó una capa de abstracción que permite intercambiar OpenAI, Gemini y Ollama mediante configuración, sin tocar el resto del pipeline.

</div>

<div class="decision-card__step decision-card__step--resultado">
<div class="decision-card__step-label">Resultado</div>

El pipeline RAG permanece inalterado con independencia del modelo utilizado — el LLM se trata como un componente intercambiable, no como el núcleo del sistema.

</div>

</div>

<div class="decision-card">

<div class="decision-card__title"><span class="decision-card__number">⑥</span> ¿Cómo mantener el corpus actualizado?</div>

<div class="decision-card__step decision-card__step--problema">
<div class="decision-card__step-label">Problema</div>

El conocimiento del sistema no puede quedar fijo en el momento de la ingesta: la literatura biomédica se publica, revisa y retracta de forma continua.

</div>

<div class="decision-card__step decision-card__step--decision">
<div class="decision-card__step-label">Decisión</div>

Un proceso de sincronización incremental revisa el corpus ya indexado en busca de cambios, evitando reconstruir el índice completo y sin bloquear el resto de la API mientras se ejecuta.

</div>

<div class="decision-card__step decision-card__step--resultado">
<div class="decision-card__step-label">Resultado</div>

El corpus evoluciona junto a la literatura científica sin interrumpir el servicio de consultas — el mecanismo detrás de la captura de la sección "Mantenimiento del corpus".

</div>

</div>

## Resultado

El resultado es una plataforma RAG biomédica capaz de incorporar nueva literatura científica, mantener el corpus sincronizado con PubMed y PubMed Central, adaptar automáticamente la estrategia de recuperación a cada consulta y generar respuestas fundamentadas con trazabilidad hasta el fragmento exacto utilizado como evidencia.
