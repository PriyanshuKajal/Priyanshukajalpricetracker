from flask import Flask, request, jsonify
import requests
import sqlite3
from bs4 import BeautifulSoup
import threading
import time

app = Flask(__name__)

# Database setup
conn = sqlite3.connect("products.db", check_same_thread=False)
cursor = conn.cursor()
cursor.execute("CREATE TABLE IF NOT EXISTS products (id INTEGER PRIMARY KEY, url TEXT, bot_token TEXT, chat_id TEXT)")
conn.commit()

# Amazon Price Fetch Function
def get_amazon_price(url):
    headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"}
    try:
        response = requests.get(url, headers=headers, timeout=10)
        soup = BeautifulSoup(response.text, "html.parser")
        price = soup.find("span", {"class": "a-price-whole"})
        return price.text.strip() if price else "Price not found"
    except Exception as e:
        return f"Error: {str(e)}"

# Telegram Alert Function
def send_telegram_alert(bot_token, chat_id, message):
    telegram_url = f"https://api.telegram.org/bot{bot_token}/sendMessage"
    params = {"chat_id": chat_id, "text": message}
    try:
        requests.get(telegram_url, params=params, timeout=10)
    except Exception as e:
        print(f"Telegram Error: {e}")

# Background Price Checker (Every 5 Min)
def price_checker():
    while True:
        try:
            conn = sqlite3.connect("products.db", check_same_thread=False)
            cursor = conn.cursor()
            cursor.execute("SELECT * FROM products")
            products = cursor.fetchall()
            conn.close()

            for product in products:
                product_id, url, bot_token, chat_id = product
                price = get_amazon_price(url)
                send_telegram_alert(bot_token, chat_id, f"🔔 Amazon Price Update: {url}\nCurrent Price: {price}")
            
            time.sleep(300)  # 5 Min Delay
        except Exception as e:
            print(f"Background Error: {e}")

# Start Background Thread
threading.Thread(target=price_checker, daemon=True).start()

# API Route To Add Product
@app.route("/track", methods=["POST"])
def track():
    data = request.json
    url = data.get("productUrl")
    bot_token = data.get("botToken")
    chat_id = data.get("chatId")

    if not url or not bot_token or not chat_id:
        return jsonify({"error": "Missing required fields"}), 400

    conn = sqlite3.connect("products.db")
    cursor = conn.cursor()
    cursor.execute("INSERT INTO products (url, bot_token, chat_id) VALUES (?, ?, ?)", (url, bot_token, chat_id))
    conn.commit()
    conn.close()

    return jsonify({"message": "✅ Product Tracking Started!"})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=10000, debug=True)
