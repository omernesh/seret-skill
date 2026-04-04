# Seret Movies Skill

> Give any AI agent the ability to search Israeli movies, showtimes, and cinema info from [seret.co.il](https://www.seret.co.il) — Israel's leading movie database.

<div align="center">

![License](https://img.shields.io/badge/license-MIT-green)
![Platform](https://img.shields.io/badge/platform-any%20AI%20agent-blue)
![Language](https://img.shields.io/badge/languages-Hebrew%20%2B%20English-orange)

</div>

## What is this?

A single markdown file (`SKILL.md`) that teaches any AI agent how to fetch and parse movie data from seret.co.il. No code, no dependencies, no API keys — just documented endpoints and HTML parsing patterns that any LLM with web access can follow.

## Capabilities

| Feature | How it works |
|---------|-------------|
| **Search movies** | Autocomplete search by name (Hebrew or English) |
| **Movie details** | Title, plot, cast, director, duration, genre, rating, trailer, IMDB link |
| **Now showing** | All movies currently in Israeli theaters |
| **Coming soon** | Upcoming releases with dates |
| **Showtimes** | By movie or by area + day, with theater and hall info |
| **Cinemas** | Theater listings by region with address and phone |

## Quick Start

### Claude Code

Copy the skill file to your skills directory:

```bash
# Global (all projects)
cp SKILL.md ~/.claude/skills/seret-movies.md

# Or in a Claude Code plugin
cp SKILL.md ~/.claude/plugins/my-plugin/skills/seret-movies.md
```

Then just ask naturally:

```
> What movies are playing in Tel Aviv tonight?
> Tell me about the new Superman movie
> When is Project Hail Mary coming out?
> Find cinemas in Jerusalem
```

### OpenClaw / Hermes Agent / Other

Drop `SKILL.md` into your agent's skill or prompt directory. The file uses standard markdown frontmatter — adapt the loading mechanism to your platform.

### Raw LLM Usage

Paste the contents of `SKILL.md` into your system prompt. The skill instructs the LLM how to use web fetch tools to query seret.co.il endpoints.

## How It Works

Seret.co.il has no public JSON API. This skill reverse-engineers 8 endpoints that return HTML fragments and server-rendered pages:

```
searchAUAjax.asp        GET   Search autocomplete
s_movies.asp            GET   Movie details (schema.org microdata)
newmovies.asp           GET   Currently in theaters
comingsoonmovies.asp    GET   Upcoming releases
showTimesAjaxMovies.asp POST  Showtimes by movie
movieTable.asp          POST  Showtimes by area + day
theatres.asp            GET   Cinemas by region
getExtraMovieRatingsAjax.asp POST Extra ratings
```

The skill documents the exact HTML selectors, CSS classes, and parsing patterns needed to extract structured data from each endpoint. Movie detail pages use `schema.org/Movie` microdata, making them particularly clean to parse.

## Area IDs

Use these when filtering showtimes or listing cinemas:

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

## Requirements

Your AI agent needs:
- **Web fetch capability** (HTTP GET + POST)
- **HTML parsing** (regex or DOM — the skill provides patterns for both)
- **windows-1255 encoding support** (Hebrew character set)

No API keys, authentication, or rate limits.

## License

MIT — use it however you want.
