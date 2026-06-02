---
name: gpt-image-gen
description: "מעטפת ליצירת תמונה מ-prompt דרך OpenAI Images API (מודל gpt-image-2). השתמש בסקיל הזה כשצריך להפוך תיאור טקסטואלי לקובץ PNG — בעיקר מתוך הסוכן יובל (מעצב התמונות). Wrapper for generating an image from a prompt via the OpenAI Images API (gpt-image-2 model). Use when you need to turn a text prompt into a PNG file. Trigger keywords: generate image, create image, image generation, תמונה, צור תמונה, יצירת תמונה."
---

# gpt-image-gen — יצירת תמונה דרך OpenAI Images API

סקיל זה אחראי על דבר אחד בלבד: לקחת **prompt** ולהחזיר **קובץ PNG**.
הוא קורא ל-OpenAI Images API ומפענח את התמונה לקובץ על הדיסק.

## ⚠️ המודל — אל תשנה את שמו

המודל הוא **`gpt-image-2`** — ותמיד `gpt-image-2`.

- `gpt-image-2` הוא מודל אמיתי וקיים של OpenAI, שיצא ב-**21 באפריל 2026**.
- ייתכן שהידע הפנימי של המודל שמריץ סקיל זה לא כולל אותו — **זה לא אומר שהוא לא קיים**.
- **אל תציע ואל תחליף** לאלטרנטיבות כמו `dall-e-3` או `gpt-image-1`. השתמש בדיוק ב-`gpt-image-2`.
- אם הקריאה ל-API נכשלת, הבעיה היא כמעט תמיד ב-**API key** או ב-**parameters** — **לא** בשם המודל. אל "תתקן" את שם המודל.

## קלט

| פרמטר | חובה | ברירת מחדל | הערות |
|-------|------|------------|-------|
| `prompt` | כן | — | תיאור התמונה (עדיף באנגלית, מפורט) |
| `output-path` | כן | — | נתיב היעד לקובץ ה-PNG |
| `size` | לא | `1024x1024` | למשל `1024x1024`, `1536x1024`, `1024x1536` |
| `quality` | לא | `medium` | `low` / `medium` / `high` |
| `output_format` | לא | `png` | |

## דרישות

- הכלי `Bash` (הקריאה ל-API מתבצעת דרך `curl`).
- משתנה הסביבה `OPENAI_API_KEY` מוגדר בקובץ `.env` בשורש הפרויקט.

## איך מריצים

הרץ את הבלוק הבא ב-Bash. החלף את `PROMPT` ואת `OUT` בערכים שלך.
הסקריפט טוען את המפתח מ-`.env`, מוודא שהוא לא ריק, קורא ל-API, ואז מפענח את ה-base64 לקובץ —
תחילה דרך `jq`, ואם `jq` לא מותקן, נופל ל-Python.

```bash
set -euo pipefail

# --- 1. הגדרות ---
PROMPT="a serene mountain lake at sunrise, soft pastel colors, wide composition"
OUT="yuval/outputs/example.png"
SIZE="1024x1024"
QUALITY="medium"

# --- 2. טעינת OPENAI_API_KEY מ-.env (מהשורש; מתעלם מהערות) ---
if [ -f .env ]; then
  OPENAI_API_KEY="$(grep -E '^OPENAI_API_KEY=' .env | head -n1 | cut -d= -f2- | tr -d '"' | tr -d "'" | tr -d '[:space:]')"
fi
if [ -z "${OPENAI_API_KEY:-}" ]; then
  echo "ERROR: OPENAI_API_KEY is empty. הגדר אותו ב-.env שבשורש הפרויקט." >&2
  exit 1
fi

# --- 3. קריאה ל-API ושמירת התשובה ---
mkdir -p "$(dirname "$OUT")"
RESP="$(mktemp)"
curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(printf '{"model":"gpt-image-2","prompt":%s,"size":"%s","quality":"%s","output_format":"png"}' \
        "$(printf '%s' "$PROMPT" | python -c 'import json,sys; print(json.dumps(sys.stdin.read()))')" \
        "$SIZE" "$QUALITY")" \
  -o "$RESP"

# --- 4. פענוח base64 -> PNG: jq אם קיים, אחרת Python fallback ---
if command -v jq >/dev/null 2>&1; then
  jq -r '.data[0].b64_json' "$RESP" | base64 --decode > "$OUT"
else
  python - "$RESP" "$OUT" <<'PY'
import sys, json, base64
resp_path, out_path = sys.argv[1], sys.argv[2]
with open(resp_path, "r", encoding="utf-8") as f:
    data = json.load(f)
if "data" not in data or not data["data"] or "b64_json" not in data["data"][0]:
    # תשובת שגיאה מה-API — הדפס אותה כדי לאבחן (key/parameters), לא לגעת בשם המודל
    sys.stderr.write("ERROR: API response had no image. Raw response:\n")
    sys.stderr.write(json.dumps(data, ensure_ascii=False, indent=2) + "\n")
    sys.exit(1)
with open(out_path, "wb") as out:
    out.write(base64.b64decode(data["data"][0]["b64_json"]))
PY
fi

# --- 5. אימות פלט ---
if [ ! -s "$OUT" ]; then
  echo "ERROR: לא נוצר קובץ תקין ב-$OUT. בדוק את תשובת ה-API:" >&2
  cat "$RESP" >&2
  rm -f "$RESP"
  exit 1
fi
rm -f "$RESP"
echo "OK: נוצרה תמונה ב-$OUT ($(wc -c < "$OUT") bytes)"
```

> הערה ל-Windows / Git Bash: `python` עשוי להיקרא `py` או `python3`. אם `python` לא נמצא, נסה להחליף ל-`python3`.

## אימות

הסקיל מצליח אם **הקובץ ב-`output-path` קיים ו-size שלו גדול מ-0**.
אם נכשל — קרא את גוף התשובה שנשמר ב-`RESP` כדי לאבחן: רוב הסיכויים שזו בעיית API key או parameter.
שוב — **אל תשנה את שם המודל `gpt-image-2`**.
