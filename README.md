# Bibliografía Zotero: espejo local y respaldo en GitHub

Este directorio `/mnt/Compartida/bibliografia` funciona como **espejo local** de la carpeta activa de Zotero ubicada en `/home/jjlealg/Zotero`. Zotero guarda su biblioteca local en la carpeta de datos que contiene `zotero.sqlite` y `storage`, por lo que el respaldo útil debe incluir ambos. [web:10]

## Estructura de este flujo

Hay dos operaciones distintas y conviene no mezclarlas:

1. **Monitoreo / espejo local**: copiar cambios desde `/home/jjlealg/Zotero` hacia `/mnt/Compartida/bibliografia` apenas cambien archivos.
2. **Respaldo a GitHub**: registrar cambios en Git y subirlos a la nube con `git push`.

`rsync` sirve para mantener un espejo incremental del contenido, y la barra `/` al final del origen hace que copie el contenido de la carpeta, no la carpeta contenedora como subdirectorio extra. [web:1][web:24]

## Rutas usadas en este equipo

- Origen activo de Zotero: `/home/jjlealg/Zotero`
- Espejo local y repo Git: `/mnt/Compartida/bibliografia`

## Requisitos

Verifica que existan estas herramientas:

```bash
which rsync git inotifywait
```

Si `inotifywait` no existe, en Arch se instala con `inotify-tools`. `inotifywait` puede vigilar cambios recursivos como escritura cerrada, creación, borrado y movimientos, que es justo lo que necesitamos para disparar `rsync`. [web:36][web:4]

```bash
sudo pacman -S inotify-tools
```

## 1) Verificar que la base activa exista

Antes de cualquier copia, confirma que la base activa de Zotero está en el origen esperado:

```bash
find /home/jjlealg/Zotero -maxdepth 1 -type f -name zotero.sqlite
```

También puedes revisar que exista el directorio de adjuntos:

```bash
find /home/jjlealg/Zotero -maxdepth 1 -type d -name storage
```

La carpeta de datos de Zotero relevante para backup contiene la base SQLite y el contenido de `storage`. [web:10]

## 2) Hacer copia espejo manual

Este comando actualiza el espejo local completo desde Zotero hacia la partición compartida:

```bash
rsync -a --delete /home/jjlealg/Zotero/ /mnt/Compartida/bibliografia/
```

- `-a`: preserva estructura, tiempos y permisos.
- `--delete`: elimina del destino lo que ya no exista en el origen.
- La `/` final en `Zotero/` importa: copia el contenido dentro de `bibliografia/`. [web:1][web:24]

Usa este comando cuando quieras forzar una sincronización manual inmediata.

## 3) Monitorear cambios automáticamente

Abre una terminal aparte y deja este bucle corriendo mientras trabajas en Zotero:

```bash
while inotifywait -r -e close_write,create,delete,move /home/jjlealg/Zotero; do
  rsync -a --delete /home/jjlealg/Zotero/ /mnt/Compartida/bibliografia/
done
```

Este comando vigila recursivamente la carpeta de Zotero y, cuando detecta cambios relevantes, ejecuta `rsync` para mantener el espejo actualizado. `inotifywait` soporta precisamente estos eventos de vigilancia en Linux. [web:4][web:5]

### Cómo usarlo bien

- Déjalo ejecutándose en una terminal dedicada.
- Para detenerlo, usa `Ctrl+C`.
- Si reinicias el equipo, tendrás que volver a lanzarlo manualmente, a menos que luego lo conviertas en un servicio de usuario.

## 4) Respaldo seguro a GitHub

Git no es el mejor mecanismo para una base SQLite que cambia constantemente, así que conviene usar Git como **respaldo puntual**, no como escritura continua en tiempo real. Para copias de seguridad, lo más seguro es cerrar Zotero antes de hacer el commit y el push. [web:10][web:161]

### Respaldo recomendado

1. Cierra Zotero.
2. Sincroniza el espejo local:
```bash
rsync -a --delete /home/jjlealg/Zotero/ /mnt/Compartida/bibliografia/
```
3. Entra al repo y sube cambios:
```bash
cd /mnt/Compartida/bibliografia && git add . && git commit -m "Backup Zotero $(date '+%F %H:%M')" && git push
```

