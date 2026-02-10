![Cron job status](https://api.cron-job.org/jobs/7252968/e21ef3fbf2b1d94d/status-3.svg)

# Update schedule & reliability notice

This site is generated automatically by workflows that are triggered externally via a scheduled HTTP request service (**cron-job.org**) and then execute a **GitHub Actions workflow** (`workflow_dispatch`) that refreshes all data and publishes the site via GitHub Pages.

**Scheduled updates typically run on weekdays between ~06:00 and 22:00 Germany time.**
Since external schedulers (like *cron-job.org*) and GitHub Actions both operate on a best‑effort basis, **there is no strict guarantee of exact timing**. Delays of a few minutes are normal, and under rare conditions an expected update may be delayed or skipped.

If your calendar app still shows old data:

* wait for the next scheduled update — typically within ~15 minutes
* re‑subscribe if necessary.

You can check the current GitHub system status — including **Actions** and **Pages** — here: [https://www.githubstatus.com/](https://www.githubstatus.com/)

Workflows used for updates:

```
.github/workflows/*.yml
```
## Alternative: GitHub Cron (Legacy)

GitHub Actions supports a built‑in **`schedule`** trigger using POSIX cron syntax, and workflows can be defined to run on a schedule (e.g., every hour or at set minutes). However:

* The GitHub scheduler only operates on a best‑effort basis — **it does not guarantee exact timing**, and can delay or skip runs under high system load or at busy times.
* Scheduled workflows are typically limited to a **minimum interval of ~5 minutes** and will not always run precisely at the configured cron time.

For these reasons, automated external triggering via cron‑job.org (or other external cron services) is used here to improve the reliability of regular updates.

---

## Why This Approach

While GitHub’s native cron scheduling is convenient, many users find its reliability insufficient for frequent interval updates (e.g., every ~15 minutes). Official documentation states:

> *The schedule event can be delayed during periods of high loads of GitHub Actions workflow runs. High load times include the start of every hour. If the load is sufficiently high enough, some queued jobs may be dropped.*

In practice, this means that relying solely on GitHub’s internal scheduler can result in missed or delayed runs — even if the workflow appears to be configured correctly. Using an external scheduler to call the `workflow_dispatch` API helps mitigate this and gives more consistent timing.


# ASW Calendar Exporter

This tool fetches ASW block schedules, parses the HTML plans and generates iCalendar (`.ics`) files.

A GitHub Actions workflow refreshes the output on a schedule and publishes a small landing page via GitHub Pages, including:
- Block navigation (derived from filenames)
- Class sub-grouping (letter + numeric styles)
- Subscribe (webcal / iOS best supported), Copy URL, and Download file actions
- A short Google/Android help page

## Transformation Preview

The parser converts the raw HTML schedule table into standard iCalendar format suitable for Outlook, Google Calendar, and Apple Calendar.

**Input (HTML Snippet strongly simplified):**
```html
<td id="zf160234" class="v" rowspan="18">
    9:00 - 10:30 Uhr<br>
    Vorlesung<br>
    IBL III<br>
    EXT: Online
</td>
```
**Output (iCalendar Event):**
```ics
BEGIN:VEVENT
UID:DBBWL-A03-1765267200-0
SUMMARY:IBL III (Vorlesung)
LOCATION:EXT: Online
DESCRIPTION:Course: DBBWL-A03_7_7.Block\nType: Vorlesung\nModule/Group: IBL III
DTSTART:20251209T080000Z
DTEND:20251209T093000Z
END:VEVENT
```

---

## Quickstart (recommended)

A prebuilt Docker image is available via GitHub Container Registry:

- `ghcr.io/umsername/asw-exporter:latest`

### Run (HTTPS mode)

#### Bash (Linux/macOS/Git-Bash)

```bash
mkdir -p out
docker run --rm \
  -v "$PWD/out:/data" \
  -e ASW_OUTPUT_DIR="/data/ics_files" \
  -e ASW_PUBLIC_DIR="/data/public" \
  ghcr.io/umsername/asw-exporter:latest
````

#### PowerShell (Windows)

```powershell
New-Item -ItemType Directory -Force -Path .\out | Out-Null

docker run --rm `
  -v "$($PWD.Path)\out:/data" `
  -e ASW_OUTPUT_DIR="/data/ics_files" `
  -e ASW_PUBLIC_DIR="/data/public" `
  ghcr.io/umsername/asw-exporter:latest
```

Output:

```txt
./out/ics_files/
./out/public/
```

---

## Local development

### Requirements

* Go installed and available in your `PATH`

### Install dependencies

```bash
go mod tidy
```

### Run

```bash
go run .
```

---

## Configuration

The app uses environment variables with sensible defaults:

* `ASW_SCHEDULE_URL`
  Default: ASW overview page (HTTPS)

* `ASW_BASE_URL`
  Default: `https://www.asw-ggmbh.de`

* `ASW_OUTPUT_DIR`
  Default: `ics_files`

* `ASW_PUBLIC_DIR`
  Default: `public`

* `ASW_USER_AGENT`
  Default: `ASW-ICS-Exporter/1.0 (+github.com/umsername/aswCalender)`

---

## Use a `.env` file (optional)

A sample file is provided as `.env.example`.

For local builds on Windows PowerShell:

```powershell
Get-Content .env | ForEach-Object {
    if ($_ -match '^\s*#' -or $_ -match '^\s*$') { return }
    $k, $v = $_ -split '=', 2
    $env:$k = $v
}

go build -o asw-parser.exe .
.\asw-parser.exe
```

---

## Publishing (GitHub)

* **Workflow:** `.github/workflows/publish-ics.yml`
* **GitHub Pages source:** GitHub Actions

The workflow generates the `.ics` files and the landing page automatically.

---

## Notes

* This repository provides code and generated calendar feeds.
* The underlying schedule content remains owned by the original provider.
* If the ASW website structure changes, the parser may require updates.


## Deep dive to the Parsing Logic
## Parsing Logic & HTML Structure
The source HTML (generated by *sked campus*) uses a complex, legacy table-based layout. It relies heavily on `rowspan` to visualize time blocks, making straightforward scraping impossible. The parser has to reconstruct the grid logic to map events to the correct days.

### The Challenge: "Rowspan Hell"

Events are floating inside a grid where a single 90-minute lecture might span 18 table rows (`rowspan="18"`), displacing cells in neighboring columns. The content itself is unstructured text separated by `<br>` tags.

**Raw Input Structure (Simplified):**

```html
<!-- Week Definition -->
<div class="w1">Veranstaltungsplan...</div>
<div class="w2">7. Studienwoche: 08. - 14.12.2025</div>

<table>
  <!-- Day Headers -->
  <tr>
    <td class="t">Mo, 08.12.2025</td>
    ...
  </tr>
  
  <!-- The Grid -->
  <tr>
    <!-- Time Label -->
    <td class="rz1" rowspan="12">9:00</td>
    
    <!-- An Event Cell (spanning 18 visual rows!) -->
    <!-- Class 'v' identifies an event -->
    <td id="zf160234" class="v" rowspan="18">
      9:00 - 10:30 Uhr<br>  <!-- Line 1: Time -->
      Vorlesung<br>         <!-- Line 2: Type -->
      IBL III<br>           <!-- Line 3: Module -->
      EXT: Online           <!-- Line 4: Location -->
    </td>
    
    <!-- Empty filler cells for other days follow... -->
  </tr>
</table>
```

### The Solution

The parser implements a stateful table reader that:
1.  **Extracts Headers:** Identifies the week range and maps table columns to specific dates (Mon-Sat).
2.  **Handling Rowspans:** Maintains a "column pointer" to account for vertical overlaps caused by `rowspan`. If a column is blocked by a previous row's event, the parser skips it for the current row.
3.  **Parsing Content:** Splits the inner HTML of `.v` cells by `<br>` to extract:
    *   **Time:** Parsed into UTC (e.g., `20251209T080000Z`).
    *   **Summary:** Module Name + Type (e.g., "IBL III (Vorlesung)").
    *   **Location:** Room number or Online status.

**Final ICS Output:**

```ics
BEGIN:VEVENT
UID:DBBWL-A03-1765267200-0
DTSTART:20251209T080000Z
DTEND:20251209T093000Z
SUMMARY:IBL III (Vorlesung)
LOCATION:EXT: Online
DESCRIPTION:Course: DBBWL-A03...
END:VEVENT
```

