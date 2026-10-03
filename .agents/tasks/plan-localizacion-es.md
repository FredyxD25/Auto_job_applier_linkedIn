# Plan de implementación — Localización ES de la interacción con LinkedIn

Objetivo: hacer que TODOS los selectores que dependen de texto en inglés acepten también el
equivalente en español de la UI de LinkedIn, para que el bot funcione cuando la cuenta ve
LinkedIn en español. Estrategia general: **selectores bilingües (inglés OR español)**, nunca
reemplazo ciego. Donde el texto venga de `config/search.py` (sort_by, date_posted, salary),
traducir el valor de config a una lista de labels aceptados (EN + ES) antes de clickar.

## Contexto verificado del proyecto

- Lenguaje: Python 3.14.7 (venv en `.venv\Scripts\python.exe`). **pytest NO está instalado** en
  el venv (confirmado: `import pytest` → ImportError). No hay suite de tests ejecutable.
- Verificación disponible: comprobación de sintaxis con `ast.parse`. Es la verificación real de
  cada ítem de este plan. "grep" no cuenta como verificación.
- Comando de verificación por archivo:
  `& ".venv\Scripts\python.exe" -c "import ast; ast.parse(open(r'RUTA','r',encoding='utf-8').read())"`
  Salida esperada: sin error, exit code 0.
- CONTRIBUTING.md: funciones en snake_case con docstring `''' ... '''` y tipos en firma. Al
  tocar funciones hay que mantener ese estilo. (No afecta a este plan porque no se crean
  funciones nuevas salvo un helper de traducción, que seguirá la guía.)
- NO TOCAR: lógica de login (`sign_in_button_labels` ya es bilingüe), filtrado de
  sponsorship/bad_words, perfil de Chrome, `config/*.py` (salvo comentarios si fuese
  estrictamente necesario), ni la pref de idioma del navegador en `modules/open_chrome.py`.

## Decisión sobre `text_xpath` (clickers_and_finders.py)

`text_xpath(tag, text)` hace `contains(lower(normalize-space(.)), text.lower())` con UN solo
texto y solo pasa a minúsculas ASCII. Es un helper genérico usado por `wait_span_click`,
`multi_sel`, `multi_sel_noWait` y `boolean_button_click`. **Decisión: NO cambiar su firma**
(rompería muchos llamadores y el matcheo de caracteres acentuados en minúscula ASCII no aporta
aquí). En su lugar, los llamadores en `runAiBot.py` que usan texto en inglés pasarán el label
en español (o se resolverán vía el mapa de traducción). Razón: mantener el helper simple y
localizar el conocimiento de idioma en el punto de uso, que es donde se sabe qué label aplica.

Nota sobre acentos: `text_xpath` compara en minúsculas y con `contains`, así que un `text` en
español con acentos se compara correctamente contra el texto real de la página (ambos lados
conservan los acentos; solo cambia A-Z→a-z ASCII). Es seguro pasar labels con acentos.

## Decisión sobre valores de config (`sort_by`, `date_posted`, `salary`)

Estos valores los fija el usuario en `config/search.py` en inglés
(`sort_by="Most recent"`, `date_posted="Past week"`, `salary="$80,000+"`) y hoy se clican vía
`wait_span_click(driver, <valor>)`. **Decisión: traducir en runtime, no reemplazar config.**
Se añade en `runAiBot.py` un diccionario `filter_label_translations: dict[str, list[str]]` que
mapea cada valor en inglés a `[inglés, español]`, y un helper `click_filter_option(driver, value)`
que intenta cada variante con `wait_span_click`. Así el `config/search.py` del usuario sigue en
inglés (no se le pide reconfigurar) y funciona con la UI en español. `salary` usa el mismo
símbolo `$` y formato en ambos idiomas, así que se deja pasar tal cual pero a través del mismo
helper por consistencia (una sola variante).

Traducciones a incluir en el mapa (verificadas como las etiquetas visibles habituales de la UI
de LinkedIn en español; marcadas las que necesitan verificación en runtime):
- sort_by: `"Most recent"` → `["Most recent", "Más recientes"]`; `"Most relevant"` →
  `["Most relevant", "Más relevantes"]`. (NECESITA VERIFICACIÓN EN RUNTIME)
