# Plan de implementación: `apply_filters()` para la NUEVA UI de filtros de LinkedIn

## Contexto verificado (leído del código y del HTML real)

- Archivo a modificar: `runAiBot.py` (único archivo). Helpers en `modules/clickers_and_finders.py` NO se tocan (se reutiliza `try_xp`, `buffer`, `scroll_to_view`).
- `apply_filters()` empieza en la línea ~376 y el bloque `try/except` con diagnóstico (captura + volcado de HTML) y el `pyautogui.confirm` de `pause_after_filters` están al final (líneas ~421-472). AMBOS deben conservarse textualmente.
- Globales disponibles en `runAiBot.py` (vía `from config.search import *` + overrides de `user_config.json`): `sort_by`, `date_posted`, `salary`, `easy_apply_only`, `experience_level`, `job_type`, `on_site`, `companies`, `location`, `industry`, `job_function`, `job_titles`, `under_10_applicants`, `in_your_network`, `fair_chance_employer`, `pause_after_filters`, `search_location`, `click_gap`, `driver`, `wait`, `actions`.
- La URL de búsqueda (en `apply_to_jobs`, línea ~1340) YA codifica keywords + location con `urllib.parse.urlencode`. NO se toca. `set_search_location()` queda como respaldo y se sigue llamando al inicio de `apply_filters()`.
- Tras los filtros, `apply_to_jobs` espera `//li[@data-occludable-job-id]` — ese comportamiento posterior NO cambia.

### Estructura real de la UI (confirmada en `logs/screenshots/apply_filters_panel.html`)

Cada filtro desplegable (dropdown) tiene:
- Un botón "pill" con `id="searchFilter_<param>"` (p.ej. `searchFilter_sortBy`). Fallback: el contenedor `//div[@data-basic-filter-parameter-name='<param>']`.
- Al abrirlo, un `<form>` con inputs `radio`/`checkbox` de id estable e idioma-independiente, cada uno con `<label for="<id>">`.
- Un botón de aplicar con `aria-label="Aplicar el filtro actual para mostrar resultados"` y clase `artdeco-button--primary` (texto visible "Mostrar resultados"). Hay que clickarlo para aplicar y cerrar ese dropdown.

IDs confirmados en el HTML:
- `sortBy` (radio): `sortBy-DD` (Más recientes), `sortBy-R` (Más relevantes).
- `experience` (checkbox): `experience-1`..`experience-6`.
- `timePostedRange` (radio): `timePostedRange-r86400`, `-r604800`, `-r2592000`, y `timePostedRange-` (Cualquier momento, se omite).
- `workplaceType` (checkbox): `workplaceType-1` (Presencial/On-site), `workplaceType-2` (En remoto/Remote), `workplaceType-3` (Híbrido/Hybrid).

Casos especiales confirmados:
- **Easy Apply** = `searchFilter_applyWithLinkedin`: NO es un dropdown, es un pill toggle directo (`role="radio"`, `aria-checked="false"`). Se clickea una vez para activarlo; NO tiene botón "Mostrar resultados" propio.
- **jobType** (Tipo de empleo): NO existe como pill en la barra. Solo está dentro del panel "Todos los filtros" (botón `id="ember299"`, id inestable; se localiza por `aria-label` que contiene "Mostrar todos los filtros" / texto "Todos los filtros").

## Diccionarios de mapeo config -> id (constantes a nivel de módulo, cerca de `filter_label_translations`)

```python
# config value -> sufijo del id del input dentro del dropdown de LinkedIn (idioma-independiente).
SORT_BY_ID_MAP = {
    "Most recent": "DD",
    "Most relevant": "R",
}
EXPERIENCE_ID_MAP = {
    "Internship": "1",
    "Entry level": "2",
    "Associate": "3",
    "Mid-Senior level": "4",
    "Director": "5",
    "Executive": "6",
}
DATE_POSTED_ID_MAP = {
    "Past 24 hours": "r86400",
    "Past week": "r604800",
    "Past month": "r2592000",
    # "Any time" -> se omite (no se selecciona ningún input).
}
WORKPLACE_TYPE_ID_MAP = {
    "On-site": "1",
    "Remote": "2",
    "Hybrid": "3",
}
```

## Firma y pseudocódigo de `select_pill_filter`

