# Buzón de reclamos y sugerencias · Estudio de fotografía

Página única para un estudio de fotografía social (quince, bodas, embarazo y familia, fotolibros) con un formulario de reclamos y sugerencias que detecta solo las entregas fuera de plazo. Todo funciona en el navegador con `localStorage`, sin backend.

**Demo:** https://armivatech.github.io/buzon-estudio-fotos/

> «Estudio Ámbar» es un nombre de prueba. Todos los datos son sintéticos.

## Cómo correrlo en local

No necesita instalación ni dependencias.

1. Clonar el repositorio: `git clone https://github.com/ArmivaTech/buzon-estudio-fotos.git`
2. Abrir `index.html` en el navegador (doble clic).
   Si se prefiere servirlo: `npx serve .` o `python3 -m http.server` y abrir `http://localhost:8000`.

## Estructura

| Archivo | Contenido |
|---|---|
| `index.html` | Página completa: estilos, marcado y lógica en un solo archivo |
| `README.md` | Este documento |

Orden de la página: encabezado con el nombre de prueba y el botón «Dejar un mensaje», presentación en dos párrafos, grilla de los cinco servicios, «Cómo trabajamos» en tres pasos y el formulario. Debajo hay una vista «Mensajes recibidos» para revisar lo guardado durante la demo.

## Cambiar servicios, precios y plazos

Todo está en un único arreglo al principio del `<script>` de `index.html`:

```js
const SERVICIOS = [
  { id:'quince', nombre:'Fiesta de quince', incluye:'…', precio:450000, plazoDias:30 },
  …
];
```

- `id`: identificador interno. No cambiarlo si ya hay mensajes guardados con ese servicio.
- `nombre`, `incluye`: textos de la grilla y del desplegable.
- `precio`: precio de ejemplo en ARS (se formatea solo).
- `plazoDias`: plazo de entrega en días de calendario. Es el valor que usa la regla de prioridad.

La grilla, el desplegable del formulario y el cálculo de plazos se generan a partir de este arreglo.

## Datos de ejemplo

La primera vez que se abre la página (cuando la clave de `localStorage` no existe), se cargan los cinco servicios y estos cuatro mensajes, con fechas relativas al día de hoy para que el resultado sea siempre el mismo:

| ID | Tipo | Servicio | Motivo | Fecha del evento | Prioritario |
|---|---|---|---|---|---|
| RC-0001 | Reclamo | Book de quince (15 días) | Calidad de edición | hoy − 20 | No (vencido, pero el motivo no hereda la prioridad) |
| SG-0002 | Sugerencia | Boda (45 días) | — | hoy − 60 | No |
| RC-0003 | Reclamo | Fiesta de quince (30 días) | Entrega de fotos | hoy − 41 | **Sí** |
| SG-0004 | Sugerencia | Embarazo y familia (10 días) | — | hoy − 5 | No |

La numeración sigue desde 5 y es compartida entre tipos: el siguiente mensaje será `RC-0005` o `SG-0005`.

**Reemplazar los datos de ejemplo:** editar la función `datosEjemplo()` en `index.html` y borrar la clave de `localStorage` (o pulsar «Restablecer datos de ejemplo» en la vista del estudio). Para arrancar sin datos, hacer que devuelva `{ mensajes: [], siguiente: 1 }`.

## Dónde mirar los mensajes en localStorage

- **Clave:** `estudioAmbar.buzon.v1`
- **Formato:** `{ "mensajes": [ … ], "siguiente": 5 }`

Cada mensaje tiene: `id`, `tipo`, `servicio` (id del servicio), `motivo` (vacío en sugerencias), `fecha` (AAAA-MM-DD), `nombre`, `email`, `mensaje`, `creado` (fecha de alta) y `prioritario` (booleano calculado al guardar).

En el navegador: DevTools → Application (Aplicación) → Local Storage → el origen de la página → `estudioAmbar.buzon.v1`.
Por consola: `JSON.parse(localStorage.getItem('estudioAmbar.buzon.v1'))`.

## Reglas y pruebas

### Modelo de datos

```
Servicio  { id, nombre, incluye, precio, plazoDias }
Mensaje   { id, tipo: 'Reclamo'|'Sugerencia', servicio, motivo, fecha, nombre, email, mensaje, creado, prioritario }
```

### Validaciones del formulario

| Campo | Regla |
|---|---|
| Tipo | Obligatorio: Reclamo o Sugerencia |
| Servicio | Obligatorio: uno de los cinco |
| Motivo | Solo aparece en Reclamo. Obligatorio: Entrega de fotos, Calidad de edición, Fotolibro o Atención |
| Fecha | Obligatoria, no futura y no anterior a dos años desde hoy |
| Nombre | Obligatorio |
| Email | Obligatorio en Reclamo y opcional en Sugerencia. Si se completa, debe tener formato válido |
| Mensaje | Obligatorio, de 20 a 600 caracteres, con contador |

