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
    headers = {"User-Agent": "Mozilla/5.0"}
    response = requests.get(url, headers=headers)
    soup = BeautifulSoup(response.text, "html.parser")
    price = soup.find("span", {"class": "a-price-whole"})
    return price.text if price else "Price not found"

# Telegram Alert Function
def send_telegram_alert(bot_token, chat_id, message):
    telegram_url = f"https://api.telegram.org/bot{bot_token}/sendMessage?chat_id={chat_id}&text={message}"
    requests.get(telegram_url)

# Background Price Checker (Every 5 Min)
def price_checker():
    while True:
        cursor.execute("SELECT * FROM products")
        products = cursor.fetchall()
        for product in products:
            product_id, url, bot_token, chat_id = product
            price = get_amazon_price(url)
            send_telegram_alert(bot_token, chat_id, f"🔔 Amazon Price Update: {url} \nCurrent Price: {price}")
        time.sleep(300)  # 5 Min Delay

# Start Background Thread
threading.Thread(target=price_checker, daemon=True).start()

# API Route To Add Product
@app.route("/track", methods=["POST"])
def track():
    data = request.json
    url = data.get("productUrl")
    bot_token = data.get("botToken")
    chat_id = data.get("chatId")

    cursor.execute("INSERT INTO products (url, bot_token, chat_id) VALUES (?, ?, ?)", (url, bot_token, chat_id))
    conn.commit()

    return jsonify({"message": "✅ Product Tracking Started!"})

if __name__ == "__main__":
    app.run(debug=True)
