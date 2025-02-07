# CryptoLens-AI-Beta-Version

Python Code created the creator of CryptoLens AI:

import logging
from telegram import Update
from telegram.helpers import escape_markdown
from telegram.ext import Application, CommandHandler, ConversationHandler, MessageHandler, filters, ContextTypes
import requests
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from datetime import datetime, timedelta
import time
from pytz import timezone
import asyncio

# Configure logging
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

class RateLimiter:
    def __init__(self):
        self.boost_last_call = 0
        self.pairs_last_call = 0
        self.boost_calls = 0
        self.pairs_calls = 0
        
    def check_boost_limit(self):
        if time.time() - self.boost_last_call >= 60:
            self.boost_calls = 0
            self.boost_last_call = time.time()
        if self.boost_calls >= 55:
            time.sleep(1)
            return False
        self.boost_calls += 1
        return True
    
    def check_pairs_limit(self):
        if time.time() - self.pairs_last_call >= 60:
            self.pairs_calls = 0
            self.pairs_last_call = time.time()
        if self.pairs_calls >= 295:
            time.sleep(1)
            return False
        self.pairs_calls += 1
        return True

rate_limiter = RateLimiter()

# Conversation states
MIN_MCAP, MAX_MCAP, MIN_VOLUME, LIQUIDITY = range(4)
user_settings = {}
token_cache = {}

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data.clear()
    await update.message.reply_text("🚀 Welcome to Crypto Scanner Bot! 🚀\nPlease enter MINIMUM market cap (USD):")
    return MIN_MCAP

