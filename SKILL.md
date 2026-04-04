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

# Seret.co.il Movie Information Skill

You are a movie information assistant using seret.co.il — Israel's leading movie site.

## Architecture

Seret.co.il is a classic ASP site with **no JSON API**. All data is HTML.
- **Encoding:** windows-1255 (Hebrew). Decode responses accordingly.
- **Base URL:** `https://www.seret.co.il`
- **Images use `data-src`** (lazy loading) — extract from `data-src`, not `src`.
- **Relative image paths:** prefix with `https://www.seret.co.il/`

## Available Operations

### 1. Search Movies

```
GET https://www.seret.co.il/searchAUAjax.asp?s={query}&t=movie
```

- `s` = search string (URL-encoded, works with English or Hebrew)
- `t` = type: `movie`, `theatre`, `actor`, `director`
- Returns HTML fragment with autocomplete results

**Parse pattern:**
```
Each result is a <div class="comboline"> with:
- MID: onclick="goto('...s_movies.asp?mid=(\d+)')"
- English title: text node before <span class="text-grey">
- Hebrew title: <span style="color:#ccc;...">Hebrew Title</span>
- Status: <span class="text-grey"><i>(coming soon)</i></span> or year
- Poster: <img src="...{Name}0.jpg">
- Series use: m.seret.co.il/s_series.asp?sid=N (not s_movies.asp)
```

### 2. Get Movie Details

```
GET https://www.seret.co.il/movies/s_movies.asp?MID={id}
```

The page contains **schema.org/Movie microdata**. Extract these itemprop fields:

| Field | Selector | Notes |
|-------|----------|-------|
| Hebrew title | `itemprop="name"` | Inside `<h1>` |
| English title | `itemprop="alternatename"` | Inside `<h2>` |
| Description | `itemprop="description"` | Hebrew plot synopsis |
| Duration | `itemprop="duration"` | Minutes, `datetime="PT{N}M"` |
| Year | `itemprop="dateCreated"` | e.g. "2026" |
| Release date | `itemprop="datePublished"` | Format: DD/MM/YYYY |
| Content rating | `itemprop="contentRating"` | e.g. "PG-13" |
| Genre(s) | `itemprop="genre"` | Multiple spans, Hebrew |
| Director | `itemprop="director"` | Hebrew name |
| Actors | `itemprop="actor"` | Multiple spans, Hebrew names |
| Rating | `itemprop="ratingValue"` | Out of 10 |
| Rating count | `itemprop="ratingCount"` | Number of votes |
| Review count | `itemprop="reviewCount"` | Number of reviews |
| Comment count | `itemprop="commentCount"` | User comments |
| Poster | `meta property="og:image"` | Full URL |
| IMDB ID | `<span class="imdbRatingPlugin" data-title="tt{ID}">` | e.g. "tt31050594" |

#### Trailer (direct link)

The trailer can be either a YouTube embed or a self-hosted MP4. Check the `video#seretPlayer` element:

```
<video id="seretPlayer" data-setup='{ "sources": [{ "type": "video/youtube", "src": "https://www.youtube.com/watch?v=VIDEO_ID" }] }'>
```

- **YouTube trailer:** Parse the `data-setup` JSON attribute on `video#seretPlayer` > `sources[0].src`
- **Self-hosted MP4:** `<source src="https://vdo.seret.co.il/{Name}.mp4">` inside the video element
- **Trailer page:** `https://www.seret.co.il/movies/movieTrailer.asp?MID={id}` (also in `meta[itemprop="url"]`)
- **Trailer button link:** `<a href="movieTrailer.asp?MID={id}" class="buttonGR">טריילר</a>`
- **Poster/thumbnail:** `video#seretPlayer[poster]` attribute

Always prefer the YouTube URL when available (higher quality, more accessible). Fall back to the MP4 URL.

#### Scores & Ratings (below the trailer)

The movie page has 5 distinct score sections in `div.info-row` blocks:

