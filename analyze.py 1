"""
analyze.py (v4)
================
المرحلة 1: Analysis Mode — الآن مع بوت Telegram للتنبيهات التلقائية!

الجديد في هذه النسخة:
  1) عند ظهور فرصة مؤهلة (score >= 80)، يرسل رسالة للبوت الخاص بك على Telegram.
  2) الرسالة تحتوي على كل التفاصيل: العملة، السعر، نقطة الدخول، الوقف، الهدف، النسبة، الدرجة.
  3) الإشارة تُسجَّل في قاعدة البيانات أيضًا.
  4) لا داعي تفتح Colab يدويًا — فقط شغّل السكربت مرة أو مرتين يوميًا.

المتطلبات:
  pip install pandas requests

التشغيل:
  python3 analyze.py
  python3 analyze.py history
"""

import sys
import os
import sqlite3
import requests
import pandas as pd
from datetime import datetime, timezone

# =========================================================
# إعدادات Telegram
# =========================================================
TELEGRAM_BOT_TOKEN = "8978922278:AAGq-g6ojc0RcbFaBR_u21TC0UdW3sY6WqI"
TELEGRAM_CHAT_ID = "8747233123"
TELEGRAM_API_URL = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"

# =========================================================
# إعدادات التحليل
# =========================================================
SYMBOLS = [
    "BTCUSDT", "ETHUSDT", "BNBUSDT", "SOLUSDT",
    "XRPUSDT", "ADAUSDT", "DOGEUSDT", "AVAXUSDT",
]
INTERVAL = "1h"
CANDLE_LIMIT = 200
BINANCE_KLINES_URL = "https://api.binance.com/api/v3/klines"

MIN_SCORE_TO_ALERT = 80
SUPPORT_RESISTANCE_LOOKBACK = 20
MIN_RISK_REWARD = 2.0

DB_PATH = "/content/drive/MyDrive/signals.db"


# =========================================================
# Telegram
# =========================================================
def send_telegram_alert(symbol: str, result: dict) -> bool:
    """
    يرسل رسالة للبوت عند ظهور فرصة مؤهلة.
    يعيد True إن نجحت، False إن فشلت.
    """
    message = (
        f"🟢 <b>فرصة محتملة</b>\n\n"
        f"<b>العملة:</b> {symbol}\n"
        f"<b>الاتجاه:</b> {result['trend']}\n"
        f"<b>السعر الحالي:</b> {result['price']}\n"
        f"<b>نقطة الدخول:</b> {result['entry']}\n"
        f"<b>وقف الخسارة:</b> {result['stop_loss']}\n"
        f"<b>الهدف:</b> {result['target']}\n"
        f"<b>نسبة المخاطرة/العائد:</b> 1:{result['risk_reward']}\n"
        f"<b>درجة الإشارة:</b> {result['score']}/100\n\n"
        f"⏰ <i>{datetime.now(timezone.utc).isoformat()}</i>"
    )
    try:
        response = requests.post(
            TELEGRAM_API_URL,
            json={"chat_id": TELEGRAM_CHAT_ID, "text": message, "parse_mode": "HTML"},
            timeout=10,
        )
        return response.status_code == 200
    except Exception as e:
        print(f"⚠️ فشل إرسال التنبيه للبوت: {e}")
        return False


