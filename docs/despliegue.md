# Manual de despliegue

Guía paso a paso para levantar y probar el **CV Ranker** de cero. La solución está repartida en tres repositorios; este manual los orquesta en orden.

| Componente | Repositorio |
|---|---|
| Backend (Lambdas + infra) | https://github.com/RafaCH1906/Back_read_CV |
| Frontend (React) | https://github.com/AE00NN/Frontend-Hackathon-CC |
| Integración + docs (este repo) | repo de entrega |

---

## Requisitos previos

- Cuenta de **AWS Academy Learner Lab** (región **us-east-1**).
- Una **API key de Groq** — se obtiene gratis en https://console.groq.com/keys
- **Python 3.12** (para empaquetar el backend).
- **Node.js 18+** y npm (para buildear el frontend).

### Nota importante sobre AWS Academy (LabRole)

El **backend** corre en Learner Lab, donde **no se pueden crear roles IAM**: solo existe `LabRole`. Por eso su infraestructura:

- Usa **CloudFormation plano** (no SAM/CDK, que fallan al intentar crear roles).
- Referencia `LabRole` como execution role de todas las Lambdas.

El **frontend** se hostea en **AWS Amplify** conectado a GitHub. Como es una app estática que solo consume la API pública del backend, no necesita vivir en la misma cuenta del Learner Lab.

Además, **las credenciales del lab rotan en cada sesión**. Lo más cómodo es trabajar desde **CloudShell** (ya viene autenticado). Si usas la CLI local, recopia las 3 llaves desde *AWS Details* al reiniciar el lab.

---

## Parte 1 — Backend

