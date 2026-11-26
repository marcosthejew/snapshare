# snapshare: diagnóstico y recomendaciones

Ejercicio DevOps, 25 y 26 de noviembre de 2026. Clúster k3s de 3 nodos (cp-1 con taint de control plane,
worker-1 y worker-2), namespace `marcos-snapshare`. Redis 7.4 detrás de `serverless-redis-http` (`srh`), que
emula la API REST de Upstash.

Todas las horas están en UTC. Cada hallazgo indica síntoma, evidencia, causa, corrección y resultado medido.
Las correcciones son configuración, objetos de Kubernetes, pipeline y versiones de dependencias: el código de la
aplicación no se tocó (ver la sección 2.1 sobre las dependencias).

## Resumen

| # | Problema | Corrección | Estado |
|---|---|---|---|
| 1 | Vulnerabilidades críticas (ejecución remota de código en `next` 16.3.5) bloqueaban el despliegue | `next` 16.4.1 (arrastra `sharp` 0.35.5) | Verificado |
| 2 | Los pods no arrancaban: `runAsNonRoot` con un usuario por nombre | `runAsUser: 1001` | Verificado |
| 3 | Crear un secreto devolvía siempre 503 mientras el health check daba 200 | `@upstash/ratelimit` fijado en 2.0.8 | Verificado |
| 4 | Imposible usar la app desde un navegador: el cifrado falla por HTTP | Certificado de Let's Encrypt en el Ingress | Verificado por un usuario |
| 5 | El límite por IP no ve la IP real: todos los usuarios comparten ~10 escrituras/min | Requiere cambiar Traefik (`kube-system`) | Documentado |
| 6 | Reparto entre nodos no garantizado tras un rollout; nodo caído tarda ~5 min | `matchLabelKeys`, `tolerationSeconds: 30` | Aplicado |

## 1. Pipeline (`.github/workflows/ci.yml`)

1. **Verificaciones:** lint, typecheck (`next typegen` + `tsc --noEmit`) y tests (Node 22).
2. **Imagen:** build → escaneo con Trivy (falla con HIGH o CRITICAL) → push a GHCR con `GITHUB_TOKEN`, etiquetada
   con el SHA del commit. En PR no se publica.
3. **Despliegue** (solo `main`): `kubectl apply -k deploy` con la imagen del commit → `rollout status` → prueba
   de humo por HTTPS en `/api/health`.

### Workflows heredados

- **Eliminados:** despliegue a Vercel, release-please y previews. Requieren secretos que el fork no tiene y
  desplegarían fuera del clúster.
- **Dependabot se conserva:** sus PR solo ejecutan verificaciones (sin push ni despliegue), así que no estorban,
  y mantiene las dependencias al día. Se le agregó una excepción para `@upstash/ratelimit` ≥ 2.1.0 (ver 3.3).
- **Concurrencia:** se cancelan ejecuciones solo en PR; los despliegues de `main` no se cancelan entre sí.

### Seguridad del pipeline

Endurecimiento aplicado al final (commit `f6dd1c5`, 05:18), verificado con actionlint y con un despliegue completo:

- **Acciones de terceros fijadas por SHA** (con la versión en un comentario); Dependabot (`github-actions`) los mantiene al día actualizando SHA y comentario. Una etiqueta como `@v4` se puede
  mover; si el repositorio de la acción se viera comprometido, el código nuevo correría junto a las credenciales
  del clúster.
- **Secretos por `env:`** en los pasos de shell, en lugar de interpolarlos en el script.
- **`permissions: contents: read`** declarado arriba; solo el job que publica imágenes agrega `packages: write`.

Auditoría (los repositorios son públicos, así que también lo son los logs de Actions):

- Ningún secreto en el historial de git (patrones de tokens, claves privadas y campos de kubeconfig: 0).
- Logs de despliegue revisados: los tokens aparecen enmascarados (`***`) y el kubeconfig decodificado nunca se
  imprime.
- Los despliegues no corren en PR ni desde forks ajenos, y no se usa `pull_request_target`: un PR malicioso no
  alcanza los secretos del clúster.
- El `GITHUB_TOKEN` por defecto del repositorio es de solo lectura; los kubeconfig de CI solo actúan en su
  namespace.

### Imágenes privadas

El paquete de GHCR hereda la visibilidad del fork (público). Se pasó a privado; los pods descargan con el Secret
`ghcr-pull` (token `read:packages`, creado fuera del pipeline). Verificado: un pull anónimo devuelve 403.

## 2. Despliegue en el clúster (`deploy/app.yaml`)

