# Heura Accesos — Contexto para Claude Code

## Estado actual
- Hecho: `.gitignore` (excluye todos los CSV salvo `usuarios_sinteticos.csv`).
- Hecho: propuesta de diseño en `DISENO.md` (modelo de datos, estructura, cifrado, búsqueda, pantallas).
- **Pendiente:** que la persona responsable valide `DISENO.md` y responda a sus "Dudas abiertas"
  (o acepte los valores por defecto). **No empezar a programar hasta esa validación.**

## Reglas del proyecto
- Idioma de la app, del README y de las conversaciones: castellano.
- **Nunca** añadir, leer ni subir CSV con datos reales. Solo se usa `usuarios_sinteticos.csv` (datos inventados).
- Ninguna petición de red en la app salvo la carga de su propio código: nada de CDNs, analíticas,
  fuentes remotas ni APIs externas.
- Mantener la capa de almacenamiento tras la interfaz `Repositorio` descrita en `DISENO.md`.
- Todos los textos de la interfaz centralizados en el objeto `TEXTOS`.

---

## Encargo original (completo)

### Contexto
Eres un desarrollador senior. Vas a construir el MVP de una app de registro de
accesos para una pequeña asociación de Barcelona que atiende a personas sin hogar.
Ofrece dos servicios con control de entrada: MERIENDA y DUCHA. Hoy no hay
registro fiable de quién usa cada servicio. La asociación gestiona a sus usuarios
en un CRM (ClinicCloud) del que exporta un CSV.

La app la usarán voluntarios, no técnicos, en la puerta, a menudo con cola y
prisa. Prioridad absoluta: registrar a una persona en menos de 10 segundos.

### Arquitectura y requisitos técnicos
- Aplicación 100% cliente: un único fichero HTML autocontenido (HTML + CSS + JS)
  más los ficheros mínimos para PWA (manifest.json y service worker). Sin backend.
- Se servirá desde GitHub Pages por HTTPS. En el servidor solo está el código;
  los datos NUNCA salen del dispositivo.
- Ninguna petición de red salvo la carga inicial del propio código: nada de
  CDNs, analíticas, fuentes remotas ni APIs externas. Todas las librerías, si las
  hay, van embebidas.
- PWA instalable y funcional offline una vez instalada. Debe funcionar en
  Chrome de PC y de tablet Android.
- Persistencia en IndexedDB. Al primer arranque, solicitar almacenamiento
  persistente con navigator.storage.persist() y mostrar en ajustes si se ha concedido.
- Interfaz táctil: botones grandes, texto legible, sin gestos complicados.
- Idioma: castellano, con todos los textos centralizados en un objeto para
  poder traducirlos al catalán después.

### Datos de entrada
CSV separado por punto y coma, con la misma estructura que el export de
ClinicCloud (subconjunto de columnas):
IDCONTACTO;ESTADO;NOMBRE;APELLIDOS;SEXO;"F. NACIMIENTO";DNI;MOTIVO

- El fichero se carga con un selector de archivos y se procesa en local; nunca
  se sube a ningún sitio.
- Codificación: aceptar UTF-8 y Windows-1252.
- F. NACIMIENTO en formato dd/mm/aaaa.
- ESTADO: "Alta" o "Baja".
- MOTIVO es texto libre. Si contiene "volunt" (sin distinguir mayúsculas ni
  acentos), la persona es voluntaria y NO se importa como usuaria.
- Ignorar cualquier columna adicional que traiga el fichero (minimización).
- La importación sustituye la lista maestra, pero NUNCA borra los fichajes ya
  registrados, que se vinculan por IDCONTACTO.
- Mostrar un resumen tras importar: usuarios cargados, voluntarios excluidos,
  filas con errores.

Genera también un fichero de prueba "usuarios_sinteticos.csv" con 100 personas
FICTICIAS con esa estructura, que incluya deliberadamente casos difíciles:
- Nombres de orígenes diversos (marroquí, argelino, senegalés, rumano,
  pakistaní, latinoamericano, español, catalán) con variantes de transcripción
  (Mohamed / Mohammed / Muhammad; Youssef / Yusuf).
- Apellidos compuestos, partículas (de, del, El, Ben, Ould) y acentos.
- Dos o tres pares de personas con el mismo nombre y apellidos pero distinta
  fecha de nacimiento.
- DNI, NIE (X/Y/Z), pasaportes y unos 15 registros sin documento.
- Unos 10 registros en estado "Baja" y 8 voluntarios (MOTIVO "Voluntari ...").
Todos los datos deben ser inventados.

