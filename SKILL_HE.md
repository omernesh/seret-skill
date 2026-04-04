---
name: seret-movies
description: >-
  Search Israeli movie showtimes, ratings, and cinema info from seret.co.il.
  Use when user asks about movies playing in Israel, Israeli cinema showtimes,
  "ma makrineem hayom", "sratim hadashim", "seret", movie tickets in Israel,
  what's showing in theaters, or upcoming Israeli film releases.
  Fetches real-time data from seret.co.il including bilingual titles (Hebrew + English),
  Seret ratings, IMDB links, trailers, and showtimes by city/area.
  Do NOT use for international movie databases without Israeli context (use IMDB/TMDB directly instead).
license: MIT
compatibility: >-
  Requires web fetch capability (HTTP GET + POST) and HTML parsing.
  Responses use windows-1255 encoding (Hebrew). No API keys needed.
  Works with Claude Code, Cursor, OpenClaw, Hermes Agent, and any agent with web access.
metadata:
  author: omernesh
  author_email: omernesher@gmail.com
  version: 1.0.0
  category: localization
  tags:
    he:
      - קולנוע
      - סרטים
      - הקרנות
      - קולנוע ישראלי
      - ישראל
    en:
      - cinema
      - movies
      - showtimes
      - israeli-cinema
      - israel
  display_name:
    he: "סרט - מידע על סרטים בישראל"
    en: Seret - Israeli Movie Information
  display_description:
    he: "חיפוש שעות הקרנה, דירוגים ומידע על סרטים מ-seret.co.il — מאגר הסרטים המוביל בישראל"
    en: >-
      Search Israeli movie showtimes, ratings, and cinema info from seret.co.il —
      Israel's leading movie database
  supported_agents:
    - claude-code
    - cursor
    - github-copilot
    - windsurf
    - opencode
    - codex
    - openclaw
---

# כלי מידע על סרטים מ-seret.co.il

אתה עוזר מידע על סרטים המשתמש ב-seret.co.il — אתר הסרטים המוביל בישראל.

## ארכיטקטורה

seret.co.il הוא אתר ASP קלאסי **ללא API בפורמט JSON**. כל המידע מוחזר כ-HTML.
- **קידוד:** windows-1255 (עברית). יש לפענח תגובות בהתאם.
- **כתובת בסיס:** `https://www.seret.co.il`
- **תמונות משתמשות ב-`data-src`** (טעינה עצלה) — יש לחלץ מ-`data-src`, לא מ-`src`.
- **נתיבי תמונות יחסיים:** יש להוסיף את הקידומת `https://www.seret.co.il/`

## פעולות זמינות

### 1. חיפוש סרטים

```
GET https://www.seret.co.il/searchAUAjax.asp?s={query}&t=movie
```

- `s` = מחרוזת חיפוש (מקודדת URL, עובד עם אנגלית או עברית)
- `t` = סוג: `movie`, `theatre`, `actor`, `director`
- מחזיר קטע HTML עם תוצאות השלמה אוטומטית

**תבנית פענוח:**
```
כל תוצאה היא <div class="comboline"> עם:
- MID: onclick="goto('...s_movies.asp?mid=(\d+)')"
- שם באנגלית: צומת טקסט לפני <span class="text-grey">
- שם בעברית: <span style="color:#ccc;...">שם בעברית</span>
- סטטוס: <span class="text-grey"><i>(coming soon)</i></span> או שנה
- פוסטר: <img src="...{Name}0.jpg">
- סדרות משתמשות ב: m.seret.co.il/s_series.asp?sid=N (לא s_movies.asp)
```

### 2. פרטי סרט

```
GET https://www.seret.co.il/movies/s_movies.asp?MID={id}
```

הדף מכיל **מיקרודאטה של schema.org/Movie**. יש לחלץ את שדות itemprop הבאים:

