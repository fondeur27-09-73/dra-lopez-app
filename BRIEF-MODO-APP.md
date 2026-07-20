# BRIEF — Prototipo "Dra. López" se ve como galería de marcos en el celular

Documento autocontenido para una sesión nueva. Quien lo lea no tiene contexto previo:
aquí está todo lo necesario, incluido lo que ya se probó y **falló**, para no repetirlo.

---

## 1. Qué se quiere lograr

Hay un prototipo HTML de una app móvil de pacientes para la nutricionista **Analeidy López**
(marca actual: "Dra. López", monograma AL). El dueño del proyecto quiere **enseñárselo a su
clienta desde el celular como si fuera la app ya instalada**: una pantalla a la vez, a
pantalla completa, navegando con la barra de pestañas de abajo.

**Lo que NO quiere, bajo ninguna circunstancia:** que en el celular se vean las pantallas
como tarjetitas dentro de marcos de teléfono dibujados, en una cuadrícula, con títulos y
textos explicativos alrededor. Eso se ve a "presentación de diseñador", no a producto.
Esta corrección la ha pedido muchas veces y sigue sin resolverse. Es el punto más sensible
del proyecto: cualquier respuesta que diga "listo" sin prueba real es peor que no responder.

---

## 2. Estado actual y URLs

Repo público: **`fondeur27-09-73/dra-lopez-app`**, rama `master`, servido por GitHub Pages.
Para actualizar: editar, commit, push; Pages reconstruye solo (tarda ~1–3 minutos).

| Archivo | URL | Qué hace |
|---|---|---|
| `index.html` | https://fondeur27-09-73.github.io/dra-lopez-app/ | Decide entre app y galería según el dispositivo. **Es el que sigue fallando.** |
| `app.html` | https://fondeur27-09-73.github.io/dra-lopez-app/app.html | Modo app **siempre activo, sin ninguna detección**. Añadido como salida de emergencia. Falta confirmación del usuario en su teléfono real. |

Copia local del prototipo (fuente histórica):
`C:\Users\fonde\.claude\projects\analeidy APP\prototype\nutrinova-prototype.html`

> **Advertencia sobre archivos locales:** durante el trabajo, tanto ese archivo local como
> un archivo de memoria se **revirtieron solos** a versiones viejas entre sesiones
> (posiblemente sobrescritos desde el editor abierto o por sincronización). Varias veces
> pareció que "no se había hecho nada" cuando sí estaba hecho y publicado.
> **La fuente de verdad es el repo, no el disco local.** Clonar y comparar antes de asumir.

El prototipo tiene 34 pantallas, es autocontenido (~332 KB, dos videos incrustados en
base64) y no depende de red. Paleta: salvia `#35594A` sobre blanco cálido `#FAF9F6`,
azul `#4E7E96` solo para tratamiento/medicación. Datos demo ficticios (paciente Mariel Peña).

---

## 3. Cómo está construido (importante para entender el bug)

El HTML original fue diseñado como **presentación de diseño**: 34 bloques
`<figure class="ph">`, cada uno con un `.ph-shell` (el marco del teléfono, ancho fijo
**302px**), dentro un `.ph-scr` (la pantalla, alto 632px) y debajo un `<figcaption>` con
el título y la explicación. Todo eso dentro de `<section class="sec">` con encabezados `<h2>`.

Encima de esa estructura se añadió un "modo app" al final del archivo, en dos bloques:
`<style id="app-mode-css">` y `<script id="app-mode-js">`. La idea: en móvil, ocultar todos
los `figure` menos uno, ponerlo `position:fixed; inset:0`, ocultar marcos y figcaptions, y
escalar el lienzo de 302px con `transform: scale(innerWidth/302)` para llenar la pantalla.
El JS maneja la navegación real (índice de 34 pantallas, pila de navegación, tabs, botones
atrás, `popstate`).

**La navegación funciona bien y está verificada.** El problema es exclusivamente
*cuándo se activa el modo app*.

---

## 4. Qué se intentó y qué pasó (no repetir)

### Intento 1 — detección por ancho
```css
@media (max-width: 820px) { /* modo app */ }
```
Activado desde JS con `matchMedia('(max-width: 820px)')` que añadía una clase al `<html>`.

**Falló por dos motivos comprobados:**

1. **"Solicitar versión de escritorio".** Safari y Chrome guardan ese ajuste **por sitio**.
   Con él activo el viewport pasa a ~980px → la media query no dispara → aparecen los 34
   marcos. Reproducido: viewport 980px = 34 figuras visibles.
2. **Dependencia total del JavaScript.** El modo app se activaba añadiendo una clase desde
   JS. Si el JS fallaba o tardaba, quedaba visible la galería completa. Reproducido con
   `javaScriptEnabled: false` = 34 figuras visibles.

### Intento 2 — detección por puntero táctil + CSS sin JS
```css
@media (max-width:820px), (pointer:coarse) and (hover:none) { /* modo app */ }
```
Más un respaldo en CSS puro (`html:not(.js-app) section.sec:first-of-type figure.ph:first-child`)
para que, aunque el JS no corra, se vea la primera pantalla a pantalla completa. El JS solo
añade `html.js-app` cuando ya montó una pantalla, más el escalado fino y la navegación.
También se limitó el escalado por alto en pantallas anchas para no recortar.

**Verificado con Playwright contra la URL pública**, todo correcto:

| Escenario | Resultado |
|---|---|
| iPhone 390×844 | 1 pantalla, ocupa 390×844 |
| Modo escritorio 980×2120 | 1 pantalla, ocupa 980×2120 |
| Teléfono chico 360×640 | 1 pantalla, ocupa 345×640 |
| iPad 820×1180 | 1 pantalla, centrada |
| Horizontal 844×390 | 1 pantalla, centrada |
| Sin JavaScript | 1 pantalla, ocupa 390×844 |
| Escritorio 1400×900 con mouse | 34 marcos (correcto, es lo deseado ahí) |

**Y aun así el usuario sigue viendo la galería de marcos en su teléfono, también en
incógnito.** Ahí está el bug sin resolver.

---

## 5. Hipótesis vivas, en orden de probabilidad

1. **iOS Safari en "modo escritorio" también miente en las media features.** Cuando se
   solicita versión de escritorio, iOS suplanta un navegador de macOS; es plausible que
   además de ensanchar el viewport a 980px reporte `hover: hover` y `pointer: fine`.
   Si es así, **fallan las dos condiciones a la vez** y se cae a la galería — encaja
   exactamente con lo que describe el usuario. **La emulación de Chromium con
   `hasTouch:true` SÍ reporta `pointer:coarse`, por eso las pruebas pasaron y el teléfono
   real falla.** Esta es la sospecha principal y explica por qué las pruebas engañaron.
2. **El usuario está abriendo una URL distinta a la que se está arreglando**: un enlace
   viejo guardado, la vista previa de un artifact anterior, o el archivo local. Hay un
   artifact antiguo del proyecto dando vueltas. Conviene confirmar **exactamente** qué
   dirección aparece en su barra de direcciones.
3. **CDN/proxy intermedio sirviendo HTML viejo** pese al incógnito (menos probable:
   `curl` con parámetro aleatorio ya devuelve la versión nueva).

---

## 6. Lo único que falta para cerrar el caso

**Saber qué reporta el teléfono real del usuario.** Nada de esto se resuelve adivinando ni
emulando: la emulación ya demostró que engaña. Publicar una página de diagnóstico mínima
y pedirle una captura resuelve la duda en 5 segundos:

```html
<meta name="viewport" content="width=device-width,initial-scale=1">
<body style="font:16px/1.6 system-ui;padding:24px">
<h2>Diagnóstico</h2>
<pre id="o"></pre>
<script>
o.textContent =
 'innerWidth: '+innerWidth+'\n'+
 'screen.width: '+screen.width+'\n'+
 'devicePixelRatio: '+devicePixelRatio+'\n'+
 'pointer coarse: '+matchMedia('(pointer:coarse)').matches+'\n'+
 'hover none: '+matchMedia('(hover:none)').matches+'\n'+
 'max-width 820: '+matchMedia('(max-width:820px)').matches+'\n'+
 'maxTouchPoints: '+navigator.maxTouchPoints+'\n'+
 'UA: '+navigator.userAgent;
</script>
```

---

## 7. Recomendación

**Dejar de detectar el dispositivo.** La detección es la única fuente del bug y ya falló dos
veces por razones distintas. La estructura correcta es separar los dos productos:

- **`app.html` — solo la app.** Modo app incondicional (`@media all`), sin ninguna rama de
  código que pueda renderizar marcos. Ya está creado y publicado como salida de emergencia
  (commit `519f721`); verificado: muestra **1 sola pantalla incluso en escritorio 1400×900
  con mouse y sin JavaScript**. **Este es el enlace que debe usar el usuario para su
  clienta.**
- **`index.html` — solo la galería** para revisar diseño en computadora, sin lógica de
  modo app.

Si además se quiere que la raíz del sitio funcione en el celular, lo robusto **no** es
detectar mejor, sino **invertir el valor por defecto**: que el modo app sea lo predeterminado
y la galería solo aparezca bajo una condición explícita y estrecha. Así, si la detección
vuelve a fallar, el usuario cae en la app (aceptable) y nunca en la galería (inaceptable).

Tareas concretas sugeridas:
1. Confirmar con el usuario **qué URL exacta** abre y qué ve en `app.html`.
2. Publicar el diagnóstico de arriba y pedir captura.
3. Según el resultado, dejar `index.html` como galería pura e `app.html` como la app, y
   entregarle un solo enlace.

---

## 8. Verificación obligatoria antes de decir "listo"

Hay Playwright instalado en `C:\Users\fonde\playwright-biu` (`node_modules/playwright`,
Chromium disponible). Probar **contra la URL pública**, no contra el archivo local, en:
390×844, 980×2120, 360×640, 820×1180, 844×390, con `javaScriptEnabled:false`, y 1400×900.

Criterio de aprobación en móvil: **exactamente 1 elemento `figure.ph` visible**, su
`.ph-shell` ocupando el viewport completo, y **0 elementos `figcaption` visibles**.

Y aun con todo eso en verde: **la emulación ya engañó una vez**. La confirmación que vale
es una captura del teléfono del usuario. No declarar el problema resuelto sin ella.

---

## 9. Nota sobre el trato con el usuario

Está muy molesto, con razón: se le dijo "listo, verificado" dos veces y el problema seguía
ahí, porque la verificación no cubría su caso real. No prometer, no adornar: mostrar la
evidencia, decir explícitamente qué **no** se probó, y pedir la captura del teléfono antes
de dar nada por cerrado.
