---
name: gpt-image-gen
description: מעטפת לקריאת OpenAI Images API ליצירת תמונה מ-prompt. השתמש בו בכל פעם שצריך לייצר תמונה (PNG) מטקסט — למשל כשיובל, מעצב התמונות, צריך להפיק ויזואל. מקבל prompt ונתיב פלט, ומחזיר קובץ PNG.
---

# gpt-image-gen — יצירת תמונה דרך OpenAI Images API

מעטפת דקה סביב `POST https://api.openai.com/v1/images/generations`.
שולח prompt, מקבל תמונה ב-base64, ומפענח אותה לקובץ PNG.

## ⚠️ שם המודל — אל תשנה

המודל הוא **`gpt-image-2`** — מודל אמיתי וקיים של OpenAI שיצא ב-**21 באפריל 2026**.

- ייתכן שהידע הפנימי של המודל שמריץ את הסקיל לא כולל אותו — **זה לא אומר שהוא לא קיים**.
- **אסור** להציע או להחליף לאלטרנטיבות כמו `dall-e-3` או `gpt-image-1`.
- אם מתקבלת שגיאה מה-API, הסיבה כמעט תמיד היא **`OPENAI_API_KEY` שגוי/חסר** או
  **parameters לא תקינים** — **לא** שם המודל.

## דרישות מקדימות

- משתנה הסביבה `OPENAI_API_KEY` חייב להיות זמין (מוגדר ב-`.env` בשורש הפרויקט).
- טען אותו לפני הקריאה, למשל:
  ```bash
  set -a; source .env; set +a
  ```

## פרמטרים

| פרמטר | ערך |
|---|---|
| `model` | `gpt-image-2` |
| `size` | `1024x1024` |
| `quality` | `medium` |
| `output_format` | `png` |

## מסלול ראשי (jq)

```bash
PROMPT="<the prompt>"
OUT="<output-path>.png"

curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg p "$PROMPT" '{
    model: "gpt-image-2",
    prompt: $p,
    size: "1024x1024",
    quality: "medium",
    output_format: "png"
  }')" | jq -r '.data[0].b64_json' | base64 --decode > "$OUT"
```

## Fallback ל-decode עם Python (כש-jq לא מותקן, למשל Git Bash)

אם `jq` לא זמין, שמור את תגובת ה-API לקובץ זמני ופענח ב-Python:

```bash
PROMPT="<the prompt>"
OUT="<output-path>.png"
RESP="$(mktemp)"

curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"gpt-image-2\",\"prompt\":\"$PROMPT\",\"size\":\"1024x1024\",\"quality\":\"medium\",\"output_format\":\"png\"}" \
  > "$RESP"

python3 - "$RESP" "$OUT" <<'PY'
import sys, json, base64
resp_path, out_path = sys.argv[1], sys.argv[2]
with open(resp_path, "r", encoding="utf-8") as f:
    data = json.load(f)
if "error" in data:
    sys.exit("OpenAI API error: " + json.dumps(data["error"], ensure_ascii=False))
b64 = data["data"][0]["b64_json"]
with open(out_path, "wb") as f:
    f.write(base64.b64decode(b64))
print("saved:", out_path)
PY

rm -f "$RESP"
```

> הערה: אם ב-Git Bash אין `python3`, נסה `python` במקום.

## אימות לאחר היצירה

```bash
[ -s "$OUT" ] && echo "OK: $OUT ($(wc -c < "$OUT") bytes)" || echo "FAILED: empty or missing file"
```