| שדה | בורר | הערות |
|-----|-------|-------|
| שם בעברית | `itemprop="name"` | בתוך `<h1>` |
| שם באנגלית | `itemprop="alternatename"` | בתוך `<h2>` |
| תקציר | `itemprop="description"` | תקציר עלילה בעברית |
| משך | `itemprop="duration"` | דקות, `datetime="PT{N}M"` |
| שנה | `itemprop="dateCreated"` | לדוגמה "2026" |
| תאריך יציאה | `itemprop="datePublished"` | פורמט: DD/MM/YYYY |
| דירוג תוכן | `itemprop="contentRating"` | לדוגמה "PG-13" |
| ז'אנר | `itemprop="genre"` | מספר תגיות, בעברית |
| במאי | `itemprop="director"` | שם בעברית |
| שחקנים | `itemprop="actor"` | מספר תגיות, שמות בעברית |
| דירוג | `itemprop="ratingValue"` | מתוך 10 |
| מספר מדרגים | `itemprop="ratingCount"` | מספר הצבעות |
| מספר ביקורות | `itemprop="reviewCount"` | מספר ביקורות |
| מספר תגובות | `itemprop="commentCount"` | תגובות משתמשים |
| פוסטר | `meta property="og:image"` | כתובת מלאה |
| מזהה IMDB | `<span class="imdbRatingPlugin" data-title="tt{ID}">` | לדוגמה "tt31050594" |

#### טריילר (קישור ישיר)

הטריילר יכול להיות סרטון YouTube מוטמע או קובץ MP4 מאוחסן. יש לבדוק את האלמנט `video#seretPlayer`:

```
<video id="seretPlayer" data-setup='{ "sources": [{ "type": "video/youtube", "src": "https://www.youtube.com/watch?v=VIDEO_ID" }] }'>
```

- קישור YouTube: לפענח את תכונת `data-setup` (JSON) ב-`video#seretPlayer` > `sources[0].src`
- קובץ MP4 מאוחסן: `<source src="https://vdo.seret.co.il/{Name}.mp4">` בתוך אלמנט הווידאו
- דף טריילר: `https://www.seret.co.il/movies/movieTrailer.asp?MID={id}`
- כפתור טריילר: `<a href="movieTrailer.asp?MID={id}" class="buttonGR">טריילר</a>`

יש להעדיף תמיד את קישור YouTube כשזמין (איכות גבוהה יותר, נגיש יותר). אם לא זמין, להשתמש בקישור MP4.

#### ציונים ודירוגים (מתחת לטריילר)

דף הסרט מכיל 5 מדדי ציון נפרדים בבלוקים של `div.info-row`:

**1. Seret Score (מדד משולב)**
```
מיכל: div.badge-col > div.badge-container
ציון: אלמנט SVG <text> (ילד ראשון, לדוגמה "5.1")
תווית: SVG <text> מתחת לציון (לדוגמה "ראוי לצפייה")
פירוט: div#badgetip (לדוגמה "מבקר: 5.0 · קהל: 5.2 · IMDb: — · כוונה: — · מומנטום: 6.5")
```
זהו מדד האיכות המשולב של האתר — שילוב של ציון מבקר, קהל, IMDB, כוונת צפייה ומומנטום.

**2. דירוג הגולשים**
```
ציון: span[itemprop="ratingValue"] (לדוגמה "4")
מקסימום: meta[itemprop="bestRating"] content="10"
מספר מצביעים: meta[itemprop="reviewCount"] או "N כבר הצביעו"
פס חזותי: span.RateScaleGreen (לדוגמה width:40% = 4/10)
```

**3. ציון המבקר**
```
ציון: div.critic-score (לדוגמה "5/10")
שם המבקר: div.critic-name (לדוגמה "יאיר הוכנר")
ציטוט מהביקורת: div.critic-publication a
קישור לביקורת המלאה: div.critic-publication a[href]
כוכבים: div.critic-stars — ספירת span.starw.on (מלא), span.starg (ריק), span.starh (חצי)
```

**4. מדד פופולריות**
```
ציון: div.pop-row div.DarkGreenStrong30 (לדוגמה "3.7/10")
מגמה: span.trend-arrow (לדוגמה "144.9%" עם חץ עולה/יורד)
תווית איכות: div.seg-bar[title] (לדוגמה "התעניינות קהל בינונית")
```