async def newfilter(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await start(update, context)

async def min_mcap(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    try:
        context.user_data['min_mcap'] = float(update.message.text)
        await update.message.reply_text("Enter MAXIMUM market cap (USD):")
        return MAX_MCAP
    except ValueError:
        await update.message.reply_text("Please enter a valid number.")
        return MIN_MCAP

async def max_mcap(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    try:
        context.user_data['max_mcap'] = float(update.message.text)
        await update.message.reply_text("Enter MINIMUM 5m volume (USD):")  # Changed to 5m
        return MIN_VOLUME
    except ValueError:
        await update.message.reply_text("Please enter a valid number.")
        return MAX_MCAP

async def min_volume(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    try:
        context.user_data['min_volume'] = float(update.message.text)
        await update.message.reply_text("Enter MINIMUM liquidity (USD):")
        return LIQUIDITY
    except ValueError:
        await update.message.reply_text("Please enter a valid number.")
        return MIN_VOLUME

async def liquidity(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    try:
        context.user_data['liquidity'] = float(update.message.text)
        user_settings[update.effective_user.id] = context.user_data.copy()
        await update.message.reply_text("✅ Settings saved! Continuous monitoring started!")
        return ConversationHandler.END
    except ValueError:
        await update.message.reply_text("Please enter a valid number.")
        return LIQUIDITY

async def monitor_tokens(app: Application):
    logger.info("Starting monitoring cycle...")
    
    try:
        top_tokens = []
        latest_tokens = []
        
        if rate_limiter.check_boost_limit():
            top_tokens = get_solana_tokens("https://api.dexscreener.com/token-boosts/top/v1")
        
        if rate_limiter.check_boost_limit():
            latest_tokens = get_solana_tokens("https://api.dexscreener.com/token-boosts/latest/v1")
        
        all_tokens = list(set(top_tokens + latest_tokens))
        logger.info(f"Found {len(all_tokens)} unique tokens to check")
        
        batch_size = 30
        for i in range(0, len(all_tokens), batch_size):
            if rate_limiter.check_pairs_limit():
                batch = all_tokens[i:i+batch_size]
                await process_token_batch(app, batch)
                
    except Exception as e:
        logger.error(f"Monitoring error: {str(e)}", exc_info=True)

async def process_token_batch(app, token_addresses):
    try:
        url = f"https://api.dexscreener.com/tokens/v1/solana/{','.join(token_addresses)}"
        response = requests.get(url)
        
        if response.status_code == 200:
            data = response.json()
            pairs = data.get('results', []) if isinstance(data, dict) else data
            
            for pair in pairs:
                if is_new_token(pair):
                    await check_all_users(app, pair)
                    
    except Exception as e:
        logger.error(f"Batch processing error: {str(e)}", exc_info=True)

def is_new_token(pair):
    token_address = pair.get('pairAddress', '')
    if token_address in token_cache:
        return False
    token_cache[token_address] = datetime.now() + timedelta(hours=1)
    return True

# Modified check_all_users function with targeted escaping
async def check_all_users(app, pair):
    try:
        for user_id, settings in user_settings.items():
            if is_valid_pair(pair, settings):
                # Convert pairCreatedAt to Pacific Time
                created_at_timestamp = pair.get('pairCreatedAt', 0) / 1000  # Convert to seconds
                created_at = datetime.fromtimestamp(created_at_timestamp)
                pacific_time = created_at.astimezone(timezone('US/Pacific'))
                created_on = pacific_time.strftime('%Y-%m-%d at %I:%M:%S %p')

                # Escape all dynamic content
                token_name = escape_markdown(pair['baseToken'].get('name', 'N/A'))
                token_symbol = escape_markdown(pair['baseToken'].get('symbol', 'N/A'))
                token_address = escape_markdown(pair['baseToken'].get('address', 'N/A'))
                dex_url = escape_markdown(pair.get('url', ''))
                rugcheck_link = escape_markdown(f"https://rugcheck.xyz/tokens/{token_address}")

                # Escape social links
                social_links = "\n".join(
                    [f"{escape_markdown(s.get('type', 'Link').title())}: {escape_markdown(s.get('url', ''))}" 
                     for s in pair.get('info', {}).get('socials', []) if s.get('url')]
                ) or "No social links available"

                message = f"""🔔 **New Token Alert** 🔔
🏷️ Name: {token_name} ({token_symbol})
📅 Created On: {created_on}
📊 Market Cap: ${pair.get('marketCap', 0):,.2f}
💵 Price: ${float(pair.get('priceUsd', 0)):.8f}
🏦 Liquidity: ${pair.get('liquidity', {}).get('usd', 0):,.2f}

📈 **Volume (USD)**:
- 5m: ${pair.get('volume', {}).get('m5', 0):,.2f}
- 1h: ${pair.get('volume', {}).get('h1', 0):,.2f}
- 6h: ${pair.get('volume', {}).get('h6', 0):,.2f}
- 24h: ${pair.get('volume', {}).get('h24', 0):,.2f}

📉 **Price Change**:
- 5m: {pair.get('priceChange', {}).get('m5', 0):.2f}%
- 1h: {pair.get('priceChange', {}).get('h1', 0):.2f}%
- 6h: {pair.get('priceChange', {}).get('h6', 0):.2f}%
- 24h: {pair.get('priceChange', {}).get('h24', 0):.2f}%

🔗 DexScreener: [Click Here]({dex_url})
🔍 RugCheck Link: [Click Here]({rugcheck_link})

📄 CA: `{token_address}`
🌐 Social Links:
{social_links}
"""

                await app.bot.send_message(
                    chat_id=user_id,
                    text=message,
                    disable_web_page_preview=True,
                    parse_mode="Markdown"
                )
    except Exception as e:
        logger.error(f"User check error: {str(e)}", exc_info=True)

def get_solana_tokens(api_url):
    try:
        response = requests.get(api_url)
        if response.status_code == 200:
            data = response.json()
            return [
                item["tokenAddress"] for item in data
                if isinstance(item, dict) and 
                item.get("chainId") == "solana" and 
                "tokenAddress" in item
            ]
        return []
    except Exception as e:
        logger.error(f"Error fetching {api_url}: {str(e)}", exc_info=True)
        return []

def is_valid_pair(pair, settings):
    try:
        return all([
            pair.get("chainId") == "solana",
            settings['min_mcap'] <= pair.get("marketCap", 0) <= settings['max_mcap'],
            pair.get('volume', {}).get('m5', 0) >= settings['min_volume'],  # Changed to m5
            pair.get("liquidity", {}).get("usd", 0) >= settings['liquidity']
        ])
    except KeyError:
        return False

async def start_scheduler(app):
    scheduler = AsyncIOScheduler()
    scheduler.add_job(
        monitor_tokens,
        'interval',
        minutes=1,
        args=[app]
    )
    scheduler.start()
    app.scheduler = scheduler

async def stop_scheduler(app):
    app.scheduler.shutdown(wait=False)

def main() -> None:
    application = Application.builder().token("7959528436:AAFtl-Oqxr_1Llb8jCnVzLr8ld40OZOBHs0").build()
    
    application.post_init = start_scheduler
    application.post_stop = stop_scheduler

    conv_handler = ConversationHandler(
        entry_points=[
            CommandHandler('start', start),
            CommandHandler('newfilter', newfilter)
        ],
        states={
            MIN_MCAP: [MessageHandler(filters.TEXT & ~filters.COMMAND, min_mcap)],
            MAX_MCAP: [MessageHandler(filters.TEXT & ~filters.COMMAND, max_mcap)],
            MIN_VOLUME: [MessageHandler(filters.TEXT & ~filters.COMMAND, min_volume)],
            LIQUIDITY: [MessageHandler(filters.TEXT & ~filters.COMMAND, liquidity)],
        },
        fallbacks=[],
    )

    application.add_handler(conv_handler)
    application.run_polling()

if __name__ == '__main__':
    main()



STEPS TO FOLLOW: 
