**Code**
```
import json
import time
import paho.mqtt.client as mqtt
from google import genai
import requests

# ─── Config ────────────────────────────────────────────────
GEMINI_API_KEY   = "AIzaSyBSPUqoamlfpthaVtCzFqvdSHXkFhQK42A"        # your Gemini key
MQTT_BROKER      = "broker.hivemq.com"
MQTT_PORT        = 1883
MQTT_TOPICS      = [
    "solar/fault/panel1",
    "solar/fault/panel2",
    "solar/fault/panel3",
]

# ─── Telegram Config ───────────────────────────────────────
TELEGRAM_TOKEN   = "8951414252:AAH1QBPGd8vYtue5mKOAeLyAngEux6H1Czs"
TELEGRAM_CHAT_ID = "1769790851"

# ─── Gemini setup ──────────────────────────────────────────
client_ai    = genai.Client(api_key=GEMINI_API_KEY)
GEMINI_MODEL = "gemini-3.5-flash"

# ─── Cooldown ──────────────────────────────────────────────
last_fault_time  = {}
COOLDOWN_SECONDS = 300

# ─── Telegram sender ───────────────────────────────────────
def send_telegram(message):
    try:
        url  = f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendMessage"
        data = {
            "chat_id"    : TELEGRAM_CHAT_ID,
            "text"       : message,
            "parse_mode" : "Markdown"
        }
        response = requests.post(url, data=data, timeout=10)
        if response.status_code == 200:
            print("[Telegram] Message sent successfully")
        else:
            print(f"[Telegram] Failed — status {response.status_code}: {response.text}")
    except Exception as e:
        print(f"[Telegram] Error: {e}")

# ─── Gemini API call with retry ────────────────────────────
def ask_gemini(panel_num, data):
    fault_names = {
        "F1": "Low voltage (dust/sand)",
        "F2": "Zero voltage (panel dead)",
        "F3": "Hotspot (overheating)"
    }
    fault_description = fault_names.get(data["fault"], "Unknown fault")

    prompt = f"""You are a solar panel maintenance expert AI.
A fault has been detected on Panel {panel_num} of a 3-panel 24V solar monitoring system.

Sensor readings:
- Fault code  : {data['fault']} — {fault_description}
- Voltage     : {data['v']:.2f} V  (nominal: 24V)
- Temperature : {data['t']:.1f} °C
- Humidity    : {data['h']:.1f} %
- Efficiency  : {data['eff']} %

Respond in exactly 3 short lines:
1. CAUSE: One sentence on the most likely root cause.
2. RISK: One sentence on what will happen if ignored.
3. ACTION: One concrete maintenance step to take right now."""

    wait_times = [10, 30, 60]

    for attempt, wait in enumerate(wait_times, 1):
        try:
            response = client_ai.models.generate_content(
                model=GEMINI_MODEL,
                contents=prompt
            )
            return response.text.strip()

        except Exception as e:
            err = str(e)
            if "429" in err or "exhausted" in err.lower() or "RESOURCE_EXHAUSTED" in err:
                if attempt < len(wait_times):
                    print(f"[Gemini] Rate limited — waiting {wait}s (retry {attempt}/{len(wait_times)})...")
                    time.sleep(wait)
                else:
                    print("[Gemini] All retries exhausted — returning fallback message")
                    return (
                        f"CAUSE: Fault {data['fault']} detected but AI quota exceeded.\n"
                        f"RISK: Panel condition unknown until next AI check.\n"
                        f"ACTION: Inspect panel manually — V={data['v']:.2f}V  T={data['t']:.1f}C"
                    )
            else:
                print(f"[Gemini] Unexpected error: {e}")
                raise

# ─── MQTT callbacks ────────────────────────────────────────
def on_connect(client, userdata, connect_flags, reason_code, properties):
    if not reason_code.is_failure:
        print("[MQTT] Connected to HiveMQ broker")
        for topic in MQTT_TOPICS:
            client.subscribe(topic)
            print(f"[MQTT] Subscribed: {topic}")
    else:
        print(f"[MQTT] Connection failed: {reason_code}")

def on_message(client, userdata, msg, properties=None):
    topic     = msg.topic
    panel_num = topic.split("panel")[1]

    try:
        data = json.loads(msg.payload.decode())
    except Exception as e:
        print(f"[ERROR] Bad MQTT payload: {e}")
        return

    fault = data.get("fault", "OK")
    print(f"[Panel {panel_num}] V={data['v']:.2f}V  T={data['t']:.1f}C  H={data['h']:.1f}%  Eff={data['eff']}%  Fault={fault}")

    if fault == "OK":
        print(f"[Panel {panel_num}] Status normal — no AI call needed")
        return

    now  = time.time()
    last = last_fault_time.get(panel_num, 0)
    if now - last < COOLDOWN_SECONDS:
        remaining = int(COOLDOWN_SECONDS - (now - last))
        print(f"[Panel {panel_num}] Cooldown active — skipping AI ({remaining}s remaining)")
        return

    last_fault_time[panel_num] = now

    print(f"[Panel {panel_num}] FAULT {fault} detected — asking Gemini AI...")
    try:
        diagnosis = ask_gemini(panel_num, data)

        print(f"\n{'='*52}")
        print(f"  AI DIAGNOSIS — Panel {panel_num}  |  Fault: {fault}")
        print(f"{'='*52}")
        print(diagnosis)
        print(f"{'='*52}\n")

        # Build Telegram message with Markdown formatting
        fault_emoji = {"F1": "🟡", "F2": "🔴", "F3": "🔥"}.get(fault, "⚠️")
        alert = (
            f"{fault_emoji} *SOLAR FAULT ALERT*\n"
            f"{'─'*25}\n"
            f"*Panel*    : {panel_num}\n"
            f"*Fault*    : {fault}\n"
            f"*Voltage*  : {data['v']:.2f} V\n"
            f"*Temp*     : {data['t']:.1f} °C\n"
            f"*Humidity* : {data['h']:.1f} %\n"
            f"*Eff*      : {data['eff']} %\n"
            f"{'─'*25}\n"
            f"{diagnosis}"
        )
        send_telegram(alert)

    except Exception as e:
        print(f"[ERROR] Gemini call failed: {e}")

def on_disconnect(client, userdata, disconnect_flags, reason_code, properties):
    print(f"[MQTT] Disconnected — reason: {reason_code}")

# ─── Main ──────────────────────────────────────────────────
def main():
    print("=" * 52)
    print("  Solar AI Monitor — Gemini + Telegram")
    print(f"  Model   : {GEMINI_MODEL}")
    print(f"  Broker  : {MQTT_BROKER}")
    print(f"  Cooldown: {COOLDOWN_SECONDS}s per panel")
    print("=" * 52)

    # Send startup message to Telegram
    send_telegram(
        "✅ *Solar AI Monitor Started*\n"
        f"Model: `{GEMINI_MODEL}`\n"
        "Monitoring panels 1, 2 and 3\n"
        "You will be alerted on any fault."
    )

    client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
    client.on_connect    = on_connect
    client.on_message    = on_message
    client.on_disconnect = on_disconnect

    try:
        client.connect(MQTT_BROKER, MQTT_PORT, keepalive=60)
        client.loop_forever()
    except KeyboardInterrupt:
        print("\n[INFO] Stopped by user")
        send_telegram("⛔ *Solar AI Monitor Stopped* by user.")
    except Exception as e:
        print(f"[ERROR] Could not connect to broker: {e}")

if __name__ == "__main__":
    main()

```
