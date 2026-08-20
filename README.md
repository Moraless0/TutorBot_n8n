# TutorBot IA

Sistema automatizado de gestión de tutorías académicas a través de Telegram, con ayuda de n8n, Google Sheets y Gemini AI.

## ¿Qué es TutorBot?

TutorBot es un asistente virtual que permite a los estudiantes:

1. Solicitar una tutoría.
2. Consultar sus tutorías.
3. Confirmar una tutoría asignada.
4. Cancelar una tutoría.

El sistema busca un tutor disponible según la materia y la fecha, evita dobles reservas, notifica al tutor y lleva un registro de todo en Google Sheets.

## Tecnologías usadas

- **n8n:** para conectar y automatizar todo el flujo.
- **Telegram Bot:** la interfaz con la que interactúan estudiantes y tutores.
- **Google Sheets:** la base de datos del proyecto.
- **Gemini AI:** el modelo de lenguaje que entiende los mensajes y decide las acciones.

## Paso a paso de lo que se hizo

### 1. Análisis del flujo original
Se revisó el workflow existente en n8n y se detectaron errores, funciones faltantes y posibles mejoras sin reconstruir el proyecto desde cero.

### 2. Corrección de errores principales
- Se corrigió el filtro para listar tutores activos (`Activo` con mayúscula).
- Se fijó que toda tutoría registrada tenga estado `Asignada` y un ID único.
- Se hizo que el `id_estudiante` y el `telegram_user` se tomen directamente del mensaje de Telegram, sin confiar en la IA.

### 3. Protección contra dobles reservas
Se agregó un nodo que verifica si el tutor ya tiene una tutoría a la misma fecha y hora antes de registrar una nueva.

### 4. Confirmación y cancelación seguras
- Confirmar: solo cambia tutorías `Asignadas` a `Confirmadas` y verifica que pertenezcan al estudiante.
- Cancelar: solo permite cancelar tutorías activas y también verifica la propiedad.

### 5. Notificaciones al tutor
Se ajustó el aviso al tutor para que use el `telegram_tutor` correcto, no el ID interno del sistema.

### 6. Control de sesiones
Se agregaron nodos que leen y guardan automáticamente el paso del usuario, así se sabe en qué parte del flujo va cada estudiante.

### 7. Mensajes de Telegram seguros
Se configuraron los mensajes para usar `parse_mode = HTML` y se evitó que se borren guiones bajos, por ejemplo en nombres de usuario como `0_0`.


## Update: Examen 1

### Sistema de Alerta de Disponibilidad Crítica

Antes de este cambio la hoja `DISPONIBILIDAD` nunca se actualizaba: el flujo solo leía las franjas `Libre`
(`buscar_disponibilidad`) y registraba la tutoría, así que la coordinación se enteraba de que un tutor estaba
saturado únicamente cuando un estudiante intentaba agendar y no aparecía ninguna opción. Por eso este update
incluye tanto el descuento de la franja como la alerta.

La lógica se agregó como una rama propia en el canvas, disparada por cada tutoría nueva:

1. **Nueva Tutoria Registrada** (Google Sheets Trigger, `rowAdded` sobre `TUTORIAS`): se activa cuando el agente
   registra una tutoría con estado `Asignada`.
2. **Leer Datos Tutor**: busca en `TUTORES` el `id_tutor` de la tutoría para obtener su `nombre`.
3. **Leer Disponibilidad Tutor**: trae todas las franjas de ese tutor en `DISPONIBILIDAD`.
4. **Calcular Franjas Libres** (Code): calcula el `dia_semana` numérico a partir de la `fecha` de la tutoría,
   identifica la franja consumida (mismo día y misma `hora_inicio`) y cuenta cuántas franjas `Libre` quedan
   (`libres_restantes`).
