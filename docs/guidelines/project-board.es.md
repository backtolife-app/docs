# Panel del Proyecto en GitHub

El proyecto **BackToLife** en GitHub es el único lugar donde se rastrea todo el trabajo de los tres repositorios. Esta guía explica cómo usarlo tanto si creas issues como si eres un desarrollador que recoge tareas.

---

## Columnas del Panel

| Columna | Significado |
|---------|-------------|
| **Backlog** | Issues que existen pero aún no están priorizados ni listos para empezar |
| **Ready** | Priorizados y completamente definidos — seguros para empezar |
| **In Progress** | Alguien está trabajando activamente en ello |
| **In Review** | Hay un PR abierto esperando revisión |
| **Done** | Fusionado y cerrado |

---

## Para Quienes Crean Issues

### 1. Crear el issue

Abre un nuevo issue usando la plantilla **Standard Task**. Rellena el título, la descripción bilingüe, las especificaciones técnicas y los cinco grupos de etiquetas. Ver la guía de [Formato de Issues y PRs](issues.es.md) para las reglas completas.

### 2. Añadirlo al panel

Tras crear el issue, añádelo al proyecto **BackToLife** desde la barra lateral derecha del issue en **Projects**. Los nuevos issues aterrizan en **Backlog** por defecto.

### 3. Mover a Ready cuando esté definido

Solo mueve un issue a **Ready** cuando:
- La descripción y las especificaciones técnicas estén completas
- Todas las etiquetas (Plataforma, SO, Prioridad, Sizing, Tipo de Tarea) estén asignadas
- Esté realmente priorizado para el ciclo de trabajo actual o próximo

No dejes issues a medio definir en Ready — muévelos de vuelta a Backlog si falta algo.

---

## Para Desarrolladores

### Recoger una tarea

1. Busca en la columna **Ready** tu próxima tarea — elige por etiqueta de prioridad si no estás seguro
2. Asígnate el issue
3. Arrastra la tarjeta a **In Progress**
4. Crea la rama usando el botón **"Create a branch"** en la página del issue (ver [Contribución](contributing.es.md#nomenclatura-de-ramas))

### Abrir un PR

1. Sube tu rama y abre un Pull Request
2. En el cuerpo del PR incluye `Closes #<número-del-issue>` — esto vincula el PR al issue y lo moverá automáticamente a **Done** al fusionarse
3. Arrastra la tarjeta del issue a **In Review** (GitHub no lo hace automáticamente)
4. Solicita una revisión

### Después de fusionar

GitHub cierra automáticamente el issue vinculado y mueve la tarjeta a **Done** cuando se fusiona el PR, siempre que `Closes #<número>` esté en el cuerpo del PR.

---

## Resumen de Reglas por Columna

| Columna | Quién la mueve | Condición |
|---------|---------------|-----------|
| Backlog | Creador del issue | Issue creado |
| Ready | Creador del issue / responsable del equipo | Completamente definido y priorizado |
| In Progress | Desarrollador | Rama creada, trabajo iniciado |
| In Review | Desarrollador | PR abierto |
| Done | GitHub (automático) | PR fusionado con `Closes #` |
