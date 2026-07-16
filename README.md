# AI Developer Experience PoC

## RAG + MCP + GitHub Copilot sobre una API Mock

> Proyecto educativo que demuestra como integrar RAG, MCP y algun modelo o asistente como Claude o Copilot para transformar documentacion tecnica en un asistente inteligente capaz de responder preguntas y ejecutar acciones reales, todo desde el editor.

---

## Tabla de contenidos

1. [Que es este proyecto](#que-es-este-proyecto)
2. [Arquitectura](#arquitectura)
3. [Prerequisitos](#prerequisitos)
4. [Instalacion](#instalacion)
5. [Inicio rapido con tmux](#inicio-rapido-con-tmux)
6. [Verificacion por capas](#verificacion-por-capas)
7. [Conectar con clientes MCP](#conectar-con-clientes-mcp)
8. [Prompts de prueba](#prompts-de-prueba)
9. [Como extender el sistema](#como-extender-el-sistema)
10. [Troubleshooting](#troubleshooting)

---

## Que es este proyecto

Este PoC demuestra una evolucion en cuatro etapas de la experiencia del desarrollador:

```
Etapa 1   Documentacion Markdown estatica
    ↓
Etapa 2   Documentacion + RAG semantico → el conocimiento responde preguntas
    ↓
Etapa 3   RAG + MCP Server → el asistente tambien puede ejecutar acciones
    ↓
Etapa 4   Todo integrado en Copilot / Claude / Cline → sin salir del editor
```

**Principio fundamental:**

| Componente | Responsabilidad |
|---|---|
| **RAG** | Responde preguntas usando busqueda semantica sobre documentacion |
| **MCP** | Ejecuta acciones reales: crear ordenes, procesar pagos, consultar estado |
| **Copilot / Claude / Cline** | Interfaz conversacional que orquesta todo |

Para la arquitectura tecnica detallada ver [ARCHITECTURE.md](./ARCHITECTURE.md).

---

## Arquitectura

```
Developer
    │
    ▼
Cliente MCP (Copilot / Claude Desktop / Cline)
    │
    │  Streamable HTTP — POST /mcp
    ▼
MCP Server  :3001
    │                    │
    ▼                    ▼
RAG Engine          API Mock  :3000
    │                    │
    ▼                    ▼
index.json          /orders
(embeddings)        /payments
```

---

## Prerequisitos

### Node.js 22+

```bash
node --version   # debe mostrar v22.x.x o superior
```

Descarga: https://nodejs.org

---

### Ollama

Ollama corre los modelos de embeddings de forma local.

```bash
# Instalar (macOS)
brew install ollama

# Verificar
ollama --version
```

Descarga para otros sistemas: https://ollama.com

---

### Modelo de embeddings

```bash
ollama pull nomic-embed-text

# Verificar que el modelo fue descargado
ollama list
# debe mostrar: nomic-embed-text:latest
```

---

### VS Code + GitHub Copilot (opcional — solo para Fase 6)

- VS Code: https://code.visualstudio.com
- Extension GitHub Copilot instalada y activada

---

## Instalacion

### 1. Instalar dependencias de cada modulo

```bash
# API Mock
cd api && npm install

# RAG
cd ../rag && npm install

# MCP Server
cd ../mcp-server && npm install
```

---

### 2. Configurar variables de entorno

Los archivos `.env` ya estan incluidos con valores por defecto para desarrollo local. Si necesitas cambiar algo, copia los `.env.example`:

```bash
# rag/
cp rag/.env.example rag/.env

# mcp-server/
cp mcp-server/.env.example mcp-server/.env
```

Variables disponibles:

| Archivo | Variable | Default | Descripcion |
|---|---|---|---|
| `rag/.env` | `OLLAMA_BASE_URL` | `http://localhost:11434` | URL de Ollama |
| `mcp-server/.env` | `API_BASE_URL` | `http://localhost:3000` | URL de la API Mock |
| `mcp-server/.env` | `PORT` | `3001` | Puerto del MCP Server |

---

### 3. Construir el indice RAG

Este paso genera los embeddings de la documentacion y los persiste en `index.json`. Solo necesitas correrlo una vez, o cuando modifiques archivos en `api/docs/`.

```bash
# Ollama debe estar corriendo antes de este paso
ollama serve

# En otra terminal:
cd rag && npm run build-index
```

Salida esperada:
```
Building index from docs...
Index saved to /ruta/al/proyecto/index.json (12 chunks)
Done. 12 chunks indexed.
```

---

## Inicio rapido con tmux

Si tienes `tmux` instalado, el script `start.sh` levanta todos los servicios en paneles separados con un solo comando:

```bash
./start.sh
```

Esto crea el siguiente layout en tu terminal:

```
┌─────────────────────────┬─────────────────────────┐
│   API Mock              │   MCP Server            │
│   localhost:3000        │   localhost:3001/mcp    │
│                         │                         │
│   > npm run dev         │   > npm run dev         │
└─────────────────────────┴─────────────────────────┘
```

**Controles tmux:**

| Accion | Teclas |
|---|---|
| Moverse entre paneles | `Ctrl+B` luego flecha direccional |
| Salir sin matar servicios | `Ctrl+B` luego `D` |
| Ver sesiones activas | `tmux ls` |
| Reconectar a la sesion | `tmux attach -t mcp-rag-test` |

Para detener todos los servicios:

```bash
./stop.sh
```

### Instalar tmux (si no lo tienes)

```bash
# macOS
brew install tmux

# Ubuntu / Debian
sudo apt install tmux
```

---

### Sin tmux — inicio manual

Si prefieres no usar tmux, abre tres terminales separadas:

```bash
# Terminal 1 — API Mock
cd api && npm run dev

# Terminal 2 — MCP Server
cd mcp-server && npm run dev

# Terminal 3 — disponible para busquedas RAG directas
cd rag && npm run search -- "tu pregunta"
```

---

## Verificacion por capas

Una vez que los servicios estan corriendo, verifica cada capa individualmente.

---

### Capa 1 — API Mock

```bash
# Health check
curl http://localhost:3000/health
```

Respuesta esperada:
```json
{ "status": "ok" }
```

```bash
# Crear una orden
curl -X POST http://localhost:3000/orders
```

Respuesta esperada:
```json
{
  "orderId": "ord_abc123",
  "status": "created",
  "createdAt": "2026-01-01T00:00:00Z"
}
```

```bash
# Consultar una orden (reemplaza el ID)
curl http://localhost:3000/orders/ord_abc123

# Crear un pago
curl -X POST http://localhost:3000/payments \
  -H "Content-Type: application/json" \
  -d '{"orderId": "ord_abc123"}'
```

---

### Capa 2 — RAG

```bash
cd rag

# Busqueda semantica directa
npm run search -- "What statuses can an order return?"

# Otra busqueda de prueba
npm run search -- "When should I create an MCP tool?"
```

Resultado esperado: array JSON con fragmentos relevantes de la documentacion.

---

### Capa 3 — MCP Server

```bash
# Listar todos los tools disponibles
curl -s -X POST http://localhost:3001/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

Deberias ver los 5 tools: `health_check`, `search_docs`, `create_order`, `get_order`, `create_payment`.

```bash
# Verificar estado de todos los componentes
curl -s -X POST http://localhost:3001/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"health_check","arguments":{}}}'
```

Respuesta esperada:
```
✅ API Mock — http://localhost:3000
✅ RAG Index — 12 chunks loaded
✅ Ollama — http://localhost:11434
✅ Ollama model — nomic-embed-text

All systems operational.
```

---

## Conectar con clientes MCP

El MCP Server es independiente del cliente. Cualquier herramienta que implemente el protocolo MCP puede conectarse. **No hay vendor lock-in.**

---

### GitHub Copilot (VS Code)

La configuracion ya esta incluida en `.vscode/mcp.json`:

```json
{
  "servers": {
    "mcp-rag-test": {
      "type": "http",
      "url": "http://localhost:3001/mcp"
    }
  }
}
```

**Pasos para activar:**

1. Abre VS Code en la raiz del proyecto
2. Asegurate de que el MCP Server este corriendo
3. Abre Copilot Chat (`Ctrl+Alt+I` / `Cmd+Alt+I`)
4. Cambia al modo **Agent** en el selector de modo
5. Los tools aparecen automaticamente

---

### Claude Desktop

**Edita el archivo de configuracion:**

```bash
# macOS — abre la carpeta de configuracion
open ~/Library/Application\ Support/Claude/
```

Agrega o edita `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "mcp-rag-test": {
      "type": "http",
      "url": "http://localhost:3001/mcp"
    }
  }
}
```

**Nota:** el MCP Server debe estar corriendo antes de abrir Claude Desktop. Si lo inicias despues, reinicia Claude Desktop.

**Verificacion:**

1. Abre Claude Desktop
2. Inicia una nueva conversacion
3. Los tools aparecen disponibles en el chat

---

### Cline / Roo Code (VS Code)

**Instalacion de la extension:**

- Cline: busca `saoudrizwan.claude-dev` en el marketplace de VS Code
- Roo Code: busca `RooVeterinaryInc.roo-cline` en el marketplace de VS Code

**Configuracion:**

1. Abre la extension en VS Code
2. Ve a **Settings** → **MCP Servers**
3. Agrega un nuevo servidor:

```json
{
  "mcp-rag-test": {
    "type": "http",
    "url": "http://localhost:3001/mcp",
    "disabled": false
  }
}
```

**Verificacion:**

Los tools `health_check`, `search_docs`, `create_order`, `get_order` y `create_payment` apareceran listados en el panel de la extension.

---

### Tabla resumen de clientes

| Cliente | Tipo | Archivo de config | Modo |
|---|---|---|---|
| GitHub Copilot | VS Code extension | `.vscode/mcp.json` | Agent mode |
| Claude Desktop | App nativa | `~/Library/Application Support/Claude/claude_desktop_config.json` | Chat normal |
| Cline | VS Code extension | Panel Settings de la extension | Auto (agente) |
| Roo Code | VS Code extension | Panel Settings de la extension | Auto (agente) |

---

## Prompts de prueba

### Consultas RAG

| Prompt | Tool invocado | Tipo |
|---|---|---|
| `What statuses can an order return?` | `search_docs` | Conocimiento de API |
| `When should I create an MCP tool instead of using RAG?` | `search_docs` | Guidelines de IA |
| `How do I add a new endpoint to the API?` | `search_docs` | Guia de desarrollo |
| `How is the system architecture organized?` | `search_docs` | Arquitectura |

### Acciones sobre la API

| Prompt | Tool(s) invocados | Tipo |
|---|---|---|
| `Create a new order` | `create_order` | Accion simple |
| `Check the status of order ord_abc123` | `get_order` | Consulta con parametro |
| `Process a payment for order ord_abc123` | `create_payment` | Accion con input |
| `Are all systems working?` | `health_check` | Diagnostico |

### Flujos multi-accion

| Prompt | Que hace el agente |
|---|---|
| `Create an order and process its payment` | `create_order` → extrae `orderId` → `create_payment` |
| `Create an order, pay it, and check its final status` | `create_order` → `create_payment` → `get_order` |

---

## Como extender el sistema

### Agregar documentacion al RAG

1. Crea archivos `.md` en `api/docs/` en la subcarpeta correspondiente
2. Reconstruye el indice:

```bash
cd rag && npm run build-index
```

3. Verifica con una busqueda:

```bash
npm run search -- "tema de tu nuevo documento"
```

---

### Agregar un nuevo tool MCP

1. Crea `mcp-server/src/tools/mi-tool.ts`:

```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { z } from 'zod';

export function registerMiTool(server: McpServer) {
  server.registerTool(
    'mi_tool',
    {
      title: 'Mi Tool',
      description: 'Descripcion clara de lo que hace este tool.',
      inputSchema: {
        parametro: z.string().describe('Descripcion del parametro'),
      },
    },
    async ({ parametro }) => {
      return {
        content: [{ type: 'text', text: `Resultado: ${parametro}` }],
      };
    }
  );
}
```

2. Registralo en `mcp-server/src/server.ts`:

```typescript
import { registerMiTool } from './tools/mi-tool.js';
registerMiTool(server);
```

3. Reinicia el MCP Server — el tool estara disponible en todos los clientes.

---

## Troubleshooting

### El MCP Server no arranca

```bash
# Verifica que el puerto 3001 no este ocupado
lsof -i :3001

# Si hay algo corriendo, terminalo
kill -9 <PID>
```

---

### search_docs no retorna resultados

```bash
# Verifica Ollama
curl http://localhost:11434/api/tags

# Reconstruye el indice
cd rag && npm run build-index
```

---

### Claude Desktop o Cline no detectan el servidor

- Verifica que el MCP Server este corriendo: `curl http://localhost:3001/mcp`
- Reinicia el cliente despues de guardar la configuracion
- Confirma que la URL sea exactamente `http://localhost:3001/mcp`

---

### El modelo de Ollama no esta disponible

```bash
ollama list                        # lista modelos instalados
ollama pull nomic-embed-text       # descarga el modelo
```

---

### El indice RAG esta desactualizado

Si agregaste o modificaste archivos en `api/docs/` y los cambios no aparecen en busquedas:

```bash
cd rag && npm run build-index
```

---

## Estructura del repositorio

```
mcp-rag-test/
├── README.md               # Esta guia
├── ARCHITECTURE.md         # Documentacion tecnica detallada
├── POST.md                 # Articulo tecnico (espanol)
├── POST.en.md              # Articulo tecnico (ingles)
├── start.sh                # Inicia todos los servicios con tmux
├── stop.sh                 # Detiene la sesion tmux
├── index.json              # Cache del indice RAG (generado — no editar)
│
├── api/                    # Mock REST API (:3000)
│   └── docs/               # Base de conocimiento Markdown
├── rag/                    # Motor RAG con persistencia
├── mcp-server/             # MCP Server (:3001) con 5 tools
└── .vscode/
    └── mcp.json            # Configuracion para GitHub Copilot
```






_______________________


# AI Developer Experience PoC

## RAG + MCP + GitHub Copilot on a Mock API

> Educational project demonstrating how to integrate RAG, MCP, and claude or some assitan like copilot to transform technical documentation into an intelligent assistant capable of answering questions and executing real actions, all from within the editor.

---

## Table of Contents

1. [What is this project](#what-is-this-project)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Installation](#installation)
5. [Quick start with tmux](#quick-start-with-tmux)
6. [Layer verification](#layer-verification)
7. [Connect with MCP clients](#connect-with-mcp-clients)
8. [Test prompts](#test-prompts)
9. [How to extend the system](#how-to-extend-the-system)
10. [Troubleshooting](#troubleshooting)

---

## What is this project

This PoC demonstrates a four-stage evolution of the developer experience:

```
Stage 1   Static Markdown documentation
    ↓
Stage 2   Documentation + semantic RAG → knowledge answers questions
    ↓
Stage 3   RAG + MCP Server → agent can also execute actions
    ↓
Stage 4   Everything integrated in Copilot / Claude / Cline → without leaving the editor
```

**Fundamental principle:**

| Component | Responsibility |
|---|---|
| **RAG** | Answers questions using semantic search over documentation |
| **MCP** | Executes real actions: create orders, process payments, check status |
| **Copilot / Claude / Cline** | Conversational interface that orchestrates everything |

For detailed technical architecture, see [ARCHITECTURE.md](./ARCHITECTURE.md).

---

## Architecture

```
Developer
    │
    ▼
MCP Client (Copilot / Claude Desktop / Cline)
    │
    │  Streamable HTTP — POST /mcp
    ▼
MCP Server  :3001
    │                    │
    ▼                    ▼
RAG Engine          Mock API  :3000
    │                    │
    ▼                    ▼
index.json          /orders
(embeddings)        /payments
```

---

## Prerequisites

### Node.js 22+

```bash
node --version   # should display v22.x.x or higher
```

Download: https://nodejs.org

---

### Ollama

Ollama runs embedding models locally.

```bash
# Install (macOS)
brew install ollama

# Verify
ollama --version
```

Download for other systems: https://ollama.com

---

### Embedding model

```bash
ollama pull nomic-embed-text

# Verify the model was downloaded
ollama list
# should display: nomic-embed-text:latest
```

---

### VS Code + GitHub Copilot (optional — only for Stage 6)

- VS Code: https://code.visualstudio.com
- GitHub Copilot extension installed and activated

---

## Installation

### 1. Install dependencies for each module

```bash
# API Mock
cd api && npm install

# RAG
cd ../rag && npm install

# MCP Server
cd ../mcp-server && npm install
```

---

### 2. Configure environment variables

The `.env` files are already included with default values for local development. If you need to change something, copy from `.env.example`:

```bash
# rag/
cp rag/.env.example rag/.env

# mcp-server/
cp mcp-server/.env.example mcp-server/.env
```

Available variables:

| File | Variable | Default | Description |
|---|---|---|---|
| `rag/.env` | `OLLAMA_BASE_URL` | `http://localhost:11434` | Ollama URL |
| `mcp-server/.env` | `API_BASE_URL` | `http://localhost:3000` | Mock API URL |
| `mcp-server/.env` | `PORT` | `3001` | MCP Server port |

---

### 3. Build the RAG index

This step generates embeddings of the documentation and persists them in `index.json`. You only need to run it once, or when you modify files in `api/docs/`.

```bash
# Ollama must be running before this step
ollama serve

# In another terminal:
cd rag && npm run build-index
```

Expected output:
```
Building index from docs...
Index saved to /path/to/project/index.json (12 chunks)
Done. 12 chunks indexed.
```

---

## Quick start with tmux

If you have `tmux` installed, the `start.sh` script launches all services in separate panels with a single command:

```bash
./start.sh
```

This creates the following layout in your terminal:

```
┌─────────────────────────┬─────────────────────────┐
│   API Mock              │   MCP Server            │
│   localhost:3000        │   localhost:3001/mcp    │
│                         │                         │
│   > npm run dev         │   > npm run dev         │
└─────────────────────────┴─────────────────────────┘
```

**Tmux controls:**

| Action | Keys |
|---|---|
| Switch between panels | `Ctrl+B` then arrow keys |
| Exit without killing services | `Ctrl+B` then `D` |
| View active sessions | `tmux ls` |
| Reconnect to session | `tmux attach -t mcp-rag-test` |

To stop all services:

```bash
./stop.sh
```

### Install tmux (if you don't have it)

```bash
# macOS
brew install tmux

# Ubuntu / Debian
sudo apt install tmux
```

---

### Without tmux — manual startup

If you prefer not to use tmux, open three separate terminals:

```bash
# Terminal 1 — API Mock
cd api && npm run dev

# Terminal 2 — MCP Server
cd mcp-server && npm run dev

# Terminal 3 — available for direct RAG searches
cd rag && npm run search -- "your question"
```

---

## Layer verification

Once the services are running, verify each layer individually.

---

### Layer 1 — Mock API

```bash
# Health check
curl http://localhost:3000/health
```

Expected response:
```json
{ "status": "ok" }
```

```bash
# Create an order
curl -X POST http://localhost:3000/orders
```

Expected response:
```json
{
  "orderId": "ord_abc123",
  "status": "created",
  "createdAt": "2026-01-01T00:00:00Z"
}
```

```bash
# Query an order (replace the ID)
curl http://localhost:3000/orders/ord_abc123

# Create a payment
curl -X POST http://localhost:3000/payments \
  -H "Content-Type: application/json" \
  -d '{"orderId": "ord_abc123"}'
```

---

### Layer 2 — RAG

```bash
cd rag

# Direct semantic search
npm run search -- "What statuses can an order return?"

# Another test search
npm run search -- "When should I create an MCP tool?"
```

Expected result: JSON array with relevant documentation fragments.

---

### Layer 3 — MCP Server

```bash
# List all available tools
curl -s -X POST http://localhost:3001/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

You should see 5 tools: `health_check`, `search_docs`, `create_order`, `get_order`, `create_payment`.

```bash
# Verify status of all components
curl -s -X POST http://localhost:3001/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"health_check","arguments":{}}}'
```

Expected response:
```
✅ API Mock — http://localhost:3000
✅ RAG Index — 12 chunks loaded
✅ Ollama — http://localhost:11434
✅ Ollama model — nomic-embed-text

All systems operational.
```

---

## Connect with MCP clients

The MCP Server is client-independent. Any tool that implements the MCP protocol can connect. **No vendor lock-in.**

---

### GitHub Copilot (VS Code)

Configuration is already included in `.vscode/mcp.json`:

```json
{
  "servers": {
    "mcp-rag-test": {
      "type": "http",
      "url": "http://localhost:3001/mcp"
    }
  }
}
```

**Steps to activate:**

1. Open VS Code in the project root
2. Make sure the MCP Server is running
3. Open Copilot Chat (`Ctrl+Alt+I` / `Cmd+Alt+I`)
4. Switch to **Agent** mode in the mode selector
5. Tools appear automatically

---

### Claude Desktop

**Edit the configuration file:**

```bash
# macOS — open the configuration folder
open ~/Library/Application\ Support/Claude/
```

Add or edit `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "mcp-rag-test": {
      "type": "http",
      "url": "http://localhost:3001/mcp"
    }
  }
}
```

**Note:** The MCP Server must be running before opening Claude Desktop. If you start it later, restart Claude Desktop.

**Verification:**

1. Open Claude Desktop
2. Start a new conversation
3. Tools appear as available in the chat

---

### Cline / Roo Code (VS Code)

**Extension installation:**

- Cline: search for `saoudrizwan.claude-dev` in the VS Code marketplace
- Roo Code: search for `RooVeterinaryInc.roo-cline` in the VS Code marketplace

**Configuration:**

1. Open the extension in VS Code
2. Go to **Settings** → **MCP Servers**
3. Add a new server:

```json
{
  "mcp-rag-test": {
    "type": "http",
    "url": "http://localhost:3001/mcp",
    "disabled": false
  }
}
```

**Verification:**

The tools `health_check`, `search_docs`, `create_order`, `get_order`, and `create_payment` will be listed in the extension panel.

---

### Client summary table

| Client | Type | Config file | Mode |
|---|---|---|---|
| GitHub Copilot | VS Code extension | `.vscode/mcp.json` | Agent mode |
| Claude Desktop | Native app | `~/Library/Application Support/Claude/claude_desktop_config.json` | Normal chat |
| Cline | VS Code extension | Extension Settings panel | Auto (agent) |
| Roo Code | VS Code extension | Extension Settings panel | Auto (agent) |

---

## Test prompts

### RAG queries

| Prompt | Tool invoked | Type |
|---|---|---|
| `What statuses can an order return?` | `search_docs` | API knowledge |
| `When should I create an MCP tool instead of using RAG?` | `search_docs` | AI guidelines |
| `How do I add a new endpoint to the API?` | `search_docs` | Development guide |
| `How is the system architecture organized?` | `search_docs` | Architecture |

### Actions on the API

| Prompt | Tool(s) invoked | Type |
|---|---|---|
| `Create a new order` | `create_order` | Simple action |
| `Check the status of order ord_abc123` | `get_order` | Query with parameter |
| `Process a payment for order ord_abc123` | `create_payment` | Action with input |
| `Are all systems working?` | `health_check` | Diagnostics |

### Multi-action flows

| Prompt | What the agent does |
|---|---|
| `Create an order and process its payment` | `create_order` → extract `orderId` → `create_payment` |
| `Create an order, pay it, and check its final status` | `create_order` → `create_payment` → `get_order` |

---

## How to extend the system

### Add documentation to the RAG

1. Create `.md` files in `api/docs/` in the corresponding subfolder
2. Rebuild the index:

```bash
cd rag && npm run build-index
```

3. Verify with a search:

```bash
npm run search -- "topic of your new document"
```

---

### Add a new MCP tool

1. Create `mcp-server/src/tools/my-tool.ts`:

```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { z } from 'zod';

export function registerMyTool(server: McpServer) {
  server.registerTool(
    'my_tool',
    {
      title: 'My Tool',
      description: 'Clear description of what this tool does.',
      inputSchema: {
        parameter: z.string().describe('Parameter description'),
      },
    },
    async ({ parameter }) => {
      return {
        content: [{ type: 'text', text: `Result: ${parameter}` }],
      };
    }
  );
}
```

2. Register it in `mcp-server/src/server.ts`:

```typescript
import { registerMyTool } from './tools/my-tool.js';
registerMyTool(server);
```

3. Restart the MCP Server — the tool will be available to all clients.

---

## Troubleshooting

### MCP Server won't start

```bash
# Check if port 3001 is already in use
lsof -i :3001

# If something is running, kill it
kill -9 <PID>
```

---

### search_docs returns no results

```bash
# Check Ollama
curl http://localhost:11434/api/tags

# Rebuild the index
cd rag && npm run build-index
```

---

### Claude Desktop or Cline don't detect the server

- Verify the MCP Server is running: `curl http://localhost:3001/mcp`
- Restart the client after saving the configuration
- Confirm the URL is exactly `http://localhost:3001/mcp`

---

### Ollama model not available

```bash
ollama list                        # list installed models
ollama pull nomic-embed-text       # download the model
```

---

### RAG index is outdated

If you added or modified files in `api/docs/` and the changes don't appear in searches:

```bash
cd rag && npm run build-index
```

---

## Repository structure

```
mcp-rag-test/
├── README.md               # This guide
├── ARCHITECTURE.md         # Detailed technical documentation
├── POST.md                 # Technical article (Spanish)
├── POST.en.md              # Technical article (English)
├── start.sh                # Launch all services with tmux
├── stop.sh                 # Stop the tmux session
├── index.json              # RAG index cache (generated — do not edit)
│
├── api/                    # Mock REST API (:3000)
│   └── docs/               # Knowledge base Markdown
├── rag/                    # RAG engine with persistence
├── mcp-server/             # MCP Server (:3001) with 5 tools
└── .vscode/
    └── mcp.json            # Configuration for GitHub Copilot
```

