---
name: digitize-pounce-scoresheet
description: Digitize scanned Pounce card game scoresheets into structured YAML format for Hugo static site integration with automated analysis
---

# Digitize Pounce Card Game Scoresheets

This skill converts scanned Pounce scoresheet images into structured YAML data for use in a Hugo static site with automated score analysis.

## Overview

**Input**: One or more scanned scoresheet images
**Output**: Hugo markdown file with YAML frontmatter containing session data and inline shortcode for analysis

## Game Rules Context

**Pounce** is a multi-player competitive solitaire card game where:
- Players compete simultaneously to play cards to shared center piles
- Each player has a "pounce pile" of 13 cards they try to empty first
- **Scoring per hand**:
  - Cards played to center: +1 point each
  - Cards remaining in pounce pile: -2 points each
  - Winner (first to empty pounce pile): +bonus points (typically 5)
- **Ruleset variations**: Different sessions may use different pile sizes and bonus points

## Scoresheet Format

Scoresheets typically contain:
- **Session date**: When the games were played
- **Ruleset**: Pile size (default: 13) and win bonus (default: 5)
- **Player names**: Listed in columns or rows
- **Hand scores**: Individual game scores for each player
- **Win indicators**: Marked winner for each hand (checkmark, circle, star, etc.)
- **Timestamps**: Optional time for each hand (can be estimated if not recorded)

## Processing Steps

### 1. Read the Scoresheet Image(s)

Use the Read tool to view scanned scoresheet images. Carefully examine:
- Player names (may be handwritten)
- Score values for each hand
- Win indicators (marks, circles, highlights)
- Dates and times if recorded
- Any ruleset notes (e.g., "15+3" means 15-card pile with 3-point bonus)

### 2. Extract Session Information

Identify:
- **Session date**: Date games were played (format: YYYY-MM-DD)
- **Ruleset**:
  - `pile`: Number of cards in pounce pile (default: 13)
  - `bonus`: Points awarded to winner (default: 5)

### 3. Extract Hand Data

For each hand (individual game):
- **Timestamp**: When the hand was played
  - Format: `YYYY-MM-DDTHH:MM:SS-HHMM` (ISO 8601 with timezone)
  - If not recorded, estimate based on session start time and typical 6-minute hand duration
- **Players**: For each player record:
  - `name`: Player name (lowercase)
  - `score`: Raw score for the hand (integer, can be negative)
  - `win`: true (only include for winning player)

### 4. Determine Winner

The winner of each hand is:
- Player who emptied their pounce pile first (usually marked on scoresheet)
- If unmarked, typically the player with the highest score
- Only ONE player per hand should have `win: true`

### 5. Generate Output File

Create a Hugo markdown file with:
- **Frontmatter**: YAML data structure with all session/hand data
- **Content**: Inline Hugo shortcode for analysis (see template below)

## Output Format Template