# =========================================================
# قاعدة البيانات
# =========================================================
def init_db(db_path: str = DB_PATH) -> None:
    conn = sqlite3.connect(db_path)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS signals (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            timestamp_utc TEXT NOT NULL,
            symbol TEXT NOT NULL,
            trend TEXT,
            entry_price REAL,
            stop_loss REAL,
            target REAL,
            risk_reward REAL,
            score REAL,
            reason TEXT,
            status TEXT DEFAULT 'open',
            result_pct REAL,
            closed_at TEXT
        )
    """)
    conn.commit()
    conn.close()


def save_signal(symbol: str, result: dict, db_path: str = DB_PATH) -> int:
    b = result["breakdown"]
    reason = (
        f"اتجاه({b['trend_alignment']}) + زخم({b['momentum_confirmation']}) + "
        f"حجم({b['volume_quality']}) + دخول({b['entry_quality']}) + "
        f"R:R({b['risk_reward']}) + سوق_عام({b['market_conditions']})"
    )
    conn = sqlite3.connect(db_path)
    cur = conn.execute(
        """
        INSERT INTO signals
            (timestamp_utc, symbol, trend, entry_price, stop_loss, target,
             risk_reward, score, reason, status)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, 'open')
        """,
        (
            datetime.now(timezone.utc).isoformat(),
            symbol,
            result["trend"],
            result["entry"],
            result["stop_loss"],
            result["target"],
            result["risk_reward"],
            result["score"],
            reason,
        ),
    )
    conn.commit()
    new_id = cur.lastrowid
    conn.close()
    return new_id


def update_signal_result(signal_id: int, status: str, result_pct: float, db_path: str = DB_PATH) -> None:
    conn = sqlite3.connect(db_path)
    conn.execute(
        """
        UPDATE signals
        SET status = ?, result_pct = ?, closed_at = ?
        WHERE id = ?
        """,
        (status, result_pct, datetime.now(timezone.utc).isoformat(), signal_id),
    )
    conn.commit()
    conn.close()


def show_history(db_path: str = DB_PATH) -> None:
    conn = sqlite3.connect(db_path)
    df = pd.read_sql_query("SELECT * FROM signals ORDER BY timestamp_utc DESC", conn)
    conn.close()

    if df.empty:
        print("لا توجد إشارات مسجَّلة بعد.")
        return

    print(f"\n📊 إجمالي الإشارات المسجَّلة: {len(df)}\n")
    print(df[["timestamp_utc", "symbol", "score", "entry_price", "stop_loss", "target", "status", "result_pct"]]
          .to_string(index=False))

    closed = df[df["status"].isin(["win", "loss"])]
    if not closed.empty:
        win_rate = (closed["status"] == "win").mean() * 100
        avg_result = closed["result_pct"].mean()
        print(f"\n--- ملخص أداء الصفقات المغلَقة ({len(closed)}) ---")
        print(f"نسبة الصفقات الرابحة: {win_rate:.1f}%")
        print(f"متوسط الربح/الخسارة: {avg_result:.2f}%")


# =========================================================
# جلب البيانات
# =========================================================
def fetch_klines(symbol: str, interval: str = INTERVAL, limit: int = CANDLE_LIMIT) -> pd.DataFrame:
    params = {"symbol": symbol, "interval": interval, "limit": limit}
    response = requests.get(BINANCE_KLINES_URL, params=params, timeout=10)
    response.raise_for_status()
    raw = response.json()

    columns = [
        "open_time", "open", "high", "low", "close", "volume",
        "close_time", "quote_asset_volume", "num_trades",
        "taker_buy_base", "taker_buy_quote", "ignore",
    ]
    df = pd.DataFrame(raw, columns=columns)
    for col in ["open", "high", "low", "close", "volume"]:
        df[col] = df[col].astype(float)
    df["open_time"] = pd.to_datetime(df["open_time"], unit="ms", utc=True)
    return df[["open_time", "open", "high", "low", "close", "volume"]]


# =========================================================
# المؤشرات الفنية
# =========================================================
def compute_ema(series: pd.Series, period: int) -> pd.Series:
    return series.ewm(span=period, adjust=False).mean()


def compute_rsi(series: pd.Series, period: int = 14) -> pd.Series:
    delta = series.diff()
    gain = delta.clip(lower=0)
    loss = -delta.clip(upper=0)
    avg_gain = gain.ewm(alpha=1 / period, min_periods=period, adjust=False).mean()
    avg_loss = loss.ewm(alpha=1 / period, min_periods=period, adjust=False).mean()
    rs = avg_gain / avg_loss
    return 100 - (100 / (1 + rs))


def compute_macd(series: pd.Series, fast: int = 12, slow: int = 26, signal: int = 9):
    ema_fast = compute_ema(series, fast)
    ema_slow = compute_ema(series, slow)
    macd_line = ema_fast - ema_slow
    signal_line = compute_ema(macd_line, signal)
    histogram = macd_line - signal_line
    return macd_line, signal_line, histogram


def compute_atr(df: pd.DataFrame, period: int = 14) -> pd.Series:
    high_low = df["high"] - df["low"]
    high_close = (df["high"] - df["close"].shift()).abs()
    low_close = (df["low"] - df["close"].shift()).abs()
    true_range = pd.concat([high_low, high_close, low_close], axis=1).max(axis=1)
    return true_range.ewm(alpha=1 / period, min_periods=period, adjust=False).mean()


def nearest_support_resistance(df: pd.DataFrame, lookback: int = SUPPORT_RESISTANCE_LOOKBACK):
    recent = df.tail(lookback)
    return recent["low"].min(), recent["high"].max()


# =========================================================
# محرك التسجيل
# =========================================================
def score_opportunity(df: pd.DataFrame, btc_trend_up: bool) -> dict:
    df = df.copy()
    df["ema20"] = compute_ema(df["close"], 20)
    df["ema50"] = compute_ema(df["close"], 50)
    df["rsi14"] = compute_rsi(df["close"], 14)
    df["macd"], df["macd_signal"], df["macd_hist"] = compute_macd(df["close"])
    df["atr14"] = compute_atr(df, 14)
    df["vol_avg20"] = df["volume"].rolling(20).mean()

    last = df.iloc[-1]
    price = last["close"]
    support, resistance = nearest_support_resistance(df)

    breakdown = {}

    ema_gap_pct = (last["ema20"] - last["ema50"]) / last["ema50"] * 100
    if last["ema20"] > last["ema50"]:
        trend_score = max(10, min(20, 10 + ema_gap_pct * 4))
        trend_direction = "صاعد"
    else:
        trend_score = 0
        trend_direction = "هابط/محايد"
    breakdown["trend_alignment"] = round(trend_score, 1)

    momentum_score = 0
    if 45 <= last["rsi14"] <= 65:
        momentum_score += 10
    elif 40 <= last["rsi14"] < 45 or 65 < last["rsi14"] <= 70:
        momentum_score += 5
    if last["macd_hist"] > 0 and last["macd_hist"] > df["macd_hist"].iloc[-2]:
        momentum_score += 10
    elif last["macd_hist"] > 0:
        momentum_score += 5
    breakdown["momentum_confirmation"] = momentum_score

    volume_score = 0
    if pd.notna(last["vol_avg20"]) and last["vol_avg20"] > 0:
        vol_ratio = last["volume"] / last["vol_avg20"]
        if vol_ratio >= 1.5:
            volume_score = 15
        elif vol_ratio >= 1.1:
            volume_score = 10
        elif vol_ratio >= 0.8:
            volume_score = 5
    breakdown["volume_quality"] = volume_score

    range_size = resistance - support if resistance > support else 1
    distance_from_support_pct = (price - support) / range_size
    if distance_from_support_pct <= 0.25:
        entry_score = 20
    elif distance_from_support_pct <= 0.4:
        entry_score = 12
    elif distance_from_support_pct <= 0.6:
        entry_score = 5
    else:
        entry_score = 0
    breakdown["entry_quality"] = entry_score

    atr = last["atr14"] if pd.notna(last["atr14"]) else (price * 0.01)
    suggested_entry = price
    suggested_stop = min(support, price - 1.5 * atr)
    suggested_target = resistance if resistance > price else price + 2 * atr

    risk = suggested_entry - suggested_stop
    reward = suggested_target - suggested_entry
    rr_ratio = reward / risk if risk > 0 else 0

    if rr_ratio >= 3:
        rr_score = 15
    elif rr_ratio >= MIN_RISK_REWARD:
        rr_score = 10
    elif rr_ratio >= 1.5:
        rr_score = 5
    else:
        rr_score = 0
    breakdown["risk_reward"] = rr_score

    market_score = 10 if btc_trend_up else 3
    breakdown["market_conditions"] = market_score

    total_score = sum(breakdown.values())

    return {
        "price": round(price, 6),
        "trend": trend_direction,
        "rsi": round(last["rsi14"], 2),
        "support": round(support, 6),
        "resistance": round(resistance, 6),
        "entry": round(suggested_entry, 6),
        "stop_loss": round(suggested_stop, 6),
        "target": round(suggested_target, 6),
        "risk_reward": round(rr_ratio, 2),
        "score": round(total_score, 1),
        "breakdown": breakdown,
    }


# =========================================================
# التشغيل الرئيسي
# =========================================================
def run_analysis():
    init_db()
    print(f"\n⏰ تشغيل التحليل — {datetime.now(timezone.utc).isoformat()} UTC")
    print(f"📊 العملات: {', '.join(SYMBOLS)} | الإطار الزمني: {INTERVAL}")
    print(f"📈 الحد الأدنى: {MIN_SCORE_TO_ALERT}/100 | البوت: مفعّل ✅\n")

    btc_trend_up = True
    try:
        btc_df = fetch_klines("BTCUSDT")
        btc_ema20 = compute_ema(btc_df["close"], 20).iloc[-1]
        btc_ema50 = compute_ema(btc_df["close"], 50).iloc[-1]
        btc_trend_up = btc_ema20 > btc_ema50
    except requests.RequestException as e:
        print(f"⚠️ تعذّر تحديد اتجاه BTC: {e}\n")

    opportunities = []
    for symbol in SYMBOLS:
        try:
            df = fetch_klines(symbol)
            result = score_opportunity(df, btc_trend_up)
            opportunities.append((symbol, result))
        except requests.RequestException as e:
            print(f"⚠️ تعذّر جلب {symbol}: {e}")
        except Exception as e:
            print(f"⚠️ خطأ في {symbol}: {e}")

    opportunities.sort(key=lambda x: x[1]["score"], reverse=True)
    qualified = [(s, r) for s, r in opportunities if r["score"] >= MIN_SCORE_TO_ALERT]

    if not qualified:
        print("🔴 لا توجد صفقة مناسبة حاليًا.\n")
        print("أفضل الدرجات:")
        for symbol, result in opportunities[:3]:
            print(f"  {symbol}: {result['score']}/100")
    else:
        print(f"✅ عثرت على {len(qualified)} فرصة مؤهلة!\n")
        for symbol, result in qualified:
            signal_id = save_signal(symbol, result)
            alert_sent = send_telegram_alert(symbol, result)
            status = "✓" if alert_sent else "⚠️"
            print(f"{status} {symbol} | الدرجة: {result['score']}/100 | نسبة R:R: 1:{result['risk_reward']}")
        print(f"\n✅ تم حفظ {len(qualified)} إشارة في قاعدة البيانات + إرسال تنبيهات.")


if __name__ == "__main__":
    if len(sys.argv) > 1 and sys.argv[1] == "history":
        show_history()
    else:
        run_analysis()