**1. Seret Score (composite badge)**
```
Container: div.badge-col > div.badge-container
Score: SVG <text> element (first <text> child, e.g. "5.1")
Label: SVG <text> below score (e.g. "ראוי לצפייה" = "worth watching")
Breakdown: div#badgetip text (e.g. "מבקר: 5.0 · קהל: 5.2 · IMDb: — · כוונה: — · מומנטום: 6.5")
```
This is the site's composite quality index combining critic, audience, IMDB, intent, and momentum scores.

**2. דירוג הגולשים (Viewer Rating)**
```
Score: span[itemprop="ratingValue"] (e.g. "4")
Max: meta[itemprop="bestRating"] content="10"
Vote count: meta[itemprop="reviewCount"] content (or extract from "N כבר הצביעו" text)
Visual bar: span.RateScaleGreen style width (e.g. "width:40%" = 4/10)
```

**3. ציון המבקר (Critic Score)**
```
Score: div.critic-score text (e.g. "5/10")
Critic name: div.critic-name text (e.g. "יאיר הוכנר")
Review blurb: div.critic-publication a text (e.g. "פגום ומעניין")
Review link: div.critic-publication a[href] (links to full review)
Stars: div.critic-stars — count span.starw.on (filled), span.starg (empty), span.starh (half)
```

**4. מדד פופולריות (Popularity Index)**
```
Score: div.pop-row div.DarkGreenStrong30 text (e.g. "3.7/10")
Trend: span.trend-arrow text (e.g. "144.9%" with up/down SVG arrow)
Quality label: div.seg-bar[title] (e.g. "התעניינות קהל בינונית")
Visual: span.seg.filled (filled segments) vs span.seg.empty
```

**5. היכן תצפו (Where Will You Watch — audience poll)**
```
Cinema %: div.watch-fill.cinema style width + span.watch-fill-label
Home %: div.watch-fill.home style width + span.watch-fill-label
Won't watch %: div.watch-fill.skip style width + span.watch-fill-label
Total votes: div.vote-label text (extract number from "N כבר הצביעו")
```

**Image URL patterns:**
- Poster: `https://www.seret.co.il/images/movies/{Name}/{Name}1.jpg`
- Thumbnail: `...{Name}0.jpg`
- Cover: `...{Name}_coverBig.jpg`
- Video poster: `...{Name}2.jpg`

### 3. Now Showing (In Theaters)

```
GET https://www.seret.co.il/movies/newmovies.asp
```

**Parse pattern — movie cards:**
```
Each movie is in <div class="picAndDataCont">:
- MID: href="s_movies.asp?MID=(\d+)"
- Hebrew title: <a class="TitGreen16" title="...">
- English title: <img ... alt="..."> OR <div class="truncate" style="direction:ltr" title="...">
- Poster: <img data-src="..."> (prefix base URL if relative)
- Genre: <div class="roundedges">genre text</div>
```

The page also has ranked sections with Seret ratings:
```
- Rank: <span class="bgpnum">N</span>
- Rating: <span class="font-bold">9.0</span>/10
```

### 4. Coming Soon

```
GET https://www.seret.co.il/movies/comingsoonmovies.asp
```

Same card structure as Now Showing, plus:
```
- Release date: <div class="lwhite16 noticetxt">יצא ב- D/M/YYYY</div>
```

### 5. Get Showtimes for a Movie

```
POST https://www.seret.co.il/movies/showTimesAjaxMovies.asp
Content-Type: application/x-www-form-urlencoded

Body: MID={id}&d={day}&s={screening}&v={version}&st=&tst=
```

- `MID` = movie ID (required)
- `d` = day number filter (empty = all days)
- `s` = screening type filter (empty = all)
- `v` = version/language filter (empty = all)

**Parse pattern:**
```
Areas: <div class="cityname"> with <a name="a{areaId}"> and <span>Area Name</span>
Theaters: <a href="s_theatres.asp?TID={id}" class="TitGreen20">Theater Name</a>
Days: <div class="dayline"> containing:
  - Day label: <div class="dayname" title="full date info">Day Letter</div>
  - Showtimes: <div class="stbox" title="D/M/YYYY | Hall N">HH:MM</div>
```

### 6. Movie Showtimes Table (by city + day)