**5. היכן תצפו (סקר קהל)**
```
קולנוע: div.watch-fill.cinema + span.watch-fill-label
בבית: div.watch-fill.home + span.watch-fill-label
לא אצפה: div.watch-fill.skip + span.watch-fill-label
סה"כ מצביעים: div.vote-label (לחלץ מספר מ-"N כבר הצביעו")
```

**תבניות כתובות תמונות:**
- פוסטר: `https://www.seret.co.il/images/movies/{Name}/{Name}1.jpg`
- תמונה ממוזערת: `...{Name}0.jpg`
- תמונת כיסוי: `...{Name}_coverBig.jpg`
- תמונת וידאו: `...{Name}2.jpg`

### 3. מוקרנים עכשיו (בבתי קולנוע)

```
GET https://www.seret.co.il/movies/newmovies.asp
```

**תבנית פענוח — כרטיסי סרטים:**
```
כל סרט נמצא ב-<div class="picAndDataCont">:
- MID: href="s_movies.asp?MID=(\d+)"
- שם בעברית: <a class="TitGreen16" title="...">
- שם באנגלית: <img ... alt="..."> או <div class="truncate" style="direction:ltr" title="...">
- פוסטר: <img data-src="..."> (יש להוסיף כתובת בסיס אם יחסי)
- ז'אנר: <div class="roundedges">טקסט ז'אנר</div>
```

הדף כולל גם חלקי דירוג לפי דירוג סרט:
```
- דירוג: <span class="bgpnum">N</span>
- ציון: <span class="font-bold">9.0</span>/10
```

### 4. בקרוב

```
GET https://www.seret.co.il/movies/comingsoonmovies.asp
```

אותו מבנה כרטיסים כמו מוקרנים עכשיו, בתוספת:
```
- תאריך יציאה: <div class="lwhite16 noticetxt">יצא ב- D/M/YYYY</div>
```

### 5. שעות הקרנה לסרט

```
POST https://www.seret.co.il/movies/showTimesAjaxMovies.asp
Content-Type: application/x-www-form-urlencoded

Body: MID={id}&d={day}&s={screening}&v={version}&st=&tst=
```

- `MID` = מזהה סרט (חובה)
- `d` = מסנן מספר יום (ריק = כל הימים)
- `s` = מסנן סוג הקרנה (ריק = הכל)
- `v` = מסנן גרסה/שפה (ריק = הכל)

**תבנית פענוח:**
```
אזורים: <div class="cityname"> עם <a name="a{areaId}"> ו-<span>שם אזור</span>
בתי קולנוע: <a href="s_theatres.asp?TID={id}" class="TitGreen20">שם בית קולנוע</a>
ימים: <div class="dayline"> המכיל:
  - תווית יום: <div class="dayname" title="מידע תאריך מלא">אות יום</div>
  - שעות הקרנה: <div class="stbox" title="D/M/YYYY | אולם N">HH:MM</div>
```

### 6. טבלת הקרנות (לפי עיר + יום)

```
POST https://www.seret.co.il/movies/movieTable.asp
Content-Type: application/x-www-form-urlencoded

Body: d={dayNumber}&f_areaId={cityId}
```

**חשוב:** נקודת קצה זו משתמשת **במזהי עיר**, לא במזהי אזור מסעיף 5. כדי לגלות מזהי עיר ומספרי ימים תקפים, יש לבצע תחילה GET ל-`movieTable.asp` ולפענח את תפריטי ה-`<select>`:
- `<select name="f_areaId">` — רשימת כל הערים עם מזהיהן
- `<select name="d">` — רשימת ימים זמינים עם ערכיהם המספריים

מזהי ערים נפוצים (אלה שונים ממזהי האזורים ב-`showTimesAjaxMovies.asp`):

| מזהה עיר | עיר |
|-----------|-----|
| 41 | תל אביב / יפו |
| 44 | אור יהודה |
| 1 | אשדוד |
| 2 | אשקלון |
| 3 | באר שבע |
| 8 | חיפה |

לרשימה המלאה, יש תמיד לפענח את התפריט מתגובת ה-GET — מזהי ערים עשויים להשתנות.

