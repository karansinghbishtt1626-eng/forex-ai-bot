from http.server import BaseHTTPRequestHandler, HTTPServer
import os
import threading
import time
import feedparser
import openai
import requests
import telebot
from telebot import types

# --- Dummy Web Server (Render Free Tier Ke Liye) ---
class SimpleHTTPRequestHandler(BaseHTTPRequestHandler):

  def do_GET(self):
    self.send_response(200)
    self.end_headers()
    self.wfile.write(b"Bot Server is Active!")


def run_web_server():
  port = int(os.environ.get("PORT", 8080))
  server_address = ("0.0.0.0", port)
  httpd = HTTPServer(server_address, SimpleHTTPRequestHandler)
  httpd.serve_forever()


# --- Main Bot Setup ---
TELEGRAM_BOT_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")
ADMIN_CHAT_ID = os.getenv("ADMIN_CHAT_ID")
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")

bot = telebot.TeleBot(TELEGRAM_BOT_TOKEN)
openai.api_key = OPENAI_API_KEY

RSS_FEEDS = {
    "Forex Factory": "https://www.forexfactory.com/news.xml",
    "Fed Reserve": "https://www.federalreserve.gov/feeds/press_all.xml",
    "Reuters": (
        "https://www.reutersagency.com/feed/?taxonomy=best-sectors&post_type=best"
    ),
}


def analyze_market_news(title, summary):
  prompt = f"""
    You are an expert Forex and Commodity (XAU/USD, Dollar Index) analyst.
    Analyze this news:
    Title: {title}
    Summary: {summary}
    
    Give output in strict format:
    Impact: [HIGH/MEDIUM/LOW]
    Category: [Central Bank / War / Political / Macro]
    XAU_USD: [BULLISH/BEARISH/NEUTRAL]
    Reason: (1 sentence in Hinglish explaining why)
    """
  try:
    response = openai.ChatCompletion.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.3,
    )
    return response.choices[0].message.content
  except Exception as e:
    return f"AI Analysis Error: {str(e)}"


@bot.message_handler(commands=["start"])
def send_welcome(message):
  markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
  markup.add(
      types.KeyboardButton("/latest_news"), types.KeyboardButton("/gold_status")
  )
  bot.reply_to(
      message,
      "🤖 *AI Market Bot Online!*",
      parse_mode="Markdown",
      reply_markup=markup,
  )


@bot.message_handler(commands=["latest_news"])
def fetch_latest_news(message):
  bot.reply_to(message, "🔍 Latest news fetch ho rahi hai...")
  for source, url in RSS_FEEDS.items():
    feed = feedparser.parse(url)
    if feed.entries:
      entry = feed.entries[0]
      analysis = analyze_market_news(
          entry.title, getattr(entry, "summary", "")
      )
      bot.send_message(
          ADMIN_CHAT_ID,
          f"📰 *{source}*\n*Title:* {entry.title}\n\n{analysis}",
          parse_mode="Markdown",
      )
      break


@bot.message_handler(commands=["gold_status"])
def gold_status(message):
  prompt = (
      "Give a quick summary of current XAU/USD (Gold) market sentiment, key"
      " levels, and major driving factors today."
  )
  try:
    response = openai.ChatCompletion.create(
        model="gpt-4o-mini", messages=[{"role": "user", "content": prompt}]
    )
    bot.reply_to(
        message,
        f"🪙 *XAU/USD Outlook:*\n\n{response.choices[0].message.content}",
        parse_mode="Markdown",
    )
  except Exception as e:
    bot.reply_to(message, f"Error: {str(e)}")


def background_monitor():
  seen_links = set()
  while True:
    try:
      for source, url in RSS_FEEDS.items():
        feed = feedparser.parse(url)
        for entry in feed.entries[:3]:
          if entry.link not in seen_links:
            seen_links.add(entry.link)
            analysis = analyze_market_news(
                entry.title, getattr(entry, "summary", "")
            )
            if "HIGH" in analysis or "MEDIUM" in analysis:
              bot.send_message(
                  ADMIN_CHAT_ID,
                  f"🚨 *HIGH IMPACT ALERT* [{source}]\n\n*Title:*"
                  f" {entry.title}\n\n{analysis}",
                  parse_mode="Markdown",
              )
    except Exception as e:
      print(f"Error: {e}")
    time.sleep(300)


if __name__ == "__main__":
  # Start Web Server in background thread
  threading.Thread(target=run_web_server, daemon=True).start()
  # Start Background News Monitor thread
  threading.Thread(target=background_monitor, daemon=True).start()

  print("Bot server running...")
  bot.infinity_polling()