Como ya quedó configurada la rama `main` con upstream, después de cada commit te bastará con `git push` para subir al remoto. [web:139][web:147]

## 5) Flujo diario recomendado

### Opción A: trabajo normal con espejo local

Usa esta opción cuando estés editando mucho la biblioteca y quieras espejo continuo en el otro disco.

1. Inicia el monitor:
```bash
while inotifywait -r -e close_write,create,delete,move /home/jjlealg/Zotero; do
  rsync -a --delete /home/jjlealg/Zotero/ /mnt/Compartida/bibliografia/
done
```
2. Trabaja normalmente en Zotero.
3. Al final del día, cierra Zotero.
4. Ejecuta:
```bash
cd /mnt/Compartida/bibliografia && git add . && git commit -m "Backup Zotero $(date '+%F %H:%M')" && git push
```

### Opción B: respaldo manual sin monitor

Usa esta opción si no quieres dejar una terminal vigilando.

1. Cierra Zotero.
2. Ejecuta:
```bash
rsync -a --delete /home/jjlealg/Zotero/ /mnt/Compartida/bibliografia/
```
3. Luego:
```bash
cd /mnt/Compartida/bibliografia && git add . && git commit -m "Backup Zotero $(date '+%F %H:%M')" && git push
```

## 6) Verificaciones rápidas

### Ver estado del repo

```bash
cd /mnt/Compartida/bibliografia && git status
```

### Ver remoto configurado

```bash
cd /mnt/Compartida/bibliografia && git remote -v
```

### Ver rama actual

```bash
cd /mnt/Compartida/bibliografia && git branch --show-current
```

### Ver último commit

```bash
cd /mnt/Compartida/bibliografia && git log --oneline -n 5
```

## 7) Problemas comunes

### `inotifywait: command not found`

Instala el paquete:

```bash
sudo pacman -S inotify-tools
```

En Arch, `inotifywait` viene en `inotify-tools`. [web:36]

### `fatal: not a git repository`

Eso significa que en la carpeta no existe `.git` o no estás parado dentro del repo correcto. Git reconoce un repositorio local por la presencia del directorio `.git`. [web:68][web:64]

Verifica:

```bash
find /mnt/Compartida/bibliografia -maxdepth 1 -name .git -ls
```

### El push falla por tamaño de archivos

GitHub avisa por archivos grandes y bloquea archivos de más de 100 MiB. Si el push falla, normalmente el problema estará en adjuntos dentro de `storage`. [web:49]

Para identificar archivos muy grandes dentro del repo:

```bash
find /mnt/Compartida/bibliografia/storage -type f -size +90M -ls
```

Si eso ocurre, tendrás que decidir entre:
- excluir adjuntos muy grandes,
- usar Git LFS,
- o respaldar a GitHub solo la base y la configuración, no todos los adjuntos.

### Quiero forzar una sincronización antes del commit

Haz siempre esto:

```bash
rsync -a --delete /home/jjlealg/Zotero/ /mnt/Compartida/bibliografia/
```

## 8) Comandos más importantes

### Espejo manual
```bash
rsync -a --delete /home/jjlealg/Zotero/ /mnt/Compartida/bibliografia/
```

### Monitor continuo
```bash
while inotifywait -r -e close_write,create,delete,move /home/jjlealg/Zotero; do
  rsync -a --delete /home/jjlealg/Zotero/ /mnt/Compartida/bibliografia/
done
```

### Backup completo a GitHub
```bash
cd /mnt/Compartida/bibliografia && rsync -a --delete /home/jjlealg/Zotero/ . && git add . && git commit -m "Backup Zotero $(date '+%F %H:%M')" && git push
```

## 9) Recomendación práctica

- Mantén Zotero trabajando desde `/home/jjlealg/Zotero`.
- Usa `/mnt/Compartida/bibliografia` como espejo local y repo Git.
- Usa el monitor para tener copia local casi inmediata.
- Usa GitHub para respaldos puntuales, mejor con Zotero cerrado. Zotero documenta la carpeta de datos como la ubicación relevante a copiar, y para backup consistente conviene no hacerlo con la base activa en uso. [web:10][web:161]