- date_posted: `"Any time"` → `["Any time", "En cualquier momento"]`; `"Past month"` →
  `["Past month", "Último mes"]`; `"Past week"` → `["Past week", "Última semana"]`;
  `"Past 24 hours"` → `["Past 24 hours", "Últimas 24 horas"]`. (NECESITA VERIFICACIÓN EN RUNTIME)
- salary: una sola variante (`"$80,000+"` etc.), idéntica en ambos idiomas.

El mapa es tolerante: si un valor no está en el diccionario, el helper usa `[value]` tal cual,
de modo que valores desconocidos o ya en español siguen funcionando.

---

## Puntos de edición (ordenados por dependencia)

Orden: primero las infra-piezas compartidas en runAiBot.py (mapa + helper) porque de ellas
dependen los filtros; luego cada bloque de selectores. Cada ítem deja el archivo parseable.

- [x] 1. Añadir el mapa de traducción de filtros y el helper `click_filter_option` en runAiBot.py.
      Insertar cerca del inicio del bloque de filtros (antes de `apply_filters`, ~línea 288):
      un `dict` `filter_label_translations` (sort_by/date_posted como arriba) y
      `def click_filter_option(driver: WebDriver, value: str, time: float=5.0) -> bool:` que
      recorra `filter_label_translations.get(value, [value])` llamando a `wait_span_click` y
      devuelva True al primer acierto. Docstring según CONTRIBUTING. Documentar en comentario la
      decisión de traducir valores de config.
      Archivos: runAiBot.py
      Verificar: `ast.parse(runAiBot.py)` sin error.

- [x] 2. Usar `click_filter_option` para sort_by, date_posted y salary en `apply_filters()`.
      Reemplazar `wait_span_click(driver, sort_by)` → `click_filter_option(driver, sort_by)`,
      `wait_span_click(driver, date_posted)` → `click_filter_option(driver, date_posted)`,
      `wait_span_click(driver, salary)` → `click_filter_option(driver, salary)` (~líneas 310, 311, 338).
      Archivos: runAiBot.py
      Verificar: `ast.parse(runAiBot.py)` sin error.

- [x] 3. Hacer bilingüe el botón "All filters" en `apply_filters()` (~línea 307).
      Actual: `'//button[normalize-space()="All filters"]'`.
      Propuesto: `'//button[normalize-space()="All filters" or normalize-space()="Todos los filtros"]'`.
      Archivos: runAiBot.py
      Verificar: `ast.parse(runAiBot.py)` sin error.

- [x] 4. Hacer bilingües los boolean toggles en `apply_filters()` (~líneas 320, 332, 333, 334).
      `boolean_button_click` matchea un `<h3>` vía `text_xpath` (contains, un solo texto), así que
      para aceptar ES hay que pasar el label español. Decisión: añadir en runAiBot.py un pequeño
      mapa `boolean_filter_labels` o pasar el label ES directamente por llamada. Preferido:
      invocar `boolean_button_click` con el texto que corresponda al idioma activo NO es posible a
      ciegas (no sabemos el idioma), así que se extiende `boolean_button_click` para aceptar
      varios textos. Ver ítem 10 (cambio en clickers) del que depende este ítem; aquí solo se
      actualizan las llamadas para pasar la lista EN+ES:
        - "Easy Apply" → `["Easy Apply", "Solicitud sencilla"]`
        - "Under 10 applicants" → `["Under 10 applicants", "Menos de 10 solicitantes"]` (NECESITA VERIFICACIÓN RUNTIME)
        - "In your network" → `["In your network", "En tu red"]` (NECESITA VERIFICACIÓN RUNTIME)
        - "Fair Chance Employer" → `["Fair Chance Employer", "Empleador con igualdad de oportunidades"]` (NECESITA VERIFICACIÓN RUNTIME)
      Depende de: ítem 10.
      Archivos: runAiBot.py
      Verificar: `ast.parse(runAiBot.py)` sin error.

