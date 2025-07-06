import requests
from bs4 import BeautifulSoup
import time

# === KONFIGURATION ===
TELEGRAM_TOKEN = "DEIN_BOT_TOKEN_HIER"
allowed_users = [123456789]  # <- Deine Telegram-ID hier eintragen

# === GLOBAL STATE ===
gesehen = set()
bot_paused = False
waiting_for_search = {}
suchbegriffe = ["yamaha aerox", "jetforce", "speedfight", "beta", "roller 50cc", "beta rr50"]

def telegram_senden(text, chat_id=None):
    if chat_id:
        url = f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendMessage"
        requests.post(url, data={"chat_id": chat_id, "text": text})
    else:
        for uid in allowed_users:
            url = f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendMessage"
            requests.post(url, data={"chat_id": uid, "text": text})

def befehle_pruefen():
    global bot_paused, waiting_for_search, suchbegriffe

    updates = requests.get(f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/getUpdates").json().get("result", [])
    for update in updates:
        if not "message" in update:
            continue
        msg = update["message"]
        chat_id = msg["chat"]["id"]
        user_id = msg["from"]["id"]
        text = msg.get("text", "").strip().lower()

        if user_id not in allowed_users:
            continue

        if user_id in waiting_for_search:
            begriff = msg["text"].strip()
            suchbegriffe.clear()
            suchbegriffe.append(begriff)
            waiting_for_search.pop(user_id)
            telegram_senden(f"🔍 Suche gestartet nach: {begriff}", chat_id)
            continue

        if text == "/start":
            telegram_senden("✅ Befehle:\n/search - Neue Suche\n/pause - Bot pausieren\n/resume - Bot starten", chat_id)
        elif text == "/pause":
            bot_paused = True
            telegram_senden("⏸ Bot pausiert.", chat_id)
        elif text == "/resume":
            bot_paused = False
            telegram_senden("▶ Bot läuft wieder.", chat_id)
        elif text == "/search":
            waiting_for_search[user_id] = True
            telegram_senden("Was möchtest du suchen?", chat_id)

def suche_starten():
    global gesehen
    gefunden = False
    headers = {"User-Agent": "Mozilla/5.0"}

    for begriff in suchbegriffe:
        url = "https://www.kleinanzeigen.de/s-motorradroller/c305"
        params = {"keywords": begriff, "maxPrice": 500}
        response = requests.get(url, params=params, headers=headers)
        soup = BeautifulSoup(response.text, "html.parser")
        angebote = soup.select(".aditem")

        for angebot in angebote:
            a_tag = angebot.select_one("a")
            titel_tag = angebot.select_one(".ellipsis")
            preis_tag = angebot.select_one(".aditem-main--middle--price")

            if a_tag and titel_tag and preis_tag:
                link = "https://www.kleinanzeigen.de" + a_tag["href"]
                if link in gesehen:
                    continue

                titel = titel_tag.get_text(strip=True)
                preis = preis_tag.get_text(strip=True)

                # Detailinfos
                detail_resp = requests.get(link, headers=headers)
                detail_soup = BeautifulSoup(detail_resp.text, "html.parser")
                beschr_tag = detail_soup.select_one(".description-text, .ad-description, #descText")
                beschreibung = beschr_tag.get_text(strip=True) if beschr_tag else "Keine Beschreibung."
                bild_tag = detail_soup.select_one(".gallery img, .image-gallery img, img.ad-image")
                bild_url = bild_tag["src"] if bild_tag and "src" in bild_tag.attrs else None

                nachricht = f"🚨 {begriff.upper()} gefunden!\n\n🛵 {titel}\n💶 {preis}\n📄 {beschreibung[:300]}...\n🔗 {link}"
                if bild_url:
                    nachricht += f"\n🖼️ Bild: {bild_url}"

                telegram_senden(nachricht)
                gesehen.add(link)
                gefunden = True
                print(f"✅ {titel} gesendet")

    if not gefunden:
        print("❌ Nichts Neues gefunden")

# === MAIN LOOP ===
while True:
    try:
        befehle_pruefen()
        if not bot_paused:
            suche_starten()
        else:
            print("⏸ Pausiert...")
        time.sleep(60)
    except Exception as e:
        print("❌ Fehler:", e)
        time.sleep(30)
