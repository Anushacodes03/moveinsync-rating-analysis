#!/usr/bin/env python3
"""
MoveInSync Play Store Rating Monitor
------------------------------------
Creates a repeatable scorecard from public Google Play review data.

Install:
    pip install -r requirements.txt

Run demo mode (offline):
    python play_ratings_monitor.py --demo

Fetch recent reviews:
    python play_ratings_monitor.py

Options:
    python play_ratings_monitor.py --days 30 --max 1000
    python play_ratings_monitor.py --apps com.moveinsync.ets

Outputs are saved in ./out/:
    scorecard_YYYY-MM-DD.md
    reviews_YYYY-MM-DD.csv

Important:
- Live fetching depends on Google Play access and the scraper package.
- Review theme labels are simple keyword-based estimates, not ground truth.
- Verify live figures against the current Play Store listing / Play Console.
"""

from __future__ import annotations

import argparse
import datetime as dt
import os
import re
import sys
from typing import Dict, Tuple, Any

import pandas as pd


APPS = {
    "com.moveinsync.ets": "Employee app (main)",
    "field.moveinsync.ets": "Field app",
    "com.moveinsync.driver.dco": "Driver Onboarding",
    "com.moveinsync.driver.app": "Partner app",
    "com.moveinsync.driver.lite": "Partner Lite",
    "com.workinsync.meeting": "Workplace",
    "com.moveinsync.publictransit": "TransitInSync",
}

# First matching category is assigned as the primary theme.
# Review these labels manually before using them to make decisions.
THEMES = [
    ("otp_login_crash", r"\botp\b|log ?in|sign ?in|crash|force close|not opening|stuck|seat ?belt|buckle|verification"),
    ("tracking_eta", r"\bgps\b|track|live location|location (is|not)|\beta\b|late|delay|shuttle.{0,20}location"),
    ("no_show_checkin", r"no[- ]?show|check[- ]?in|can't make it|cannot make it|won't be able|cancellation"),
    ("ui_regression", r"update|upgrade|new (ui|version|design)|redesign|toggle|confus|clicks?|tab|button|2 step|two step|hidden|hide"),
    ("booking_notification", r"\bbook(ing|ed)?\b|schedule|roster|trip not|not booked|notification"),
    ("routing_service", r"route|pick ?up point|nodal|last drop|detour|driver|cab|helpdesk|support|complaint|transport"),
    ("positive", r"great|good|excellent|amazing|helpful|convenient|easy|best|game changer|nice"),
]


def tag_review(text: str) -> str:
    """Return one primary theme using transparent keyword matching."""
    content = (text or "").lower()
    for theme, pattern in THEMES:
        if re.search(pattern, content):
            return theme
    return "other"


def to_naive_datetime(value: Any) -> pd.Timestamp:
    """Convert a timestamp to a timezone-naive pandas Timestamp."""
    stamp = pd.to_datetime(value, errors="coerce", utc=True)
    if pd.isna(stamp):
        return pd.NaT
    return stamp.tz_convert(None)


def fetch_live(package_id: str, days: int, max_reviews: int) -> Tuple[dict, pd.DataFrame]:
    """Fetch recent reviews for one app, including the previous comparison window."""
    from google_play_scraper import app, reviews, Sort

    metadata = app(package_id, lang="en", country="in")
    cutoff = dt.datetime.now() - dt.timedelta(days=days * 2)
    collected = []
    continuation_token = None

    while len(collected) < max_reviews:
        remaining = max_reviews - len(collected)
        batch_size = min(200, remaining)
        batch, continuation_token = reviews(
            package_id,
            lang="en",
            country="in",
            sort=Sort.NEWEST,
            count=batch_size,
            continuation_token=continuation_token,
        )
        if not batch:
            break

        collected.extend(batch)
        oldest_in_batch = min((item.get("at") for item in batch if item.get("at")), default=None)

        if oldest_in_batch is not None and oldest_in_batch < cutoff:
            break
        if continuation_token is None:
            break

    if not collected:
        return metadata, pd.DataFrame(
            columns=["package", "date", "stars", "text", "version", "replied_at", "reply_text"]
        )

    rows = []
    for item in collected:
        rows.append({
            "package": package_id,
            "date": to_naive_datetime(item.get("at")),
            "stars": item.get("score"),
            "text": item.get("content") or "",
            "version": item.get("reviewCreatedVersion") or "unknown",
            "replied_at": to_naive_datetime(item.get("repliedAt")),
            "reply_text": item.get("replyContent") or "",
        })

    df = pd.DataFrame(rows)
    df = df.dropna(subset=["date", "stars"])
    df["stars"] = pd.to_numeric(df["stars"], errors="coerce")
    df = df.dropna(subset=["stars"])
    return metadata, df