```python
def select_pill_filter(param_name: str, option_ids: list[str]) -> bool:
    '''
    Abre el dropdown "pill" del filtro `param_name` en la barra de filtros nueva de LinkedIn,
    marca cada opción de `option_ids` (por id de input estable) y pulsa "Mostrar resultados".
    - `param_name`: nombre del parámetro del filtro, p.ej. "sortBy", "experience",
      "timePostedRange", "workplaceType". Se usa para localizar el pill `searchFilter_<param_name>`.
    - `option_ids`: lista de ids de input a seleccionar dentro del dropdown, p.ej. ["experience-2", "experience-3"].
    - Devuelve `True` si al menos una opción se seleccionó, `False` si no se pudo abrir el
      dropdown o no se marcó ninguna opción. No lanza excepción: registra warning y sigue.
    '''
```

Pseudocódigo:

1. Si `not option_ids`: devolver `False` (nada que hacer).
2. Abrir el dropdown:
   - `pill = try_xp(driver, f".//button[@id='searchFilter_{param_name}']", False)`.
   - Fallback si no está: `try_xp(driver, f".//div[@data-basic-filter-parameter-name='{param_name}']//button", False)`.
   - Si no hay pill: `logger.warning("No se encontró el filtro pill '%s'", param_name)`; devolver `False`.
   - `scroll_to_view(driver, pill)`; click vía `pill.click()` con fallback `driver.execute_script("arguments[0].click();", pill)` ante `ElementClickInterceptedException`.
   - `buffer(recommended_filter_wait(click_gap))` o `buffer(1)` para dejar abrir el dropdown.
3. Para cada `option_id` en `option_ids` (envuelto en try/except por opción):
   - Preferir el label clicable: `label = try_xp(driver, f".//label[@for='{option_id}']", False)`.
   - Si existe label visible: `scroll_to_view` + `label.click()`; ante intercept, `execute_script` click.
   - Si no hay label, buscar el input por id y hacer `driver.execute_script("arguments[0].click();", input_el)` (los inputs suelen estar ocultos; el click JS por id es lo más fiable).
   - Si todo falla para esa opción: `logger.warning("No se pudo seleccionar la opción '%s' del filtro '%s'", option_id, param_name)` y continuar (no abortar).
   - Marcar `selected_any = True` en el primer éxito. `buffer(click_gap or 1)` entre clics (ritmo humano, anti-ban).
4. Aplicar el dropdown: localizar el botón primario DENTRO del dropdown abierto y clickarlo:
   - XPath robusto (bilingüe + case-insensitive sobre aria-label, más fallback por clase):
     `//button[contains(translate(@aria-label,'ABCDEFGHIJKLMNOPQRSTUVWXYZ','abcdefghijklmnopqrstuvwxyz'),'mostrar resultados') or contains(translate(@aria-label,...),'aplicar el filtro actual') or contains(translate(@aria-label,...),'apply current filters to show')]`
   - Preferir el que sea visible (reutilizar `try_xp(..., True)`; si hay varios, el primero clicable sirve porque solo el dropdown abierto lo expone visible). Alternativamente anclar por clase `artdeco-button--primary` dentro del contenedor visible.
   - `buffer(recommended_filter_wait(click_gap))` tras aplicar para dejar recargar resultados.
5. Devolver `selected_any`.

Notas de robustez (anti-ban, confirmado por la práctica del repo):
- Usar `buffer(...)` entre acciones (nunca ráfagas de clics). `buffer` ya introduce la pausa configurable.
- Reutilizar `scroll_to_view` para que el elemento esté en viewport antes de clickar (comportamiento más humano y evita intercept).
- No fabricar ids: solo los de los diccionarios de arriba, verificados contra el HTML real.

## Nueva estructura de `apply_filters()` paso a paso

Mantener la firma `def apply_filters() -> None:` y el docstring. Reemplazar SOLO el cuerpo del bloque `try` que hoy abre "Todos los filtros"; conservar intactos el `set_search_location()` inicial, el `except` de diagnóstico y el bloque `pause_after_filters`.

