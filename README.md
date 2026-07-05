# חבילת סקילים לפיתוח מקצועי עם Claude — m.eylon studio

שישה סקילים שמכסים את שרשרת הבנייה המלאה: אפליקציות, תוכנות, אתרים ומשחקים.
נבנו להשלים את הסקילים הקיימים בסטודיו (עיצוב, פרונט, RTL, אנימציות) — לא לשכפל אותם.

## מפת הסקילים

| סקיל | מכסה | מתי מופעל |
|---|---|---|
| `backend-api-design` | ארכיטקטורת שרת, API, ולידציה, שגיאות, הרשאות, idempotency | כל קוד צד-שרת, endpoint, server action |
| `app-security` | אימות, סודות, הזרקות, XSS/CSRF, העלאת קבצים, rate limiting + וורקפלואו ביקורת מלא | כל קוד auth/תשלומים/דאטה של משתמשים, ולפני כל דיפלוי |
| `database-design` | מידול סכמה, מפתחות, אינדקסים, מיגרציות, multi-tenancy, RLS | כל טבלה/מודל חדש, ובתחילת כל פרויקט |
| `testing-strategy` | מה לבדוק לפי ערך, Vitest/Playwright, לולאת אימות, TDD לבאגים | כל פיצ'ר לא-טריוויאלי וכל תיקון באג |
| `web-game-dev` | game loop נכון, קלט, קוליז'ן, state machines, juice, ביצועים | כל משחק או דבר "משחיק" |
| `ship-and-deploy` | סביבות, env vars, CI, Vercel/Docker/Supabase, ניטור, צ'קליסט השקה | "לעלות לאוויר", ומראש בכל פרויקט |

## איך הסקילים מדברים אחד עם השני

הם בנויים כרשת, לא כרשימה:
- backend מפנה ל-app-security בכל נושא auth, ל-testing בהגדרת "done", ול-ship בנושא סודות
- database ↔ backend חולקים את עקרון ה-RLS וה-transactions
- ship-and-deploy דורש שה-security review וה-CI ירוצו לפני השקה

בשיעור, זו הנקודה הפדגוגית: סקיל טוב לא חי לבד — הוא מגדיר חוזה מול השכנים שלו.

## התקנה

**Claude Code — כ-plugin (מומלץ):** הריפו הזה הוא plugin תקני (`.claude-plugin/plugin.json` + הסקילים תחת `skills/`). בסשן אינטראקטיבי:
```
/plugin marketplace add <path-to-this-folder>
/plugin install claude-dev-skills@meylon-studio
```

**Claude Code — העתקה ידנית:**
```bash
# גלובלי (כל הפרויקטים)
cp -r skills/* ~/.claude/skills/
# או פר-פרויקט
cp -r skills/* .claude/skills/
```

**Claude Desktop / Claude.ai:** העלאת ה-SKILL.md דרך Settings → Capabilities → Skills, או הצגת קובץ ה-.skill ולחיצה על Save skill.

## סקילים חיצוניים מומלצים (לא לכתוב מחדש — להתקין)

- **obra/superpowers** — github.com/obra/superpowers — שכבת ה-workflow: brainstorming → spec → תכנון → ביצוע ב-subagents → TDD → code review. הקולקציה הקהילתית הגדולה ביותר. משלים את החבילה הזו מלמעלה.
- **anthropics/skills** — github.com/anthropics/skills — הרפו הרשמי, כולל frontend-design ו-mcp-builder.
- **Trail of Bits security skills** — ניתוח סטטי עם CodeQL/Semgrep וביקורת קוד, מבית חברת אבטחה. שכבה מעל app-security כשצריך עומק.
- **Vercel web-design-guidelines** — quality gate של 100+ חוקי נגישות/ביצועים/UX, משלים את frontend-design-il מהצד של הבקרה.

## הערה למתקדמים (חומר לשיעור)

הסקילים כתובים לפי עקרונות skill-creator של Anthropic:
1. **Description "דוחפני"** — כולל טריגרים בעברית ובאנגלית, כי המודל נוטה לתת-הפעלה (undertrigger)
2. **הסבר "למה" ולא רק "מה"** — הוראה עם רציונל שורדת מצבים שהכותב לא צפה
3. **צ'קליסט מדיד בסוף** — הופך את הסקיל מעצות ל-definition of done שאפשר לדווח עליו
4. **מתחת ל-500 שורות** — progressive disclosure; ידע עמוק יותר עובר לקבצי reference