def demo_data() -> Dict[str, Tuple[dict, pd.DataFrame]]:
    """Small illustrative dataset for testing the report format without network access."""
    today = dt.datetime.now()
    examples = [
        (1, 2, "OTP not received after many tries; cannot log in", "12.19.0"),
        (1, 5, "Trip booked but app says trip is not booked", "12.19.0"),
        (1, 9, "Seat belt check hides OTP and app keeps crashing", "12.19.0"),
        (1, 12, "Shuttle live location was removed after the update", "12.16.0"),
        (2, 15, "Recent update is confusing; booking tab is hard to find", "12.16.0"),
        (2, 20, "Booking a shuttle now takes two steps and is confusing", "12.16.0"),
        (1, 25, "GPS tracking is slow and pickup timings are wrong", "12.16.0"),
        (3, 28, "Need a clearer no-show option for the driver", "12.16.0"),
        (1, 30, "No check-in reminder; marked as no-show unexpectedly", "12.15.0"),
        (1, 33, "Pickup point is far away and helpdesk did not help", "12.15.0"),
        (5, 3, "Game changer for office transport; easy tracking", "12.19.0"),
        (5, 5, "Very convenient and helpful", "12.19.0"),
        (5, 8, "Amazing experience", "12.19.0"),
        (5, 22, "Good app and nice tracking", "12.16.0"),
        (5, 35, "Great and helpful", "12.15.0"),
    ]

    rows = []
    for stars, days_ago, text, version in examples:
        rows.append({
            "package": "com.moveinsync.ets",
            "date": today - dt.timedelta(days=days_ago),
            "stars": stars,
            "text": text,
            "version": version,
            "replied_at": pd.NaT,
            "reply_text": "",
        })

    df = pd.DataFrame(rows)
    # These figures are illustrative only and must not be represented as live data.
    metadata = {
        "title": "MoveInSync (DEMO DATA)",
        "score": 4.3,
        "ratings": 114607,
        "histogram": [14000, 960, 2200, 15000, 82000],  # 1-star through 5-star
    }
    return {"com.moveinsync.ets": (metadata, df)}


def pct(numerator: int, denominator: int) -> str:
    return f"{100 * numerator / denominator:.1f}%" if denominator else "n/a"


def safe_score(metadata: dict) -> str:
    value = metadata.get("score")
    try:
        return f"{float(value):.2f}"
    except (TypeError, ValueError):
        return "n/a"