- [x] 5. Verificar/ajustar el botón "show results" en `apply_filters()` (~línea 341).
      Actual: `contains(lower(@aria-label), "apply current filters to show")`. El aria-label en
      español probablemente es "Aplicar los filtros actuales para mostrar X resultados" o similar.
      Propuesto: aceptar substring EN **o** ES en minúsculas, p. ej.
      `contains(lower(@aria-label), "apply current filters to show") or contains(lower(@aria-label), "mostrar") or contains(lower(@aria-label), "resultados")`.
      El substring español EXACTO del aria-label NECESITA VERIFICACIÓN EN RUNTIME; usar un
      substring robusto ("resultados") y dejar comentario.
      Archivos: runAiBot.py
      Verificar: `ast.parse(runAiBot.py)` sin error.

- [x] 6. Hacer bilingüe `set_search_location()` (~líneas 272, 277, 282).
      Actual: input `@aria-label='City, state, or zip code'` y botón `@aria-label='Cancel'`.
      Propuesto input:
      `.//input[(@aria-label='City, state, or zip code' or @aria-label='Ciudad, estado o código postal') and not(@disabled)]`.
      Propuesto botón Cancel (2 ocurrencias):
      `.//button[@aria-label='Cancel' or @aria-label='Cancelar']`.
      El aria-label ES del input NECESITA VERIFICACIÓN EN RUNTIME; dejar comentario. "Cancelar" es
      estándar.
      Archivos: runAiBot.py
      Verificar: `ast.parse(runAiBot.py)` sin error.

- [x] 7. Hacer bilingües los botones del modal Easy Apply (~líneas 1043-1047).
      Actual:
        `next_button_xpath = './/button[@aria-label="Continue to next step" or contains(normalize-space(.), "Next")]'`
        `review_button_xpath = './/button[@aria-label="Review your application" or normalize-space(.)="Review"]'`
        `submit_button_xpath = './/button[@aria-label="Submit application" or normalize-space(.)="Submit application"]'`
      Propuesto (añadir variantes ES con `or`):
        next: añadir `@aria-label="Continuar al siguiente paso"` y `contains(normalize-space(.), "Siguiente")`.
        review: añadir `@aria-label="Revisar tu solicitud"` y `normalize-space(.)="Revisar"`.
        submit: añadir `@aria-label="Enviar solicitud"` y `normalize-space(.)="Enviar solicitud"`.
      Labels ES NECESITAN VERIFICACIÓN EN RUNTIME; dejar comentario (son los más críticos del flujo).
      Archivos: runAiBot.py
      Verificar: `ast.parse(runAiBot.py)` sin error.

- [x] 8. Hacer bilingües `apply_button_xpath`, `easy_apply_locators` y el "Continue" externo
      (~líneas 1023, 1037, 1060-1066).
      - `easy_apply_locators`: las variantes que buscan el texto "Easy Apply" en aria-label y en
        span deben aceptar también "Solicitud sencilla":
        aria-label: `contains(@aria-label, "Easy Apply") or contains(@aria-label, "Solicitud sencilla")`;
        span: `contains(normalize-space(.), "Easy Apply") or contains(normalize-space(.), "Solicitud sencilla")`.
        (Las variantes por `@id='jobs-apply-button-id'`, clase y URL flag ya son idioma-independientes; no tocar.)
      - `wait_span_click(driver, "Continue", 1, True, False)` en `external_apply` (~línea 1023):
        reemplazar por un intento EN y uno ES ("Continuar"), p. ej. llamar dos veces o añadir
        helper. Preferido: `wait_span_click(driver, "Continue", 1, True, False) or wait_span_click(driver, "Continuar", 1, True, False)`.
      "Solicitud sencilla" NECESITA VERIFICACIÓN EN RUNTIME.
      Archivos: runAiBot.py
      Verificar: `ast.parse(runAiBot.py)` sin error.

