# MXD
Op
import requests
import pandas as pd
import numpy as np
import time
import ta

# === Configuration ===
BROKER_API_KEY = "https://qxbroker.com/en/demo-trade"
BROKER_API_URL = "https://qxbroker.com/en/demo-trade  # Replace with the actual broker API URL
TRADING_PAIR = "EUR/USD"
TIMEFRAME = "1m"  # Change to preferred timeframe (1m, 5m, 15m, etc.)
TRADE_AMOUNT = 10  # Amount per trade
RISK_REWARD_RATIO = 2  # Define risk-reward ratio
STOP_LOSS_PERCENT = 0.02  # 2% stop-loss
TAKE_PROFIT_PERCENT = STOP_LOSS_PERCENT * RISK_REWARD_RATIO

# === Function to Get Market Data ===
def fetch_market_data():
    url = f"{BROKER_API_URL}/market_data?pair={TRADING_PAIR}&timeframe={TIMEFRAME}"
    headers = {"Authorization": f"Bearer {BROKER_API_KEY}"}
    response = requests.get(url, headers=headers)
    data = response.json()
    
    if "candles" in data:
        df = pd.DataFrame(data["candles"])
        df["timestamp"] = pd.to_datetime(df["timestamp"], unit="s")
        df.set_index("timestamp", inplace=True)
        return df
    else:
        print("Error fetching market data:", data)
        return None

# === Function to Compute Technical Indicators ===
def apply_indicators(df):
    df["SMA_50"] = ta.trend.sma_indicator(df["close"], window=50)
    df["SMA_200"] = ta.trend.sma_indicator(df["close"], window=200)
    df["RSI"] = ta.momentum.rsi(df["close"], window=14)
    
    macd = ta.trend.MACD(df["close"])
    df["MACD"] = macd.macd()
    df["MACD_Signal"] = macd.macd_signal()
    
    return df

# === Function to Generate Buy/Sell Signals ===
def generate_signal(df):
    latest = df.iloc[-1]
    
    # Buy Signal: RSI < 30, MACD > MACD Signal, SMA_50 > SMA_200 (Uptrend)
    if latest["RSI"] < 30 and latest["MACD"] > latest["MACD_Signal"] and latest["SMA_50"] > latest["SMA_200"]:
        return "BUY"
    
    # Sell Signal: RSI > 70, MACD < MACD Signal, SMA_50 < SMA_200 (Downtrend)
    elif latest["RSI"] > 70 and latest["MACD"] < latest["MACD_Signal"] and latest["SMA_50"] < latest["SMA_200"]:
        return "SELL"
    
    return "HOLD"

# === Function to Place Trades via API ===
def place_trade(order_type):
    url = f"{BROKER_API_URL}/place_order"
    headers = {"Authorization": f"Bearer {BROKER_API_KEY}"}
    order = {
        "pair": TRADING_PAIR,
        "order_type": order_type,
        "amount": TRADE_AMOUNT,
        "stop_loss": STOP_LOSS_PERCENT,
        "take_profit": TAKE_PROFIT_PERCENT,
    }
    
    response = requests.post(url, json=order, headers=headers)
    result = response.json()
    print(f"Trade executed: {order_type} - Response: {result}")

# === Main Bot Execution Loop ===
def trading_bot():
    while True:
        print("Fetching market data...")
        df = fetch_market_data()
        
        if df is not None:
            df = apply_indicators(df)
            signal = generate_signal(df)
            
            if signal == "BUY":
                print("BUY Signal detected! Placing trade...")
                place_trade("BUY")
            elif signal == "SELL":
                print("SELL Signal detected! Placing trade...")
                place_trade("SELL")
            else:
                print("No trade signal. Waiting for next candle...")
        
        time.sleep(60)  # Wait for the next minute candle

# Run the bot
if __name__ == "__main__":
    trading_bot()
