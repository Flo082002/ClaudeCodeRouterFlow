# ClaudeCodeRouterFlow
Hier eine kleine ergänzung zum Patreon video von The Morpheus Tutorials (https://www.patreon.com/posts/work-in-progress-136000549) zum verwenden von Claude Flow mit Claude Router zusammen.



# Claude Code Router mit Claude Flow

Diese Anleitung beschreibt, wie du das Plugin **stripReasoningConflict** für den Claude Code Router einrichtest.  

---

## 0. Claude Code, Claude Code Router und Claude Flow installiren. Dann die Anleitung befolgen https://www.patreon.com/posts/work-in-progress-136000549

---

## 1. Plugin-Verzeichnis erstellen

Im Verzeichnis `.claude-code-router` einen neuen Ordner `plugins` anlegen:

```bash
mkdir -p ~/.claude-code-router/plugins
```

---

## 2. Plugin-Datei erstellen

Im Ordner `plugins` die Datei `stripReasoningConflict.cjs` erstellen und **folgenden Code unverändert einfügen**:

```javascript
// ~/.claude-code-router/plugins/stripReasoningConflict.cjs
// CJS-Transformer: Fix reasoning-Konflikt + rekursives Patchen ALLER Tool-Parameter-Schemata
// - Entfernt reasoning.max_tokens, wenn effort gesetzt ist
// - Für jedes Tool: ergänzt fehlende "items" bei Arrays (inkl. nested)
// - Defensiver Re-Assign des Request-Bodys (Deep Clone), damit Patches garantiert ankommen
// - Dumps der Tool-Parameter nach ~/.claude-code-router/tool_params/tool{index}.json
// - Optional: Tools per ENV droppen: export CCR_DROP_TOOL_INDICES="32,42"

const fs = require("fs");
const path = require("path");

const LOGFILE = process.env.CCR_DEBUG_FILE || "~/.claude-code-router/stripReasoningConflict.log";
const DUMP_DIR = "~/.claude-code-router/tool_params";
const DROP_INDICES_ENV = process.env.CCR_DROP_TOOL_INDICES || ""; // z. B. "32,42"

function log(msg, obj) {
  try {
    fs.appendFileSync(
      LOGFILE,
      `[${new Date().toISOString()}] ${msg}${obj ? " " + JSON.stringify(obj) : ""}\n`
    );
  } catch {}
}

function ensureDir(p) {
  try { fs.mkdirSync(p, { recursive: true }); } catch {}
}

function deepClone(v) {
  try { return JSON.parse(JSON.stringify(v)); } catch { return v; }
}

function parseDropIndices(envStr) {
  return envStr
    .split(",")
    .map(s => s.trim())
    .filter(Boolean)
    .map(n => Number.parseInt(n, 10))
    .filter(Number.isFinite);
}

function stripReasoning(body) {
  if (!body || typeof body !== "object") return body;
  const r = body.reasoning;
  if (r && typeof r === "object") {
    const hasEffort = Object.prototype.hasOwnProperty.call(r, "effort");
    const hasMax    = Object.prototype.hasOwnProperty.call(r, "max_tokens");
    if (hasEffort && hasMax) { delete r.max_tokens; log("ACTION", { removed: "reasoning.max_tokens" }); }
    if (Object.keys(r).length === 0) { delete body.reasoning; log("ACTION", { removed: "empty reasoning" }); }
  }
  return body;
}

// Rekursiver Patch: überall dort, wo type:"array" aber kein items vorhanden ist,
// wird ein tolerantes items-Schema gesetzt. Zusätzlich werden verschachtelte Schemas traversiert.
function fixArraySchemas(schema, pathStr = "$") {
  if (!schema || typeof schema !== "object") return;
  if (schema.type === "array" && schema.items === undefined) {
    // Tolerantes Default, das OpenAI akzeptiert und selten stört:
    schema.items = { type: "object", additionalProperties: true };
    log("PATCH items added", { path: pathStr });
  }
  // Traverse gängige Keywords
  if (schema.properties && typeof schema.properties === "object") {
    for (const k of Object.keys(schema.properties)) {
      fixArraySchemas(schema.properties[k], `${pathStr}.properties.${k}`);
    }
  }
  for (const key of ["allOf","oneOf","anyOf"]) {
    if (Array.isArray(schema[key])) {
      schema[key].forEach((sub, i) => fixArraySchemas(sub, `${pathStr}.${key}[${i}]`));
    }
  }
  if (schema.items && typeof schema.items === "object") {
    fixArraySchemas(schema.items, `${pathStr}.items`);
  }
  if (schema.additionalProperties && typeof schema.additionalProperties === "object") {
    fixArraySchemas(schema.additionalProperties, `${pathStr}.additionalProperties`);
  }
}

// Patcht alle Tools (oder dropt bestimmte Indizes) und legt Dumps an
function patchAllToolsOnBody(body, phase) {
  const tools = body?.tools;
  if (!Array.isArray(tools)) return;

  // Tools droppen?
  const dropIndices = parseDropIndices(DROP_INDICES_ENV);
  if (dropIndices.length) {
    const removed = [];
    const kept = [];
    tools.forEach((t, i) => {
      if (dropIndices.includes(i)) {
        removed.push({ index: i, name: t?.function?.name || "<unknown>" });
      } else {
        kept.push(t);
      }
    });
    if (removed.length) {
      body.tools = kept;
      log("DROP tools by env", { removed });
    }
  }

  // Nach evtl. Drop erneut referenzieren
  const newTools = body.tools;
  if (!Array.isArray(newTools)) return;

  ensureDir(DUMP_DIR);

  newTools.forEach((tool, i) => {
    try {
      const params = tool?.function?.parameters;
      if (!params || typeof params !== "object") return;

      // Dump vor Patch (Original sehen)
      const dumpPath = path.join(DUMP_DIR, `tool${i}.parameters.json`);
      try {
        fs.writeFileSync(dumpPath, JSON.stringify(params, null, 2));
        log("DUMPED tool.parameters", { phase, index: i, tool: tool?.function?.name || "<unknown>", path: dumpPath });
      } catch (e) {
        log("DUMP ERROR", { phase, index: i, msg: String(e && e.message || e) });
      }

      // Rekursiv fehlende items ergänzen
      fixArraySchemas(params, `$.tools[${i}].parameters`);
    } catch (e) {
      log("TOOLS PATCH ERROR", { phase, index: i, msg: String(e && e.message || e) });
    }
  });
}

class StripReasoningConflict {
  constructor() {
    this.name = "stripReasoningConflict";
    this.endpoint = "/v1/chat/completions"; // an OpenRouter-Endpunkt binden
    log("LOADED stripReasoningConflict (CJS class, endpoint=/v1/chat/completions)");
  }

  // Typisch: wird vor Provider-Transformer aufgerufen
  transformRequestIn(req) {
    const body = req && req.body ? req.body : req;
    log("transformRequestIn ENTER", { hasBody: !!body });

    if (body) {
      stripReasoning(body);
      patchAllToolsOnBody(body, "pre");

      // *** WICHTIG ***: defensiv reassignen, damit In-Place-Mutationen nicht verloren gehen
      if (req && req.body) req.body = deepClone(body);
    }
    return req;
  }

  // Fallback – falls die CCR-Version statt transformRequestIn -> transform/run verwendet
  async transform(ctx) {
    log("transform ENTER", { hasCtx: !!ctx });

    // Body aus versch. Pfaden ziehen
    let body = null, source = "unknown";
    if (ctx && ctx.body && typeof ctx.body === "object") { body = ctx.body; source = "ctx.body"; }
    else if (ctx && ctx.fetchInit && typeof ctx.fetchInit.body === "string") {
      try { body = JSON.parse(ctx.fetchInit.body); source = "ctx.fetchInit.body"; } catch {}
    }
    else if (ctx && ctx.requestBody && typeof ctx.requestBody === "object") { body = ctx.requestBody; source = "ctx.requestBody"; }

    log("transform BODY SOURCE", { source, hasBody: !!body });
    if (!body) return ctx;

    stripReasoning(body);
    patchAllToolsOnBody(body, "fallback");

    // defensiv zurückschreiben
    const cloned = deepClone(body);
    if (ctx && ctx.body && typeof ctx.body === "object") ctx.body = cloned;
    if (ctx && ctx.requestBody && typeof ctx.requestBody === "object") ctx.requestBody = cloned;
    if (ctx && ctx.fetchInit && typeof ctx.fetchInit.body === "string") ctx.fetchInit.body = JSON.stringify(cloned);

    return ctx;
  }

  async run(ctx) { return this.transform(ctx); }
  transformResponseOut(res) { return res; }
}

module.exports = StripReasoningConflict;

```

---

## 3. Konfiguration anpassen

In der Datei `~/.claude-code-router/config.json`  
bei **`transformers`** folgendes hinzufügen:

```json
{
  "path": "~/.claude-code-router/plugins/stripReasoningConflict.cjs"
}
```

Bei den **`Providers`** ergänzen:

```json
"transformer": {
  "use": ["stripReasoningConflict", "openrouter", "stripReasoningConflict"]
}
```

---

Beispiel config:
https://github.com/Flo082002/ClaudeCodeRouterFlow/blob/main/config.json

## 4. Bestimmte Tools droppen

Falls gewünscht, bestimmte Tool-Indizes droppen:

```bash
export CCR_DROP_TOOL_INDICES="32,42"
```

Anschließend den Claude Code Router neu starten:

```bash
ccr restart
```

---

## 5. Verbindung zu Claude Code Flow einrichten

1. Token-Variablen setzen (Inhalt des Tokens ist egal):

   ```bash
   export ANTHROPIC_AUTH_TOKEN="DerInhaltDiesesTokensIstEgal"
   ```

2. Basis-URL anpassen:  
   Die IP-Adresse und den Port aus `config.json` verwenden.

   ```bash
   export ANTHROPIC_BASE_URL="http://127.0.0.1:3456"
   ```

3. Claude Code Flow wie gewohnt im selben Terminal verwenden.

---

## ⚠️ Achtung

> **Hinweis:**  
> Dieser Text die Anleitung und der Code wurden vollständig mit ChatGPT erstellt.  
> Ich selbst habe keine Ahnung, wie der Code funktioniert – bei mir läuft er aber.

---

## Lizenz / Haftungsausschluss

Diese Anleitung erfolgt **ohne Garantie**.  
Verwendung auf eigene Verantwortung.
Diese Anleitung unterliegt der [MIT License](https://github.com/Flo082002/ClaudeCodeRouterFlow/blob/main/LICENSE)
