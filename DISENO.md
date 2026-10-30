# Heura Accesos — Propuesta de diseño (pendiente de validar)

> Documento de diseño previo al desarrollo del MVP. **No se ha escrito código todavía.**
> Las dudas del final deben resolverse (o aceptarse los valores por defecto) antes de programar.

## 1. Ficheros entregables

```
index.html            App completa (HTML + CSS + JS, sin dependencias externas)
manifest.json         PWA
sw.js                 Service worker: guarda la app para uso offline y avisa de versiones nuevas
icon-192.png, icon-512.png   Iconos (Chrome los necesita en PNG para permitir instalar)
usuarios_sinteticos.csv
README.md
```

## 2. Organización del código dentro de `index.html`

El código va en capas. Cada una solo habla con la de debajo:

| Capa | Qué hace |
|---|---|
| `TEXTOS` | Todos los textos de la interfaz en un objeto `{ es: {...} }`. Así el catalán solo será añadir `ca` |
| `Util` | Normalizar texto (quitar acentos, mayúsculas y guiones), fechas, CSV con `;` y comillas, detectar UTF-8 o Windows-1252 |
| `Cripto` | PBKDF2 (210.000 iteraciones), AES-GCM, protección de claves, cifrado de copias |
| `Repositorio` | **Interfaz única de almacenamiento.** Implementación actual: `RepositorioIndexedDB` |
| `Servicios` | Importar, buscar, fichar, temporales, conciliar, copias, sesión y bloqueo |
| `UI` | Pantallas y componentes táctiles |

### Interfaz `Repositorio`

Todo asíncrono y en términos del dominio, sin nada propio de IndexedDB:

```js
abrir(clave) / cerrar()
leerMeta(k) / guardarMeta(k, v)
personas.todas() / personas.sustituirMaestra(lista)
temporales.todos() / .guardar(t) / .siguienteId()
fichajes.anadir(f) / .consultar({desde, hasta, servicio, personaId}) / .reasignar(idTmp, idDef)
exportarTodo() / importarTodo(datos) / borrarTodo()
```

Un futuro `RepositorioSheets` implementaría lo mismo llamando a Apps Script. El cifrado va dentro de la implementación de IndexedDB, no en la lógica de la app.

## 3. Modelo de datos

```js
Persona   { id: "12345", estado: "Alta"|"Baja", nombre, apellidos, sexo,
            fechaNac: "1987-03-14"|null, documento: "X1234567L"|"",
            claveBusqueda (precalculada, solo en memoria) }

Temporal  { id: "TMP-0001", nombre, apellidos, sexo, fechaNac, documento,
            observaciones, creado: ISO, creadoPor,
            estado: "pendiente"|"fusionado", fusionadoCon: "12345"|null }

Fichaje   { id: uuid, personaId: "12345"|"TMP-0001", servicio: "MERIENDA"|"DUCHA",
            fechaHora: ISO, dia: "2026-10-30" (hora local), voluntario,
            revisarEstado: bool, repetidoConfirmado: bool,
            idOriginal: "TMP-0001"|null   // si venía de un temporal ya fusionado
          }

Meta (sin cifrar): salt, iteraciones, claves protegidas, contador de intentos fallidos,
                   fecha de última copia, si el almacenamiento persistente está concedido
```

### Almacenamiento

- **IndexedDB:** almacenes `personas`, `temporales`, `fichajes` y `meta`.
- Cada registro se guarda como `{id, iv, datos cifrados}`.
- Al desbloquear, todo se descifra a memoria. Con unos cientos de personas la búsqueda es instantánea.
- Al bloquear (o tras 5 minutos sin uso), se borran la clave y los datos de memoria.

### Claves

- Se genera una **clave de datos** aleatoria (AES-256).
- Esa clave se guarda protegida con otra derivada del **PIN de acceso** (PBKDF2, salt aleatorio en local).
- También se guarda, por separado, protegida con otra derivada del **PIN de coordinación**.
- Ventajas: cambiar un PIN no obliga a volver a cifrar todos los datos, y la clave solo vive en memoria.
- A partir del intento fallido 10, espera creciente entre intentos.

### Copia de seguridad

- Fichero `.heura` (JSON) con `{formato, versión, salt, iteraciones, iv, datos cifrados}`.
- Se cifra con una clave derivada **del PIN de coordinación**, así se restaura en otro dispositivo solo con ese PIN.

## 4. Búsqueda

