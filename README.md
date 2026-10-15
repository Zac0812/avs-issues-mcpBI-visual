# MCP Power BI Modeling — Guía rápida

Configuración de **Power BI Modeling MCP Server** para Claude Code en VS Code.

> Al final están las notas personales con los atajos para arreglar el bug de mayúscula de unidad.

Versión vigente probada: **1.0.0** (14-oct-2026).

---

## 1. Instalar

Extensión de VS Code **Power BI Modeling MCP Server** (Marketplace, `analysis-services.powerbi-modeling-mcp`, Microsoft). Se instala una sola vez por máquina, no por proyecto.

Para ver qué versión quedó instalada:

```powershell
Get-ChildItem "$env:USERPROFILE\.vscode\extensions" -Filter "analysis-services.powerbi-modeling-mcp-*" | Select-Object -ExpandProperty Name
```

## 2. Registrar (por cada proyecto nuevo, o tras actualizar la extensión)

1. Abre Power BI Desktop con el `.pbix`/`.pbip` del proyecto.
2. Abre la carpeta del proyecto como workspace de VS Code.
3. Desde el **Bash/PowerShell tool de una sesión de Claude Code ya abierta en esa ventana** (no desde una terminal externa):

```powershell
claude mcp add powerbi-modeling-mcp -- "C:\Users\luis.meza\.vscode\extensions\analysis-services.powerbi-modeling-mcp-1.0.0-win32-x64\server\powerbi-modeling-mcp.exe" --start --skip-confirmation --accept-eula
```

- Ajusta `1.0.0` a la carpeta real instalada (comando del paso 1).
- `--accept-eula` evita que el server bloquee las tools pidiendo aceptar la licencia (ver sección 5). Úsalo solo si ya revisaste el EULA: https://go.microsoft.com/fwlink/?LinkId=2381247

> ⚠️ **Por qué "desde dentro de la sesión"**: si lo registras desde otra terminal, el `cwd` puede quedar guardado con distinta mayúscula de unidad (`C:` vs `c:`) a la que usa la sesión de chat, y esta nunca verá el server. Registrar desde el propio tool de Claude Code evita ese desfase.

> ⚠️ **Cuando VS Code actualiza la extensión**, la carpeta con la versión vieja desaparece y el MCP deja de cargar. Hay que remover y volver a registrar con la ruta nueva:
>
> ```powershell
> claude mcp remove powerbi-modeling-mcp -s local
> claude mcp add powerbi-modeling-mcp -- "<ruta con la versión nueva>" --start --skip-confirmation --accept-eula
> ```

## 3. Validar

```powershell
claude mcp list
claude mcp get powerbi-modeling-mcp
```

Ambos deben mostrar el servidor `√ Connected` y la ruta con la versión vigente. Luego, **en una conversación nueva** de Claude Code (no basta `claude mcp list` en una terminal aparte; puede decir "Connected" aunque la sesión de chat no vea nada):

> Conéctate a `<archivo>.pbix` en Power BI Desktop. Crea una medida de prueba `Test = 1` y luego bórrala.

Si funciona sin pedir confirmación adicional, quedó listo.

## 4. Usar

Lenguaje natural, por ejemplo:

- Conéctate a `P&L.pbix` en Power BI Desktop
- Lista las medidas de la tabla X
- Crea la medida `<nombre> = <DAX>` en la tabla `<tabla>`
- Ejecuta este DAX y dame las métricas de memoria/tiempo

Si hay varios `.pbix` abiertos, cada uno tiene su propio puerto. Claude debe verificar a cuál se conectó (por ejemplo, listando tablas) antes de escribir.

## 5. Si algo falla — revisar en este orden

1. **Las tools responden "The Power BI Authoring MCP EULA must be accepted…"** → desde la versión 1.0.0 hay que aceptar la licencia de Microsoft.
   - Opción rápida: revisar el EULA y decirle a Claude en el chat *"revisé el EULA, acéptalo"*. Claude llama a la tool `accept_eula`.
   - Opción permanente: registrar con `--accept-eula` (sección 2) o definir la variable de entorno `PBI_MODELING_MCP_ACCEPT_EULA=true`.
   - Nota: justo después de aceptar, la primera llamada puede volver a fallar con el mismo mensaje; basta repetirla.
2. **El MCP no carga / no aparece tras una actualización de la extensión** → la ruta registrada apunta a una versión que ya no existe. Revisa con `claude mcp get powerbi-modeling-mcp` y vuelve a registrar (sección 2).
3. **Pide confirmación pese a `--skip-confirmation`** → el flag solo aplica al reiniciar el proceso del server. Cierra la conversación de Claude Code y abre una nueva (no hace falta re-registrar).
4. **Ninguna tool aparece ni en una sesión nueva** → bug de mayúscula de unidad en `~/.claude.json`. Busca la ruta del proyecto ahí (`grep -n "<ruta>" ~/.claude.json`); si hay dos entradas (`C:` y `c:`) con `mcpServers` distinto, copia el bloque de la que funciona a la que está vacía (`"mcpServers": {}`) y abre sesión nueva. Ver notas personales abajo.

---

## Notas personales: dos métodos rápidos para arreglar el bug de mayúscula de unidad

Para cuando `~/.claude.json` queda con la entrada `C:\...` poblada y la `c:\...` vacía (`"mcpServers": {}`).

### Método 1 — "truco del chat interno" (recomendado, más fácil)

1. En una terminal de Windows normal (PowerShell), borra el registro viejo:
   `claude mcp remove powerbi-modeling-mcp`
2. Vuelve a VS Code y abre una sesión de chat de Claude Code en el panel integrado.
3. En vez de correr el comando tú mismo, pídele a Claude en el chat que lo ejecute con su propia herramienta de terminal, por ejemplo:

   > Por favor, ejecuta este comando usando tu herramienta de terminal interna para registrar mi servidor MCP:
   > `claude mcp add powerbi-modeling-mcp -- "C:\Users\luis.meza\.vscode\extensions\analysis-services.powerbi-modeling-mcp-1.0.0-win32-x64\server\powerbi-modeling-mcp.exe" --start --skip-confirmation --accept-eula`

4. Al ejecutarlo desde su propio entorno, Claude hereda el `cwd` con la `c:` minúscula de VS Code, y el registro queda en el casillero correcto del JSON.

### Método 2 — quirúrgico (editar el JSON a mano)

1. Abre `C:\Users\luis.meza\.claude.json` (archivo oculto; OJO: en `luis.meza` debes abrir el **archivo**, no entrar a la carpeta `.claude`) con el Bloc de notas o VS Code.
2. En la sección `projects`, localiza los dos bloques con la misma ruta: uno con `C:` y otro con `c:`.
3. El bloque `C:` tiene el `mcpServers` poblado; el bloque `c:` tiene `"mcpServers": {}` vacío.
4. Copia el contenido de `mcpServers` del bloque `C:` y pégalo dentro del bloque `c:`.
5. Guarda el archivo.

### Paso final obligatorio para ambos métodos

1. Reinicia la ventana de VS Code (`Ctrl+R` o `F1` → *Developer: Reload Window*) para matar el proceso viejo del server MCP colgado en memoria.
2. Abre una conversación **completamente nueva** de Claude Code; la sesión ya abierta no recoge el cambio aunque el JSON ya esté corregido.

> Para remover el server: `claude mcp remove powerbi-modeling-mcp -s local`