def scorecard(results: Dict[str, Tuple[dict, pd.DataFrame]], days: int):
    now = pd.Timestamp.now().floor("s")
    lines = [
        f"# MoveInSync Play Store scorecard — {now:%d %b %Y}",
        f"Comparison: latest {days} days vs preceding {days} days.",
        "",
        "> Note: Public review data and keyword themes are indicative. Verify figures and manually review themes before making decisions.",
        "",
    ]
    all_frames = []

    for package_id, (metadata, raw_df) in results.items():
        label = APPS.get(package_id, package_id)
        df = raw_df.copy()
        lines.append(f"## {label} (`{package_id}`)")
        lines.append(f"- Store score reported by scraper: **{safe_score(metadata)}**")
        lines.append(f"- Total ratings reported by scraper: {metadata.get('ratings', 'n/a')}")

        histogram = metadata.get("histogram") or []
        if len(histogram) == 5:
            total_hist = sum(histogram)
            # Google Play histogram order is 1-star through 5-star.
            hist_summary = " | ".join(
                f"{star}★ {pct(int(histogram[star - 1]), total_hist)}"
                for star in range(1, 6)
            )
            lines.append(f"- Rating distribution (1★ to 5★): {hist_summary}")
        else:
            lines.append("- Rating distribution: not available from returned metadata.")

        if df.empty:
            lines.append("- No reviews were retrieved for this app.\n")
            continue

        df["date"] = pd.to_datetime(df["date"], errors="coerce")
        df["stars"] = pd.to_numeric(df["stars"], errors="coerce")
        df = df.dropna(subset=["date", "stars"])
        df["theme"] = df["text"].fillna("").map(tag_review)

        current_start = now - pd.Timedelta(days=days)
        previous_start = now - pd.Timedelta(days=2 * days)
        current = df[df["date"] >= current_start]
        previous = df[(df["date"] < current_start) & (df["date"] >= previous_start)]

        lines.append(f"- Reviews retrieved: {len(df)}")
        lines.append(
            f"- Reviews in latest window: {len(current)}; average stars: "
            f"{current['stars'].mean():.2f}" if not current.empty
            else "- Reviews in latest window: 0; average stars: n/a"
        )
        if not previous.empty:
            lines.append(
                f"- Reviews in previous window: {len(previous)}; average stars: {previous['stars'].mean():.2f}"
            )
        else:
            lines.append("- Reviews in previous window: 0; average stars: n/a")

        if not current.empty:
            one_star = int((current["stars"] == 1).sum())
            lines.append(f"- One-star share of latest reviews: **{pct(one_star, len(current))}**")
        else:
            lines.append("- One-star share of latest reviews: n/a")

        negative_current = current[current["stars"] <= 3]
        negative_previous = previous[previous["stars"] <= 3]
        if not negative_current.empty:
            lines.append("")
            lines.append("| Primary theme among reviews rated 1–3★ | Latest window | Previous window |")
            lines.append("|---|---:|---:|")
            themes = sorted(set(negative_current["theme"]) | set(negative_previous["theme"]))
            for theme in themes:
                cur_count = int((negative_current["theme"] == theme).sum())
                prev_count = int((negative_previous["theme"] == theme).sum())
                lines.append(
                    f"| {theme} | {pct(cur_count, len(negative_current))} "
                    f"({cur_count}) | {pct(prev_count, len(negative_previous))} ({prev_count}) |"
                )

        # Reply timing is calculated only where both timestamps exist.
        if "replied_at" in current.columns:
            replied = current.dropna(subset=["replied_at"]).copy()
            negative = current[current["stars"] <= 2]
            if not negative.empty:
                reply_rate = pct(len(replied[replied.index.isin(negative.index)]), len(negative))
                lines.append(f"- Developer reply rate on 1–2★ reviews: {reply_rate}")
                reply_subset = replied[replied.index.isin(negative.index)]
                if not reply_subset.empty:
                    days_to_reply = (
                        pd.to_datetime(reply_subset["replied_at"]) - pd.to_datetime(reply_subset["date"])
                    ).dt.total_seconds() / 86400
                    lines.append(f"- Median reply time on replied 1–2★ reviews: {days_to_reply.median():.1f} days")

        version_summary = current.groupby("version")["stars"].agg(["count", "mean"])
        version_summary = version_summary[version_summary["count"] >= 3].sort_index(ascending=False)
        if not version_summary.empty:
            lines.extend(["", "| App version | Reviews | Average stars |", "|---|---:|---:|"])
            for version, row in version_summary.iterrows():
                lines.append(f"| {version} | {int(row['count'])} | {row['mean']:.2f} |")
        lines.append("")
        all_frames.append(df)

    combined = pd.concat(all_frames, ignore_index=True) if all_frames else pd.DataFrame()
    return "\n".join(lines), combined


def main():
    parser = argparse.ArgumentParser(description="Create a MoveInSync Play Store rating scorecard.")
    parser.add_argument("--days", type=int, default=30, help="Length of each comparison window (default: 30)")
    parser.add_argument("--max", dest="max_reviews", type=int, default=1000, help="Maximum reviews to fetch per app")
    parser.add_argument("--demo", action="store_true", help="Use illustrative offline data; no network access")
    parser.add_argument("--apps", nargs="*", default=list(APPS), help="Package IDs to fetch")
    args = parser.parse_args()

    if args.days < 1:
        parser.error("--days must be at least 1")
    if args.max_reviews < 1:
        parser.error("--max must be at least 1")

    if args.demo:
        results = demo_data()
    else:
        results = {}
        for package_id in args.apps:
            try:
                print(f"Fetching {package_id}...", file=sys.stderr)
                results[package_id] = fetch_live(package_id, args.days, args.max_reviews)
            except Exception as exc:
                print(f"[warning] Could not fetch {package_id}: {exc}", file=sys.stderr)

    report, review_data = scorecard(results, args.days)
    os.makedirs("out", exist_ok=True)
    stamp = dt.date.today().isoformat()
    report_path = os.path.join("out", f"scorecard_{stamp}.md")
    csv_path = os.path.join("out", f"reviews_{stamp}.csv")

    with open(report_path, "w", encoding="utf-8") as handle:
        handle.write(report + "\n")
    if not review_data.empty:
        review_data.to_csv(csv_path, index=False, encoding="utf-8")

    print(report)
    print(f"\nSaved scorecard: {report_path}")
    if not review_data.empty:
        print(f"Saved review data: {csv_path}")


if __name__ == "__main__":
    main()
