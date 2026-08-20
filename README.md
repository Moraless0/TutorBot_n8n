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


## Update: Examen 1

### Sistema de Alerta de Disponibilidad Crítica

Se agregó una rama automática de post-asignación que descuenta la disponibilidad del tutor y alerta a la coordinación cuando quedan 1 o menos franjas libres.

**Nodos nuevos en el workflow (`Ver_Final_Examen1.json`)**

1. **actualizar_disponibilidad** — Marca la franja seleccionada como `Ocupado` en la hoja `DISPONIBILIDAD` justo después de `registrar_tutoria`.
2. **contar_franjas_libres** — Consulta las franjas `Libre` restantes del mismo tutor.
3. **preparar_alerta** — Cuenta las franjas libres y construye el mensaje de alerta.
4. **IF Disponibilidad Crítica** — Verifica si la cantidad de franjas libres es `<= 1`.
5. **alertar_coordinacion** — Envía el mensaje de Telegram al Chat ID de coordinación configurado.
6. **IF Desactivar Tutor** — Verifica si la cantidad de franjas libres es exactamente `0`.
7. **desactivar_tutor** — Cambia el estado del tutor a `Inactivo` en la hoja `TUTORES`.

**Lógica del flujo**

- Tras una asignación exitosa, el agente llama a `actualizar_disponibilidad` con `id_dispo` e `id_tutor`.
- Luego llama a `contar_franjas_libres` con el mismo `id_tutor` y cuenta cuántas filas `Libre` quedan.
- Si el resultado es `<= 1`, el agente ejecuta `alertar_coordinacion` con: `⚠️ ALERTA DE DISPONIBILIDAD: El tutor [Nombre] solo tiene [Cantidad] franja(s) libre(s). Favor gestionar refuerzo.`
- Si el resultado es `0`, el agente ejecuta `desactivar_tutor` y cambia el estado del tutor a `Inactivo`.
- Los nodos `IF Disponibilidad Crítica` e `IF Desactivar Tutor` quedan en el canvas como evidencia visual de la validación de umbral solicitada en el examen.

**Ajustes necesarios antes de probar**

- En el nodo `alertar_coordinacion` reemplaza el `chatId` `8246775248` por el Chat ID real de la coordinación.
- La alerta usa el **nombre** del tutor obtenido por `listar_tutores`; si no está disponible, usa el `id_tutor`.

## Autor

Henry Morales
