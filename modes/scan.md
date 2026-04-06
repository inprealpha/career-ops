# Modo: scan — Portal Scanner (Descubrimiento de Ofertas)

Escanea portales de empleo configurados, filtra por relevancia de título, y añade nuevas ofertas al pipeline para evaluación posterior.

## Ejecución recomendada

Ejecutar como subagente para no consumir contexto del main:

```
Agent(
    subagent_type="general-purpose",
    prompt="[contenido de este archivo + datos específicos]",
    run_in_background=True
)
```

## Configuración

Leer `portals.yml` que contiene:
- `search_queries`: Lista de queries WebSearch con `site:` filters por portal (descubrimiento amplio)
- `tracked_companies`: Empresas específicas con `careers_url` para navegación directa
- `title_filter`: Keywords positive/negative/seniority_boost para filtrado de títulos

## Estrategia de descubrimiento (3 niveles)

### Nivel 1 — Playwright directo (PRINCIPAL)

**Para cada empresa en `tracked_companies`:** Navegar a su `careers_url` con Playwright (`browser_navigate` + `browser_snapshot`), leer TODOS los job listings visibles, y extraer título + URL de cada uno. Este es el método más fiable porque:
- Ve la página en tiempo real (no resultados cacheados de Google)
- Funciona con SPAs (Ashby, Lever, Workday)
- Detecta ofertas nuevas al instante
- No depende de la indexación de Google

**Cada empresa DEBE tener `careers_url` en portals.yml.** Si no la tiene, buscarla una vez, guardarla, y usar en futuros scans.

### Nivel 2 — Greenhouse API (COMPLEMENTARIO)

Para empresas con Greenhouse, la API JSON (`boards-api.greenhouse.io/v1/boards/{slug}/jobs`) devuelve datos estructurados limpios. Usar como complemento rápido de Nivel 1 — es más rápido que Playwright pero solo funciona con Greenhouse.

### Nivel 3 — WebSearch queries (DESCUBRIMIENTO AMPLIO)

Los `search_queries` con `site:` filters cubren portales de forma transversal (todos los Ashby, todos los Greenhouse, etc.). Útil para descubrir empresas NUEVAS que aún no están en `tracked_companies`, pero los resultados pueden estar desfasados.

**Prioridad de ejecución:**
1. Nivel 1: Playwright → todas las `tracked_companies` con `careers_url`
2. Nivel 2: API → todas las `tracked_companies` con `api:`
3. Nivel 3: WebSearch → todos los `search_queries` con `enabled: true`

Los niveles son aditivos — se ejecutan todos, los resultados se mezclan y deduplicar.

## Adaptación regional (ej: India)

El scanner está diseñado para ser **config-driven**. Para adaptarlo a India o cualquier otro mercado:

1. **Cambiar `title_filter`** para incluir títulos comunes en tu mercado objetivo.
2. **Añadir geo-keywords** en `search_queries` como `"India"`, `"Bengaluru"`, `"Bangalore"`, `"Hyderabad"`, `"Pune"`, `"Mumbai"`, `"Gurugram"`, `"Chennai"`.
3. **Priorizar portales públicos del mercado local** en `search_queries` y `tracked_companies`.
4. **Guardar `careers_url` locales** cuando una empresa tenga una página de jobs específica por país o región.
5. **Usar notas en `tracked_companies`** para recordar si una empresa contrata en India, pide presencialidad en una ciudad concreta, o tiene una entidad legal local.
6. **Mantener variantes de ciudad** cuando existan (`Bengaluru`/`Bangalore`) para mejorar la recuperación, ya que los portales no usan siempre el mismo nombre.

Ejemplo:

```yaml
- name: India - AI Roles
  query: 'site:job-boards.greenhouse.io OR site:jobs.ashbyhq.com OR site:naukri.com ("AI Engineer" OR "Solutions Architect" OR "Product Manager") ("India" OR "Bengaluru" OR "Bangalore" OR "Hyderabad" OR "Pune" OR "Mumbai" OR "Gurugram" OR "Chennai")'
  enabled: true
```

La lógica del scanner no cambia: solo cambian las URLs, queries, y keywords de filtrado.

## Workflow

1. **Leer configuración**: `portals.yml`
2. **Leer historial**: `data/scan-history.tsv` → URLs ya vistas
3. **Leer dedup sources**: `data/applications.md` + `data/pipeline.md`

4. **Nivel 1 — Playwright scan** (paralelo en batches de 3-5):
   Para cada empresa en `tracked_companies` con `enabled: true` y `careers_url` definida:
   a. `browser_navigate` a la `careers_url`
   b. `browser_snapshot` para leer todos los job listings
   c. Si la página tiene filtros/departamentos, navegar las secciones relevantes
   d. Para cada job listing extraer: `{title, url, company}`
   e. Si la página pagina resultados, navegar páginas adicionales
   f. Acumular en lista de candidatos
   g. Si `careers_url` falla (404, redirect), intentar `scan_query` como fallback y anotar para actualizar la URL

5. **Nivel 2 — Greenhouse APIs** (paralelo):
   Para cada empresa en `tracked_companies` con `api:` definida y `enabled: true`:
   a. WebFetch de la URL de API → JSON con lista de jobs
   b. Para cada job extraer: `{title, url, company}`
   c. Acumular en lista de candidatos (dedup con Nivel 1)

