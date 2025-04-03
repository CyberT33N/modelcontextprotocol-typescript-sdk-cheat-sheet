# MCP TypeScript SDK Cheatsheet


# Good 2 Know
- Da der MCP-Server über STDIO läuft, ist nur JSON zulässig. Das heißt alle Arten von UTF-8-Symbolen und so weiter führen bei Konsole-Log, Warn-Error und so weiter zum Absturz des Servers.


---


# MCP TypeScript SDK Cheatsheet

![NPM Version](https://img.shields.io/npm/v/%40modelcontextprotocol%2Fsdk) ![MIT licensed](https://img.shields.io/npm/l/%40modelcontextprotocol%2Fsdk)

## Inhaltsverzeichnis
- [Was ist MCP?](#was-ist-mcp)
- [Installation](#installation)
- [Grundlegende Konzepte](#grundlegende-konzepte)
  - [MCP-Server](#mcp-server)
  - [Ressourcen](#ressourcen)
  - [Tools](#tools)
  - [Prompts](#prompts)
- [Einen MCP-Server starten](#einen-mcp-server-starten)
  - [Über stdio (Kommandozeile)](#uber-stdio-kommandozeile)
  - [Über HTTP mit SSE](#uber-http-mit-sse)
- [Beispiele](#beispiele)
  - [Echo-Server](#echo-server)
  - [SQLite Explorer](#sqlite-explorer)

---

## Was ist MCP?

Das **Model Context Protocol (MCP)** ist ein Standard, mit dem Anwendungen **Daten und Funktionen für KI-Modelle bereitstellen** können. Man kann es sich wie eine API für Künstliche Intelligenz vorstellen.

Ein MCP-Server kann:
- **Daten bereitstellen** (Ressourcen, ähnlich wie GET-Anfragen in einer REST-API)
- **Funktionen anbieten** (Tools, ähnlich wie POST-Anfragen, um Aktionen auszuführen)
- **Interaktionsmuster definieren** (Prompts, vordefinierte Kommunikationsabläufe für KI-Modelle)

Der große Vorteil: Durch MCP kann man verschiedene LLMs einfach mit Kontext versorgen, ohne ihre interne Funktionsweise ändern zu müssen.

---

## Installation

Das MCP TypeScript SDK kann über NPM installiert werden:

```bash
npm install @modelcontextprotocol/sdk
```

---

## Grundlegende Konzepte

### MCP-Server

Ein MCP-Server ist das Herzstück der Architektur. Er stellt Ressourcen bereit, bietet Tools an und verarbeitet Anfragen von Clients.

So erstellt man einen MCP-Server:

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

const server = new McpServer({
  name: "Mein Server",
  version: "1.0.0"
});
```

Hier:
- **`name`** gibt dem Server einen Namen.
- **`version`** gibt an, welche Version des Servers läuft.

### Ressourcen

Ressourcen sind Daten, die der Server für KI-Modelle bereitstellt. Sie sind vergleichbar mit **GET-Endpunkten** in einer REST-API.

Ein statisches Beispiel:

```typescript
server.resource("config", "config://app", async (uri) => ({
  contents: [{ uri: uri.href, text: "App-Konfiguration" }]
}));
```

Ein dynamisches Beispiel:

```typescript
server.resource("user-profile",
  new ResourceTemplate("users://{userId}/profile", { list: undefined }),
  async (uri, { userId }) => ({
    contents: [{ uri: uri.href, text: `Profil für Nutzer ${userId}` }]
  })
);
```

Hier:
- **Die Ressource "config"** gibt eine feste Konfiguration zurück.
- **Die Ressource "user-profile"** nutzt `{userId}` als Platzhalter für Nutzer-IDs.

### Tools

Tools führen Aktionen aus, ähnlich wie **POST-Anfragen**. Sie berechnen Werte oder ändern Daten.

Ein einfaches Beispiel:

```typescript
server.tool("add",
  { a: z.number(), b: z.number() },
  async ({ a, b }) => ({
    content: [{ type: "text", text: String(a + b) }]
  })
);
```

Hier:
- Das Tool **"add"** nimmt zwei Zahlen `a` und `b` entgegen.
- Es gibt die Summe als Text zurück.

### Prompts

Prompts sind vordefinierte Anfragen für KI-Modelle, damit sie konsistente Antworten geben.

```typescript
server.prompt("review-code",
  { code: z.string() },
  ({ code }) => ({
    messages: [{
      role: "user",
      content: { type: "text", text: `Bitte überprüfe diesen Code:\n\n${code}` }
    }]
  })
);
```

Hier:
- Das Prompt **"review-code"** gibt einem KI-Modell den Auftrag, Code zu überprüfen.

---

## Einen MCP-Server starten

MCP-Server müssen mit einem Transport gestartet werden. Es gibt zwei gängige Möglichkeiten:

### Über stdio (Kommandozeile)

```typescript
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const transport = new StdioServerTransport();
await server.connect(transport);
```

Hier:
- **Stdio** bedeutet "Standard Input/Output" – der Server kommuniziert über die Kommandozeile.

### Über HTTP mit SSE

**SSE (Server-Sent Events)** ermöglicht es dem Server, in Echtzeit Nachrichten an den Client zu senden.

```typescript
import express from "express";
import { SSEServerTransport } from "@modelcontextprotocol/sdk/server/sse.js";

const app = express();
const server = new McpServer({ name: "example-server", version: "1.0.0" });

const transports = {};

app.get("/sse", async (req, res) => {
  const transport = new SSEServerTransport("/messages", res);
  transports[transport.sessionId] = transport;
  res.on("close", () => delete transports[transport.sessionId]);
  await server.connect(transport);
});

app.post("/messages", async (req, res) => {
  const sessionId = req.query.sessionId;
  const transport = transports[sessionId];
  if (transport) await transport.handlePostMessage(req, res);
  else res.status(400).send("Keine Verbindung gefunden");
});

app.listen(3001);
```

Hier:
- **SSE** wird verwendet, damit Clients den Server-Updates in Echtzeit lauschen können.
- **Ein Express-Server** stellt `/sse` für Verbindungen und `/messages` für Client-Anfragen bereit.

---

## Beispiele

### Echo-Server

Ein einfacher Server, der Nachrichten zurücksendet:

```typescript
server.resource("echo", new ResourceTemplate("echo://{message}", { list: undefined }),
  async (uri, { message }) => ({
    contents: [{ uri: uri.href, text: `Echo: ${message}` }]
  })
);
```

### SQLite Explorer

Ein Server, der Daten aus einer SQLite-Datenbank abfragt:

```typescript
import sqlite3 from "sqlite3";
import { promisify } from "util";

const db = new sqlite3.Database("database.db");
const getAll = promisify(db.all.bind(db));

server.tool("query", { sql: z.string() }, async ({ sql }) => {
  try {
    const results = await getAll(sql);
    return { content: [{ type: "text", text: JSON.stringify(results, null, 2) }] };
  } catch (err) {
    return { content: [{ type: "text", text: `Fehler: ${err.message}` }], isError: true };
  }
});
```

Hier:
- Der Server nimmt SQL-Abfragen entgegen und gibt die Ergebnisse zurück.