| Aspecto | Implementación |
|---|---|
| Réplicas y despliegues seguros | 2 réplicas, `maxUnavailable: 0`, `maxSurge: 1`, `preStop` de 5 s. |
| Probes | Readiness en `/api/health` (hace ping a Redis); liveness y startup por TCP, para que una caída de Redis no reinicie todos los pods en bucle. |
| Pérdida de un nodo | Reparto obligatorio entre workers (`DoNotSchedule`, `nodeTaintsPolicy: Honor`, `matchLabelKeys: [pod-template-hash]`), PDB `minAvailable: 1`, `tolerationSeconds: 30`. |
| Recursos | 50m / 128Mi de request, 256Mi de límite de memoria. |
| Seguridad | UID 1001, sin escalada de privilegios, todas las capabilities eliminadas. |
| HTTPS | Ingress con el Secret `snapshare-tls`. |

**Evidencia:** durante un reinicio completo, 202/202 peticiones devolvieron 200. Borrando a la vez todos los
pods de un worker, `/api/health` respondió 59/59.

### 2.1 Cambios de dependencias, no de código

El enunciado no permite modificar el código de la aplicación (salvo Dockerfiles y pipeline). Se cambiaron dos
versiones en `package.json` y `package-lock.json`, sin tocar ningún archivo de código:

| Cambio | Por qué no había alternativa |
|---|---|
| `next` 16.3.5 → 16.4.1 | Sin él, la imagen tenía 3 vulnerabilidades críticas de ejecución remota de código y el propio escaneo del pipeline bloqueaba el despliegue: había que elegir entre desplegar algo vulnerable o no desplegar. |
| `@upstash/ratelimit` ^2.1.0 → 2.0.8 | Sin él, snapshare no podía crear ningún secreto. La alternativa (cambiar el Redis) no existe: ninguna versión de Redis reconoce la bandera que usa 2.1.0+. |

Son el mismo tipo de cambio que propone Dependabot, que el fork ya trae configurado precisamente para modificar
esos archivos. Lint, tests (65/65) y typecheck pasaron tras cada cambio.

## 3. Problemas encontrados

### 3.1 Vulnerabilidades críticas

- **Síntoma:** el primer despliegue no llegó a publicar la imagen: Trivy la bloqueó.
- **Evidencia:** 3 CRITICAL y 3 HIGH. En `next` 16.3.5: ejecución remota de código sin autenticación en Image
  Optimization (SVG), en funciones `use cache` y en `next/og`; denegación de servicio en Server Actions; SSRF en
  Image Optimization. En `sharp` 0.35.4: vulnerabilidad en su dependencia librsvg.
- **Corrección:** `next` 16.4.1 (y su configuración de lint); `sharp` pasa a 0.35.5 como dependencia de `next`.
  Lint, tests (65/65) y typecheck en verde. Es una actualización de dependencias, equivalente a aceptar un PR de
  Dependabot.
- `npm audit` sigue listando `@grpc/grpc-js` y `brace-expansion`: son herramientas de desarrollo que no entran en
  la imagen, por eso Trivy no las marca.

### 3.2 Los pods no arrancaban

- **Síntoma:** `CreateContainerConfigError`: "container has runAsNonRoot and image has non-numeric user
  (nextjs), cannot verify user is non-root".
- **Causa:** el Dockerfile usa el usuario `nextjs` (un nombre); Kubernetes solo puede verificar un UID numérico.
- **Corrección:** `runAsUser: 1001` y `runAsGroup: 1001`, los IDs que el Dockerfile asigna a `nextjs`.

### 3.3 Crear un secreto siempre devolvía 503

- **Síntoma:** todo `POST /api/secret` respondía `503 {"error":"temporarily unavailable"}` con `Retry-After: 5`,
  mientras `/api/health` daba 200. Kubernetes veía la app sana y la función principal estaba rota.
- **Evidencia:**
  1. Ese 503 sale del `catch` que rodea al limitador de peticiones (`lib/secret-guard.ts`); el error se descarta
     sin registrarlo, por eso ni snapshare ni `srh` mostraban nada en los logs.
  2. Redis respondía bien a mano a través de `srh` desde un pod (PING, SET, EVAL, EVALSHA, SCRIPT LOAD).
  3. Reproduciendo el limitador con la misma librería contra `srh` (port-forward):
     `ERR Unexpected flag in script shebang: allow-key-locking`.
- **Causa:** `@upstash/ratelimit` 2.1.0 (presente desde el repositorio original) envía sus 12 scripts Lua con la
  bandera `#!lua flags=allow-key-locking`, propia del Redis administrado de Upstash. Redis 7.4 la rechaza. La
  bandera está fija en el código de la librería: 2.0.8 no la usa; 2.1.0 la introduce; 2.2.2 la mantiene.
