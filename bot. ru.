import os
import asyncio
import logging
from aiogram import Bot, Dispatcher, types
from aiogram.filters import Command
from aiogram.types import Message
from binance.client import Client
import pandas as pd
import ta
from datetime import datetime

BOT_TOKEN = os.getenv("BOT_TOKEN")
BINANCE_API_KEY = os.getenv("BINANCE_API_KEY", "")
BINANCE_API_SECRET = os.getenv("BINANCE_API_SECRET", "")

bot = Bot(token=BOT_TOKEN)
dp = Dispatcher()
client = Client(BINANCE_API_KEY, BINANCE_API_SECRET)

logging.basicConfig(level=logging.INFO)

def get_klines(symbol: str, interval="15m", limit=100):
    klines = client.get_klines(symbol=symbol, interval=interval, limit=limit)
    df = pd.DataFrame(klines, columns=[
        'timestamp', 'open', 'high', 'low', 'close', 'volume',
        'close_time', 'quote_asset_volume', 'number_of_trades',
        'taker_buy_base', 'taker_buy_quote', 'ignore'
    ])
    df['close'] = pd.to_numeric(df['close'])
    df['high'] = pd.to_numeric(df['high'])
    df['low'] = pd.to_numeric(df['low'])
    return df

def generate_signal(symbol: str):
    try:
        df = get_klines(symbol)
        
        df['rsi'] = ta.momentum.RSIIndicator(df['close'], window=14).rsi()
        df['sma_fast'] = ta.trend.SMAIndicator(df['close'], window=9).sma_indicator()
        df['sma_slow'] = ta.trend.SMAIndicator(df['close'], window=21).sma_indicator()
        
        last = df.iloc[-1]
        prev = df.iloc[-2]
        
        price = last['close']
        rsi = last['rsi']
        
        signal = "NEUTRAL"
        reason = []
        
        if rsi < 30 and last['sma_fast'] > last['sma_slow']:
            signal = "LONG 🟢"
            reason.append("RSI перепродан + быстрая MA выше медленной")
        elif rsi > 70 and last['sma_fast'] < last['sma_slow']:
            signal = "SHORT 🔴"
            reason.append("RSI перекуплен + быстрая MA ниже медленной")
        elif last['sma_fast'] > last['sma_slow'] and prev['sma_fast'] <= prev['sma_slow']:
            signal = "LONG 🟢"
            reason.append("Пересечение MA вверх")
        elif last['sma_fast'] < last['sma_slow'] and prev['sma_fast'] >= prev['sma_slow']:
            signal = "SHORT 🔴"
            reason.append("Пересечение MA вниз")
        else:
            reason.append("Нет чёткого сигнала")
        
        if "LONG" in signal:
            entry = price
            stop = round(price * 0.985, 4)
            tp1 = round(price * 1.015, 4)
            tp2 = round(price * 1.03, 4)
        elif "SHORT" in signal:
            entry = price
            stop = round(price * 1.015, 4)
            tp1 = round(price * 0.985, 4)
            tp2 = round(price * 0.97, 4)
        else:
            entry = stop = tp1 = tp2 = price
        
        text = f"""
<pre>
Trading AI Bot [Version 10.0.19570.1000]
(c) 2025 Aignals Corporation. All rights reserved.

Bot:\\> signal {symbol}

══════════════════════════════════════
СИМВОЛ:     {symbol}
ЦЕНА:       {price:.4f} USDT
RSI(14):    {rsi:.1f}
СИГНАЛ:     {signal}
══════════════════════════════════════
ВХОД:       {entry:.4f}
СТОП:       {stop:.4f}
ТЕЙК 1:     {tp1:.4f}
ТЕЙК 2:     {tp2:.4f}
══════════════════════════════════════
ПРИЧИНА:    {', '.join(reason)}
ВРЕМЯ:      {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
══════════════════════════════════════

TRY IT FOR FREE NOW
</pre>
"""
        return text
    except Exception as e:
        return f"<pre>Ошибка получения данных по {symbol}:\n{str(e)}</pre>"

@dp.message(Command("start"))
async def cmd_start(message: Message):
    text = """
<pre>
Trading AI Bot [Version 10.0.19570.1000]
(c) 2025 Aignals Corporation. All rights reserved.

Bot:\\> 

Добро пожаловать в Aignals Trading AI Bot.

Доступные команды:
/signal BTCUSDT   — сигнал по BTC
/signal ETHUSDT   — сигнал по ETH
/status           — статус бота
/help             — помощь

TRY IT FOR FREE NOW
</pre>
"""
    await message.answer(text, parse_mode="HTML")

@dp.message(Command("help"))
async def cmd_help(message: Message):
    text = """
<pre>
Bot:\\> help

Команды:
/signal СИМВОЛ   — получить сигнал (пример: /signal BTCUSDT)
/status          — статус системы
/start           — перезапуск

Поддерживаются все пары Binance USDT.
Сигналы носят исключительно информационный характер.
</pre>
"""
    await message.answer(text, parse_mode="HTML")

@dp.message(Command("status"))
async def cmd_status(message: Message):
    text = f"""
<pre>
Bot:\\> status

Система:          ONLINE
Биржа:            Binance
Режим:            Только сигналы
Версия:           10.0.19570.1000
Время сервера:    {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}

TRY IT FOR FREE NOW
</pre>
"""
    await message.answer(text, parse_mode="HTML")

@dp.message(Command("signal"))
async def cmd_signal(message: Message):
    args = message.text.split()
    if len(args) < 2:
        await message.answer("<pre>Использование: /signal BTCUSDT</pre>", parse_mode="HTML")
        return
    
    symbol = args[1].upper()
    if not symbol.endswith("USDT"):
        symbol += "USDT"
    
    await message.answer("<pre>Bot:\\> Анализирую рынок...</pre>", parse_mode="HTML")
    signal_text = generate_signal(symbol)
    await message.answer(signal_text, parse_mode="HTML")

async def main():
    print("Bot started...")
    await dp.start_polling(bot)

if __name__ == "__main__":
    asyncio.run(main())
