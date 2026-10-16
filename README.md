# Webhooks de generación de PDF (Inspira)

**Versión:** 1.0.0 · **Requiere:** WordPress 5.8+ · PHP 7.4+ · **Text domain:** `inspira-pdf-webhooks`

> Centraliza en wp-admin las URL de los webhooks que generan los PDF de Inspira y permite probarlas con datos reales de empresa, área o trabajador.

## ¿Qué es?

Antes de este plugin, las URL de los webhooks de n8n que convierten los resultados del test en PDF estaban escritas directamente en el código. Unas estaban en el tema hijo (`download-results.js`) y otras en el plugin **Áreas Asignadas**. Este plugin las reúne en un solo sitio:

- Una pantalla en wp-admin (**Webhooks de PDF**) para editarlas.
- Un **envío de prueba** con datos reales, que muestra qué responde cada webhook.

Va en un plugin aparte a propósito, para que el tema y Áreas Asignadas lean la URL del mismo lugar sin depender el uno del otro. Si se desactiva, cada uno vuelve a usar su URL de siempre y las descargas siguen funcionando.

---

## Informes que gestiona

| Slug | Informe | Dónde se usa | URL por defecto |
|---|---|---|---|
| `personal` | Resultado personal completo | Botón «Descargar resultado» en `/resultado-test-personal-completo/` | `…/webhook/pdf_inspira` |
| `area` | Informe general (área y empresa) | Botones «Informe general» de `/mi-cuenta/areas/`: el de cada fila y el del menú «Acciones» | `…/webhook/pdf_aprobandome` |
| `usuario` | Informe individual (trabajador) | Botón «Ver resultados» de cada usuario en el panel de áreas | `…/webhook/pdf_aprobandome` |

*Todas las URL están en `https://api.efrenmartinezortiz-ia.com`.*

El webhook `area` se usa para los dos botones de «Informe general», el de un área y el de toda la empresa. Si se cambia esa URL, cambian los dos.

Todo se define en `IPW_Settings::definitions()`. Para añadir un informe nuevo basta con añadir una entrada ahí: el formulario y el probador se generan solos a partir de esa lista.

---

## Funcionalidades

### 1. Configuración de URL
- Menú **Webhooks de PDF** en wp-admin, visible solo con `manage_options`.
- Hay una tarjeta por informe, con:
  - El nombre del informe.
  - Dónde se usa.
  - El campo de la URL.
  - La URL por defecto como referencia.
- Las URL se guardan en la opción **`ipw_webhooks`** (un array por slug).
- **Valores por defecto:** mientras nadie guarde nada, cada informe usa la URL que había antes en el código. Por eso instalar el plugin no cambia nada. Si se deja un campo vacío y se guarda, ese informe vuelve a su URL por defecto.
- **Validación:** solo acepta URL `http`/`https` absolutas. Si una URL no es válida, no se guarda, se conserva la anterior y aparece un mensaje de error.
- Al guardar se abre un modal de «Cambios guardados».
- En esta pantalla se ocultan los avisos de otros plugins, para que no tapen los mensajes propios.

### 2. Cómo llegan las URL a los botones
- **Front-end:** en el `<head>` se imprime este objeto:
  ```js
  window.ipwPdfWebhooks = { personal: "…", area: "…", usuario: "…" };
  ```
  Solo contiene URL públicas de webhooks, ningún token.
- **PHP:** la función pública **`ipw_get_webhook( $slug )`** devuelve la URL del informe, o `''` si el slug no existe.
- Si falta una URL, los consumidores prueban en este orden:
  - **Tema hijo:** `ipwPdfWebhooks.personal` y, si no está, la URL escrita en el código.
  - **Áreas Asignadas:** `ipwPdfWebhooks[tipo]`, después `apAjax.webhook_pdf` y, por último, la URL escrita en el código.

### 3. Envío de prueba
- Cada tarjeta tiene un bloque **«Enviar prueba»** con tres selectores encadenados: **empresa → área → trabajador**. El selector de trabajador solo aparece en los informes `personal` y `usuario`.
- Solo se listan empresas, áreas y trabajadores **que ya respondieron el test**. Se exige que el usermeta `question_test_ids` exista y no esté vacío (`''`, `[]`, `null`, `{}`), para no generar PDF en blanco que hagan parecer que el webhook falla.
- El contenido que se envía es **el mismo que en producción**:
  - Informes de trabajador: `get_results_user( $user_id )`.
  - Informes de área: `get_results_area( $area_id )`.
- La petición sale **desde el servidor** (`wp_remote_post`, con 45 s de espera). Así se evitan los errores de CORS y se puede distinguir un error real del webhook de un bloqueo del navegador.
- Se usa la URL **guardada**, no la que esté escrita sin guardar en el campo. Así se prueba lo que está en producción.
- El resultado muestra:
  - Código HTTP y duración en ms.
  - La URL a la que se envió.
  - El cuerpo de la respuesta, recortado a 4 000 caracteres y mostrado como texto.
  - Si la respuesta trae `pdfUrl`, un enlace **«Abrir el PDF generado»**.

### 4. Validaciones de la prueba
- Comprueba que el área pertenece a la empresa y que el trabajador pertenece al área.
- Si el tema hijo no define `get_results_user()` o `get_results_area()`, la prueba avisa y no envía datos inventados. Guardar URL sigue funcionando.
- Si el área o el trabajador no devuelven resultados, avisa en lugar de enviar.

---

## Seguridad
- Para configurar o probar hace falta la capacidad `manage_options`.
- Los tres endpoints AJAX (`ipw_get_areas`, `ipw_get_trabajadores` e `ipw_probar_envio`) comprueban la capacidad y el nonce `ipw_test`.
- Todas las consultas SQL usan `$wpdb->prepare()`.
- La respuesta del webhook se muestra como texto, nunca como HTML.

## Arquitectura

```
inspira-pdf-webhooks/
├── inspira-pdf-webhooks.php     # Bootstrap, ipw_get_webhook(), config en <head>
├── includes/
│   ├── class-ipw-settings.php   # Definiciones, opción ipw_webhooks, saneado
│   ├── class-ipw-data.php       # SQL de empresas/áreas/trabajadores con test
│   ├── class-ipw-tester.php     # Endpoints AJAX y envío de prueba
│   └── class-ipw-admin.php      # Menú, pantalla, assets
└── assets/
    ├── admin.css
    └── admin.js                 # Selectores en cascada y panel de resultado
```

## Dependencias
- Tablas `{prefix}areas` y `{prefix}user_members`, del ecosistema de Áreas Asignadas.
- `get_results_user()` y `get_results_area()` del tema hijo. Solo hacen falta para el envío de prueba.
- Si existe el ajuste `rol_empresa_admin` en la opción `ap_ajustes` de Áreas Asignadas, se usa ese rol. Si no, se usa `empresa_admin`.
