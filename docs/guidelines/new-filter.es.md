# Checklist de filtro nuevo

Todo lo que toca un filtro de contenido nuevo, en los cuatro repositorios y en el orden de hacerlo. Escrito a partir del último filtro añadido de punta a punta, Ocultar Vídeos Sugeridos (`youtube_suggested`, user-app #353, 16-10-2026). Copia la lista en el issue o en una nota y ve marcando: una casilla sin marcar es un sitio al que el filtro no llega sin avisar.

---

## 0. Decidir antes de escribir código

- [ ] **Nombre** tal como lo ve la persona, en EN, ES e IT, con la forma de sus hermanos (`Hide …` / `Ocultar …` / `Nascondi …`). La app del usuario pone mayúscula en cada palabra (`Ocultar Inicio`); la app de padres deja minúscula tras el verbo (`Ocultar inicio`).
- [ ] **Clave del flag** `plataforma_cosa` en snake case (`youtube_suggested`). Nunca una clave camelCase: las dos que existen (`instagram_likeComments`, `instagramDmOnly`) son contrato heredado.
- [ ] **Clave del provider** en la app del usuario, con prefijo por plataforma (`ytHideSuggested`, `igHide…`, `twHide…`, `tkHide…`), ya con prefijo, sin clave antigua que migrar.
- [ ] **Filtro de elementos o de superficie** (regla dura 10 de la app del usuario). Un filtro de elementos quita entradas de una lista que el sitio descarga: una fila de regla en el `feedItemFilter` de la plataforma tras un flag `window.__btl…`, que falla en abierto. Un filtro de superficie quita una parte de la interfaz: una hoja de estilos o un script de DOM. Nunca un `MutationObserver` nuevo ni un script de texto de etiqueta para un filtro de elementos. Excepción: Facebook no tiene paso de datos (el sitio móvil es un árbol de componentes renderizado en el servidor, nada que la página descargue como lista), así que sus filtros de unidad del feed son scripts de DOM sobre el contenedor de unidad `[data-type="vscroller"] > *`, solo en el feed, con las palabras marcadoras del propio sitio en los idiomas de la app (user-app#379).
- [ ] **Gratis o Premium** en la app del usuario. Premium añade el cableado de apagado de abajo.
- [ ] **Por defecto**: apagado hasta activarlo, activado por defecto, o fijo para todos. Fijo significa `kLockedFilterFlags` en la app de padres y una constante en el provider de la app del usuario.
- [ ] **Configuración recomendada**: en qué nivel del asistente de la app de padres va, si va (nivel siempre activo, nivel de días de clase, o fuera).
- [ ] **Captura en dispositivo primero**: abre el sitio en la app, en el emulador o en un móvil, y lee el DOM o el payload real (chrome://inspect o el socket de devtools) antes de escribir un selector o un nombre de campo. Los nombres publicados suelen estar mal.
- [ ] Sin espejo heredado: el flag va solo en `FLAGS`. `strict_shedules` se eliminó el 04-09-2026.

## 1. Backend primero, porque el despliegue ejecuta la migración

- [ ] Migración que añade `boolean('<flag>')->default(false)` a **las dos** tablas `strict_schedule_filters` y `parent_strict_schedule_filters` (modelo: `2026_10_16_000001_add_youtube_suggested_filter_column.php`).
- [ ] `app/Models/ScheduleFilters.php`: `FLAGS` (con un comentario: qué oculta, gratis o Premium, seleccionable por padres), `$fillable`, `$casts`.
- [ ] `app/Models/ParentScheduleFilters.php`: las mismas tres listas.
- [ ] `app/Http/Controllers/API/StrictSchedule/StrictScheduleController.php`: `'filters.<flag>' => 'nullable'`.
- [ ] `app/Http/Controllers/API/Parent/ParentChildController.php`: la misma línea. Un flag que falte en cualquiera de los dos controladores se descarta sin avisar.
- [ ] `app/Http/Controllers/Web/Backend/RulesController.php` `FILTER_LABELS`: `'<flag>' => 'Plataforma · Cosa'`. Da nombre al flag en la página de reglas, en el bloque de la reunión y en el informe semanal de uso; el informe acepta el flag a través de `FLAGS` sin más cambios.
- [ ] `resources/views/backend/layouts/strict/show.blade.php`: una fila en el grupo de la plataforma.
- [ ] Tests: un caso en `tests/Feature/ChildOwnConfigTest.php` y en `tests/Feature/ParentOverlayScheduleTest.php` (flag guardado, devuelto, aceptado en `always_on`). Suite completa en el contenedor: `docker compose exec laravel.test php artisan test --compact`.
- [ ] Base de datos local de desarrollo: `docker compose exec laravel.test php artisan migrate`. Nunca en producción, el pipeline migra.
- [ ] `docs/api/endpoints.md` y `endpoints.es.md`: una fila en la tabla de filtros de contenido.
- [ ] Entrada en `CHANGELOG.md`.

## 2. App del usuario

Copia al hermano más cercano (Ocultar Inicio para un filtro de superficie Premium de YouTube, Ocultar Posts con IA para un filtro de elementos): busca su flag y su clave de provider y toca cada fichero que salga.

**Script**

- [ ] `lib/features/webviews/scripts/<plataforma>_scripts.dart`. Filtro de elementos: una regla en `feedItemFilter` más los envoltorios de `FlagSeeds` (seed y at-start). Filtro de superficie: un script autoinvocado e idempotente con guarda de instalación (`window.__btl<Nombre>Installed`), una hoja de estilos puesta con `textContent` (youtube.com exige Trusted Types, así que `innerHTML` lanza error), un `MutationObserver` agrupado con `requestAnimationFrame`, y un docblock con los selectores capturados, la fecha de captura y `VERIFY ON DEVICE`.

**Estado y menú**

- [ ] `lib/features/webviews/widget/<plataforma>_provider.dart`: campo, lectura en el constructor, setter, y `resetPremiumFeatures` (condición, reinicio, escritura) si es Premium.
- [ ] `lib/features/webviews/widget/<plataforma>_toggle.dart`: `FilterToggleRow`, `PremiumFilterToggleRow` o `AlwaysOnFilterRow`, con `lock: strict.lockFor('<flag>')` y `lockReason`, más el callback `on…Toggle`.
- [ ] `lib/features/webviews/presentation/<plataforma>_screen.dart`: el campo `_strict…`; asignado desde la caché de `LocalStrictMode` (dos sitios) y desde la respuesta del backend (solo el objeto canónico `filters`, nunca el campo heredado de nivel superior); las inyecciones en vivo en los dos caminos de `_applySchedule`; la entrada en `_surfaceScripts()` (superficie) o la semilla de inicio de documento en `_userScripts()` (elementos); las tres ramas de `onLoadStop` (estricto, interruptores, siempre activo resuelto); el callback del interruptor a través de `_applyFilterToggle`.
- [ ] `lib/features/home/presentation/app_settings_screen.dart`: `row` o `premiumRow`.
- [ ] `lib/features/more/presentation/filters_features_screen.dart`: la entrada de la guía (nombre, descripción).

**Horarios y siempre activo**

- [ ] `lib/features/block/presentation/always_on_screen.dart`: `_FilterDef`.
- [ ] `lib/features/block/presentation/strict_schedule_screen.dart`: campo, lectura del mapa de caché, lectura de la respuesta, reinicio, mapa de `_saveLocalCache`, payload, `ToggleRowData`.
- [ ] `lib/features/block/data/local_strict_mode.dart`: campo, valor por defecto en el constructor, `buildAt` con `on(filters['<camel>'] == true, '<flag>')`.
- [ ] `lib/features/block/data/rx_get_schedule/rx.dart`: el `ScheduleFilters(...)` de respaldo heredado y la entrada del mapa de caché.
- [ ] `lib/features/block/data/rx_post_strick_schedule/api.dart`: campo de `ScheduleFiltersPayload` y `toJson`.
- [ ] `lib/features/block/model/get_schedule_model.dart`: `Data` y `ScheduleFilters`: campo, constructor, `fromJson`, `toJson`.

**Informe y Premium**

- [ ] `lib/features/webviews/service/filter_usage_reporter.dart`: clave de provider → flag.
- [ ] `lib/navigation_screen.dart` `_maybeReportFirstFilter`: añadirlo cuando la persona pueda activarlo, nunca uno fijo.
- [ ] `lib/features/premium/presentation/premium_shutoff_watcher.dart` `_anyToggleOn`: solo Premium.

**Textos, tests, docs**

- [ ] `lib/l10n/app_en.arb`, `app_es.arb`, `app_it.arb`: `filtersHide<Nombre>` y `filtersFeatures<Plataforma><Nombre>Desc`, y después `flutter gen-l10n`.
- [ ] Tests: `test/providers/toggle_provider_test.dart` (lista de claves, valor por defecto, persistencia, reinicio Premium), `test/features/block/local_strict_mode_test.dart` (se resuelve desde siempre activo, se queda apagado fuera de una ventana si solo está en los flags de Focus), `test/features/webviews/<plataforma>_scripts_test.dart` (marcadores, sin `innerHTML`, `node --check` sobre la cadena emitida, y el arnés de Node con formas reales de payload para un filtro de elementos).
- [ ] `docs/features/webview_filters.md` y `.es.md`; `docs/features/overview.md` y `.es.md` (la lista por plataforma).
- [ ] Entrada en `CHANGELOG.md`.
- [ ] `flutter analyze --no-fatal-infos --no-fatal-warnings` y el `flutter test --reporter compact` completo.
- [ ] Dispositivo: inyectar el script emitido por el socket de devtools en la página viva antes de la build, y después comprobar la app compilada con el interruptor activado y desactivado, dentro de una ventana de Focus, como filtro siempre activo, y tras perder Premium.

## 3. App de padres

- [ ] `lib/features/controls/model/schedule_model.dart`: `ScheduleData` y `ScheduleFilters`: campo, constructor, `fromJson`.
- [ ] `lib/features/controls/data/rx_put_schedule/api.dart`: campo de `ScheduleFiltersPayload` como `required`, para que un punto de llamada olvidado no compile, y `toJson`.
- [ ] `lib/features/rules/presentation/rule_always_on_screen.dart` (`_f`), `rule_allday_filters_screen.dart` (`_F`), `rule_strict_schedule_screen.dart` (campo, hidratación, payload, `ScheduleFilterToggleData`). Filtro fijo: `ScheduleFilterToggleData.locked` y `kLockedFilterFlags` en `lib/constants/locked_filters.dart`.
- [ ] `lib/features/settings/presentation/how_app_works_filters_screen.dart`: la fila de la guía.
- [ ] `lib/features/setup_wizard/data/pending_setup.dart`: el campo del payload de Focus.
- [ ] Configuración recomendada, cuando se decida: `lib/features/setup_wizard/data/wizard_filter_keys.dart` (lista del nivel y mapa de flags), `lib/features/setup_wizard/presentation/setup_wizard_screen.dart` (valor por defecto en `_filters` y `_filterLabel`), `test/unit/pending_setup_test.dart`.
- [ ] `lib/l10n/app_en.arb`, `app_es.arb`, `app_it.arb`: `rulesSocialHide<Nombre>` y `howWorksDesc<Plataforma><Nombre>`, y después `flutter gen-l10n`.
- [ ] Tests: `test/unit/schedule_model_test.dart` (claves del payload, parseo anidado), `test/unit/put_schedule_rx_test.dart`, `test/unit/pending_setup_test.dart`.
- [ ] `docs/features/overview.md` y `.es.md` (la tabla de `ScheduleData`), `docs/api/models.md` y `.es.md`, `docs/api/endpoints.md` (el ejemplo del PUT).
- [ ] Entrada en `CHANGELOG.md`; analyze y la suite completa.

## 4. Repositorio de docs

- [ ] `docs/database/schema.md` y `schema.es.md`: la columna en la fila de la plataforma, con la fecha.

## 5. Issue y tablero

- [ ] El issue lleva una etiqueta de plataforma por cada repositorio tocado (`backend`, `parent app`, `user app`), la etiqueta de SO, la de tipo de tarea, y Priority y Size en el tablero.
- [ ] Cuando se pida reformular el issue, reescribir título y cuerpo como una petición nueva del alcance final (gratis o Premium, valor por defecto, qué oculta, padres, informe), nunca como un informe de lo construido.

## 6. Orden de los commits

Un bloque para copiar y pegar por repositorio: backend primero, porque el despliegue migra, y después la app del usuario, la app de padres y el repositorio de docs.

## 7. Tras la siguiente build

- [ ] App del usuario: interruptor activado y desactivado, el candado dentro de una ventana de Focus, la píldora de siempre activo, la fila puesta por el padre o madre, el apagado Premium.
- [ ] App de padres: la fila en Siempre activo, Todo el día y Focus, el asistente si está incluido, y la guía.
- [ ] Admin del backend: la fila en la página strict, la etiqueta en la página de reglas y en el siguiente resumen semanal.

---

Referencia: los commits de `youtube_suggested` del 16-10-2026 en los cuatro repositorios, un filtro cableado de punta a punta.