- [x] 9. Hacer bilingües "This is today", "Done" y el discard dentro del flujo Easy Apply
      (~líneas 989/1419 contexto, 1419/1422 "Done", discard_button_xpath ~línea 1148).
      - "This is today" en `answer_questions` (`try_xp(modal, ".//button[contains(@aria-label, 'This is today')]")`):
        `.//button[contains(@aria-label, 'This is today') or contains(@aria-label, 'hoy')]`.
        El texto ES exacto del date picker NECESITA VERIFICACIÓN EN RUNTIME; "hoy" como substring
        robusto + comentario.
      - `wait_span_click(driver, "Done", 2)` (2 ocurrencias en `apply_to_jobs`): intentar EN y ES:
        `wait_span_click(driver, "Done", 2) or wait_span_click(driver, "Listo", 2)` (y análogo en la
        otra ocurrencia). "Listo"/"Hecho" NECESITA VERIFICACIÓN EN RUNTIME; usar "Listo".
      - `discard_button_xpath` (~línea 1148): ya ancla en `@data-control-name='discard_application_confirm_btn'`
        (idioma-independiente) con fallback a `contains(normalize-space(.), 'Discard')`. Añadir
        `or contains(normalize-space(.), 'Descartar')` al fallback. El ancla principal ya cubre ES.
      Archivos: runAiBot.py
      Verificar: `ast.parse(runAiBot.py)` sin error.

- [x] 10. Extender `boolean_button_click` en clickers_and_finders.py para aceptar múltiples textos.
      Actual: `boolean_button_click(driver, actions, text: str)` busca `text_xpath("h3", text)`.
      Propuesto: aceptar `text: str | list[str]`; si es lista, construir el xpath del `<h3>` como
      la unión (`... or ...`) de `text_xpath` por cada variante, o iterar probando cada una.
      Mantener compatibilidad con llamadas que pasan un `str`. Docstring y tipos según CONTRIBUTING.
      Este ítem habilita el ítem 4 (debe hacerse antes o junto con él, pero como son archivos
      distintos cada uno queda parseable por separado).
      Archivos: modules/clickers_and_finders.py
      Verificar: `ast.parse(clickers_and_finders.py)` sin error.

- [x] 11. Verificación final de sintaxis de los dos archivos modificados.
      Ejecutar `ast.parse` sobre `runAiBot.py` y `modules/clickers_and_finders.py`.
      Archivos: (ninguno nuevo)
      Verificar: ambos `ast.parse` sin error, exit code 0.

---

## Puntos revisados que se dejan como están (con justificación)

- `label[normalize-space()='{answer}']` en `answer_questions` (~línea 810 del rango original):
  `answer` viene de `config/questions.py` (texto del usuario), no de la UI de LinkedIn. No es un
  label de UI a localizar. **Se deja igual.**
- Lógica de login (`sign_in_button_labels`, `login_email_css`, `login_password_css`,
  `is_logged_in_LN`): ya es bilingüe/idioma-independiente por diseño. **No tocar.**
- `multi_sel_noWait` para `experience_level`, `job_type`, `on_site`, `location`, `industry`,
  `job_function`, `job_titles`, `benefits`, `commitments`: estos valores también vienen de
  `config/search.py` en inglés y se clican como spans. **Fuera del alcance mínimo de esta tarea**
  (la tarea enumera sort_by/date_posted/salary y los toggles booleanos como los puntos críticos).
  Si se quiere cobertura total, se extendería `filter_label_translations` para incluir estas
  listas; se deja anotado como mejora futura y NECESITA VERIFICACIÓN EN RUNTIME de cada label ES.
  No se implementa en este plan para no inflar el cambio ni inventar traducciones no verificadas.
- `data-occludable-job-id`, clases artdeco, `data-test-form-element`, `data-control-name`,
  selectores por `@type`/`@id`: idioma-independientes. **No tocar.**

## Resumen de labels que NECESITAN VERIFICACIÓN EN RUNTIME

Marcar en los comentarios del código:
1. sort_by ES: "Más recientes" / "Más relevantes".
2. date_posted ES: "En cualquier momento" / "Último mes" / "Última semana" / "Últimas 24 horas".
3. Toggles ES: "Menos de 10 solicitantes", "En tu red", "Empleador con igualdad de oportunidades",
   "Solicitud sencilla".
