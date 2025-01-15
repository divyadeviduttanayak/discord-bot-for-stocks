import discord
import os
from alpha_vantage.timeseries import TimeSeries
import asyncio

# Initialize your bot and API
TOKEN = 'MTI5NjQ5NzkwNTAzMjc1NzI0OA.Gff72E.tTIAIcZTKwiCgc-6kDpvpstDv5LQrUIfe_qtjk'  # Replace with your Discord bot token
ALPHA_VANTAGE_API_KEY = '13AMT55EDWG7QRRN'  # Replace with your Alpha Vantage API key

# Setting up Alpha Vantage API
ts = TimeSeries(key=ALPHA_VANTAGE_API_KEY, output_format='json')

# Initialize the Discord client
intents = discord.Intents.default()
client = discord.Client(intents=intents)

# Prefix for bot commands
COMMAND_PREFIX = '!'

# For price notifications
price_watchlist = {}  # To store user requests for price thresholds

# Function to get stock data from Alpha Vantage
def get_stock_price(symbol):
    try:
        data, meta_data = ts.get_quote_endpoint(symbol)
        return {
            'symbol': data['01. symbol'],
            'price': float(data['05. price']),
            'open': float(data['02. open']),
            'high': float(data['03. high']),
            'low': float(data['04. low']),
            'volume': int(data['06. volume']),
            'previous_close': float(data['08. previous close']),
            'change': float(data['09. change']),
            'percent_change': float(data['10. change percent'].replace('%', ''))
        }
    except Exception as e:
        return None

# Event when the bot is ready
@client.event
async def on_ready():
    print(f'Logged in as {client.user}')

# Event to handle messages
@client.event
async def on_message(message):
    # Ignore the bot's own messages
    if message.author == client.user:
        return

    # Command to get stock price
    if message.content.startswith(f'{COMMAND_PREFIX}stock'):
        try:
            # Extract the stock symbol from the message
            symbol = message.content.split()[1].upper()
            stock_data = get_stock_price(symbol)

            if stock_data:
                # Prepare and send stock details
                response = (f"**{stock_data['symbol']} Stock Information**\n"
                            f"Current Price: ${stock_data['price']:.2f}\n"
                            f"Open: ${stock_data['open']:.2f}\n"
                            f"High: ${stock_data['high']:.2f}\n"
                            f"Low: ${stock_data['low']:.2f}\n"
                            f"Volume: {stock_data['volume']:,}\n"
                            f"Previous Close: ${stock_data['previous_close']:.2f}\n"
                            f"Change: {stock_data['change']} ({stock_data['percent_change']}%)")
                await message.channel.send(response)
            else:
                await message.channel.send("Error: Could not retrieve stock data. Make sure the symbol is correct.")

        except IndexError:
            await message.channel.send("Please provide a stock symbol like this: `!stock AAPL`")
        except Exception as e:
            await message.channel.send(f"An error occurred: {e}")

    # Command to set a price threshold notification
    if message.content.startswith(f'{COMMAND_PREFIX}watch'):
        try:
            # Extract the stock symbol and target price
            args = message.content.split()
            symbol = args[1].upper()
            target_price = float(args[2])

            # Add to the price watchlist
            price_watchlist[symbol] = {
                'user': message.author,
                'target_price': target_price
            }
            await message.channel.send(f"Watching {symbol} for price threshold of ${target_price:.2f}. You'll be notified when it reaches that.")

        except IndexError:
            await message.channel.send("Please provide a stock symbol and target price like this: `!watch AAPL 150`")
        except Exception as e:
            await message.channel.send(f"An error occurred: {e}")

# Background task to monitor stock prices
async def check_price_thresholds():
    await client.wait_until_ready()

    while not client.is_closed():
        for symbol, watch_data in list(price_watchlist.items()):
            stock_data = get_stock_price(symbol)
            if stock_data and stock_data['price'] >= watch_data['target_price']:
                user = watch_data['user']
                await user.send(f"{symbol} has reached your target price of ${watch_data['target_price']:.2f}. Current price: ${stock_data['price']:.2f}")
                del price_watchlist[symbol]  # Remove the notification after triggering

        await asyncio.sleep(60)  # Check every 60 seconds

# Start the background task for price monitoring
@client.event
async def on_ready():
    # Start the background task only when the bot is ready
    client.loop.create_task(check_price_thresholds())
    print(f'Price monitoring task started.')

# Run the bot
client.run(TOKEN)
            
