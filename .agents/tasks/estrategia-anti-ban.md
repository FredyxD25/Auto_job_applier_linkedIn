# Estrategia anti-ban de LinkedIn

Resumen de cómo LinkedIn detecta la automatización y qué hacemos en este bot para reducir el
riesgo. Fuentes (contenido parafraseado para cumplir con las licencias):
- https://kakiyo.com/blog/is-linkedin-automation-safe
- https://blog.closelyhq.com/linkedin-detect-automation-stay-under-radar/
- https://www.outx.ai/blog/linkedin-automation-safety-guide-best-practices-2026
- https://www.hyperclapper.com/blog-posts/generate-leads-on-linkedin-without-getting-banned-in-2026

## Cómo detecta LinkedIn un bot

1. **Análisis de comportamiento ("Activity DNA").** No miran solo el volumen, sino los
   patrones. Lo que más delata: velocidad y UNIFORMIDAD. Tiempos idénticos entre clics,
   navegación robótica sin pausas, cero scroll "humano".
2. **Fingerprinting del navegador.** Detectan navegadores headless y automatizados.
3. **Monitoreo de IP y picos de actividad.** Ráfagas súbitas disparan el riesgo.
4. **Puntuación acumulativa de riesgo.** Es la suma de señales a lo largo del tiempo, no un
   único evento. Si llega una restricción, hay que PARAR de inmediato.

## Qué ya hace este bot a favor nuestro

- Usa `undetected_chromedriver`, no Selenium plano (menos fingerprintable).
- NO usa `--headless` (headless es trivialmente detectable). Corre un Chrome real.
- `human_type()` escribe carácter por carácter con jitter (~40-180ms) y pausas de "pensamiento".
- `buffer()` usa rangos ALEATORIOS, y ahora suma un jitter extra (0-250ms) para que no haya
  dos pausas idénticas (rompe la uniformidad, que es la señal #1).
- Perfil de Chrome persistente y dedicado (sesión estable, no re-login constante).

## Límites recomendados (conservadores, para una cuenta real)

Basado en las fuentes para 2025-2026 (son para outreach, pero el principio de "poco volumen y
ritmo humano" aplica igual a aplicaciones):

- **Aplicaciones Easy Apply**: LinkedIn ya limita de hecho a ~25/día. No forzar más.
- **Warm-up**: las primeras corridas, poco volumen (p. ej. `switch_number` bajo). Subir gradual.
- **Ritmo**: `click_gap = 2` o más (nunca 0). Pausas variadas, nunca fijas.
- **Horario**: correr en horas laborales normales, no de madrugada ni 24/7.
- **Actividad manual en paralelo**: seguir usando LinkedIn a mano (ver feed, etc.).
- **Una restricción = parar.** Si LinkedIn muestra un aviso de restricción o checkpoint, detener
  el bot inmediatamente y no reintentar en días. Apelar si corresponde.

## Config actual del usuario (user_config.json) relevante al riesgo

- `click_gap = 2` → bien (pausa entre acciones).
- `switch_number = 15` → razonable; bajar a 5-8 en el warm-up inicial si se quiere ser cauto.
- `run_in_background = false` → bien (Chrome visible, no headless).
- `safe_mode = true` → perfil dedicado persistente.

## Pendiente / mejoras futuras posibles (no implementadas aún)

- Scroll aleatorio ocasional en la lista de resultados (más "humano").
- Pausa larga ocasional entre trabajos (simular distracción).
- Tope diario de aplicaciones configurable con corte automático.
- Detección de página de restricción/checkpoint para abortar y avisar.

Estas se implementarán DESPUÉS de que el flujo completo (buscar + filtrar + aplicar) funcione de
principio a fin, porque un bot que falla y reintenta de forma errática es MÁS sospechoso que uno
que funciona limpio.
