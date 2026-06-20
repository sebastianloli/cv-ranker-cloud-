# Contexto del problema, casos de uso e impacto

## El problema

Seleccionar candidatos a partir de CVs es uno de los cuellos de botella más caros del proceso de contratación. Para una sola vacante, un reclutador puede recibir entre **50 y 100 CVs** (y en convocatorias masivas, varios cientos), y debe leerlos uno por uno para armar un shortlist.

El dato incómodo es cuánto tiempo real se le dedica a cada uno. Estudios de seguimiento ocular (Ladders, 2018) midieron que un reclutador dedica en promedio **~7 segundos** al escaneo inicial de un CV. Universidades como Tufts reportan que no es raro recibir **300-500+ CVs por puesto**. La combinación de alto volumen y poco tiempo produce tres consecuencias concretas:

1. **Se pierden buenos candidatos.** Un escaneo de 7 segundos no alcanza para evaluar profundidad técnica, potencial o experiencia transferible. Perfiles valiosos se descartan por un formato pobre o por no usar exactamente las palabras esperadas.
2. **Sesgo en la decisión.** En esos primeros segundos el ojo se va al nombre, la foto, la universidad o la antigüedad. Eso abre la puerta a sesgos demográficos (género, origen, edad) que nada tienen que ver con la idoneidad para el puesto.
3. **Fatiga e inconsistencia.** El CV número 80 no recibe la misma atención que el número 1. La calidad de la evaluación se degrada con el cansancio y depende del orden de revisión.

En resumen: el reclutador está obligado a elegir entre **rapidez** y **rigor**, y hoy gana la rapidez a costa de la calidad y la equidad.

## La solución

Un sistema que recibe la **descripción del puesto** y un **lote de CVs en PDF**, y devuelve un **ranking pre-ordenado por score** donde cada candidato trae fortalezas, gaps, seniority y un resumen. El reclutador deja de leer 80 CVs en crudo y pasa directo a evaluar a los mejores, con contexto.

Tres decisiones de diseño atacan directamente los tres problemas:

- **Evaluación con un LLM (Groq), no con keywords.** Un ATS tradicional filtra por coincidencia de palabras clave y se equivoca con sinónimos y contexto: no entiende que "lideré la migración a microservicios" implica experiencia en arquitectura aunque no aparezca esa palabra exacta. El LLM evalúa el CV completo contra los requisitos del puesto y entrega una valoración explicable (fortalezas y gaps), no un match binario.
- **Anonimización antes de evaluar.** El sistema elimina nombre, email y teléfono de cada CV antes de enviarlo al modelo. La evaluación se hace sobre experiencia y habilidades, no sobre la identidad, mitigando el sesgo demográfico que se cuela en la revisión humana.
- **Profundidad consistente.** Cada CV recibe exactamente el mismo nivel de análisis, sin fatiga ni sesgo de orden. El candidato 80 se evalúa igual que el 1.

## Casos de uso

1. **Reclutador con alto volumen (caso principal).** Una vacante con 50-100 postulantes; el reclutador obtiene un shortlist ordenado en minutos y se concentra en los top.
2. **Convocatorias masivas (prácticas, trainees, graduate programs).** Cientos de postulantes con perfiles homogéneos donde el filtrado manual es inviable.
3. **Agencias y consultoras de reclutamiento.** Procesan lotes para múltiples clientes con distintos perfiles; necesitan rapidez sin sacrificar calidad.
4. **Pre-filtro antes de entrevistas técnicas.** Reducir la lista de candidatos que pasan a una evaluación técnica costosa, priorizando por fit real.

## Impacto esperado

**Tiempo.** Una revisión por fit cuidadosa (no el escaneo de 7 segundos, sino una lectura real del CV contra los requisitos) toma del orden de 2-4 minutos por CV. Para una vacante de 80 CVs, eso son aproximadamente **3 a 5 horas de trabajo manual por vacante**. El sistema procesa el lote completo en **paralelo y de forma asíncrona**, entregando el ranking en **minutos**. El reclutador recupera esas horas para las etapas que sí requieren juicio humano: entrevistas y decisión final.

**Calidad y consistencia.** Al dar a cada CV la misma profundidad de análisis, se reduce el riesgo de descartar buenos candidatos por fatiga, formato o sesgo de orden. El ranking prioriza por idoneidad, no por quién llegó primero a la bandeja.

**Equidad.** La anonimización previa a la evaluación ataca de raíz el sesgo demográfico del escaneo manual. La decisión se justifica con fortalezas y gaps concretos, lo que hace el proceso **auditable y defendible**, no una impresión de 7 segundos.

**Trazabilidad.** Cada evaluación queda registrada con su score, fortalezas, gaps y resumen, permitiendo revisar y justificar por qué un candidato quedó arriba o abajo.

---

### Fuentes

- Ladders, *Eye-Tracking Study* (2018): tiempo promedio de escaneo inicial de un CV ≈ 7.4 segundos.
- Tufts University, Career Center: volumen típico de 300-500+ CVs por puesto.