1. `set_search_location()` (sin cambios, respaldo).
2. Dentro de `try:`:
   - `recommended_wait = recommended_filter_wait(click_gap)`.
   - Esperar a que la barra de filtros nueva esté presente antes de tocar pills:
     `wait.until(EC.presence_of_element_located((By.ID, "search-reusables__filters-bar")))` (envuelto defensivamente; si falla, el `except` ya maneja el fallback). Esto reemplaza el antiguo click a "All filters".
   - **sortBy**: si `sort_by` y `sort_by in SORT_BY_ID_MAP`: `select_pill_filter("sortBy", [f"sortBy-{SORT_BY_ID_MAP[sort_by]}"])`.
   - **timePostedRange**: si `date_posted` y `date_posted in DATE_POSTED_ID_MAP` (es decir, distinto de "Any time"/vacío): `select_pill_filter("timePostedRange", [f"timePostedRange-{DATE_POSTED_ID_MAP[date_posted]}"])`.
   - **experience**: construir `ids = [f"experience-{EXPERIENCE_ID_MAP[v]}" for v in experience_level if v in EXPERIENCE_ID_MAP]`; si `ids`: `select_pill_filter("experience", ids)`.
   - **workplaceType**: construir `ids = [f"workplaceType-{WORKPLACE_TYPE_ID_MAP[v]}" for v in on_site if v in WORKPLACE_TYPE_ID_MAP]`; si `ids`: `select_pill_filter("workplaceType", ids)`.
   - **easy_apply_only** (toggle directo, NO dropdown): si `easy_apply_only`, click sobre `//button[@id='searchFilter_applyWithLinkedin']` solo si `aria-checked != 'true'`, vía `try_xp` + fallback JS click; `buffer(recommended_wait)`. Comentario: no requiere "Mostrar resultados".
   - **job_type** (sin pill en la barra): intentar primero el pill `searchFilter_jobType` por si en runtime existe:
     `if job_type:` -> construir ids `jobType-F/P/C/T/V/O` con un mapa `JOB_TYPE_ID_MAP = {"Full-time":"F","Part-time":"P","Contract":"C","Temporary":"T","Volunteer":"V","Internship":"I","Other":"O"}` y llamar `select_pill_filter("jobType", ids)`. Si devuelve `False` (no hay pill), hacer fallback al panel "Todos los filtros": abrir el botón por aria-label ("mostrar todos los filtros" / "all filters"), usar el flujo viejo `multi_sel_noWait(driver, translate_filter_list(job_type))` dentro del panel y pulsar el botón "Mostrar/Ver resultados" del panel, y cerrar. Dejar comentario:
     `# TODO: jobType no aparece como pill en el HTML capturado; los ids jobType-* NO están verificados. Si el pill no existe se usa el panel "Todos los filtros" como fallback. Verificar en runtime.`
3. Conservar TAL CUAL el botón global "Ver/Mostrar resultados" solo si se usó el panel "Todos los filtros"; los dropdowns de pills ya aplican cada uno su propio "Mostrar resultados", así que para el flujo de pills NO hace falta un botón global.
4. Conservar TAL CUAL el bloque `pause_after_filters` (`global pause_after_filters` + `pyautogui.confirm(...)`).
5. Conservar TAL CUAL el bloque `except Exception as e:` con diagnóstico (URL, título, screenshot, volcado de HTML del panel/barra, cierre con ESCAPE y warning de continuación).

### Qué pasa EXACTAMENTE con jobType y Easy Apply (y por qué)

- **Easy Apply**: resuelto como pill toggle directo `searchFilter_applyWithLinkedin` (confirmado en el HTML). Se activa con un click; respeta `aria-checked` para no desactivarlo si ya estaba activo.
- **jobType**: NO hay pill en el HTML capturado (solo aparece en "Todos los filtros"). Enfoque: intentar `select_pill_filter("jobType", ...)` por robustez a futuro; si no existe el pill, caer al panel "Todos los filtros" con el flujo antiguo `multi_sel_noWait` + `translate_filter_list`. Si el panel tampoco está disponible, se omite con warning claro y el `# TODO` indicado. NO se inventan ids estables para jobType.

## Funciones que NO se borran (pueden quedar inertes)

`set_search_location`, `wait_span_click`, `multi_sel_noWait`, `boolean_button_click`, `click_filter_option`, `translate_filter_list`. `translate_filter_list`/`multi_sel_noWait` siguen usándose en el fallback de jobType.

## XPaths/ids concretos verificados contra el HTML real

- Barra: `//div[@id='search-reusables__filters-bar']`.
- Pill por parámetro: `//button[@id='searchFilter_sortBy']`, `//button[@id='searchFilter_experience']`, `//button[@id='searchFilter_timePostedRange']`, `//button[@id='searchFilter_workplaceType']`. Fallback: `//div[@data-basic-filter-parameter-name='<param>']//button`.
- Opción (label): `//label[@for='experience-2']`, etc. Input: `//input[@id='experience-2']`.
- Aplicar dropdown: `//button[contains(translate(@aria-label,'ABCDEFGHIJKLMNOPQRSTUVWXYZ','abcdefghijklmnopqrstuvwxyz'),'mostrar resultados') or contains(...,'aplicar el filtro actual') or contains(...,'apply current filters to show')]`.
- Easy Apply: `//button[@id='searchFilter_applyWithLinkedin']` (leer `aria-checked`).
- Todos los filtros (fallback jobType): `//button[contains(translate(@aria-label,...),'mostrar todos los filtros') or contains(translate(@aria-label,...),'all filters') or normalize-space()='Todos los filtros' or normalize-space()='All filters']` (id `ember299` es inestable, NO usarlo).

