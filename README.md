import os
import json
import asyncio
import aiohttp
from bs4 import BeautifulSoup
import discord
from discord.ext import tasks
from discord import app_commands
from dotenv import load_dotenv
import logging

# --- Logging Setup ---
logging.basicConfig(level=logging.INFO, format='%(asctime)s | %(levelname)s | %(message)s')

# --- Load environment variables ---
load_dotenv()
TOKEN = os.getenv('DISCORD_TOKEN')
GUILD_ID = int(os.getenv('DISCORD_GUILD_ID', 0))
CHANNEL_ID = int(os.getenv('DISCORD_CHANNEL_ID', 0))

if not TOKEN or not GUILD_ID or not CHANNEL_ID:
    logging.error("Bitte setze alle Umgebungsvariablen: DISCORD_TOKEN, DISCORD_GUILD_ID, DISCORD_CHANNEL_ID")
    exit(1)

# --- Default Filters ---
default_filters = {
    'price_min': 0,
    'price_max': 600,
    'km_max': 50
}

# --- Seen listings file ---
SEEN_FILE = 'seen_listings.json'

# --- Discord Bot Setup ---
intents = discord.Intents.default()
bot = discord.Client(intents=intents)
tree = app_commands.CommandTree(bot)

# --- Helper Functions ---

def load_seen_ids():
    if os.path.isfile(SEEN_FILE):
        try:
            with open(SEEN_FILE, 'r', encoding='utf-8') as f:
                data = json.load(f)
                return set(data)
        except Exception as e:
            logging.warning(f"Fehler beim Laden der gespeicherten IDs: {e}")
            return set()
    return set()

def save_seen_ids(seen_ids):
    try:
        with open(SEEN_FILE, 'w', encoding='utf-8') as f:
            json.dump(list(seen_ids), f)
    except Exception as e:
        logging.error(f"Fehler beim Speichern der gespeicherten IDs: {e}")

def build_search_url():
    # Aktuelle Kategorie "Scooter" (109) und Suchbegriff "Yamaha Aerox" + Radius 50 km
    return "https://www.ebay-kleinanzeigen.de/s-scooter/yamaha-aerox/k0c109l3391r0?distance=50"

async def fetch_listings(filters):
    url = build_search_url()
    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko)'
                      ' Chrome/114.0.0.0 Safari/537.36'
    }
    listings = []

    try:
        async with aiohttp.ClientSession(headers=headers) as session:
            async with session.get(url, timeout=15) as response:
                if response.status != 200:
                    logging.warning(f"Unerwarteter HTTP Status: {response.status}")
                    return []
                text = await response.text()
    except Exception as e:
        logging.error(f"HTTP Fehler beim Abrufen der Seite: {e}")
        return []

    soup = BeautifulSoup(text, 'html.parser')

    # Artikel finden (Class kann sich ändern, daher fallback-Logik)
    articles = soup.find_all('article')
    if not articles:
        logging.warning("Keine Artikel auf der Seite gefunden - evtl. HTML Struktur geändert?")
        return []

    for article in articles:
        try:
            aid = article.get('data-adid')
            if not aid:
                continue

            title_tag = article.find('a', class_='ellipsis')
            if not title_tag:
                continue
            title = title_tag.get_text(strip=True)
            link = 'https://www.ebay-kleinanzeigen.de' + title_tag.get('href', '')

            price_tag = article.find('p', class_='aditem-main--middle--price')
            if not price_tag:
                continue
            price_str = price_tag.get_text(strip=True).replace('€', '').replace('.', '').replace(',', '').strip()
            if not price_str.isdigit():
                continue
            price = int(price_str)

            desc_tag = article.find('div', class_='aditem-main--bottom')
            km_val = 0
            if desc_tag:
                desc_text = desc_tag.get_text(separator='|')
                # Suche nach km-Angabe
                for part in desc_text.split('|'):
                    part = part.strip().lower()
                    if 'km' in part:
                        # Extrahiere Zahlen aus Teilstring
                        digits = ''.join(c for c in part if c.isdigit())
                        if digits.isdigit():
                            km_val = int(digits)
                            break

            # Thumbnail-Bild finden
            thumb_tag = article.find('img')
            thumbnail = None
            if thumb_tag and thumb_tag.has_attr('src'):
                thumbnail = thumb_tag['src']

            # Filter anwenden
            if (filters['price_min'] <= price <= filters['price_max']) and (km_val <= filters['km_max']):
                listings.append({
                    'id': aid,
                    'title': title,
                    'link': link,
                    'price': price,
                    'km': km_val,
                    'thumbnail': thumbnail
                })

        except Exception as e:
            logging.warning(f"Fehler beim Parsen eines Artikels: {e}")
            continue

    logging.info(f"{len(listings)} gefilterte Inserate gefunden.")
    return listings