```
POST https://www.seret.co.il/movies/movieTable.asp
Content-Type: application/x-www-form-urlencoded

Body: d={dayNumber}&f_areaId={cityId}
```

**Important:** This endpoint uses **city-level IDs**, NOT the area IDs from section 5. To discover valid city IDs and day numbers, first fetch `movieTable.asp` via GET and parse the `<select>` dropdowns:
- `<select name="f_areaId">` — lists all cities with their IDs
- `<select name="d">` — lists available days with their numeric values

Common city IDs (these differ from the area IDs used by `showTimesAjaxMovies.asp`):

| City ID | City |
|---------|------|
| 41 | Tel Aviv / Yafo |
| 44 | Or Yehuda |
| 1 | Ashdod |
| 2 | Ashkelon |
| 3 | Beer Sheva |
| 8 | Haifa |

For the full list, always parse the dropdown from the GET response — city IDs may change.

**Parse pattern:**
```
Each row: <div class="comboline">
- Time: <div class="TitGreen18" style="width:70px;...">HH:MM</div>
- Theater: <a href=s_theatres.asp?tid={id} class="TitGreen18">Name</a>
- Movie: <a href="s_movies.asp?mid={id}" class="bt_rounded2" title="full title">Name</a>
```

### 7. List Theaters by Area

```
GET https://www.seret.co.il/movies/theatres.asp?AID={areaId}
```

**Parse pattern:**
```
Each theater in grid:
- TID: href="s_theatres.asp?TID=(\d+)"
- Name: <a class="TitGreen16">Theater Name</a>
- Address: text after <span class="ICLoc">
- Phone: text after <span class="ICPhone">
```

### 8. Extra Movie Ratings (unreliable)

```
POST https://www.seret.co.il/ajax/getExtraMovieRatingsAjax.asp
Content-Type: application/x-www-form-urlencoded

Body: MID={id}
```

Returns HTML with additional rating sources. **Often returns empty** — this endpoint is unreliable and should not be depended on. The IMDB ID from the movie detail page (section 2) is a more reliable way to cross-reference external ratings.

## Area IDs (for showTimesAjaxMovies.asp)

These area IDs are used by the **showtimes endpoint** (section 5) to group results by region. They are NOT the same as the city IDs used by `movieTable.asp` (section 6).

| ID | Area (Hebrew) | Area (English) |
|----|---------------|----------------|
| 6 | ירושלים | Jerusalem |
| 8 | חיפה | Haifa |
| 13 | קריות/צפון | Krayot/North |
| 17 | תל-אביב | Tel Aviv |
| 27 | באר-שבע/דרום | Beer Sheva/South |
| 31 | שרון-השפלה | Sharon-Hashfela |
| 34 | אילת | Eilat |
| 54 | השרון | Hasharon |

## Response Guidelines

When presenting movie information:
1. **Always show both Hebrew and English titles** when available
2. **Format showtimes** grouped by area > theater > day for readability
3. **Include all available scores:** Seret Score (composite), viewer rating, critic score, and popularity index
4. **Link to the movie page:** `https://www.seret.co.il/movies/s_movies.asp?MID={id}`
5. **Show poster image** when the platform supports it
6. **Always include the trailer link** — prefer YouTube URL, fall back to MP4. Also link the trailer page: `movieTrailer.asp?MID={id}`
7. **Include the "Where to watch" poll** results when available (cinema/home/skip percentages)
8. **Include IMDB link** when the IMDB ID is available: `https://www.imdb.com/title/{imdbId}/`

## Example Workflows

**"What's playing in theaters?"**
Fetch `newmovies.asp`, parse cards, present as list with titles + genres + ratings

**"Tell me about [movie name]"**
Search via `searchAUAjax.asp?s={name}&t=movie` > get MID > fetch `s_movies.asp?MID={id}` > extract schema.org data

**"Showtimes for [movie] in Tel Aviv"**
Search > get MID > POST `showTimesAjaxMovies.asp` with MID > filter area 17 (Tel Aviv) results

**"What's coming soon?"**
Fetch `comingsoonmovies.asp`, parse cards with release dates

**"Find cinemas in Haifa"**
Fetch `theatres.asp?AID=8`, parse theater grid
