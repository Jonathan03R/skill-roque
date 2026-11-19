---
name: apis-roque
description: Use when creating, modifying, debugging, or integrating backend APIs, especially SUNAT integrations, external HTTP responses, AppError, apiErrorResponse, controllers, services, or frontend API clients.
---

# APIs Roque

Contrato obligatorio para errores de APIs Sunapp/SUNAT. Mantener separación entre mensaje visible, diagnóstico técnico y respuesta externa original.

## Contrato de respuesta

Toda API fallida debe responder mediante `apiErrorResponse` y conservar esta forma:

```json
{
  "success": false,
  "status": 502,
  "code": "PLAME_SUNAT_INVALID_RESPONSE",
  "message": "El contribuyente no ha sido registrado como Empleador.",
  "requestId": "uuid",
  "reportable": false,
  "errors": {
    "sunatStatus": 200,
    "sunatResponse": "respuesta original de SUNAT"
  }
}
```

Reglas:

- `message`: único texto que frontend muestra al usuario. Corto, humano y sin HTML.
- `code`: estable, en mayúsculas, específico del dominio.
- `status`: HTTP de nuestra API. Error externo normalmente usa `502`; validación externa puede conservar `4xx` si representa una condición funcional conocida.
- `requestId`: siempre generado por `apiErrorResponse` y enviado en header `X-Request-Id`.
- `errors`: datos estructurados para diagnóstico/programación; frontend no debe renderizarlo completo.
- `details` interno: conservar `sunatStatus`, `sunatResponse`, endpoint externo y contenido original sin alterar, salvo secretos.
- Nunca poner body HTML, stack, token, cookie, credenciales o headers sensibles dentro de `message`.
- Nunca repetir el mismo error concatenando `message`, `errors`, stack o body.

## Flujo backend

1. Controller valida request y llama service.
2. Service ejecuta HTTP externo.
3. Service captura status y body originales antes de parsear.
4. Service crea `AppError` con mensaje visible y detalles técnicos.
5. Controller usa `apiErrorResponse`; no arma respuestas de error manuales.

Controller:

```ts
catch (error) {
  return apiErrorResponse({
    req,
    res,
    error,
    fallbackCode: "MODULE_OPERATION_FAILED",
    fallbackUserMessage: "No pudimos completar la operación.",
  });
}
```

## Respuesta externa SUNAT

Aceptar status para inspeccionar body cuando el objetivo sea preservar error:

```ts
const response = await client.get(url, {
  validateStatus: () => true,
  responseType: "text",
});

const raw = String(response.data ?? "");
if (response.status >= 400) {
  throw new AppError(
    502,
    extractVisibleMessage(raw),
    true,
    { sunatStatus: response.status, sunatResponse: raw },
    "MODULE_SUNAT_RESPONSE_ERROR",
  );
}
```

Si SUNAT devuelve HTML, extraer solo el texto visible para `message` y conservar HTML completo en `details.sunatResponse`. Si devuelve JSON, usar campos funcionales (`message`, `msg`, `errors[].desError`) para `message`, conservando JSON original en details.

## Frontend

El cliente debe leer únicamente:

```ts
error?.response?.data?.message
```

No renderizar `errors`, `details`, `sunatResponse`, stack ni objeto completo. Si backend antiguo envía HTML en `message`, sanitizar y extraer texto antes de mostrar.

## Errores por tipo

| Caso | HTTP | `code` | `message` |
|---|---:|---|---|
| Request inválido | 400 | `*_VALIDATION_ERROR` | Parámetros inválidos. |
| Sesión app inválida | 401 | `SESSION_INVALID` | Sesión no válida. |
| Credencial SOL faltante | 422 | `SUNAT_CREDENTIALS_MISSING` | No tienes una cuenta SOL disponible. |
| SUNAT rechazó sesión | 502 | `SUNAT_AUTH_FAILED` | SUNAT rechazó la sesión. |
| SUNAT body inválido | 502 | `*_SUNAT_INVALID_RESPONSE` | Texto visible extraído de SUNAT. |
| SUNAT no disponible | 502 | `*_SUNAT_REQUEST_FAILED` | SUNAT no está disponible. |

## Errores prohibidos

- Lanzar `new Error(rawHtml)` sin `details`.
- Reemplazar body externo por mensaje genérico perdiendo diagnóstico.
- Exponer HTML completo en UI.
- Devolver un error distinto en cada capa para la misma causa.
- Loguear credenciales, cookies, Bearer o tokens.
- Cambiar `api-error-response.ts` para resolver un solo módulo; corregir el service que perdió el body.

## Checklist

- [ ] Controller usa `apiErrorResponse`.
- [ ] Service conserva status/body externo antes de parsear.
- [ ] `message` es único, visible y sin HTML.
- [ ] `errors`/`details` contienen diagnóstico original.
- [ ] Frontend muestra solo `message`.
- [ ] Secretos están redactados.
- [ ] `git diff --check` ejecutado; respetar instrucción del usuario sobre tests/build.