---

## Lista de implementación (ordenada por dependencia)

- [ ] 1. Añadir los diccionarios de mapeo config->id a nivel de módulo en `runAiBot.py`, junto a `filter_label_translations` (líneas ~314-346): `SORT_BY_ID_MAP`, `EXPERIENCE_ID_MAP`, `DATE_POSTED_ID_MAP`, `WORKPLACE_TYPE_ID_MAP`, `JOB_TYPE_ID_MAP`.
      Files: `runAiBot.py`
      Verify: `& ".venv\Scripts\python.exe" -c "import ast; ast.parse(open('runAiBot.py', encoding='utf-8').read())"` — sin errores de sintaxis.

- [ ] 2. Añadir el helper `select_pill_filter(param_name: str, option_ids: list[str]) -> bool` en `runAiBot.py`, justo antes de `def apply_filters()` (línea ~376). Implementar el pseudocódigo de arriba: abrir pill por `searchFilter_<param>` (fallback `data-basic-filter-parameter-name`), click por `//label[@for='<id>']` o JS click en el input por id, pausas con `buffer`, click en "Mostrar resultados" del dropdown, manejo de excepción por opción con warning, retorno `bool`. Docstring en español con tipos (estilo CONTRIBUTING.md).
      Files: `runAiBot.py`
      Verify: `& ".venv\Scripts\python.exe" -c "import ast; ast.parse(open('runAiBot.py', encoding='utf-8').read())"` — sin errores; revisión manual de que todos los ids/XPaths coinciden con `logs/screenshots/apply_filters_panel.html`.

- [ ] 3. Reescribir el cuerpo del bloque `try:` de `apply_filters()` para usar `select_pill_filter` por filtro (sortBy, timePostedRange, experience, workplaceType), el toggle directo para Easy Apply (`searchFilter_applyWithLinkedin` respetando `aria-checked`), y jobType con intento de pill + fallback a "Todos los filtros" + `# TODO`. CONSERVAR intactos: `set_search_location()` inicial, el bloque `pause_after_filters` (`pyautogui.confirm`) y el bloque `except Exception as e:` de diagnóstico. Eliminar las llamadas a `click_filter_option`/`multi_sel_noWait`/`boolean_button_click` del flujo principal (salvo el fallback jobType).
      Files: `runAiBot.py`
      Verify: `& ".venv\Scripts\python.exe" -c "import ast; ast.parse(open('runAiBot.py', encoding='utf-8').read())"` — sin errores; revisión manual: (a) el `pyautogui.confirm` sigue presente; (b) el `except` de diagnóstico (screenshot + volcado HTML + ESCAPE) sigue presente; (c) no se referencian variables inexistentes.

- [ ] 4. Verificación de integración (sin login real; no hay pytest): comprobar que el módulo importa y que las funciones viejas no borradas siguen existiendo.
      Files: (ninguno; solo verificación)
      Verify: `& ".venv\Scripts\python.exe" -c "import ast; m=ast.parse(open('runAiBot.py',encoding='utf-8').read()); names={n.name for n in m.body if isinstance(n, ast.FunctionDef)}; assert {'select_pill_filter','apply_filters','set_search_location','click_filter_option','translate_filter_list'} <= names, names; print('OK', 'select_pill_filter' in names)"` — imprime `OK True`. (No se intenta `import runAiBot` completo porque arrastra Selenium/driver y side effects; `ast.parse` + chequeo de nombres es la verificación acordada.)

## Notas / supuestos

- No se puede verificar con login real (2FA) ni con pytest (no existe en el proyecto). La verificación acordada es `ast.parse` + revisión de selectores contra el HTML real capturado. Esto está asumido explícitamente.
- Los ids de `jobType-*` NO se pudieron verificar (no hay pill en el HTML); por eso el fallback a "Todos los filtros" y el `# TODO`. No se inventan ids estables.
- Las etiquetas en español en `filter_label_translations` ya están confirmadas por el usuario y se reutilizan en el fallback de jobType vía `translate_filter_list`.