```yaml
---
title: "YYYY Event Name Pounce"
date: YYYY-MM-DDTHH:MM:SS-HH:MM
draft: false
toc: false
tags:
  - pounce
  - cards
data:
  sessions:
    - session: "YYYY-MM-DD"
      ruleset:
        pile: 13
        bonus: 5
      hands:
        - ts: "YYYY-MM-DDTHH:MM:SS-HHMM"
          players:
            - name: "player1"
              score: 10
            - name: "player2"
              score: 15
              win: true
            - name: "player3"
              score: -5
        - ts: "YYYY-MM-DDTHH:MM:SS-HHMM"
          players:
            - name: "player1"
              score: 8
              win: true
            - name: "player2"
              score: -2
            - name: "player3"
              score: 3
---

{{< pounce.inline >}}
{{ $sessions := .Page.Params.data.sessions }}
<p>Sessions: {{ len $sessions }}</p>

<table>
  <tr>
    <th>#</th>
    <th>Start</th>
    <th>End</th>
    <th>Hands</th>
    <th>Players</th>
    <th>Player-Hands</th>
    <th>Rules</th>
  </tr>

  {{ range $i, $session := sort $sessions "session" }}
    {{ $distinctPlayers := dict }}
    {{ $handPlayerCount := 0 }}
    {{ $startTs := time "2199-01-01T16:35:00-0700" }}
    {{ $endTs := time "1899-01-01T16:35:00-0700" }}

    {{ range $hand := $session.hands }}
      {{ $handPlayerCount = add $handPlayerCount (len $hand.players) }}
      {{ $handTs := time $hand.ts }}
      {{ if lt $handTs $startTs }}
        {{ $startTs = $handTs }}
      {{ end }}
      {{ if gt $handTs $endTs }}
        {{ $endTs = $handTs }}
      {{ end }}

      {{ range $handPlayer := $hand.players }}
        {{ $distinctPlayers = merge $distinctPlayers (dict $handPlayer.name true) }}
      {{ end }}
    {{ end }}

    <tr>
      <td>{{ add $i 1 }}</td>
      <td>{{ $startTs | time.Format "01/02 15:04" }}</td>
      <td>{{ $endTs | time.Format "01/02 15:04"  }}</td>
      <td>{{ len $session.hands }}</td>
      <td>{{ len $distinctPlayers }}</td>
      <td>{{ $handPlayerCount }}</td>
      <td>{{ $session.ruleset.pile }}+{{ $session.ruleset.bonus }}</td>
    </tr>
  {{ end }}

</table>

{{ $allHands := slice }}
{{ range $session := $sessions }}
  {{ range $hand := $session.hands }}
    {{ $hand = merge $hand (dict "ruleset" $session.ruleset) }}
    {{ $allHands = $allHands | append $hand }}
  {{ end }}
{{ end }}

<p>Hands: {{ len $allHands }}</p>

{{ $perPlayer := dict "handCount" 0 "winCount" 0 "scoreSum" 0 }}
{{ $scoreboardByPlayer := dict }}

{{ range $hand := $allHands }}
  {{ $ruleset := $hand.ruleset }}
  {{ range $handPlayer := $hand.players }}
    {{ if not (isset $scoreboardByPlayer $handPlayer.name) }}
      {{ $scoreboardByPlayer = merge $scoreboardByPlayer (dict $handPlayer.name $perPlayer)}}
    {{ end }}

    {{ $before := index $scoreboardByPlayer $handPlayer.name }}
    {{ $handPlayerScore := add $handPlayer.score (cond ($handPlayer.win | default false) $ruleset.bonus 0) }}

    {{ $after := merge $before (dict "handCount" (add (index $before "handCount") 1) "winCount" (add (index $before "winCount") (cond ($handPlayer.win | default false) 1 0)) "scoreSum" (add (index $before "scoreSum") $handPlayerScore) ) }}
    {{ $scoreboardByPlayer = merge $scoreboardByPlayer (dict $handPlayer.name $after) }}
  {{ end }}
{{ end }}

<table>
  <tr>
    <th>Rank</th>
    <th>Name</th>
    <th>Score Avg ↓</th>
    <th>Wins</th>
  </tr>

  {{ $sortable := slice }}
  {{ range $name, $stats := $scoreboardByPlayer }}
    {{ $scoreAvg := div $stats.scoreSum (float $stats.handCount) }}
    {{ $winAvg := div $stats.winCount (float $stats.handCount) }}
    {{ $row := merge $stats (dict "name" $name "scoreAvg" $scoreAvg "winAvg" $winAvg) }}
    {{ $sortable = append $sortable (slice $row)}}
  {{ end }}

  {{ range $i, $row := sort $sortable "scoreAvg" "desc" }}
    <tr>
      <td>{{ add $i 1 }}</td>
      <td>{{ .name | title }}</td>
      <td>
        {{ lang.FormatNumber 3 .scoreAvg }} ({{ .scoreSum }} / {{ .handCount }})
      </td>
      <td>
        {{ .winCount }}
      </td>
    </tr>
  {{ end }}
</table>

{{< /pounce.inline >}}
```

## Example Workflow

**User provides**: `thanksgiving-2022-pounce-scoresheet.jpg`

**Processing**:
1. Read image to identify 6 players: tom, neela, nancy, gavin, kate, alex
2. Identify 3 hands played on 2022-11-23 starting at 20:45
3. Extract scores for each player per hand
4. Identify winners (marked with circles on scoresheet)
5. Estimate timestamps at 6-minute intervals: 20:45, 20:51, 20:57
6. Note ruleset: 13+5 (standard rules)

**Output**: Create `2022-thanksgiving-pounce.md` with complete YAML frontmatter and shortcode

## Data Validation Checklist

Before finalizing output:
- [ ] All player names are lowercase and consistent across hands
- [ ] Each hand has exactly one winner with `win: true`
- [ ] Scores are integers (can be negative)
- [ ] Timestamps are in ISO 8601 format with timezone
- [ ] Session date format is YYYY-MM-DD
- [ ] Ruleset pile and bonus are integers
- [ ] Title follows pattern: "YYYY Event Name Pounce"
- [ ] File date is ISO 8601 timestamp

## Common Scoresheet Patterns

### Multiple Sessions
If scoresheet contains games from different dates/events:
- Create separate session entries in the `sessions` array
- Each session has its own ruleset and hands array

### Missing Timestamps
If times are not recorded:
- Use session date with estimated start time
- Add 6 minutes per hand for subsequent hands
- Adjust if actual duration is noted

### Ruleset Notation
Common notations on scoresheets:
- "13+5" = 13-card pile, 5-point bonus (standard)
- "15+3" = 15-card pile, 3-point bonus
- "10+7" = 10-card pile, 7-point bonus

### Player Name Variations
Normalize names to lowercase:
- "Tom" → "tom"
- "T.H." → "tom" (if context is clear)
- Maintain consistency across all hands

## Hugo Shortcode Analysis

The included shortcode calculates:
- **Session Summary**: Sessions played, date range, hand count, player count
- **Overall Leaderboard**: Average score per hand (descending), total wins
- **Score Average Formula**: `(sum of all hand scores + win bonuses) / total hands played`

The shortcode automatically:
- Adds bonus points to winner's score for average calculation
- Ranks players by score average (higher is better)
- Counts total wins per player
- Aggregates across multiple sessions if present

## Notes

- **Timezone**: Use appropriate timezone offset for session location (e.g., -0700 for PDT, -0800 for PST)
- **File naming**: Use format `YYYY-event-name-pounce.md` (lowercase, hyphenated)
- **Draft status**: Set `draft: false` for published content
- **Tags**: Always include `pounce` and `cards` tags for categorization
