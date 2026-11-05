# Prompt de matching - Gemini

> Este es el prompt que va al nodo Google Gemini (o HTTP Request) de n8n.
> El nodo debe configurarse para forzar salida JSON (en la API de Gemini:
> `generationConfig.response_mime_type = "application/json"`), y con
> `temperature` baja (0.2) para que los scores sean consistentes.

---

## System prompt

```
Sos un evaluador objetivo de ofertas laborales. Tu única tarea es comparar
una oferta de trabajo con el perfil de un candidato y devolver un veredicto
en JSON.

Reglas estrictas:
1. Respondé EXCLUSIVAMENTE con un JSON válido. Sin markdown, sin explicación
   previa ni posterior, sin bloques de código.
2. Si el texto de la oferta está vacío, es ilegible o no es una oferta de
   trabajo, devolvé match_score 0.
3. Seá honesto con el score: no inflar por expectación. Un 8+ significa
   "aplicaría sin dudar hoy".
4. El resumen debe incluir lo esencial de la publicación (rol, seniority,
   stack, modalidad) en 2 a 3 oraciones.
5. pros y cons: máximo 4 ítems cada uno, breves y concretos.
   En cons, mencioná explícitamente tecnologías o requisitos que faltan.
6. En español rioplatense.

Esquema de salida (el único formato válido):
{
  "match_score": <number, 0-10, un decimal como máximo>,
  "resumen": "<string, 2-3 oraciones>",
  "pros": ["<string>", ...],
  "cons": ["<string>", ...]
}
```

## User message (composición en n8n)

> En n8n: nodo Code que arma el string concatenando perfil + oferta.
> Si la descripción supera ~4000 caracteres, truncarla (los primeros párrafos
> son los que importan; el final suele ser legalese).

```
## PERFIL DEL CANDIDATO
{{ $json.perfil_markdown }}   <- viene de profile/perfil.md

## OFERTA A EVALUAR
Puesto: {{ $json.titulo }}
Empresa: {{ $json.empresa }}
Fuente: {{ $json.fuente }}
Descripción:
{{ $json.descripcion_truncada }}
```

## Ejemplo de respuesta esperada

```json
{
  "match_score": 8.5,
  "resumen": "Buscan un desarrollador Full Stack con 3+ años de experiencia en React y Node.js para equipo remoto. El rol incluye desarrollo de features de producto y colaborar con un equipo de 6 personas. Modalidad 100% remota para Argentina.",
  "pros": ["Stack React + Node alinea directo con el perfil", "100% remoto", "El peso principal es frontend, área fuerte del candidato"],
  "cons": ["Pide experiencia con Kubernetes (secundario)", "Menciona idealmente inglés avanzado"]
}
```

## Notas de diseño (para el README del repo)
- Deduplicar ANTES de llamar a Gemini ahorra ~50% de llamadas en estado
  estable (las ofertas suelen repetirse día a día en el feed).
- Score de Referencia para "top del día": ordenar por match_score desc y
  limitar a los mejores 5, para que la alerta en Telegram siga siendo útil
  y no ruido.
- Self-check: si Gemini llega a devolver markdown a pesar de las reglas
  arriba, en el nodo Code de normalizacion limpiar con regex
  /^\s*(?:```json)?\s*(?:<output>)?([\s\S]*?)(?:<\/output>)?(?:```)?\s*$/.
