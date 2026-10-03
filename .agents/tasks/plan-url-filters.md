# Plan de implementación: filtros de búsqueda por parámetros de URL

## Objetivo

Reescribir la aplicación de filtros del bot (`runAiBot.py`) para que los filtros se pasen
como **parámetros de URL** al navegar a la página de resultados, en lugar de abrir el panel
modal "Todos los filtros" y clicar opción por opción. El panel modal falla de forma inestable
(`NoSuchElementException`, `ElementClickInterceptedException`, a veces `InvalidSessionIdException`)
porque LinkedIn está retirando la "búsqueda clásica". La URL de resultados observada en runtime
ya incluía filtros aplicados por LinkedIn (`...&f_WT=2%2C1&...&sortBy=R...`), lo que confirma que
LinkedIn **acepta** filtros por parámetros de URL.

## Decisiones de diseño

1. **Ubicación: usar `location=<search_location>` en la URL, NO `set_search_location()`.**
   El input de ubicación en la UI nueva es frágil (la clase se comparte con el input de keywords,
   ya documentado en el código). LinkedIn resuelve `location=<texto>` al `geoId` correcto del lado
   servidor, igual que lo hace la caja de búsqueda. Es la vía más fiable y elimina una fuente de
   fallos. `set_search_location()` queda en el archivo pero deja de llamarse desde `apply_filters()`.

2. **Filtros avanzados que se OMITEN de la URL:** `salary` (f_SB), `industry` (f_I),
   `job_function` (f_F), `job_titles` (f_T), `companies` (f_C), el filtro `location`
   (lista secundaria), `benefits`, `commitments`, `in_your_network`, `fair_chance_employer`.
   Motivo: LinkedIn los indexa por **ID interno numérico** (p. ej. `f_I=96` para una industria),
   no por texto libre; no se pueden derivar de forma fiable del texto de config. Intentar
   construirlos produciría URLs inválidas o filtros silenciosamente ignorados. Se documenta en el
   docstring del helper.

3. **`under_10_applicants` (f_EA=true): se OMITE** por ahora. El parámetro `f_EA` no está
   confirmado contra la UI del usuario y su comportamiento varía; incluirlo sin confirmar
   arriesga una URL que LinkedIn ignore o malinterprete. Se documenta en el docstring. (Si en el
   futuro se confirma, se añade al mapa booleano del helper.)

4. **Parámetros SÍ soportados en la URL** (todos derivados de config, confirmados):
   `keywords`, `location`, `sortBy`, `f_TPR`, `f_E`, `f_JT`, `f_WT`, `f_AL`.

5. **`apply_filters()` queda casi vacío:** ya no abre el panel ni clica opciones. Solo conserva
   el diálogo `pause_after_filters` (pyautogui.confirm) y el bloque `try/except` de diagnóstico
   (captura de pantalla). Las funciones de clicado (`set_search_location`, `wait_span_click`,
   `multi_sel_noWait`, `boolean_button_click`, `click_filter_option`, el botón "Ver resultados")
   **no se borran**; simplemente dejan de llamarse desde `apply_filters()`.

## Mapeo de parámetros (texto config → código URL)

- `sortBy` (de `sort_by`): `"Most recent"→"DD"`, `"Most relevant"→"R"`.
- `f_TPR` (de `date_posted`): `"Past 24 hours"→"r86400"`, `"Past week"→"r604800"`,
  `"Past month"→"r2592000"`, `"Any time"→` (omitir, sin parámetro).
- `f_E` (de `experience_level`, multi, unido por coma): `"Internship"→"1"`, `"Entry level"→"2"`,
  `"Associate"→"3"`, `"Mid-Senior level"→"4"`, `"Director"→"5"`, `"Executive"→"6"`.
- `f_JT` (de `job_type`, multi): `"Full-time"→"F"`, `"Part-time"→"P"`, `"Contract"→"C"`,
  `"Temporary"→"T"`, `"Internship"→"I"`, `"Volunteer"→"V"`, `"Other"→"O"`.
- `f_WT` (de `on_site`, multi): `"On-site"→"1"`, `"Remote"→"2"`, `"Hybrid"→"3"`.
- `f_AL` (de `easy_apply_only`): si `True` → `"true"`; si `False` → omitir.

Reglas de construcción:
- Omitir cualquier parámetro cuyo valor derivado sea vacío/`None`.
- Multi-valores (`f_E`, `f_JT`, `f_WT`): mapear cada elemento de la lista; descartar los que no
  estén en el diccionario; unir los códigos resultantes con `,`. Si la lista queda vacía, omitir
  el parámetro.
- Codificar todo con `urllib.parse.urlencode` (que escapa espacios y comas: la coma de los
  multi-valores se codifica como `%2C`, igual que la URL real observada).

---

# Implementation Plan

- [ ] 1. Añadir el import de `urllib.parse` en la cabecera de imports de `runAiBot.py`.
      Insertar `import urllib.parse` junto a los imports de la stdlib (bloque `import os`...`import pyautogui`, ~líneas 20-25). Es prerequisito del helper del paso 2.
      Files: runAiBot.py
      Verify: `.venv\Scripts\python.exe -c "import ast; ast.parse(open('runAiBot.py', encoding='utf-8').read())"` termina sin error (exit 0).

