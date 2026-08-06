# avs-issues-mcpBI-visual
Configuración de **Power BI Modeling MCP Server**  en VS Code 


# MCP Power BI Modeling — Guía rápida

(Versión resumida. Detalle completo y explicación de cada bug en `instrucciones-de-mcp.md`.)

Ojo: Ver al final las notas personales donde se explica de manera rapida como anexar el mcp a la sesión de visual studio code.

## 1. Instalar

Extensión de VS Code **Power BI Modeling MCP Server** (Marketplace, `analysis-services.powerbi-modeling-mcp`, Microsoft). Una sola vez por máquina, no por proyecto.

## 2. Registrar (por cada proyecto nuevo)

1. Abre **Power BI Desktop** con el `.pbix`/`.pbip` del proyecto.
2. Abre la carpeta del proyecto como workspace de VS Code.
3. **Desde el Bash/PowerShell tool de una sesión de Claude Code ya abierta en esa ventana** (no una terminal externa del sistema):

```powershell
claude mcp add powerbi-modeling-mcp -- "C:\Users\luis.meza\.vscode\extensions\analysis-services.powerbi-modeling-mcp-<version>-win32-x64\server\powerbi-modeling-mcp.exe" --start --skip-confirmation
```

Ajusta `<version>` a la carpeta real instalada (`Get-ChildItem "$env:USERPROFILE\.vscode\extensions" -Filter "analysis-services.powerbi-modeling-mcp-*"`).

⚠️ **Por qué "desde dentro de la sesión" y no desde una terminal aparte**: si lo registras desde otra terminal, el cwd puede quedar guardado con distinta mayúscula de unidad (`C:` vs `c:`) a la que usa realmente la sesión de chat, y esta nunca verá el server. Registrar desde el propio tool de Claude Code evita ese desfase.

## 3. Validar

```powershell
claude mcp list
claude mcp get powerbi-modeling-mcp
```


Ambos deben mostrar el servidor conectado. Luego, en la conversación con Claude Code, basta pedir en
lenguaje natural: `Conéctate a 'P&L.pbix' en Power BI Desktop`.


**En esa misma sesión** (no en `claude mcp list` de una terminal aparte — puede decir "Connected" aunque la sesión de chat no vea nada):

```
Conecta a '<archivo>.pbix' en Power BI Desktop. Crea una medida de prueba Test = 1 y luego bórrala.
```

Si funciona sin pedir confirmación adicional, quedó listo.

## 4. Usar

Lenguaje natural, ejemplos:
- `Conecta a 'P&L.pbix' en Power BI Desktop`
- `Lista las medidas de la tabla X`
- `Crea la medida <nombre> = <DAX> en la tabla <tabla>`
- `Ejecuta este DAX y dame las métricas de memoria/tiempo`

## 5. Si algo falla — dos causas, en este orden

1. **Pide confirmación pese a `--skip-confirmation`** → el flag solo aplica al *reiniciar el proceso* del server. Cierra la conversación de Claude Code y abre una nueva (no hace falta re-registrar).
2. **Ninguna tool aparece ni en una sesión nueva** → bug de mayúscula de unidad en `~/.claude.json`. Busca la ruta del proyecto ahí (`grep -n "<ruta>" ~/.claude.json`); si hay dos entradas (`C:` y `c:`) con `mcpServers` distinto, copia el bloque de la que funciona a la que está vacía (`"mcpServers": {}`) y abre sesión nueva.

Detalle completo de ambos bugs, con logs y comandos de diagnóstico exactos, en `instrucciones-de-mcp.md`.




# Notas personales: dos métodos rápidos para arreglar el bug de mayúscula de unidad

Resumen práctico (ver el detalle completo del bug en Troubleshooting arriba) con los dos atajos que
me funcionaron para corregir el registro cuando `~/.claude.json` queda con la entrada `C:\...`
poblada y la `c:\...` vacía (`"mcpServers": {}`).

**Método 1 — "truco del chat interno" (recomendado, más fácil):**
1. En una terminal de Windows normal (PowerShell), borrar el registro viejo:
   `claude mcp remove powerbi-modeling-mcp`
2. Volver a VS Code, abrir una sesión de chat de Claude Code en el panel integrado.
3. En vez de correr el comando yo mismo, pedirle a Claude en el chat que lo ejecute él con su propia
   herramienta de terminal (Bash/PowerShell tool), por ejemplo:
```
Por favor, ejecuta este comando usando tu herramienta de terminal interna (terminal/bash tool) para registrar mi servidor MCP: claude mcp add powerbi-modeling-mcp -- "C:\Users\luis.meza\.vscode\extensions\analysis-services.powerbi-modeling-mcp-0.4.0-win32-x64\server\powerbi-modeling-mcp.exe" --start --skip-confirmation
```


1. Al ejecutarlo desde su propio entorno, Claude hereda automáticamente el cwd con la `c:` minúscula
   de VS Code, y el registro queda guardado en el casillero correcto del JSON.

**Método 2 — quirúrgico (editar el JSON a mano):**
1. Abrir `C:\Users\luis.meza\.claude.json` (archivo oculto) con el Bloc de notas o VS Code.
2. Buscar la sección `projects` y localizar los dos bloques con la misma ruta: uno con `C:` y otro
   con `c:`.
3. El bloque `C:` tiene el `mcpServers` poblado; el bloque `c:` tiene `"mcpServers": {}` vacío.
4. Copiar el contenido de `mcpServers` del bloque `C:` y pegarlo dentro del bloque `c:`.
5. Guardar el archivo.

**Paso final obligatorio para ambos métodos:**
- Reiniciar la ventana de VS Code (`Ctrl+R` o `F1` → `Developer: Reload Window`) para matar el
  proceso viejo del servidor MCP colgado en memoria.
- Abrir una conversación completamente **nueva** de Claude Code — la sesión ya abierta no recoge el
  cambio aunque el JSON ya esté corregido.


  Nota: Para remover podemos usar `claude mcp remove powerbi-modeling-mcp`

