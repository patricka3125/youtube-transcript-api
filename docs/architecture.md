# youtube-transcript-api — Architecture & Design

## Project Overview

A Python library + CLI tool (v1.2.4, MIT license) that fetches YouTube video transcripts/subtitles. Supports Python 3.8–3.14.

## Structure

```
youtube_transcript_api/
├── __init__.py          # Package exports
├── __main__.py          # CLI entry point
├── _api.py              # Core API class (YouTubeTranscriptApi)
├── _cli.py              # CLI implementation (YouTubeTranscriptCli)
├── _transcripts.py      # Transcript fetching, parsing, data models
├── _errors.py           # Exception hierarchy
├── _settings.py         # URL constants, InnerTube config
├── formatters.py        # Output formatters (JSON, SRT, WebVTT, etc.)
├── proxies.py           # Proxy configuration (Generic, Webshare)
└── test/                # Test suite (100% coverage target)
```

## CLI vs API Relationship

**The CLI is a thin wrapper around the API. They are not independent — the CLI depends entirely on the API for all YouTube interaction.**

```
YouTubeTranscriptCli  (argument parsing, formatting, error display)
        │ creates & calls
        ▼
YouTubeTranscriptApi  (core logic, HTTP requests, YouTube interaction)
        │ delegates to
        ▼
TranscriptListFetcher → TranscriptList → Transcript → FetchedTranscript
```

- **CLI cannot function without the API** — it imports and instantiates `YouTubeTranscriptApi` directly
- **API works perfectly without the CLI** — it's the library's primary interface, usable in any Python code
- The CLI adds: argparse argument handling, formatter selection, proxy config from flags, error collection/display

## Data Flow

```
User Input (video ID)
    ↓
YouTubeTranscriptApi.list(video_id)
    ↓
TranscriptListFetcher.fetch()
    ├── GET youtube.com/watch?v=ID  (+ consent cookie handling)
    ├── Extract INNERTUBE_API_KEY from HTML via regex
    ├── POST to InnerTube API (mimics Android client)
    │   └── Returns playability status + caption metadata
    ├── Validate playability (age-restricted? blocked? unavailable?)
    └── Build TranscriptList from captionTracks JSON
    ↓
TranscriptList (container of Transcript objects)
    ├── find_transcript(["en", "de"])           # any type, priority order
    ├── find_manually_created_transcript(...)    # manual only
    └── find_generated_transcript(...)           # auto-generated only
    ↓
Transcript.fetch()
    ├── GET transcript XML URL
    ├── _TranscriptParser.parse(xml)  (uses defusedxml for safety)
    └── Returns FetchedTranscript (list of snippets with text/start/duration)
    ↓
Optional: Transcript.translate("es")  → modifies URL with &tlang=es
    ↓
Output: FetchedTranscript → Formatter → string
```

## Key Classes

| Class | File | Responsibility |
|---|---|---|
| `YouTubeTranscriptApi` | `_api.py` | Public API. Creates HTTP session, exposes `fetch()` and `list()` |
| `YouTubeTranscriptCli` | `_cli.py` | CLI wrapper. Parses args, calls API, formats output |
| `TranscriptListFetcher` | `_transcripts.py` | Orchestrates HTML fetch, InnerTube API call, caption extraction |
| `TranscriptList` | `_transcripts.py` | Container of available transcripts with filtering methods |
| `Transcript` | `_transcripts.py` | Single transcript metadata + `fetch()` and `translate()` |
| `FetchedTranscript` | `_transcripts.py` | Iterable container of `FetchedTranscriptSnippet` dataclasses |
| `_TranscriptParser` | `_transcripts.py` | Parses YouTube XML into snippet objects |
| `Formatter` (abstract) | `formatters.py` | Base class for output formatters |
| `ProxyConfig` (abstract) | `proxies.py` | Base class for proxy configurations |

## Formatter System (Strategy Pattern)

Five implementations registered in `FormatterLoader`:

| Formatter | Output |
|---|---|
| `JSONFormatter` | JSON array of dicts |
| `PrettyPrintFormatter` | Python pprint (default for CLI) |
| `TextFormatter` | Plain text, no timestamps |
| `SRTFormatter` | SubRip subtitle format (`00:00:00,000`) |
| `WebVTTFormatter` | WebVTT subtitle format (`00:00:00.000`) |

## Error Hierarchy

```
YouTubeTranscriptApiException
├── CookieError
│   ├── CookiePathInvalid
│   └── CookieInvalid
└── CouldNotRetrieveTranscript
    ├── VideoUnavailable / VideoUnplayable / InvalidVideoId
    ├── RequestBlocked → IpBlocked  (429 responses)
    ├── TranscriptsDisabled / AgeRestricted
    ├── NoTranscriptFound / NotTranslatable / TranslationLanguageNotAvailable
    ├── PoTokenRequired
    ├── YouTubeRequestFailed / YouTubeDataUnparsable
    └── FailedToCreateConsentCookie
```

All errors include the video URL, a cause description, and a GitHub issue link for remediation.

## Proxy System

- `GenericProxyConfig`: Any HTTP/HTTPS/SOCKS proxy
- `WebshareProxyConfig`: Rotating residential proxies with auto-retry on 429, connection closing for IP rotation

## Notable Design Decisions

1. **Android client masquerade** — InnerTube requests use `clientName: "ANDROID"` to bypass some YouTube restrictions
2. **Not thread-safe by design** — one `requests.Session` per instance; documented as intentional
3. **defusedxml** — prevents XXE attacks when parsing transcript XML
4. **Language fallback chain** — `find_transcript(["de", "en"])` tries each code in order
5. **100% test coverage** enforced on all source except `__main__.py`

## Bottom Line

The CLI and API are **not** two separate components. The CLI is a convenience layer on top of the API. If you remove the API, the CLI has nothing — it's just argparse glue. The API is the entire engine: fetching pages, calling YouTube's InnerTube endpoint, parsing XML transcripts, and returning structured data. You can use the API as a Python library without ever touching the CLI.