5. **Franja Localizada?** (IF): si no se pudo identificar la franja, no se escribe nada en la hoja.
6. **Ocupar Franja**: aquí se descuenta la disponibilidad, cambiando el `estado` de esa franja a `Ocupado`.
7. **Disponibilidad Critica?** (IF): valida el umbral `libres_restantes <= 1`.
8. **Alerta Disponibilidad Coordinacion** (Telegram): si es crítico, avisa al chat de coordinación con el
   formato `⚠️ ALERTA DE DISPONIBILIDAD: El tutor [Nombre] solo tiene [Cantidad] franja(s) libre(s). Favor
   gestionar refuerzo.`
9. **Sin Franjas Libres?** (IF) + **Desactivar Tutor** (opcional): si `libres_restantes = 0`, el tutor pasa a
   `estado = Inactivo` en `TUTORES`, con lo cual `listar_tutores` deja de ofrecerlo hasta que la coordinación
   le cargue nuevas franjas.

Notas:

- El umbral es `<= 1` (no `< 1`), así la coordinación recibe el aviso con una franja de margen todavía disponible.
- Al marcar la franja como `Ocupado`, `buscar_disponibilidad` deja de ofrecerla y se refuerza la protección
  contra dobles reservas que ya existía.
- El Chat ID de coordinación (`8246775248`) es el mismo que ya usaba el nodo `Notificar Error`.

## Base de datos en Google Sheets

Crear una hoja de cálculo con cuatro pestañas:

### TUTORES

| id_tutor | nombre | especialidad_materias | estado | telegram_tutor |
|----------|--------|----------------------|--------|----------------|
| T001 | Ana Pérez | Matemáticas, Física | Activo | 123456789 |
| T002 | Luis Gómez | Química, Biología | Activo | 987654321 |

- `estado` puede ser `Activo` o `Inactivo`.
- `telegram_tutor` es el número de chat de Telegram del tutor.

### DISPONIBILIDAD

| id_dispo | id_tutor | dia_semana | hora_inicio | hora_fin | estado |
|----------|----------|------------|-------------|----------|--------|
| D001 | T001 | 1 | 08:00 | 10:00 | Libre |
| D002 | T002 | 3 | 14:00 | 16:00 | Libre |

- `dia_semana`: 1 = Lunes, 7 = Domingo.
- `estado`: `Libre` u `Ocupado`.

### TUTORIAS

| id_tutoria | id_estudiante | id_tutor | materia | fecha | hora | estado |
|------------|---------------|----------|---------|-------|------|--------|
| TUT-20260820-12345 | 11223344 | T001 | Matemáticas | 2026-08-20 | 08:00 | Asignada |

- `estado` puede ser: `Solicitada`, `Asignada`, `Confirmada`, `Finalizada`, `Cancelada`.

### SESSIONS

| telegram_user | pantalla_actual | paso_actual | datos_parciales |
|---------------|-----------------|-------------|-----------------|
| 11223344 | MENU | 0 | {"nombre":"Juan"} |

- Aquí se guarda por dónde va cada estudiante en la conversación.

## Credenciales necesarias

En n8n se deben configurar:

1. **Telegram API:** el token del bot creado con [@BotFather](https://t.me/BotFather).
2. **Google Sheets OAuth2:** la cuenta con acceso al documento compartido.
3. **Google Gemini (PaLM) API:** la clave de API de Google AI Studio.

## Cómo importar el workflow

1. Entra a n8n y ve a **Workflows**.
2. Selecciona **Import from file**.
3. Sube el archivo `TutorBot_IA_v4_fixed.json`.
4. Reconecta las credenciales si es necesario.
5. Activa el workflow.

## Funciones del bot

### Solicitar tutoría
1. El estudiante elige la materia.
2. Ingresa la fecha deseada.
3. El sistema busca tutores disponibles.
4. Valida que no haya doble reserva.
5. Muestra la mejor opción y pide confirmación.
6. Registra la tutoría como `Asignada` y notifica al tutor.

### Consultar tutorías
- El estudiante ve todas sus tutorías con materia, fecha, hora y estado.

### Confirmar tutoría
- El estudiante elige una tutoría `Asignada` y la cambia a `Confirmada`.

### Cancelar tutoría
- El estudiante elige una tutoría activa y la cambia a `Cancelada`.


## Autor

Henry Morales
