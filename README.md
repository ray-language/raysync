# raysync

Sincronizador de directorios **con delta por bloques y transporte cifrado**, escrito en [raylang](https://github.com/ray-language/raylang): un `push` de una dirección que salta los archivos idénticos (hash), reenvía solo los **bloques de 64 KiB que cambiaron** de los archivos modificados, reconstruye del lado receptor con verificación de hash y rename atómico, y con `--watch` queda aparcado en eventos de kernel (`fs.watch`). Complementa a [takeit](../takeit) (un archivo, una vez) con el caso "un árbol entero, continuamente".

```text
# Receptor
$ raysync serve ./backup --password s3cr3t

# Emisor
$ raysync push ./proyecto --host nas.local --password s3cr3t
synced 2 file(s): 52428800 B sent, 0 B reused in place

$ raysync push ./proyecto --host nas.local --password s3cr3t     # sin cambios
synced 0 file(s): 0 B sent, 0 B reused in place

# … tocas 1 byte de un archivo de 30 MB …
synced 1 file(s): 65536 B sent, 31391744 B reused in place

$ raysync push ./proyecto … --watch          # modo continuo (eventos de kernel)
$ raysync push ./proyecto … --delete         # borra en destino lo que ya no existe
```

## Medido (nativo, localhost)

- **50 MB fríos: 0.17 s** (cifrado ChaCha20-Poly1305 + sha256 incremental en
  ambos lados incluidos).
- Push sin cambios: 53 ms (escaneo + manifest + plan).
- 1 byte cambiado en 30 MB: **solo 64 KiB viajan**; resultado byte-idéntico
  (verificado con `cmp`).

## Protocolo

1. El receptor manda un salt de 16 B en claro; ambos derivan la clave con
   `pbkdf2_hmac_sha256(password, salt, 600000)` — lenta a propósito: quien
   grabe una sesión no puede probar contraseñas a velocidad de HMAC (el
   protocolo v1 usaba HKDF; emisor y receptor v1/v2 no interoperan: el
   handshake falla la autenticación). Todo lo demás son frames
   `[len BE32][ChaCha20-Poly1305]` con nonce = dirección + contador (un frame
   reordenado o repetido falla la autenticación; la contraseña nunca viaja).
2. El emisor manda su manifiesto (ruta, tamaño, sha256 real del archivo — incremental, contrastable con `shasum -a 256`).
3. El receptor responde el plan: `full` (no lo tengo), nada (idéntico), o
   `blocks` (lo tengo distinto: aquí van los hashes de MIS bloques de 64 KiB).
4. Para cada archivo con delta, el emisor compara bloque a bloque (alineado
   por posición) y manda ops `COPY(i)` / `DATA` + los bloques nuevos; el
   receptor reconstruye en un `.tmp` (sus propios bloques via `seek` + los
   recibidos), **verifica el hash y solo entonces renombra** — una
   transferencia mala jamás pisa un archivo bueno. Rutas con `..` rechazadas,
   y **contención real** (`fs.is_within_real`, raylang 1.26): un symlink dentro
   del destino que apunte fuera nunca se usa para escribir, leer bloques ni
   borrar (`--delete`) — la sesión falla con "path escapes the sync root".

Delta de bloques FIJOS: perfecto para appends y ediciones in-place; una
inserción desplaza todo lo posterior (el rolling checksum de rsync queda
fuera de v1).

## Estado actual

| Capacidad | Estado |
|-----------|--------|
| Push cifrado E2E (clave por contraseña, nonces direccionales) | ✅ |
| Skip por hash + delta por bloques 64 KiB + reconstrucción verificada | ✅ |
| Árboles anidados (mkdir -p en destino), rutas relativas | ✅ |
| `--delete` (espejo) y `--watch` (fs.watch + debounce; degrada a sondeo) | ✅ |
| Contraseña errónea = fallo de autenticación limpio | ✅ |
| Binario nativo (50 MB en 0.17 s) | ✅ |
| Tests (scan/hash/bloques + E2E con delta, borrado y symlink que escapa) | ✅ 4 |
| Contención del destino ante symlinks (`fs.is_within_real`) | ✅ |
| Rolling checksum (inserciones), sync bidireccional | 📋 fuera de v1 |
| Replicar permisos/symlinks/metadatos | 📋 v2 (`fs.stat`/`fs.chmod` ya existen) |

## Hallazgos de dogfood (necesidades confirmadas del lenguaje)

Anotados en `raylang/IDEAS.md` §69:

1. **[RESUELTO — raylang M115.4]** Sin watch de filesystem: `fs.watch`
   existe y `--watch` aparca en eventos de kernel con debounce.
2. **[RESUELTO — raylang M115.3 / 1.26]** Sin metadatos: `fs.stat` (lstat:
   kind/mode/size/mtime) distingue symlinks y `fs.real_path`/`is_within_real`
   dan la contención que usa el receptor. Replicar permisos y symlinks queda
   para v2 (es trabajo de la app, ya no del lenguaje); crear un symlink ya
   tiene `fs.symlink` (raylang 1.27.13; el test lo usa en vez de `ln -s`).
3. **[RESUELTO — raylang M115]** Sin `fs.write_bytes(handle)`: existe; la
   reconstrucción sigue por temp + `append_file_bytes` + rename porque es lo
   que da la atomicidad.
4. **[RESUELTO — raylang M126]** Sin hasher incremental: `sha256_init` +
   `hash_update` + `hash_final` existen — el hash de archivo es el sha256
   REAL (boundary-independiente, contrastable con `shasum`), y el encadenado
   casero desapareció del protocolo entero.
5. **Positivo**: la cripto de `ring` vuela (50 MB cifrados+hasheados en
   0.17 s), `seek`+`read_bytes` componen la lectura de bloques limpia, y
   `fs.rename` vuelve a ser la pieza de atomicidad que todo lo salva.

## Desarrollo

Requiere raylang 1.27.13+ (sin dependencias externas).

```sh
ray test                # 4 tests
ray build --native src/main.ray -o raysync --release
```

Estructura: `src/main.ray` (CLI) · `scan.ray` (walk + hashes) · `frames.ray`
(transporte cifrado) · `proto.ray` (mensajes + plan de delta) · `server.ray`
(receptor/reconstrucción) · `push.ray` (emisor + watch).
