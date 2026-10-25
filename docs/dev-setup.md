# Cómo compilar y correr el fork (Linux)

Verificado el 2026-10-25 en un contenedor Linux x86_64 con Go 1.24 (se baja solo 1.26), Node 22 y bun.

Estructura esperada, con los dos forks al lado:

```
anytype-heart/   chemworks/anytype-heart  (rama develop)
anytype-ts/      chemworks/anytype-ts     (rama develop)
```

## 1. Middleware (anytype-heart)

```bash
cd anytype-heart
make build-server          # unos 3 minutos la primera vez; deja dist/server
```

Bindings JS. `make install-dev-js` hace todo, pero su `npm install` se puede colgar detrás de un proxy, porque el post-install de `protoc-gen-js` baja binarios con el `https` de Node, que no respeta `HTTPS_PROXY`. Si pasa, hacerlo a mano:

```bash
# protoc >= 33 (bajar el zip de protobuf releases si no está en el sistema)
PATH="./node_modules/.bin:$PATH" protoc -I ./ -I <protoc>/include \
  --js_out=import_style=commonjs,binary:./dist/js/pb \
  pb/protos/service/*.proto pb/protos/*.proto pkg/lib/pb/model/protos/*.proto
PATH="./node_modules/.bin:$PATH" protoc -I ./ -I <protoc>/include \
  --grpc-web_out=import_style=commonjs+dts,mode=grpcwebtext:./dist/js/pb \
  pb/protos/service/*.proto pb/protos/*.proto pkg/lib/pb/model/protos/*.proto

cp dist/server ../anytype-ts/dist/anytypeHelper
cp -r dist/js/pb/* ../anytype-ts/dist/lib
mkdir -p ../anytype-ts/dist/lib/json/generated
cp pkg/lib/bundle/system*.json pkg/lib/bundle/internal*.json ../anytype-ts/dist/lib/json/generated
```

Push al fork: el repo tiene hooks propios (`.githooks`). Hace falta `pip install pre-commit` (corre gitleaks antes del push), y las ramas tienen que llevar una clave tipo `GO-0000` en el nombre.

## 2. Cliente (anytype-ts)

```bash
cd anytype-ts
ELECTRON_SKIP_BINARY_DOWNLOAD=1 bun install
bun run generate:protos    # genera middleware/ y src/ts/lib/api/service.ts desde ../anytype-heart
```

`bun install` puede terminar con error al recompilar `keytar` para Electron si no llega a electronjs.org. En ese caso la app de escritorio no arranca, pero el modo web sí.

## 3. Correr en modo web

```bash
WEB_DATA_PATH=/ruta/a/datos node scripts/start-web.js
# abre http://127.0.0.1:3030/?server=http%3A%2F%2F127.0.0.1%3A31008&dataPath=...
```

Levanta el middleware compilado en el paso 1, un proxy gRPC-web y Vite. Verificado: aparece la pantalla de inicio de Anytype.

## 4. Escritorio (en tu máquina)

En una máquina con acceso normal a internet, `bun install` recompila `keytar` sin problemas y `bun run start:dev` abre la app Electron.

## 4. Gotchas del modo web (2026-10-25)

- Abrí la URL completa que imprime `start-web.js` (`http://127.0.0.1:3030/?server=…&dataPath=…`). Sin `dataPath` la creación de cuenta falla con "failed to create local repo", y después el helper responde "initial parameters already set" hasta reiniciarlo.
- `bun run typecheck` se queda sin memoria con el heap por defecto. Usá `NODE_OPTIONS=--max-old-space-size=8192 bunx tsc --noEmit -p tsconfig.json`.
- La primera vez que se abre una tabla de Tipo aparece un aviso con botón "Ok" que tapa el `+` de columnas.
