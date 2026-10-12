# MoveInSync Play Store Rating Analysis

**Candidate Case Study | Forward Deployed Engineer (Intern to FTE)**

## Overview

This project explores how MoveInSync can improve its Play Store ratings by identifying potential sources of user dissatisfaction and translating them into actionable product and engineering priorities.

The central idea is simple: improving ratings starts with reducing the failures that make employees lose trust in their daily commute.

## Problem Areas Investigated

- Booking confirmation and OTP reliability
- Shuttle booking usability
- Live vehicle tracking and ETA accuracy
- Check-in and no-show experience
- Transport operations and support

These are preliminary themes identified from publicly available information and a limited sample of reviews. They require validation against a larger sample and internal product data.

## Proposed Solution

The recommendation prioritises critical booking and login failures, followed by improvements to booking usability, tracking, and complaint resolution.

It also proposes a recurring scorecard to track ratings, recent review sentiment, recurring complaints, and potential release-related regressions.

## Technical Implementation

The Python monitoring script is designed to retrieve Play Store reviews and generate a Markdown scorecard using:

- Python
- google-play-scraper
- pandas

## Setup

Install the dependencies:

```bash
pip install -r requirements.txt
```

Run the offline demonstration:

```bash
python play_ratings_monitor.py --demo
```

Run live collection, after validating the script and package identifiers:

```bash
python play_ratings_monitor.py
```

Live collection depends on external access and the availability of the relevant Play Store data. Its outputs should be validated before being used for business decisions.

## Deliverables

- Case study recommendation PDF
- Python monitoring script
- Dependency list
- Example scorecard, if included

## Methodology and Limitations

I used publicly available information and a small sample of user reviews for this analysis. The review categories are based on keyword matching, so they may not always be accurate. The script still needs to be tested with live Play Store data to make sure the results are correct before using them for further analysis.

#