4. Botón show results: substring ES del aria-label (se usa "resultados" como robusto).
5. Input de ubicación: aria-label ES ("Ciudad, estado o código postal").
6. Modal Easy Apply: "Continuar al siguiente paso"/"Siguiente", "Revisar tu solicitud"/"Revisar",
   "Enviar solicitud".
7. Date picker: texto ES de "This is today" (se usa substring "hoy").
8. "Done" ES: "Listo"/"Hecho".

Donde la verificación no sea posible antes de ejecutar, el selector queda bilingüe con `or` y/o
usa un substring raíz robusto (`contains`), de modo que un label ES ligeramente distinto no
rompa el inglés ni al revés.

---

## Registro de implementación (iteración 1 — COMPLETADA)

Archivos modificados: `runAiBot.py`, `modules/clickers_and_finders.py`.

Labels en español USADOS (bilingües EN or ES):
- "All filters" → también "Todos los filtros".
- "Cancel" → también "Cancelar".
- "Continue" (externo) → también "Continuar".
- "Discard" (fallback) → también "Descartar" (ancla principal `data-control-name` ya cubre ES).

Labels en español USADOS pero marcados `# NECESITA VERIFICACIÓN EN RUNTIME`:
1. sort_by: "Más recientes" / "Más relevantes" (mapa `filter_label_translations`).
2. date_posted: "En cualquier momento" / "Último mes" / "Última semana" / "Últimas 24 horas" (mapa).
3. Toggles booleanos: "Solicitud sencilla", "Menos de 10 solicitantes", "En tu red",
   "Empleador con igualdad de oportunidades" (vía `boolean_button_click` con lista EN+ES).
4. Botón "Ver resultados": substrings robustos "mostrar"/"resultados" además del EN
   ("apply current filters to show"); substring exacto ES sin confirmar.
5. Input de ubicación: aria-label "Ciudad, estado o código postal".
6. Modal Easy Apply: "Continuar al siguiente paso"/"Siguiente", "Revisar tu solicitud"/"Revisar",
   "Enviar solicitud".
7. Date picker "This is today" → substring robusto "hoy".
8. "Done" → "Listo".
9. Botón Easy Apply: "Solicitud sencilla".

Decisiones de diseño:
- `filter_label_translations` (dict) + `click_filter_option(driver, value, time)` en runAiBot.py:
  la config del usuario sigue en inglés; en runtime se prueban EN y ES. Valores fuera del mapa
  (incl. `salary`) se usan tal cual.
- `boolean_button_click` ahora acepta `str | list[str]`; con lista construye el xpath del `<h3>`
  como unión `( ... | ... )` de `text_xpath` por cada variante. Compatible con llamadas `str`.
- NO se tocó: login (`sign_in_button_labels`), sponsorship/bad_words, perfil de Chrome,
  `config/*.py`, pref de idioma del navegador en `open_chrome.py`.
- `label[normalize-space()='{answer}']` se dejó igual (el valor viene de config/questions.py del
  usuario, no es un label de UI de LinkedIn).
- No existe en el código ningún selector de "Save"/"Guardar" (el close del modal usa
  `data-test-modal-close-btn`, idioma-independiente), así que no hubo nada que localizar ahí.

Verificación de sintaxis ejecutada (ast.parse, exit code 0):
- `& ".venv\Scripts\python.exe" -c "import ast; ast.parse(open(r'...\runAiBot.py','r',encoding='utf-8').read())"` → `runAiBot.py OK`.
- `& ".venv\Scripts\python.exe" -c "import ast; ast.parse(open(r'...\modules\clickers_and_finders.py','r',encoding='utf-8').read())"` → `clickers_and_finders.py OK`.
- Validación de XPaths: se intentó con lxml pero no está instalado en el venv; revisión manual
  confirma que todos los selectores usan solo construcciones XPath 1.0 válidas (or/and,
  contains, translate, normalize-space, unión `|`, ancestor::).