- Cada persona tiene sus palabras normalizadas: sin acentos, minúsculas, guiones como espacios, `ç→c`, `ñ→n`.
- Hay además una forma "fonética" sencilla para las variantes de transcripción: letras dobles reducidas a una, `ou→u`, `y→i`, `ph→f`, `kh→j`… Así `Mohammed`, `Mohamed` y `Muhammad` quedan muy cerca, y `Youssef` y `Yusuf` también.
- Puntuación: para cada palabra buscada se toma la mejor coincidencia entre las palabras de la persona, sin importar el orden. Coincidencia exacta > empieza por > contiene > parecido por distancia de edición o trigramas. Se suman las palabras y se ordena.
- Si lo que se busca tiene dígitos, también se busca dentro del documento normalizado.
- Máximo 8 resultados.
- Las fichas temporales entran en la búsqueda con la etiqueta "Temporal". Las de Baja aparecen con una marca discreta.
- En pantalla solo se ve `···67L` del documento.

## 5. Pantallas

1. **Configuración inicial:** crear PIN de acceso y PIN de coordinación, con un aviso destacado de que si se olvidan los datos se pierden. Después se pide almacenamiento persistente.
2. **Desbloqueo:** teclado numérico grande.
3. **Inicio:** botones grandes MERIENDA / DUCHA, nombre del voluntario, aviso si la copia tiene más de 7 días, aviso permanente de que borrar los datos de Chrome lo elimina todo.
4. **Buscar y fichar:** campo de búsqueda con el foco puesto y botón grande "Registrar". Confirmación verde durante 2 s, o aviso naranja si ya fichó hoy o está de Baja. Botón "No está en la lista".
5. **Alta temporal:** formulario, luego "¿Es alguna de estas?", luego guardar y fichar en el mismo paso.
6. **Coordinación** (pide el PIN): Importar CSV (con resumen y conciliación), Fichajes (filtros, totales y exportar), Temporales pendientes (exportar), Copias, Ajustes (cambiar PINes, estado del almacenamiento persistente, borrar todo).

---

## Dudas abiertas (con el valor por defecto propuesto)

1. **Alcance del PIN de coordinación.** En una app sin servidor, alguien con el PIN de acceso y conocimientos técnicos podría leer el histórico desde las herramientas de Chrome, porque la misma clave descifra todo.
   - **Opción A (recomendada para el MVP):** el PIN de coordinación controla la interfaz y cifra las copias.
   - **Opción B:** separación real. Los fichajes se guardan con una clave pública: el voluntario puede escribir pero no leer el histórico. Solo los del día quedan accesibles, para el aviso de "ya fichó". Es más segura pero más compleja.
2. **Personas que desaparecen del CSV nuevo.** Por defecto: guardar un registro archivado (solo nombre e ID) para que sus fichajes antiguos sigan mostrando un nombre. Si no, en el histórico solo se vería el ID.
3. **Restaurar una copia.** Por defecto: **sustituir** todo lo que hay en el dispositivo, con doble confirmación, porque solo hay un dispositivo.
4. **Corregir errores.** Por defecto: botón "Deshacer" durante los 2 s de confirmación, y opción de anular un fichaje desde coordinación. Los anulados quedan marcados, no borrados.
5. **Exportar fichajes.** Por defecto, por minimizar datos: solo ID, servicio, fecha, hora, voluntario y avisos, más una columna opcional con el nombre.
6. **Sexo.** ¿Qué valores trae ClinicCloud exactamente (`Hombre`/`Mujer`, `H`/`M`, otros)? Se guarda tal cual, pero el formulario necesita opciones.
7. **Volumen aproximado.** ¿Cuántos fichajes al día y cuántas personas en el CSV? Por debajo de unos 100.000 fichajes al año, descifrarlo todo en memoria va bien. Por encima, se agruparían por mes.
8. **Conservación de datos.** ¿Hay que borrar automáticamente los fichajes de más de X meses? Por defecto: no, de momento, pero conviene definirlo por protección de datos.
9. **Varios dispositivos.** Los IDs `TMP-0001` chocarían entre tablets. ¿Prefijo de dispositivo ya (`TMP-A-0001`) o más adelante? Por defecto: más adelante.
10. **Nombre de la app e icono.** Por defecto: "Heura Accesos", sin colores corporativos definidos.
11. **Fin del día.** Por defecto: el día cuenta según la hora local del dispositivo, a medianoche.
