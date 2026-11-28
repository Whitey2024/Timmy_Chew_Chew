Here's a ready-to-use Python script that downloads the playlist daily, filters out the categories you listed, and writes a cleaned M3U.
What it removes

Local TV – anything with _locals in tvg-id (ABC, CBS, NBC, FOX, CW, Telemundo, Univision, etc.)
Sports channels – ESPN*, Fox Sports*, NFL/NBA/NHL Network, ACC/Big Ten/SEC, regional sports nets, Golf, Tennis, racing, TSN, Sky Sports, etc.
PPV / live events – the entire Live Events group (all the event streams)
Children’s channels – Disney Channel/XD, Nickelodeon family, Boomerang, PBS Kids, Nick Jr, Nicktoons, MeTV Toons, etc.
Spanish-language – Univision, Telemundo, Galavisión, TUDN, ESPN Deportes, and channels whose names contain common Spanish markers

It keeps news, entertainment, movies, lifestyle, etc.
Script (filter_m3u.py)

Create a text document and paste the following, then rename to 'filter_m3u.py' make sure to remove the .txt file extension.
'''py
#!/usr/bin/env python3
"""
Daily M3U filter for BuddyChewChew/My-Streams tv.m3u
Removes: local TV, sports, PPV/live events, children's, Spanish channels
"""

import re
import urllib.request
from pathlib import Path
from datetime import datetime

# ========== CONFIG ==========
SOURCE_URL = "https://raw.githubusercontent.com/BuddyChewChew/My-Streams/main/tv.m3u"
OUTPUT_FILE = Path.home() / "filtered_tv.m3u"   # change if you want a different location
# ============================

# Keywords / patterns (case-insensitive)
SPORTS_KEYWORDS = [
    r"\bespn\b", r"fox\s*sports", r"\bnba\b", r"\bnfl\b", r"\bnhl\b", r"\bmlb\b",
    r"sports?\s*net", r"\bsec\b", r"big\s*ten", r"\bacc\b", r"altitude\s*sports",
    r"chicago\s*sports", r"\bmsg\b", r"\byes\b", r"\btsn\b", r"sky\s*sports",
    r"golf\s*channel", r"tennis\s*channel", r"fanduel", r"floracing", r"willow",
    r"nbc\s*sports", r"cbs\s*sports", r"tnt\s*sports", r"\bppv\b", r"pay.?per.?view",
    r"ufc\b", r"boxing", r"wwe\b", r"racing", r"\bgolazo\b",
]

KIDS_KEYWORDS = [
    r"disney\s*(channel|xd)", r"\bnick(elodeon|toons|jr)?\b", r"boomerang",
    r"pbs\s*kids", r"me\s*tv\s*toons", r"cartoon\s*network", r"adult\s*swim",
    r"universal\s*kids", r"discovery\s*family", r"baby\s*first", r"sesame",
]

SPANISH_KEYWORDS = [
    r"univision", r"telemundo", r"galavisión", r"galavision", r"\btudn\b",
    r"espn\s*deportes", r"\bespañol\b", r"\bespanol\b", r"\blatino\b",
    r"\bmexico\b", r"\bmx\b",
]

# Compile once
SPORTS_RE = re.compile("|".join(SPORTS_KEYWORDS), re.I)
KIDS_RE   = re.compile("|".join(KIDS_KEYWORDS), re.I)
SPANISH_RE = re.compile("|".join(SPANISH_KEYWORDS), re.I)
LOCAL_RE  = re.compile(r"_locals", re.I)

def should_exclude(extinf_line: str, group: str) -> bool:
    """Return True if this channel should be removed."""
    # Entire Live Events group = sports / PPV / events
    if group.lower() == "live events":
        return True

    # Local stations
    if LOCAL_RE.search(extinf_line):
        return True

    # Name / title text
    name_match = re.search(r'tvg-name="([^"]*)"', extinf_line, re.I)
    title_match = re.search(r',([^,]*)$', extinf_line)
    text = " ".join(filter(None, [
        name_match.group(1) if name_match else "",
        title_match.group(1).strip() if title_match else "",
        extinf_line,
    ]))

    if SPORTS_RE.search(text):
        return True
    if KIDS_RE.search(text):
        return True
    if SPANISH_RE.search(text):
        return True

    return False

def filter_playlist(content: str) -> str:
    lines = content.splitlines()
    output = []
    i = 0

    # Keep the header
    while i < len(lines) and not lines[i].startswith("#EXTINF"):
        output.append(lines[i])
        i += 1

    while i < len(lines):
        line = lines[i]

        if line.startswith("#EXTINF"):
            # Collect the full entry: EXTINF + optional EXTVLCOPT lines + URL
            entry = [line]
            i += 1
            while i < len(lines) and (lines[i].startswith("#EXTVLCOPT") or lines[i].startswith("#EXT")):
                entry.append(lines[i])
                i += 1
            if i < len(lines) and not lines[i].startswith("#"):
                entry.append(lines[i])  # the URL
                i += 1

            # Extract group-title
            group_match = re.search(r'group-title="([^"]*)"', entry[0], re.I)
            group = group_match.group(1) if group_match else ""

            if not should_exclude(entry[0], group):
                output.extend(entry)
                output.append("")  # blank line between entries (optional but nice)
        else:
            i += 1

    return "\n".join(output).rstrip() + "\n"

def main():
    print(f"[{datetime.now():%Y-%m-%d %H:%M}] Downloading playlist...")
    try:
        with urllib.request.urlopen(SOURCE_URL, timeout=30) as resp:
            raw = resp.read().decode("utf-8", errors="replace")
    except Exception as e:
        print(f"ERROR downloading: {e}")
        return 1

    print("Filtering...")
    cleaned = filter_playlist(raw)

    OUTPUT_FILE.write_text(cleaned, encoding="utf-8")
    # Rough channel count
    channels = cleaned.count("#EXTINF")
    print(f"Done. Wrote {channels} channels → {OUTPUT_FILE}")
    return 0

if __name__ == "__main__":
    raise SystemExit(main())
'''
