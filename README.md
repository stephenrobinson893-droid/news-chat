import feedparser
import requests
import time
import json
import os
import logging
from datetime import datetime

# ==============================
# CONFIG
# ==============================
WEBHOOK_URL = "[YOUR_NEW_WEBHOOK_URL_HERE](https://discord.com/api/webhooks/1487962367860932629/fTtiOk0vnpcr_PXqvsXYOqlLbiJsGqq4Mh4_qUT5c9h_-c8WaWRQtQ8DxVi8ZSU9DiJT)"  # Regenerate this in Discord — old one is compromised

FEEDS = [
    "https://www.fxstreet.com/rss/news",
    "https://www.forexlive.com/feed/",
    "https://news.google.com/rss/search?q=gold+price+fed+inflation+USD",
    "https://news.google.com/rss/search?q=stock+market+economy+finance"
]

HIGH_IMPACT_KEYWORDS = ["fed", "fomc", "cpi", "interest rate", "treasury", "yield"]
MEDIUM_IMPACT_KEYWORDS = ["inflation", "gold", "xauusd", "usd", "dollar"]
LOW_IMPACT_KEYWORDS = ["market", "stocks", "economy", "finance"]

CHECK_INTERVAL = 300        # seconds (5 minutes)
SENT_FILE = "sent_links.json"
MAX_SENT_LINKS = 5000       # prevent file growing forever
REQUEST_TIMEOUT = 10        # seconds

# ==============================
# LOGGING SETUP
# ==============================
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    handlers=[
        logging.FileHandler("news_chat.log"),
        logging.StreamHandler()
    ]
)
log = logging.getLogger(__name__)

# ==============================
# PERSISTENCE
# ==============================
def load_sent_links():
    """Load previously sent links from disk."""
    if os.path.exists(SENT_FILE):
        try:
            with open(SENT_FILE, "r") as f:
                return set(json.load(f))
        except Exception as e:
            log.warning(f"Could not load sent links file: {e}")
    return set()

def save_sent_links(sent_links):
    """Save sent links to disk, trimming if too large."""
    try:
        # Keep only the most recent MAX_SENT_LINKS entries
        links_list = list(sent_links)[-MAX_SENT_LINKS:]
        with open(SENT_FILE, "w") as f:
            json.dump(links_list, f)
    except Exception as e:
        log.warning(f"Could not save sent links: {e}")

# ==============================
# CLASSIFICATION
# ==============================
def classify_impact(title):
    """Determine the impact level based on keywords in title."""
    title_lower = title.lower()
    if any(k in title_lower for k in HIGH_IMPACT_KEYWORDS):
        return "🔥 HIGH IMPACT"
    elif any(k in title_lower for k in MEDIUM_IMPACT_KEYWORDS):
        return "⚠️ MEDIUM IMPACT"
    elif any(k in title_lower for k in LOW_IMPACT_KEYWORDS):
        return "ℹ️ LOW IMPACT"
    return None

# ==============================
# DISCORD
# ==============================
def send_to_discord(title, link, impact, pub_date=None):
    """Send a formatted news item to Discord via webhook."""
    date_str = f"\n🕐 {pub_date}" if pub_date else ""
    message = f"{impact}{date_str}\n**{title}**\n{link}"

    try:
        response = requests.post(
            WEBHOOK_URL,
            json={"content": message},
            timeout=REQUEST_TIMEOUT
        )
        if response.status_code == 429:
            # Discord rate limit — wait and retry once
            retry_after = response.json().get("retry_after", 5)
            log.warning(f"Discord rate limited. Retrying after {retry_after}s...")
            time.sleep(retry_after)
            requests.post(WEBHOOK_URL, json={"content": message}, timeout=REQUEST_TIMEOUT)
        elif response.status_code not in (200, 204):
            log.warning(f"Discord returned status {response.status_code}")
        else:
            log.info(f"Sent [{impact}]: {title[:60]}...")
    except requests.exceptions.Timeout:
        log.error("Discord request timed out.")
    except Exception as e:
        log.error(f"Failed to send to Discord: {e}")

# ==============================
# FEED FETCHING
# ==============================
def get_pub_date(entry):
    """Extract a readable publish date from a feed entry."""
    try:
        t = entry.get("published_parsed") or entry.get("updated_parsed")
        if t:
            return datetime(*t[:6]).strftime("%Y-%m-%d %H:%M UTC")
    except Exception:
        pass
    return None

def fetch_news(sent_links):
    """Fetch all feeds and post new relevant items to Discord."""
    new_count = 0

    for feed_url in FEEDS:
        try:
            log.info(f"Fetching: {feed_url}")
            feed = feedparser.parse(feed_url)

            if not feed.entries:
                log.warning(f"No entries found for: {feed_url}")
                continue

            for entry in feed.entries:
                title = entry.get("title", "").strip()
                link = entry.get("link", "").strip()

                if not title or not link:
                    continue
                if link in sent_links:
                    continue

                impact = classify_impact(title)
                if impact:
                    pub_date = get_pub_date(entry)
                    send_to_discord(title, link, impact, pub_date)
                    sent_links.add(link)
                    new_count += 1
                    time.sleep(1)  # avoid hammering Discord

        except Exception as e:
            log.error(f"Error fetching feed {feed_url}: {e}")

    log.info(f"Cycle complete. {new_count} new item(s) posted.")
    return sent_links

# ==============================
# MAIN LOOP
# ==============================
def main():
    log.info("=== News-Chat Bot Starting ===")
    sent_links = load_sent_links()
    log.info(f"Loaded {len(sent_links)} previously sent links.")

    while True:
        try:
            sent_links = fetch_news(sent_links)
            save_sent_links(sent_links)
            log.info(f"Sleeping for {CHECK_INTERVAL // 60} minutes...\n")
            time.sleep(CHECK_INTERVAL)
        except KeyboardInterrupt:
            log.info("Stopped by user.")
            save_sent_links(sent_links)
            break
        except Exception as e:
            log.error(f"Unexpected error in main loop: {e}")
            time.sleep(60)

if __name__ == "__main__":
    main()