- **Corrección:** `@upstash/ratelimit` fijado en 2.0.8, probado antes contra el `srh` del clúster (permite 5,
  bloquea la 6.ª y la 7.ª). `dependabot.yml` ignora 2.1.0+ para que no vuelva el fallo.
- **Por qué el health check no lo detectaba:** `/api/health` solo hace PING, que funciona. Una readiness que
  ejercite el limitador requeriría código.

### 3.4 Imposible crear un enlace desde el navegador

- **Síntoma (encontrado usando la app como usuario):** "Create link" mostraba "Something went wrong. Try
  again.", sin errores en la consola y **sin ninguna petición** en la pestaña Network.
- **Causa:** snapshare cifra en el navegador (AES-256-GCM) con `crypto.subtle` (`lib/crypto.ts`). Los navegadores
  solo exponen `crypto.subtle` en contextos seguros (HTTPS o localhost). Por HTTP no existe, el cifrado falla
  antes de enviar nada y la app muestra un error genérico.
- **Por qué las pruebas con `curl` sí funcionaban:** enviaban a la API un texto cifrado ya preparado, saltándose
  el cifrado del navegador.
- **Verificación previa:** Traefik ya respondía por HTTPS con su certificado por defecto; aceptando la
  advertencia, "Create link" funcionaba.
- **Corrección:** no hay cert-manager y el ACME de Traefik vive en `kube-system`. Se emitió un certificado de
  Let's Encrypt a mano desde el clúster: pod temporal con `lego` (sin root), Ingress temporal solo para
  `/.well-known/acme-challenge` y Secret `snapshare-tls`; la clave nunca se imprimió y los recursos temporales se
  borraron. La prueba de humo del pipeline pasó a HTTPS.
- **Resultado (verificado por un usuario):** el enlace se crea; al abrirlo devuelve el secreto original; al
  reabrirlo indica que expiró o ya fue abierto. "Burn after reading" funciona con 2 réplicas porque el estado vive
  en Redis.

### 3.5 El límite por IP no ve la IP real

- **Prueba:** 20 escrituras seguidas (3 aceptadas, el resto 429, tras 7 anteriores en la misma ventana) y 5 con un
  `X-Forwarded-For` inventado (todas 429).
- **Resultado:**
  - **No se puede evadir:** Traefik no confía en `X-Forwarded-For` desde fuera y lo sobrescribe.
  - **Pero no ve la IP del cliente:** la app registró solo `10.42.1.4` y `10.42.2.2`, direcciones internas (una
    por worker). El balanceador delante de Traefik reescribe la dirección de origen (SNAT).
  - **Consecuencia:** todos los usuarios comparten 2 cubetas: ~10 escrituras y ~40 lecturas por minuto **en
    total**. Con usuarios reales sería una denegación de servicio autoinfligida.
- **Corrección (fuera de nuestros permisos):** `externalTrafficPolicy: Local` en el Service de Traefik, o PROXY
  protocol. El código de la librería ya advierte que el uso de `X-Forwarded-For` debe revisarse si la app deja
  Vercel.

### 3.6 Reparto entre nodos y nodo caído

- La regla de reparto contaba también los pods de la versión anterior durante un rollout, así que podía dejar las
  dos réplicas en el mismo nodo. Corrección: `matchLabelKeys: [pod-template-hash]`.
- Con un nodo caído, Kubernetes espera ~5 min por defecto antes de reemplazar sus pods. Corrección:
  `tolerationSeconds: 30` para `not-ready` y `unreachable`.

## 4. Riesgos y recomendaciones

1. **Redirección HTTP → HTTPS:** quien entra por `http://` sigue viendo la versión rota. Requiere un Middleware
   de Traefik; no tenemos permiso para crearlo.
2. **Renovación del certificado:** se emitió a mano; vence el 2027-02-24. En producción: cert-manager.
3. **IP real del cliente** para el limitador (ver 3.5).
4. **Errores silenciosos:** el `catch` del limitador descarta el error; registrarlo habría ahorrado el
   diagnóstico de 3.3.
5. **Health check superficial:** `/api/health` no detecta que el limitador falla.
6. **Compatibilidad con Upstash:** `@upstash/ratelimit` asume el Redis de Upstash; con Redis propio hay que fijar
   versiones o cambiar de librería. Revisar al actualizar.
7. **Redis único en cp-1:** si cp-1 cae, se pierden los secretos almacenados.

## 5. Proceso y lecciones

- Probar la API no basta: los dos fallos que dejaban la app inutilizable (3.3 y 3.4) solo se vieron intentando
  usarla; el segundo, solo desde un navegador.
- Antes de cambiar una dependencia se reprodujo el fallo con la misma librería y se probó la versión corregida
  contra el Redis real.