**תבנית פענוח:**
```
כל שורה: <div class="comboline">
- שעה: <div class="TitGreen18" style="width:70px;...">HH:MM</div>
- בית קולנוע: <a href=s_theatres.asp?tid={id} class="TitGreen18">שם</a>
- סרט: <a href="s_movies.asp?mid={id}" class="bt_rounded2" title="שם מלא">שם</a>
```

### 7. רשימת בתי קולנוע לפי אזור

```
GET https://www.seret.co.il/movies/theatres.asp?AID={areaId}
```

**תבנית פענוח:**
```
כל בית קולנוע ברשת:
- TID: href="s_theatres.asp?TID=(\d+)"
- שם: <a class="TitGreen16">שם בית קולנוע</a>
- כתובת: טקסט אחרי <span class="ICLoc">
- טלפון: טקסט אחרי <span class="ICPhone">
```

### 8. דירוגים נוספים (לא אמין)

```
POST https://www.seret.co.il/ajax/getExtraMovieRatingsAjax.asp
Content-Type: application/x-www-form-urlencoded

Body: MID={id}
```

מחזיר HTML עם מקורות דירוג נוספים. **לרוב מחזיר ריק** — נקודת קצה זו אינה אמינה ואין להסתמך עליה. מזהה ה-IMDB מדף פרטי הסרט (סעיף 2) הוא דרך אמינה יותר להצלבת דירוגים חיצוניים.

## מזהי אזורים (עבור showTimesAjaxMovies.asp)

מזהי אזורים אלה משמשים את **נקודת הקצה של שעות ההקרנה** (סעיף 5) לקיבוץ תוצאות לפי אזור. הם אינם זהים למזהי הערים ב-`movieTable.asp` (סעיף 6).

| מזהה | אזור (עברית) | אזור (אנגלית) |
|------|---------------|----------------|
| 6 | ירושלים | Jerusalem |
| 8 | חיפה | Haifa |
| 13 | קריות/צפון | Krayot/North |
| 17 | תל-אביב | Tel Aviv |
| 27 | באר-שבע/דרום | Beer Sheva/South |
| 31 | שרון-השפלה | Sharon-Hashfela |
| 34 | אילת | Eilat |
| 54 | השרון | Hasharon |

## הנחיות תגובה

בעת הצגת מידע על סרטים:
1. **תמיד להציג גם שם בעברית וגם באנגלית** כשזמין
2. **לעצב שעות הקרנה** מקובצות לפי אזור > בית קולנוע > יום לקריאות
3. **לכלול את כל הציונים הזמינים:** Seret Score (משולב), דירוג גולשים, ציון מבקר ומדד פופולריות
4. **קישור לדף הסרט:** `https://www.seret.co.il/movies/s_movies.asp?MID={id}`
5. **להציג תמונת פוסטר** כשהפלטפורמה תומכת
6. **תמיד לכלול קישור לטריילר** — להעדיף YouTube, לחלופין MP4. גם לקשר לדף הטריילר: `movieTrailer.asp?MID={id}`
7. **לכלול תוצאות סקר "היכן תצפו"** כשזמין (אחוזי קולנוע/בבית/לא אצפה)
8. **לכלול קישור IMDB** כשמזהה IMDB זמין: `https://www.imdb.com/title/{imdbId}/`

## תהליכי עבודה לדוגמה

**"מה מקרינים היום?"**
לבצע GET ל-`newmovies.asp`, לפענח כרטיסים, להציג כרשימה עם שמות + ז'אנרים + דירוגים

**"ספר לי על [שם סרט]"**
חיפוש דרך `searchAUAjax.asp?s={name}&t=movie` > לקבל MID > לבצע GET ל-`s_movies.asp?MID={id}` > לחלץ נתוני schema.org

**"שעות הקרנה ל[סרט] בתל אביב"**
חיפוש > לקבל MID > POST ל-`showTimesAjaxMovies.asp` עם MID > לסנן תוצאות אזור 17 (תל אביב)

**"מה יוצא בקרוב?"**
לבצע GET ל-`comingsoonmovies.asp`, לפענח כרטיסים עם תאריכי יציאה

**"מצא בתי קולנוע בחיפה"**
לבצע GET ל-`theatres.asp?AID=8`, לפענח רשת בתי קולנוע
