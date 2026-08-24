# raysync

Sincronizador de directorios **con delta por bloques y transporte cifrado**, escrito en [raylang](https://github.com/roberto-ayala/raylang): un `push` de una dirección que salta los archivos idénticos (hash), reenvía solo los **bloques de 64 KiB que cambiaron** de los archivos modificados, reconstruye del lado receptor con verificación de hash y rename atómico, y con `--watch` queda vigilando mtimes. Complementa a [takeit](../takeit) (un archivo, una vez) con el caso "un árbol entero, continuamente".

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

$ raysync push ./proyecto … --watch          # modo continuo (sondea mtimes)
$ raysync push ./proyecto … --delete         # borra en destino lo que ya no existe
```

## Medido (nativo, localhost)

- **50 MB fríos: 0.17 s** (cifrado ChaCha20-Poly1305 + sha256 encadenado en
  ambos lados incluidos).
- Push sin cambios: 53 ms (escaneo + manifest + plan).
- 1 byte cambiado en 30 MB: **solo 64 KiB viajan**; resultado byte-idéntico
  (verificado con `cmp`).

## Protocolo

1. El receptor manda un salt de 16 B en claro; ambos derivan la clave con
   `hkdf_sha256(salt, password, "raysync-v1")`. Todo lo demás son frames
   `[len BE32][ChaCha20-Poly1305]` con nonce = dirección + contador (un frame
   reordenado o repetido falla la autenticación; la contraseña nunca viaja).
2. El emisor manda su manifiesto (ruta, tamaño, hash encadenado por chunks).
3. El receptor responde el plan: `full` (no lo tengo), nada (idéntico), o
   `blocks` (lo tengo distinto: aquí van los hashes de MIS bloques de 64 KiB).
4. Para cada archivo con delta, el emisor compara bloque a bloque (alineado
   por posición) y manda ops `COPY(i)` / `DATA` + los bloques nuevos; el
   receptor reconstruye en un `.tmp` (sus propios bloques via `seek` + los
   recibidos), **verifica el hash y solo entonces renombra** — una
   transferencia mala jamás pisa un archivo bueno. Rutas con `..` rechazadas.

Delta de bloques FIJOS: perfecto para appends y ediciones in-place; una
inserción desplaza todo lo posterior (el rolling checksum de rsync queda
fuera de v1).

## Estado actual

| Capacidad | Estado |
|-----------|--------|
| Push cifrado E2E (clave por contraseña, nonces direccionales) | ✅ |
| Skip por hash + delta por bloques 64 KiB + reconstrucción verificada | ✅ |
| Árboles anidados (mkdir -p en destino), rutas relativas | ✅ |
| `--delete` (espejo) y `--watch` (sondeo de mtimes) | ✅ |
| Contraseña errónea = fallo de autenticación limpio | ✅ |
| Binario nativo (50 MB en 0.17 s) | ✅ |
| Tests (scan/hash/bloques + E2E con delta y borrado) | ✅ 4 |
| Rolling checksum (inserciones), sync bidireccional | 📋 fuera de v1 |
| Permisos/symlinks/metadatos | ❌ bloqueado (fs no los expone) |

## Hallazgos de dogfood (necesidades confirmadas del lenguaje)

Anotados en `raylang/IDEAS.md` §69:

1. **Sin watch de filesystem** — cuarta app sondeando mtimes (tras `ray dev`,
   raycode-dev y raylogs `--follow`). La evidencia ya es plantilla.
2. **Sin metadatos**: `fs` no expone permisos ni distingue symlinks
   (`is_dir`/`is_file` y nada más) → un sync fiel a rsync no puede serlo.
3. **Sin `fs.write_bytes(handle)`** (§68): la reconstrucción escribe por
   `append_file_bytes` a un temp + rename — funciona y es atómico, pero
   reescribir un bloque in-place es inexpresable.
4. **Sin hasher incremental** en `std/crypto`: el hash de archivo es sha256
   encadenado por chunks (patrón takeit, tercera app que lo copia) — un
   `sha256_init/update/final` evitaría la variante casera.
5. **Positivo**: la cripto de `ring` vuela (50 MB cifrados+hasheados en
   0.17 s), `seek`+`read_bytes` componen la lectura de bloques limpia, y
   `fs.rename` vuelve a ser la pieza de atomicidad que todo lo salva.

## Desarrollo

```sh
ray test                # 4 tests
ray build --native src/main.ray -o raysync --release
```

Estructura: `src/main.ray` (CLI) · `scan.ray` (walk + hashes) · `frames.ray`
(transporte cifrado) · `proto.ray` (mensajes + plan de delta) · `server.ray`
(receptor/reconstrucción) · `push.ray` (emisor + watch).
