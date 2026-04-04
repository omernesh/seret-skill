# Seret Movies Skill

> Give any AI agent the ability to search Israeli movies, showtimes, ratings, trailers, and cinema info from [seret.co.il](https://www.seret.co.il) — Israel's leading movie database.

<div align="center">

![License](https://img.shields.io/badge/license-MIT-green)
![Platform](https://img.shields.io/badge/platform-any%20AI%20agent-blue)
![Language](https://img.shields.io/badge/languages-Hebrew%20%2B%20English-orange)
![skills-il](https://img.shields.io/badge/skills--il-localization-teal)

</div>

## What is this?

A single markdown file (`SKILL.md`) that teaches any AI agent how to fetch and parse movie data from seret.co.il. No code, no dependencies, no API keys — just documented endpoints and HTML parsing patterns that any LLM with web access can follow.

Also includes a Hebrew companion (`SKILL_HE.md`) with fully translated instructions.

## Capabilities

| Feature | How it works |
|---------|-------------|
| **Search movies** | Autocomplete search by name (Hebrew or English) |
| **Movie details** | Title, plot, cast, director, duration, genre, content rating, IMDB link |
| **Trailer links** | Direct YouTube URL or self-hosted MP4, plus trailer page link |
| **Seret Score** | Composite quality index (critic + audience + IMDB + momentum) |
| **Viewer rating** | User votes out of 10 with vote count |
| **Critic score** | Professional review score, critic name, review blurb + stars |
| **Popularity index** | Trending score with week-over-week change percentage |
| **"Where to watch" poll** | Audience intent: cinema vs. home vs. skip (percentages) |
| **Now showing** | All movies currently in Israeli theaters |
| **Coming soon** | Upcoming releases with dates |
| **Showtimes** | By movie (grouped by area/theater) or by city + day |
| **Cinemas** | Theater listings by region with address and phone |

## Quick Start

### Claude Code

```bash
# Global (all projects)
cp SKILL.md ~/.claude/skills/seret-movies.md

# Or in a Claude Code plugin
cp SKILL.md ~/.claude/plugins/my-plugin/skills/seret-movies.md
```

Then just ask naturally:

```
> What movies are playing in Tel Aviv tonight?
> Tell me about the new Superman movie — is it worth watching?
> Show me the trailer for The Bride
> When is Project Hail Mary coming out?
> Find cinemas in Jerusalem
```

### Hermes Agent

```bash
mkdir -p ~/.hermes/skills/seret-movies
cp SKILL.md SKILL_HE.md ~/.hermes/skills/seret-movies/
```

### OpenClaw / Cursor / Other Agents

Drop `SKILL.md` into your agent's skill or prompt directory. The file uses standard markdown frontmatter — adapt the loading mechanism to your platform.

### Raw LLM Usage

Paste the contents of `SKILL.md` into your system prompt. The skill instructs the LLM how to use web fetch tools to query seret.co.il endpoints.

## How It Works

Seret.co.il has no public JSON API. This skill reverse-engineers 8 endpoints that return HTML fragments and server-rendered pages:

```
searchAUAjax.asp        GET   Search autocomplete
s_movies.asp            GET   Movie details (schema.org microdata + scores)
newmovies.asp           GET   Currently in theaters
comingsoonmovies.asp    GET   Upcoming releases
showTimesAjaxMovies.asp POST  Showtimes by movie (grouped by area)
movieTable.asp          POST  Showtimes by city + day
theatres.asp            GET   Cinemas by region
getExtraMovieRatingsAjax.asp POST Extra ratings (unreliable)
```

Movie detail pages include schema.org/Movie microdata for clean structured extraction, plus 5 dedicated score sections (Seret Score, viewer rating, critic score, popularity index, audience watch-intent poll) and direct trailer links (YouTube or MP4).

## Area & City IDs

**Area IDs** (for `showTimesAjaxMovies.asp` — regional grouping):

| ID | Region |
|----|--------|
| 6 | Jerusalem |
| 8 | Haifa |
| 13 | Krayot / North |
| 17 | Tel Aviv |
| 27 | Beer Sheva / South |
| 31 | Sharon / Hashfela |
| 34 | Eilat |
| 54 | Hasharon |

**City IDs** (for `movieTable.asp` — different numbering):

| ID | City |
|----|------|
| 41 | Tel Aviv / Yafo |
| 3 | Beer Sheva |
| 8 | Haifa |

For the full city list, parse the `<select name="f_areaId">` dropdown from `movieTable.asp`.

## Requirements

Your AI agent needs:
- **Web fetch capability** (HTTP GET + POST)
- **HTML parsing** (regex or DOM — the skill provides patterns for both)
- **windows-1255 encoding support** (Hebrew character set)

No API keys, authentication, or rate limits.

## License

MIT — use it however you want.