# --- Background task ---

@tasks.loop(minutes=5)
async def check_new_listings():
    await bot.wait_until_ready()
    channel = bot.get_channel(CHANNEL_ID)
    if channel is None:
        logging.error(f"Channel mit ID {CHANNEL_ID} nicht gefunden!")
        return

    seen = load_seen_ids()
    new_items = []
    filters = default_filters

    try:
        listings = await fetch_listings(filters)
        for item in listings:
            if item['id'] not in seen:
                new_items.append(item)
                seen.add(item['id'])
        if new_items:
            save_seen_ids(seen)
            for item in new_items:
                embed = discord.Embed(title=item['title'], url=item['link'], color=0x0055ff)
                embed.add_field(name='Preis', value=f"{item['price']} €", inline=True)
                embed.add_field(name='Kilometerstand', value=f"{item['km']} km", inline=True)
                if item['thumbnail']:
                    embed.set_thumbnail(url=item['thumbnail'])
                await channel.send(content='@everyone Neue Yamaha Aerox gefunden!', embed=embed)
            logging.info(f"{len(new_items)} neue Inserate gepostet.")
        else:
            logging.info("Keine neuen Inserate gefunden.")
    except Exception as e:
        logging.error(f"Fehler beim Background-Task: {e}")

# --- Slash Commands ---

@tree.command(name="search", description="Suche jetzt nach neuen Yamaha Aerox Kleinanzeigen.")
async def search(interaction: discord.Interaction):
    await interaction.response.defer()
    filters = default_filters
    listings = await fetch_listings(filters)
    if not listings:
        await interaction.followup.send("Keine Ergebnisse gefunden.")
        return
    # Max 5 Ergebnisse
    for item in listings[:5]:
        embed = discord.Embed(title=item['title'], url=item['link'], color=0x0055ff)
        embed.add_field(name='Preis', value=f"{item['price']} €", inline=True)
        embed.add_field(name='Kilometerstand', value=f"{item['km']} km", inline=True)
        if item['thumbnail']:
            embed.set_thumbnail(url=item['thumbnail'])
        await interaction.followup.send(embed=embed)

@tree.command(name="setfilters", description="Setze Preis- und Kilometer-Filter.")
@app_commands.describe(
    price_min="Minimaler Preis (€, >=0)",
    price_max="Maximaler Preis (€, >= price_min)",
    km_max="Maximale Kilometer (>=0)"
)
async def setfilters(interaction: discord.Interaction, price_min: int, price_max: int, km_max: int):
    global default_filters
    if price_min < 0 or price_max < price_min or km_max < 0:
        await interaction.response.send_message("Ungültige Filterwerte! Bitte gültige Werte eingeben.", ephemeral=True)
        return
    default_filters = {
        'price_min': price_min,
        'price_max': price_max,
        'km_max': km_max
    }
    await interaction.response.send_message(
        f"Filter gesetzt: Preis {price_min}€ - {price_max}€, Kilometer ≤ {km_max} km."
    )
    logging.info(f"Filter aktualisiert: {default_filters}")

# --- Bot startup ---

@bot.event
async def on_ready():
    if GUILD_ID:
        guild = discord.Object(id=GUILD_ID)
        await tree.sync(guild=guild)
        logging.info(f"Slash Commands zu Guild {GUILD_ID} synchronisiert.")
    else:
        await tree.sync()
        logging.info("Globale Slash Commands synchronisiert.")
    check_new_listings.start()
    logging.info(f"Bot ist eingeloggt als {bot.user}")

if __name__ == '__main__':
    bot.run(TOKEN)