- [ ] 2. Crear el helper `build_search_url(search_term: str) -> str` en `runAiBot.py`, justo antes de la definición de `apply_filters()` (~línea 360, tras `translate_filter_list`).
      Debe: (a) definir cuatro diccionarios de mapeo locales (`sort_by_map`, `date_posted_map`, `experience_map`, `job_type_map`, `on_site_map`) texto→código según la sección "Mapeo de parámetros"; (b) construir un dict `params` empezando por `keywords=search_term` y `location=search_location.strip()` (este último solo si no está vacío); (c) añadir `sortBy`, `f_TPR`, `f_AL` cuando el valor mapeado no sea vacío; (d) para `f_E`/`f_JT`/`f_WT`, mapear cada elemento de la lista de config, descartar los no mapeados, y unir con `,` solo si queda algún código; (e) devolver `"https://www.linkedin.com/jobs/search/?" + urllib.parse.urlencode(params)`. Docstring en español con triple comilla y tipos en la firma (estilo CONTRIBUTING.md), documentando la decisión de `location` por URL y qué filtros avanzados se omiten y por qué (ver "Decisiones de diseño" 2 y 3).
      Files: runAiBot.py
      Verify: `.venv\Scripts\python.exe -c "import ast; ast.parse(open('runAiBot.py', encoding='utf-8').read())"` termina sin error (exit 0).

- [ ] 3. En `apply_to_jobs` (línea 1318), reemplazar
      `driver.get(f"https://www.linkedin.com/jobs/search/?keywords={searchTerm}")`
      por `driver.get(build_search_url(searchTerm))`. Depende del paso 2.
      Files: runAiBot.py
      Verify: `.venv\Scripts\python.exe -c "import ast; ast.parse(open('runAiBot.py', encoding='utf-8').read())"` sin error; además `grep` manual de revisión: la línea 1318 ya no contiene `?keywords={searchTerm}` literal.

- [ ] 4. Simplificar `apply_filters()` (~línea 360): eliminar del cuerpo la apertura del panel ("All filters"/"Todos los filtros"), todas las llamadas a `click_filter_option`, `multi_sel_noWait`, `boolean_button_click`, `set_search_location`, y el clic del botón "Ver resultados". Conservar: (a) el `global pause_after_filters` y el bloque `pyautogui.confirm` tal cual; (b) el bloque `try/except` con el diagnóstico (log de URL/título + `driver.save_screenshot`) y el cierre del panel por si acaso. El cuerpo del `try` queda reducido al diálogo de pausa (y opcionalmente un `buffer`/log informando que los filtros ya van en la URL). NO borrar las funciones helper del archivo; solo dejar de invocarlas aquí.
      Files: runAiBot.py
      Verify: `.venv\Scripts\python.exe -c "import ast; ast.parse(open('runAiBot.py', encoding='utf-8').read())"` sin error (exit 0). Revisión de lógica: `apply_filters` ya no referencia `wait.until(...All filters...)` ni `multi_sel_noWait`; `pause_after_filters` y el `except` de diagnóstico siguen presentes.

- [ ] 5. Verificación final de integración estática: confirmar que las funciones helper conservadas (`set_search_location`, `wait_span_click`, `multi_sel_noWait`, `boolean_button_click`, `click_filter_option`) siguen definidas en el archivo (no deben haberse borrado) y que `build_search_url` se referencia exactamente una vez desde `apply_to_jobs`.
      Files: runAiBot.py (solo lectura/verificación)
      Verify: `.venv\Scripts\python.exe -c "import ast; m=ast.parse(open('runAiBot.py', encoding='utf-8').read()); fns=[n.name for n in ast.walk(m) if isinstance(n, ast.FunctionDef)]; print('build_search_url' in fns, 'apply_filters' in fns, 'set_search_location' in fns)"` imprime `True True True`.

---

## URL de ejemplo esperada

Para `search_term='Ingeniero de Datos'` con la config actual del usuario
(`search_location="Bogota, Colombia"`, `sort_by="Most recent"`, `date_posted="Past 24 hours"`,
`experience_level=["Entry level","Associate","Mid-Senior level"]`,
`job_type=["Full-time","Contract"]`, `on_site=["On-site","Remote","Hybrid"]`,
`easy_apply_only=True`), `build_search_url` debe producir (orden de params según inserción en el
dict; `urlencode` escapa espacios como `+` o `%20` y comas como `%2C`):

```
https://www.linkedin.com/jobs/search/?keywords=Ingeniero+de+Datos&location=Bogota%2C+Colombia&sortBy=DD&f_TPR=r86400&f_E=2%2C3%2C4&f_JT=F%2CC&f_WT=1%2C2%2C3&f_AL=true
```

Descodificado, los filtros son: keywords="Ingeniero de Datos", location="Bogota, Colombia",
ordenar por más recientes (DD), últimas 24 h (r86400), niveles Sin experiencia/Algo de
responsabilidad/Intermedio (2,3,4), jornada completa + contrato (F,C),
presencial + remoto + híbrido (1,2,3), solicitud sencilla (true).

## Notas de verificación

- No hay `pytest` en el venv; `run_tests.*` requiere pytest, así que la verificación es
  **estática** (`ast.parse`) + revisión de lógica. No se puede hacer login real (2FA del usuario).
- No tocar: lógica de login, filtrado de sponsorship/bad_words, `modules/open_chrome.py`,
  `config/*.py`, `user_config.json`.