6. **Nivel 3 — WebSearch queries** (paralelo si posible):
   Para cada query en `search_queries` con `enabled: true`:
   a. Ejecutar WebSearch con el `query` definido
   b. De cada resultado extraer: `{title, url, company}`
      - **title**: del título del resultado (antes del " @ " o " | ")
      - **url**: URL del resultado
      - **company**: después del " @ " en el título, o extraer del dominio/path
   c. Acumular en lista de candidatos (dedup con Nivel 1+2)

6. **Filtrar por título** usando `title_filter` de `portals.yml`:
   - Al menos 1 keyword de `positive` debe aparecer en el título (case-insensitive)
   - 0 keywords de `negative` deben aparecer
   - `seniority_boost` keywords dan prioridad pero no son obligatorios

7. **Deduplicar** contra 3 fuentes:
   - `scan-history.tsv` → URL exacta ya vista
   - `applications.md` → empresa + rol normalizado ya evaluado
   - `pipeline.md` → URL exacta ya en pendientes o procesadas

8. **Para cada oferta nueva que pase filtros**:
   a. Añadir a `pipeline.md` sección "Pendientes": `- [ ] {url} | {company} | {title}`
   b. Registrar en `scan-history.tsv`: `{url}\t{date}\t{query_name}\t{title}\t{company}\tadded`

9. **Ofertas filtradas por título**: registrar en `scan-history.tsv` con status `skipped_title`
10. **Ofertas duplicadas**: registrar con status `skipped_dup`

## Extracción de título y empresa de WebSearch results

Los resultados de WebSearch vienen en formato: `"Job Title @ Company"` o `"Job Title | Company"` o `"Job Title — Company"`.

Patrones de extracción por portal:
- **Ashby**: `"Senior AI PM (Remote) @ EverAI"` → title: `Senior AI PM`, company: `EverAI`
- **Greenhouse**: `"AI Engineer at Anthropic"` → title: `AI Engineer`, company: `Anthropic`
- **Lever**: `"Product Manager - AI @ Temporal"` → title: `Product Manager - AI`, company: `Temporal`

Regex genérico: `(.+?)(?:\s*[@|—–-]\s*|\s+at\s+)(.+?)$`

## URLs privadas

Si se encuentra una URL no accesible públicamente:
1. Guardar el JD en `jds/{company}-{role-slug}.md`
2. Añadir a pipeline.md como: `- [ ] local:jds/{company}-{role-slug}.md | {company} | {title}`

## Portales con auth obligatorio (LinkedIn, Workday privado, etc.)

El scanner prioriza **fuentes públicas y reproducibles**. Si un portal requiere login:

1. **NO depender de esa URL** para scans automáticos recurrentes.
2. Intentar primero una fuente pública equivalente:
   - careers page de la empresa
   - Greenhouse API / Ashby / Lever pública
   - WebSearch con `site:` filters
3. Si el contenido solo existe tras login:
   - pedir al usuario que pegue el texto del JD, o
   - guardar el JD manualmente en `jds/` y referenciarlo como `local:jds/...`
4. Registrar en el resumen que esa fuente requiere auth para que el usuario decida si merece seguimiento manual.

**LinkedIn en particular:** usarlo como señal de descubrimiento o outreach, pero no como fuente principal del JD si la vacante no es accesible públicamente.

## Scan History

`data/scan-history.tsv` trackea TODAS las URLs vistas:

```
url	first_seen	portal	title	company	status
https://...	2026-02-10	Ashby — AI PM	PM AI	Acme	added
https://...	2026-02-10	Greenhouse — SA	Junior Dev	BigCo	skipped_title
https://...	2026-02-10	Ashby — AI PM	SA AI	OldCo	skipped_dup
```

## Resumen de salida

```
Portal Scan — {YYYY-MM-DD}
━━━━━━━━━━━━━━━━━━━━━━━━━━
Queries ejecutados: N
Ofertas encontradas: N total
Filtradas por título: N relevantes
Duplicadas: N (ya evaluadas o en pipeline)
Nuevas añadidas a pipeline.md: N

  + {company} | {title} | {query_name}
  ...

→ Ejecuta /career-ops pipeline para evaluar las nuevas ofertas.
```

## Gestión de careers_url

Cada empresa en `tracked_companies` debe tener `careers_url` — la URL directa a su página de ofertas. Esto evita buscarlo cada vez.

**Patrones conocidos por plataforma:**
- **Ashby:** `https://jobs.ashbyhq.com/{slug}`
- **Greenhouse:** `https://job-boards.greenhouse.io/{slug}` o `https://job-boards.eu.greenhouse.io/{slug}`
- **Lever:** `https://jobs.lever.co/{slug}`
- **Custom:** La URL propia de la empresa (ej: `https://openai.com/careers`)

**Si `careers_url` no existe** para una empresa:
1. Intentar el patrón de su plataforma conocida
2. Si falla, hacer un WebSearch rápido: `"{company}" careers jobs`
3. Navegar con Playwright para confirmar que funciona
4. **Guardar la URL encontrada en portals.yml** para futuros scans

**Si `careers_url` devuelve 404 o redirect:**
1. Anotar en el resumen de salida
2. Intentar scan_query como fallback
3. Marcar para actualización manual

## Mantenimiento del portals.yml

- **SIEMPRE guardar `careers_url`** cuando se añade una empresa nueva
- Añadir nuevos queries según se descubran portales o roles interesantes
- Desactivar queries con `enabled: false` si generan demasiado ruido
- Ajustar keywords de filtrado según evolucionen los roles target
- Añadir empresas a `tracked_companies` cuando interese seguirlas de cerca
- Verificar `careers_url` periódicamente — las empresas cambian de plataforma ATS