### Flujo principal
1. Pantalla de inicio: elegir servicio activo (MERIENDA / DUCHA), que queda
   fijado hasta que se cambie, y nombre del voluntario que opera (opcional,
   recordado en el dispositivo).
2. Búsqueda: un único campo de texto. Búsqueda en vivo desde el segundo
   carácter, que debe ser:
   - insensible a mayúsculas, acentos, guiones y orden de las palabras
     ("garcia maria" encuentra "María García");
   - tolerante a errores tipográficos (distancia de edición o similitud por
     trigramas) para nombres difíciles;
   - capaz de buscar también por fragmento de documento.
   Resultados: máximo 8, ordenados por similitud, mostrando nombre completo,
   año de nacimiento y SOLO los 3 últimos caracteres del documento (nunca el
   documento completo en pantalla).
3. Fichaje: al tocar un resultado, confirmar con un botón grande. Se guarda
   IDCONTACTO, servicio, fecha y hora automáticas y voluntario. Mostrar
   confirmación visual clara durante 2 segundos y volver a la búsqueda.
   - Si la persona ya fichó en ese servicio ese día, avisar y pedir
     confirmación (no bloquear).
   - Si su ESTADO es "Baja", registrar igualmente y marcar el fichaje como
     "revisar estado". Nunca se niega el acceso por el estado de la ficha.
4. Persona no encontrada: botón "No está en la lista". Formulario mínimo:
   nombre y apellidos (obligatorios), sexo, fecha de nacimiento y documento
   (opcionales) y observaciones.
   - ANTES de guardar, mostrar las personas existentes más parecidas
     ("¿Es alguna de estas?") para evitar duplicados.
   - Se guarda como "registro temporal – pendiente de alta en ClinicCloud",
     con un ID provisional (TMP-0001...), y se ficha la entrada en el mismo paso.
   - Si esa persona vuelve antes de estar en ClinicCloud, debe aparecer en la
     búsqueda (marcada como temporal).

### Consulta y gestión
- Tabla de fichajes con filtros por rango de fechas, servicio y persona.
  Totales por día y servicio.
- Exportar fichajes a CSV (con IDCONTACTO o ID provisional).
- Lista de registros temporales pendientes, exportable a CSV para darlos de
  alta en ClinicCloud.
- Conciliación: al importar un nuevo CSV maestro, proponer coincidencias entre
  registros temporales y personas nuevas del fichero (por nombre + fecha de
  nacimiento). El voluntario confirma la fusión y los fichajes temporales pasan
  al IDCONTACTO definitivo.

### Seguridad y protección de datos
- Cifrado en reposo: todos los datos personales y fichajes se guardan cifrados
  en IndexedDB con AES-GCM (Web Crypto API). La clave se deriva del PIN de
  acceso con PBKDF2 (salt aleatorio guardado en local, mínimo 200.000
  iteraciones). La clave solo existe en memoria mientras la app está desbloqueada.
- PIN de acceso (mínimo 6 dígitos) obligatorio al abrir. Bloqueo automático
  tras 5 minutos sin uso, borrando la clave de memoria.
- Tras 10 intentos fallidos de PIN, retardo progresivo entre intentos.
- PIN de coordinación adicional para importar, exportar, consultar el
  histórico, cambiar PINes y borrar datos.
- Si se olvida el PIN, los datos son irrecuperables: dejarlo claro en la
  configuración inicial y en el README.
- Opción de borrar todos los datos del dispositivo, con doble confirmación.

### Copias de seguridad
- Exportar copia completa (usuarios, temporales y fichajes) en un único fichero
  cifrado con el PIN de coordinación, e importarla en otro dispositivo o tras reinstalar.
- Recordatorio visible en la pantalla de inicio si han pasado más de 7 días
  desde la última copia.
- La app debe avisar claramente de que borrar los datos de Chrome o desinstalar
  la app elimina toda la información no copiada.

### Entregables
1. index.html, manifest.json y service worker.
2. usuarios_sinteticos.csv.
3. README breve en castellano para la persona de la asociación que la
   mantendrá: cómo publicarla en GitHub Pages paso a paso, cómo instalarla en la
   tablet, cómo importar el CSV, cómo hacer y restaurar copias, qué pasa si se
   olvida el PIN y qué hacer si algo falla.

### Fuera de alcance del MVP
Sincronización entre dispositivos, lectura de documentos con cámara, integración
automática con ClinicCloud, roles de usuario avanzados, estadísticas complejas.
Diseña el modelo de datos separando la capa de almacenamiento (una interfaz
única tipo repository) para poder sustituir IndexedDB por Google Sheets vía
Apps Script en el futuro sin rehacer la lógica de la app.