Cada error se muestra debajo de su campo.

### Regla «prioridad con límite exacto»

```
díasTranscurridos = días de calendario entre la fecha del evento y hoy
prioritario = tipo == 'Reclamo'
          && motivo == 'Entrega de fotos'
          && díasTranscurridos > plazoDias del servicio   // estrictamente mayor
```

- Se cuentan **días de calendario** (fines de semana y feriados incluidos). El cálculo usa fechas en UTC para que el cambio de horario no altere el resultado.
- **El día del límite no es prioritario:** con 30 días de plazo, 30 días transcurridos están en plazo y 31 ya son prioritarios.
- **Otros motivos no heredan la prioridad**, aunque la entrega esté vencida.
- El aviso de «Entrega fuera de plazo» se muestra antes de enviar en cuanto servicio y fecha indican atraso, y explica si el mensaje entrará como prioritario.

### Prueba guiada y resultado observado

Servicio de referencia: Fiesta de quince, con un plazo de 30 días.

| # | Entrada | Esperado | Observado |
|---|---|---|---|
| 1 | Reclamo · Entrega de fotos · fecha = hoy − 30 | Sin aviso, no prioritario | Sin aviso, guardado como RC-0005, no prioritario ✔ |
| 2 | Reclamo · Entrega de fotos · fecha = hoy − 31 | Aviso visible, prioritario | Aviso «31 días… 1 de atraso», guardado como RC-0006, prioritario ✔ |
| 3 | Reclamo · Calidad de edición · fecha = hoy − 31 | Aviso visible, **no** prioritario | Aviso con sugerencia de motivo, guardado como RC-0007, no prioritario ✔ |
| 4 | Sugerencia · fecha = hoy − 31 · sin email | Se guarda, no prioritario | Guardado como SG-0008, no prioritario ✔ |
| 5 | Fecha de mañana | Error «no puede ser futura» | Error debajo del campo ✔ |
| 6 | Fecha de hace 2 años y 1 día | Error de antigüedad | Error debajo del campo ✔ |
| 7 | Reclamo sin email | Error de email obligatorio | Error debajo del campo ✔ |
| 8 | Email `ana@correo` | Error de formato | Error debajo del campo ✔ |
| 9 | Mensaje de 19 caracteres | Error y contador en rojo | Error «llevás 19» ✔ |

Los resultados de esta tabla se observaron ejecutando la página en un navegador simulado (jsdom), partiendo de los datos de ejemplo recién cargados. La prueba de fallo de guardado también se ejecutó: el texto quedó intacto y el reintento se guardó como RC-0009, sin saltos en la numeración.

### Probar un guardado fallido sin perder lo escrito

1. Completar el formulario con datos válidos.
2. En «Mensajes recibidos», marcar **Simular fallo de guardado**.
3. Enviar. Aparece «No pudimos guardar tu mensaje. Tu texto sigue acá» y todos los campos conservan lo escrito.
4. Desmarcar la casilla y volver a enviar: se guarda con el número siguiente, sin saltos en la numeración.

Ese fallo también se da de verdad si `localStorage` no está disponible (modo privado estricto o cuota llena), porque el guardado está dentro de un `try/catch`.

## Verificación manual

- [ ] Primera carga: aparecen los cinco servicios y los cuatro mensajes de ejemplo; RC-0003 es el único prioritario.
- [ ] El botón «Dejar un mensaje» lleva al formulario.
- [ ] El motivo aparece y desaparece al cambiar entre Reclamo y Sugerencia.
- [ ] El email pasa a ser «(opcional)» en Sugerencia.
- [ ] El contador del mensaje se actualiza y se pone en rojo por debajo de 20 caracteres.
- [ ] Casos 1 a 9 de la prueba guiada.
- [ ] El guardado fallido conserva el texto.
- [ ] Al recargar la página, los mensajes siguen ahí (persistencia en `localStorage`).
- [ ] «Restablecer datos de ejemplo» vuelve al estado inicial.
- [ ] Se ve bien en móvil (≤ 820 px, una columna).

## Pendiente para producción (a cargo de Kodarvia)

Backend y persistencia real, envío de avisos al estudio, dominio, políticas legales y contenidos definitivos. La lógica de validación y la regla de prioridad están aisladas en funciones (`validar`, `estaVencido`, `esPrioritario`) para poder reutilizarlas en el servidor.
