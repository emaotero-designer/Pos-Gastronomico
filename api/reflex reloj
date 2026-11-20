// Estado compartido de la demo (lo leen y escriben todas las compus que abren el link).
// Usa la base Redis (Upstash) conectada al proyecto en Vercel.
const REDIS_URL = process.env.KV_REST_API_URL || process.env.UPSTASH_REDIS_REST_URL;
const REDIS_TOKEN = process.env.KV_REST_API_TOKEN || process.env.UPSTASH_REDIS_REST_TOKEN;

async function redis(commands) {
  const response = await fetch(`${REDIS_URL}/pipeline`, {
    method: "POST",
    headers: { Authorization: `Bearer ${REDIS_TOKEN}`, "Content-Type": "application/json" },
    body: JSON.stringify(commands),
  });
  if (!response.ok) throw new Error(`Redis respondió ${response.status}`);
  return (await response.json()).map((entry) => {
    if (entry.error) throw new Error(entry.error);
    return entry.result;
  });
}

module.exports = async (req, res) => {
  res.setHeader("Cache-Control", "no-store");
  if (!REDIS_URL || !REDIS_TOKEN) {
    return res.status(500).json({ ok: false, error: "Falta conectar la base de datos al proyecto" });
  }

  try {
    if (req.method === "GET") {
      const since = Number(req.query.since ?? -1);
      const [rawVersion] = await redis([["GET", "grilla:version"]]);
      const version = Number(rawVersion || 0);
      if (version === since) return res.status(200).json({ ok: true, version, changed: false });

      const [tv, flatTables, open] = await redis([
        ["GET", "grilla:tv"],
        ["HGETALL", "grilla:tables"],
        ["GET", "grilla:open"],
      ]);
      const tables = {};
      for (let index = 0; index < (flatTables || []).length; index += 2) {
        tables[flatTables[index]] = flatTables[index + 1];
      }
      return res
        .status(200)
        .json({ ok: true, version, changed: true, tv: tv || null, tables, open: open || null });
    }

    if (req.method === "POST") {
      const body = typeof req.body === "string" ? JSON.parse(req.body) : req.body || {};
      let commands;
      if (body.op === "table") {
        commands = body.state
          ? [["HSET", "grilla:tables", String(body.slot), JSON.stringify(body.state)]]
          : [["HDEL", "grilla:tables", String(body.slot)]];
      } else if (body.op === "tv") {
        commands = [["SET", "grilla:tv", JSON.stringify(body.tv)]];
      } else if (body.op === "open") {
        // Mesa tocada en un reloj: la abre el posnet de ese mozo (se borra sola al minuto)
        commands = [["SET", "grilla:open", JSON.stringify(body.open), "EX", "60"]];
      } else if (body.op === "reset") {
        commands = [["DEL", "grilla:tv", "grilla:tables", "grilla:open"]];
      } else {
        return res.status(400).json({ ok: false, error: "Operación desconocida" });
      }
      const results = await redis([...commands, ["INCR", "grilla:version"]]);
      return res.status(200).json({ ok: true, version: results[results.length - 1] });
    }

    return res.status(405).json({ ok: false });
  } catch (error) {
    return res.status(500).json({ ok: false, error: String(error.message || error) });
  }
};