> Pasos detallados en el [README del backend](https://github.com/RafaCH1906/Back_read_CV). Resumen del flujo:

1. Inicia el lab, abre **CloudShell** y fija la región:
   ```bash
   export AWS_DEFAULT_REGION=us-east-1
   ```
2. Clona el repo y entra:
   ```bash
   git clone https://github.com/RafaCH1906/Back_read_CV.git
   cd Back_read_CV
   ```
3. Empaqueta las Lambdas (descarga dependencias para Linux y genera los ZIP):
   ```bash
   python build_package.py
   ```
4. Crea un bucket para los artefactos de despliegue y súbelos:
   ```bash
   aws s3 mb s3://mis-deploys-cv-ranker-<ALGO_UNICO>
   aws s3 cp dist/ s3://mis-deploys-cv-ranker-<ALGO_UNICO>/ --recursive
   ```
5. Despliega la infraestructura (pasa tu bucket de deploys y tu API key de Groq):
   ```bash
   aws cloudformation deploy \
     --template-file template.yaml \
     --stack-name cv-ranker \
     --parameter-overrides \
       DeploymentBucket=mis-deploys-cv-ranker-<ALGO_UNICO> \
       GroqApiKey=gsk_TU_API_KEY_DE_GROQ
   ```
6. **Anota el endpoint de la API** (lo necesita el frontend):
   ```bash
   aws cloudformation describe-stacks --stack-name cv-ranker \
     --query "Stacks[0].Outputs" --output table
   ```
   Busca el `ApiEndpoint`, algo como `https://xxxxxxxx.execute-api.us-east-1.amazonaws.com`.

✅ Al terminar, el backend está desplegado: API Gateway, las 3 Lambdas, S3 de uploads, SQS + DLQ y DynamoDB.

---

## Parte 2 — Frontend (build)

1. Clona el repo y entra:
   ```bash
   git clone https://github.com/AE00NN/Frontend-Hackathon-CC.git
   cd Frontend-Hackathon-CC
   ```
2. Apunta el frontend al endpoint del backend. La URL está en **`src/api.js`**, primera línea:
   ```js
   const API_BASE = 'https://sf0vznhkrl.execute-api.us-east-1.amazonaws.com';
   ```
   Si tu `ApiEndpoint` de la Parte 1 es distinto, reemplázalo aquí.
3. Instala dependencias y genera el build:
   ```bash
   npm install
   npm run build
   ```
   Esto crea la carpeta **`dist/`** con la app lista para servir.

---

## Parte 3 — Publicar el frontend en AWS Amplify (URL pública)

El frontend se hostea en **AWS Amplify** conectado directamente al repo de GitHub. Amplify autodetecta el build de Vite y publica la app en una URL pública con HTTPS.

1. En la consola de AWS, entra a **AWS Amplify** → **Create new app** → **Host web app**.
2. Conecta el proveedor **GitHub** y autoriza el acceso al repositorio `AE00NN/Frontend-Hackathon-CC`.
3. Selecciona la rama **`main`**.
4. Amplify detecta el framework (Vite). Confirma la configuración de build:
   - **Build command:** `npm run build`
   - **Output directory (baseDirectory):** `dist`
5. (SPA) Para que las rutas internas no rompan al refrescar, agrega una *rewrite rule* en **App settings → Rewrites and redirects**:
   - Source: `</^[^.]+$|\.(?!(css|gif|ico|jpg|js|png|txt|svg|woff|woff2|ttf|map|json)$)([^.]+$)/>`
   - Target: `/index.html` · Type: `200 (Rewrite)`
6. **Save and deploy.** Amplify buildea y publica.

La **URL pública** queda con la forma `https://main.<APP_ID>.amplifyapp.com`. Cada push a `main` redespliega automáticamente.

> **Importante:** la URL de la API está hardcodeada en `src/api.js`. Si cambia el endpoint del backend, edita esa línea, haz commit y push, y Amplify reconstruye solo.

✅ Esa es la URL que va en el entregable (criterio 4) y en el README.

> *Alternativa (S3 + CloudFront):* si por algún motivo no usas Amplify, también puedes servir el `dist/` desde un bucket S3 con static website hosting + acceso público, configurando que los 404 redirijan a `index.html` (la app es una SPA con rutas). Amplify es la vía recomendada por simplicidad y porque ya está en uso.


---

## Parte 4 — Verificación end-to-end

1. Abre la URL pública del frontend.
2. Crea la vacante con la configuración de demo:

   | Campo | Valor |
   |---|---|
   | Título | `Backend Developer` |
   | Skills | `python, aws, sql, docker` |
   | Años | `3` |

3. Sube los **28 CVs de prueba** de `data/cvs_dummy/` (este repo).
4. El ranking se va llenando en vivo. Compáralo contra `data/cvs_dummy/_manifest.csv`: los perfiles backend Python/AWS deben quedar arriba (~75-98) y los fuera de perfil al fondo.

---

## Solución de problemas

| Síntoma | Causa / arreglo |
|---|---|
| `ExpiredToken` o `AccessDenied` en la CLI | La sesión del lab expiró. Reinicia el lab y recopia las credenciales (o usa CloudShell). |
| El despliegue del backend falla al crear roles | Estás usando SAM/CDK. Usa el `template.yaml` de CloudFormation plano de este proyecto. |
| El upload del CV a S3 falla con error de firma | El `PUT` debe incluir el header `Content-Type: application/pdf` y el archivo debe ser un PDF real. |
| El upload falla por CORS | El bucket de uploads del backend debe tener CORS con `AllowedOrigins: ["*"]` (ya viene así en el template de P1). |
| La presigned URL da error "expired" | Caduca a los 15 min. Vuelve a crear la vacante para generar URLs frescas. |
| El frontend carga pero las rutas rompen al refrescar | Falta la *rewrite rule* a `/index.html` en Amplify (Parte 3, paso 5). Es una SPA. |
| El ranking no avanza | Revisa los logs de la Lambda `worker` en CloudWatch. Verifica que `GroqApiKey` quedó seteada en el deploy. |

---

## Arquitectura

Ver el diagrama en [`arquitectura.png`](arquitectura.png) y la explicación en el [README principal](../README.md).
