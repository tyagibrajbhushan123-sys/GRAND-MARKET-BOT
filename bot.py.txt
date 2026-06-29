
from flask import Flask
from threading import Thread
import discord
import os
import asyncpg
from discord.ext import commands

app = Flask('')

@app.route('/')
def home():
    return "Grand Market Bot online hai!"

def run():
    app.run(host='0.0.0.0', port=10000)

def keep_alive():
    t = Thread(target=run)
    t.start()

import os
import discord
import asyncpg
from discord.ext import commands

intents = discord.Intents.default()
intents.message_content = True
intents.members = True
client = commands.Bot(command_prefix='!', intents=intents, help_command=None)

db_pool = None


async def init_db():
    global db_pool
    db_pool = await asyncpg.create_pool(dsn=os.getenv('DATABASE_URL'))
    async with db_pool.acquire() as conn:
        await conn.execute('''
            CREATE TABLE IF NOT EXISTS market_listings (
                id SERIAL PRIMARY KEY,
                item_name TEXT NOT NULL UNIQUE,
                price INTEGER NOT NULL,
                seller_name TEXT NOT NULL,
                seller_id TEXT NOT NULL,
                listed_at TIMESTAMP DEFAULT NOW()
            )
        ''')
        await conn.execute('''
            CREATE TABLE IF NOT EXISTS sold_history (
                id SERIAL PRIMARY KEY,
                item_name TEXT NOT NULL,
                price INTEGER NOT NULL,
                seller_name TEXT NOT NULL,
                seller_id TEXT NOT NULL,
                sold_at TIMESTAMP DEFAULT NOW()
            )
        ''')
        await conn.execute('''
            CREATE TABLE IF NOT EXISTS watchlist (
                id SERIAL PRIMARY KEY,
                user_id TEXT NOT NULL,
                user_name TEXT NOT NULL,
                keyword TEXT NOT NULL,
                created_at TIMESTAMP DEFAULT NOW(),
                UNIQUE(user_id, keyword)
            )
        ''')
        await conn.execute('''
            CREATE TABLE IF NOT EXISTS offers (
                id SERIAL PRIMARY KEY,
                item_name TEXT NOT NULL,
                buyer_id TEXT NOT NULL,
                buyer_name TEXT NOT NULL,
                offer_price INTEGER NOT NULL,
                seller_id TEXT NOT NULL,
                status TEXT NOT NULL DEFAULT 'pending',
                created_at TIMESTAMP DEFAULT NOW(),
                UNIQUE(item_name, buyer_id)
            )
        ''')
        await conn.execute('''
            CREATE TABLE IF NOT EXISTS reports (
                id SERIAL PRIMARY KEY,
                reporter_id TEXT NOT NULL,
                reporter_name TEXT NOT NULL,
                reported_id TEXT NOT NULL,
                reported_name TEXT NOT NULL,
                reason TEXT NOT NULL,
                status TEXT NOT NULL DEFAULT 'open',
                created_at TIMESTAMP DEFAULT NOW()
            )
        ''')
        await conn.execute('''
            CREATE TABLE IF NOT EXISTS bot_config (
                key TEXT PRIMARY KEY,
                value TEXT NOT NULL
            )
        ''')
        await conn.execute('''
            CREATE TABLE IF NOT EXISTS banned_users (
                user_id TEXT PRIMARY KEY,
                user_name TEXT NOT NULL,
                banned_by TEXT NOT NULL,
                reason TEXT,
                banned_at TIMESTAMP DEFAULT NOW()
            )
        ''')
    print('Database ready!')


@client.check
async def not_banned(ctx):
    if ctx.command and ctx.command.name in ('unban', 'ban'):
        return True
    async with db_pool.acquire() as conn:
        row = await conn.fetchrow(
            'SELECT reason FROM banned_users WHERE user_id = $1', str(ctx.author.id)
        )
    if row:
        await ctx.send(
            f'🚫 {ctx.author.mention} tumhe Grand Market se ban kiya gaya hai.\n'
            f'Reason: {row["reason"] or "N/A"}'
        )
        return False
    return True


@client.event
async def on_ready():
    await init_db()
    print(f'{client.user} online ho gaya bhai!')


@client.event
async def on_member_join(member):
    async with db_pool.acquire() as conn:
        config = await conn.fetchrow("SELECT value FROM bot_config WHERE key = 'welcome_channel_id'")
    if not config:
        return
    channel = client.get_channel(int(config['value']))
    if not channel:
        return
    embed = discord.Embed(
        title=f'🎉 Welcome to {member.guild.name}!',
        description=f'**{member.mention}** aa gaya Grand Market mein! Kharid, becho, kamaao! 🏪',
        color=discord.Color.green()
    )
    embed.set_thumbnail(url=member.display_avatar.url)
    embed.add_field(name='👤 Member', value=member.display_name, inline=True)
    embed.add_field(name='📅 Joined', value=member.joined_at.strftime('%d %b %Y') if member.joined_at else 'Abhi', inline=True)
    embed.add_field(name='👥 Total Members', value=str(member.guild.member_count), inline=True)
    embed.add_field(name='🚀 Shuru Karo', value='`!market` se listings dekho\n`!sell <item> <price>` se becho\n`!help` se saari commands dekho', inline=False)
    embed.set_footer(text='Grand Market — Sab milega yahan!')
    await channel.send(embed=embed)


@client.event
async def on_member_remove(member):
    async with db_pool.acquire() as conn:
        config = await conn.fetchrow("SELECT value FROM bot_config WHERE key = 'welcome_channel_id'")
    if not config:
        return
    channel = client.get_channel(int(config['value']))
    if not channel:
        return
    embed = discord.Embed(
        title=f'👋 Alvida, {member.display_name}!',
        description=f'**{member.display_name}** ne server chhod diya. 😢',
        color=discord.Color.red()
    )
    embed.set_thumbnail(url=member.display_avatar.url)
    embed.add_field(name='👥 Remaining Members', value=str(member.guild.member_count), inline=True)
    embed.set_footer(text='Grand Market — Wapas aana kabhi bhi!')
    await channel.send(embed=embed)


@client.command()
async def ping(ctx):
    await ctx.send('Pong! 🏓 GRAND MARKET BOT zinda hai')


@client.command()
async def hello(ctx):
    await ctx.send(f'Namaste {ctx.author.mention}! Grand Market mein swagat hai 🎉')


@client.command()
async def help(ctx):
    embed = discord.Embed(title='📜 GRAND MARKET BOT — Commands', color=discord.Color.gold())
    embed.add_field(name='!ping', value='Bot ka status check karo', inline=False)
    embed.add_field(name='!hello', value='Bot se milao', inline=False)
    embed.add_field(name='!market', value='Saari active listings dekho', inline=False)
    embed.add_field(name='!sell <item> <price>', value='Kuch becho — `!sell Bike 5000`', inline=False)
    embed.add_field(name='!price <item>', value='Item ka price dekho — `!price Bike`', inline=False)
    embed.add_field(name='!sold <item>', value='Item bik gayi mark karo — `!sold Bike`', inline=False)
    embed.add_field(name='!history', value='Pichhli sabhi bikri dekho', inline=False)
    embed.add_field(name='!mystuff', value='Apni saari listings aur sold items ek jagah dekho', inline=False)
    embed.add_field(name='!search <keyword>', value='Market mein item dhundho — `!search bi`', inline=False)
    embed.add_field(name='!top', value='Top sellers ki leaderboard dekho', inline=False)
    embed.add_field(name='!watch <keyword>', value='Kisi item ka alert lagao — `!watch Bike`', inline=False)
    embed.add_field(name='!unwatch <keyword>', value='Alert band karo — `!unwatch Bike`', inline=False)
    embed.add_field(name='!mywatches', value='Apne saare active alerts dekho', inline=False)
    embed.add_field(name='!offer <item> <price>', value='Kisi item pe counteroffer karo — `!offer Bike 4000`', inline=False)
    embed.add_field(name='!myoffers', value='Tumhare bheje hue offers dekho', inline=False)
    embed.add_field(name='!offers', value='Tumhare items pe aaye offers dekho', inline=False)
    embed.add_field(name='!accept <item> <@buyer>', value='Offer accept karo — `!accept Bike @Ram`', inline=False)
    embed.add_field(name='!decline <item> <@buyer>', value='Offer reject karo — `!decline Bike @Ram`', inline=False)
    embed.add_field(name='!report @user <reason>', value='Kisi ko fraud ke liye report karo', inline=False)
    embed.add_field(name='!reports', value='Saare open reports dekho (sirf admins)', inline=False)
    embed.add_field(name='!resolve <id>', value='Report close karo (sirf admins) — `!resolve 3`', inline=False)
    embed.add_field(name='!setreportchannel', value='Is channel ko admin alerts channel banao (sirf admins)', inline=False)
    embed.add_field(name='!setwelcome', value='Is channel ko welcome/goodbye channel banao (sirf admins)', inline=False)
    embed.add_field(name='!ban @user <reason>', value='User ko ban karo — saari listings hat jaayengi (sirf admins)', inline=False)
    embed.add_field(name='!unban @user', value='User ka ban hatao (sirf admins)', inline=False)
    embed.add_field(name='!banlist', value='Saare banned users dekho (sirf admins)', inline=False)
    embed.add_field(name='!stats', value='Server ki overall market stats dekho', inline=False)
    embed.add_field(name='!dm @user <message>', value='Kisi ko bot ke zariye DM bhejo — `!dm @Ram Bike abhi bhi available hai?`', inline=False)
    embed.add_field(name='!remove <item>', value='Apni listing hatao', inline=False)
    embed.set_footer(text='Grand Market — Sab milega yahan! | Data saved permanently 💾')
    await ctx.send(embed=embed)


@client.command()
async def market(ctx):
    async with db_pool.acquire() as conn:
        rows = await conn.fetch('SELECT * FROM market_listings ORDER BY listed_at DESC')
    if not rows:
        await ctx.send('📭 Abhi koi listing nahi hai. `!sell <item> <price>` se shuru karo!')
        return
    embed = discord.Embed(title='🏪 GRAND MARKET — Active Listings', color=discord.Color.green())
    for row in rows:
        listed_at = row['listed_at'].strftime('%d %b, %I:%M %p')
        embed.add_field(
            name=f'🔖 {row["item_name"]}',
            value=f'💰 ₹{row["price"]} | 👤 {row["seller_name"]} | 🕐 {listed_at}',
            inline=False,
        )
    embed.set_footer(text=f'Total listings: {len(rows)} | 💾 Permanently saved')
    await ctx.send(embed=embed)


@client.command()
async def sell(ctx, *args):
    if len(args) < 2:
        await ctx.send('❌ Sahi format: `!sell <item> <price>` — jaise `!sell Bike 5000`')
        return
    price_str = args[-1]
    item_name = ' '.join(args[:-1])
    if not price_str.isdigit():
        await ctx.send('❌ Price sirf number hona chahiye — jaise `!sell Bike 5000`')
        return
    async with db_pool.acquire() as conn:
        existing = await conn.fetchrow(
            'SELECT seller_id FROM market_listings WHERE item_name = $1', item_name
        )
        if existing:
            if existing['seller_id'] != str(ctx.author.id):
                await ctx.send(f'❌ **{item_name}** already listed hai kisi aur ke dwara!')
                return
            await conn.execute(
                'UPDATE market_listings SET price=$1, seller_name=$2, listed_at=NOW() WHERE item_name=$3',
                int(price_str), ctx.author.display_name, item_name,
            )
            await ctx.send(f'✅ **{item_name}** ka price update hua — ab ₹**{price_str}** {ctx.author.mention}!')
        else:
            await conn.execute(
                'INSERT INTO market_listings (item_name, price, seller_name, seller_id) VALUES ($1, $2, $3, $4)',
                item_name, int(price_str), ctx.author.display_name, str(ctx.author.id),
            )
            await ctx.send(
                f'✅ **{item_name}** listed for ₹**{price_str}** by {ctx.author.mention}! 💾 Permanently saved\n'
                f'Sabko dikhega `!market` se 🏪'
            )
            async with db_pool.acquire() as conn2:
                watchers = await conn2.fetch(
                    "SELECT * FROM watchlist WHERE $1 ILIKE '%' || keyword || '%' AND user_id != $2",
                    item_name, str(ctx.author.id)
                )
            for watcher in watchers:
                try:
                    user = await client.fetch_user(int(watcher['user_id']))
                    dm_embed = discord.Embed(
                        title='🔔 Grand Market — Watchlist Alert!',
                        description=f'Tumhara watched keyword **"{watcher["keyword"]}"** se match karta item aa gaya!',
                        color=discord.Color.orange()
                    )
                    dm_embed.add_field(name='🔖 Item', value=item_name, inline=True)
                    dm_embed.add_field(name='💰 Price', value=f'₹{price_str}', inline=True)
                    dm_embed.add_field(name='👤 Seller', value=ctx.author.display_name, inline=True)
                    dm_embed.set_footer(text='Grand Market mein jao aur !price ya !market se dekho!')
                    await user.send(embed=dm_embed)
                except Exception:
                    pass


@client.command()
async def price(ctx, *args):
    if not args:
        await ctx.send('❌ Sahi format: `!price <item>` — jaise `!price Bike`')
        return
    item_name = ' '.join(args)
    async with db_pool.acquire() as conn:
        row = await conn.fetchrow('SELECT * FROM market_listings WHERE item_name = $1', item_name)
    if row:
        embed = discord.Embed(title=f'🔍 {row["item_name"]}', color=discord.Color.blue())
        embed.add_field(name='💰 Price', value=f'₹{row["price"]}', inline=True)
        embed.add_field(name='👤 Seller', value=row['seller_name'], inline=True)
        embed.add_field(name='🕐 Listed on', value=row['listed_at'].strftime('%d %b, %I:%M %p'), inline=True)
        await ctx.send(embed=embed)
    else:
        await ctx.send(f'❌ **{item_name}** market mein nahi mila. `!market` se dekho kya available hai.')


@client.command()
async def sold(ctx, *args):
    if not args:
        await ctx.send('❌ Sahi format: `!sold <item>` — jaise `!sold Bike`')
        return
    item_name = ' '.join(args)
    async with db_pool.acquire() as conn:
        row = await conn.fetchrow('SELECT * FROM market_listings WHERE item_name = $1', item_name)
        if not row:
            await ctx.send(f'❌ **{item_name}** active listings mein nahi mila.')
            return
        if row['seller_id'] != str(ctx.author.id):
            await ctx.send('❌ Sirf apna item sold mark kar sakte ho!')
            return
        await conn.execute(
            'INSERT INTO sold_history (item_name, price, seller_name, seller_id) VALUES ($1, $2, $3, $4)',
            row['item_name'], row['price'], row['seller_name'], row['seller_id'],
        )
        await conn.execute('DELETE FROM market_listings WHERE item_name = $1', item_name)
    embed = discord.Embed(
        title='🎉 Item Sold!',
        description=f'**{item_name}** ₹{row["price"]} mein bik gayi!',
        color=discord.Color.green(),
    )
    embed.add_field(name='👤 Seller', value=ctx.author.mention, inline=True)
    embed.add_field(name='💰 Final Price', value=f'₹{row["price"]}', inline=True)
    embed.set_footer(text='Sold history mein save ho gaya! !history se dekho')
    await ctx.send(embed=embed)


@client.command()
async def history(ctx):
    async with db_pool.acquire() as conn:
        rows = await conn.fetch('SELECT * FROM sold_history ORDER BY sold_at DESC LIMIT 10')
    if not rows:
        await ctx.send('📭 Abhi koi item nahi bika. Pehli sale ka wait hai! 🤞')
        return
    embed = discord.Embed(title='📋 GRAND MARKET — Sold History (Last 10)', color=discord.Color.purple())
    for row in rows:
        sold_at = row['sold_at'].strftime('%d %b, %I:%M %p')
        embed.add_field(
            name=f'✅ {row["item_name"]}',
            value=f'💰 ₹{row["price"]} | 👤 {row["seller_name"]} | 🕐 {sold_at}',
            inline=False,
        )
    embed.set_footer(text=f'Total sold: {len(rows)} shown (max 10)')
    await ctx.send(embed=embed)


@client.command()
async def mystuff(ctx):
    async with db_pool.acquire() as conn:
        active = await conn.fetch(
            'SELECT * FROM market_listings WHERE seller_id = $1 ORDER BY listed_at DESC',
            str(ctx.author.id),
        )
        sold_rows = await conn.fetch(
            'SELECT * FROM sold_history WHERE seller_id = $1 ORDER BY sold_at DESC LIMIT 5',
            str(ctx.author.id),
        )
    embed = discord.Embed(title=f'🗂️ {ctx.author.display_name} ka Saman', color=discord.Color.orange())
    if active:
        active_lines = '\n'.join(
            f'🔖 **{r["item_name"]}** — ₹{r["price"]} ({r["listed_at"].strftime("%d %b")})'
            for r in active
        )
        embed.add_field(name=f'📦 Active Listings ({len(active)})', value=active_lines, inline=False)
    else:
        embed.add_field(name='📦 Active Listings', value='Koi listing nahi. `!sell <item> <price>` se shuru karo!', inline=False)
    if sold_rows:
        sold_lines = '\n'.join(
            f'✅ **{r["item_name"]}** — ₹{r["price"]} ({r["sold_at"].strftime("%d %b")})'
            for r in sold_rows
        )
        embed.add_field(name=f'💸 Recently Sold ({len(sold_rows)})', value=sold_lines, inline=False)
    else:
        embed.add_field(name='💸 Recently Sold', value='Abhi kuch nahi bika.', inline=False)
    total_earned = sum(r['price'] for r in sold_rows)
    embed.set_footer(text=f'Last 5 sales mein total kamayi: ₹{total_earned}')
    await ctx.send(embed=embed)


@client.command()
async def search(ctx, *args):
    if not args:
        await ctx.send('❌ Sahi format: `!search <keyword>` — jaise `!search bi`')
        return
    keyword = ' '.join(args)
    async with db_pool.acquire() as conn:
        rows = await conn.fetch(
            'SELECT * FROM market_listings WHERE item_name ILIKE $1 ORDER BY price ASC',
            f'%{keyword}%',
        )
    if not rows:
        await ctx.send(f'🔍 **"{keyword}"** se koi item nahi mila market mein.')
        return
    embed = discord.Embed(title=f'🔍 Search Results — "{keyword}"', color=discord.Color.teal())
    for row in rows:
        listed_at = row['listed_at'].strftime('%d %b, %I:%M %p')
        embed.add_field(
            name=f'🔖 {row["item_name"]}',
            value=f'💰 ₹{row["price"]} | 👤 {row["seller_name"]} | 🕐 {listed_at}',
            inline=False,
        )
    embed.set_footer(text=f'{len(rows)} result(s) mila')
    await ctx.send(embed=embed)


@client.command()
async def top(ctx):
    async with db_pool.acquire() as conn:
        sellers = await conn.fetch('''
            SELECT seller_name, COUNT(*) as total_sold, SUM(price) as total_earned
            FROM sold_history
            GROUP BY seller_name
            ORDER BY total_sold DESC, total_earned DESC
            LIMIT 10
        ''')
        active_count = await conn.fetch('''
            SELECT seller_name, COUNT(*) as active
            FROM market_listings
            GROUP BY seller_name
            ORDER BY active DESC
            LIMIT 5
        ''')
    embed = discord.Embed(title='🏆 GRAND MARKET — Top Sellers', color=discord.Color.gold())
    if sellers:
        medals = ['🥇', '🥈', '🥉'] + ['🔹'] * 7
        board = '\n'.join(
            f'{medals[i]} **{r["seller_name"]}** — {r["total_sold"]} sales | ₹{r["total_earned"]} kamaaye'
            for i, r in enumerate(sellers)
        )
        embed.add_field(name='💸 Most Sales', value=board, inline=False)
    else:
        embed.add_field(name='💸 Most Sales', value='Abhi koi sale nahi hui. Pehle becho!', inline=False)
    if active_count:
        active_board = '\n'.join(
            f'🔖 **{r["seller_name"]}** — {r["active"]} active listings'
            for r in active_count
        )
        embed.add_field(name='📦 Most Active Listings', value=active_board, inline=False)
    embed.set_footer(text='Grand Market Leaderboard — Sabse aage kaun hai?')
    await ctx.send(embed=embed)


@client.command()
async def watch(ctx, *args):
    if not args:
        await ctx.send('❌ Sahi format: `!watch <keyword>` — jaise `!watch Bike`')
        return
    keyword = ' '.join(args).lower()
    async with db_pool.acquire() as conn:
        try:
            await conn.execute(
                'INSERT INTO watchlist (user_id, user_name, keyword) VALUES ($1, $2, $3)',
                str(ctx.author.id), ctx.author.display_name, keyword
            )
            await ctx.send(
                f'🔔 **"{keyword}"** watch list mein add ho gaya {ctx.author.mention}!\n'
                f'Jab bhi koi iss keyword se milta item list karega, tumhe DM aayega. `!unwatch {keyword}` se band karo.'
            )
        except Exception:
            await ctx.send(f'⚠️ **"{keyword}"** pehle se hi tumhari watch list mein hai.')


@client.command()
async def unwatch(ctx, *args):
    if not args:
        await ctx.send('❌ Sahi format: `!unwatch <keyword>` — jaise `!unwatch Bike`')
        return
    keyword = ' '.join(args).lower()
    async with db_pool.acquire() as conn:
        result = await conn.execute(
            'DELETE FROM watchlist WHERE user_id = $1 AND keyword = $2',
            str(ctx.author.id), keyword
        )
    if result == 'DELETE 1':
        await ctx.send(f'🔕 **"{keyword}"** watch list se hata diya {ctx.author.mention}.')
    else:
        await ctx.send(f'❌ **"{keyword}"** tumhari watch list mein nahi mila.')


@client.command()
async def mywatches(ctx):
    async with db_pool.acquire() as conn:
        rows = await conn.fetch(
            'SELECT * FROM watchlist WHERE user_id = $1 ORDER BY created_at DESC',
            str(ctx.author.id)
        )
    if not rows:
        await ctx.send(f'📭 Tumhari koi watch nahi hai {ctx.author.mention}. `!watch <keyword>` se shuru karo!')
        return
    embed = discord.Embed(
        title=f'🔔 {ctx.author.display_name} ki Watchlist',
        color=discord.Color.orange()
    )
    watches = '\n'.join(f'👁️ **{r["keyword"]}** — {r["created_at"].strftime("%d %b")} se' for r in rows)
    embed.add_field(name=f'Active Alerts ({len(rows)})', value=watches, inline=False)
    embed.set_footer(text='Jab koi matching item list hoga, DM aayega! | !unwatch se band karo')
    await ctx.send(embed=embed)


@client.command()
async def offer(ctx, *args):
    if len(args) < 2:
        await ctx.send('❌ Sahi format: `!offer <item> <price>` — jaise `!offer Bike 4000`')
        return
    price_str = args[-1]
    item_name = ' '.join(args[:-1])
    if not price_str.isdigit():
        await ctx.send('❌ Price sirf number hona chahiye — jaise `!offer Bike 4000`')
        return
    async with db_pool.acquire() as conn:
        listing = await conn.fetchrow('SELECT * FROM market_listings WHERE item_name = $1', item_name)
        if not listing:
            await ctx.send(f'❌ **{item_name}** market mein nahi mila. `!market` se check karo.')
            return
        if listing['seller_id'] == str(ctx.author.id):
            await ctx.send('❌ Apne hi item pe offer nahi kar sakte!')
            return
        try:
            await conn.execute(
                '''INSERT INTO offers (item_name, buyer_id, buyer_name, offer_price, seller_id)
                   VALUES ($1, $2, $3, $4, $5)
                   ON CONFLICT (item_name, buyer_id)
                   DO UPDATE SET offer_price=$4, status='pending', created_at=NOW()''',
                item_name, str(ctx.author.id), ctx.author.display_name,
                int(price_str), listing['seller_id']
            )
        except Exception as e:
            await ctx.send(f'❌ Kuch gadbad ho gayi: {e}')
            return
    await ctx.send(
        f'📨 **{item_name}** ke liye ₹**{price_str}** ka offer bhej diya {ctx.author.mention}!\n'
        f'Seller ko DM gaya hai. `!myoffers` se status dekho.'
    )
    try:
        seller = await client.fetch_user(int(listing['seller_id']))
        dm_embed = discord.Embed(
            title='💬 Grand Market — Naya Offer Aaya!',
            description=f'Tumhare **{item_name}** pe ek offer aaya hai!',
            color=discord.Color.blue()
        )
        dm_embed.add_field(name='🔖 Item', value=item_name, inline=True)
        dm_embed.add_field(name='💰 Tumhara Price', value=f'₹{listing["price"]}', inline=True)
        dm_embed.add_field(name='🤝 Offer', value=f'₹{price_str}', inline=True)
        dm_embed.add_field(name='👤 Buyer', value=ctx.author.display_name, inline=True)
        dm_embed.set_footer(text=f'Accept karo: !accept {item_name} @{ctx.author.display_name} | Decline: !decline {item_name} @{ctx.author.display_name}')
        await seller.send(embed=dm_embed)
    except Exception:
        pass


@client.command()
async def myoffers(ctx):
    async with db_pool.acquire() as conn:
        rows = await conn.fetch(
            'SELECT * FROM offers WHERE buyer_id = $1 ORDER BY created_at DESC',
            str(ctx.author.id)
        )
    if not rows:
        await ctx.send(f'📭 Tumne abhi koi offer nahi bheja {ctx.author.mention}. `!offer <item> <price>` se karo!')
        return
    embed = discord.Embed(title=f'📨 {ctx.author.display_name} ke Bheje Offers', color=discord.Color.blue())
    status_icons = {'pending': '⏳', 'accepted': '✅', 'declined': '❌'}
    for row in rows:
        icon = status_icons.get(row['status'], '❓')
        embed.add_field(
            name=f'{icon} {row["item_name"]}',
            value=f'💰 Offer: ₹{row["offer_price"]} | Status: **{row["status"]}** | {row["created_at"].strftime("%d %b")}',
            inline=False,
        )
    await ctx.send(embed=embed)


@client.command()
async def offers(ctx):
    async with db_pool.acquire() as conn:
        rows = await conn.fetch(
            "SELECT * FROM offers WHERE seller_id = $1 AND status = 'pending' ORDER BY created_at DESC",
            str(ctx.author.id)
        )
    if not rows:
        await ctx.send(f'📭 Tumhare items pe abhi koi pending offer nahi {ctx.author.mention}.')
        return
    embed = discord.Embed(title=f'💬 Tumhare Items pe Offers ({len(rows)})', color=discord.Color.green())
    for row in rows:
        async with db_pool.acquire() as conn2:
            listing = await conn2.fetchrow('SELECT price FROM market_listings WHERE item_name = $1', row['item_name'])
        listed_price = listing['price'] if listing else '?'
        embed.add_field(
            name=f'🔖 {row["item_name"]}',
            value=(
                f'👤 Buyer: **{row["buyer_name"]}** | 💰 Listed: ₹{listed_price} | 🤝 Offer: ₹{row["offer_price"]}\n'
                f'Accept: `!accept {row["item_name"]} @{row["buyer_name"]}` | Decline: `!decline {row["item_name"]} @{row["buyer_name"]}`'
            ),
            inline=False,
        )
    await ctx.send(embed=embed)


@client.command()
async def accept(ctx, *args):
    if len(args) < 2:
        await ctx.send('❌ Sahi format: `!accept <item> @buyer`')
        return
    buyer_mention = args[-1]
    item_name = ' '.join(args[:-1])
    if not ctx.message.mentions:
        await ctx.send('❌ Buyer ko @mention karo — jaise `!accept Bike @Ram`')
        return
    buyer = ctx.message.mentions[0]
    async with db_pool.acquire() as conn:
        offer_row = await conn.fetchrow(
            "SELECT * FROM offers WHERE item_name = $1 AND buyer_id = $2 AND status = 'pending'",
            item_name, str(buyer.id)
        )
        if not offer_row:
            await ctx.send(f'❌ **{item_name}** pe {buyer.display_name} ka koi pending offer nahi mila.')
            return
        if offer_row['seller_id'] != str(ctx.author.id):
            await ctx.send('❌ Sirf apne item ka offer accept kar sakte ho!')
            return
        await conn.execute(
            "UPDATE offers SET status='accepted' WHERE item_name=$1 AND buyer_id=$2",
            item_name, str(buyer.id)
        )
        await conn.execute(
            'INSERT INTO sold_history (item_name, price, seller_name, seller_id) VALUES ($1, $2, $3, $4)',
            item_name, offer_row['offer_price'], ctx.author.display_name, str(ctx.author.id)
        )
        await conn.execute('DELETE FROM market_listings WHERE item_name = $1', item_name)
        await conn.execute(
            "UPDATE offers SET status='declined' WHERE item_name=$1 AND buyer_id != $2 AND status='pending'",
            item_name, str(buyer.id)
        )
    embed = discord.Embed(
        title='🤝 Deal Ho Gayi!',
        description=f'**{item_name}** ₹{offer_row["offer_price"]} mein **{buyer.display_name}** ko sell ho gayi!',
        color=discord.Color.green()
    )
    embed.add_field(name='👤 Seller', value=ctx.author.mention, inline=True)
    embed.add_field(name='🛒 Buyer', value=buyer.mention, inline=True)
    embed.add_field(name='💰 Final Price', value=f'₹{offer_row["offer_price"]}', inline=True)
    await ctx.send(embed=embed)
    try:
        dm_embed = discord.Embed(
            title='✅ Tumhara Offer Accept Ho Gaya!',
            description=f'**{item_name}** ₹{offer_row["offer_price"]} mein tumhara ho gaya! Seller se contact karo.',
            color=discord.Color.green()
        )
        await buyer.send(embed=dm_embed)
    except Exception:
        pass


@client.command()
async def decline(ctx, *args):
    if len(args) < 2:
        await ctx.send('❌ Sahi format: `!decline <item> @buyer`')
        return
    item_name = ' '.join(args[:-1])
    if not ctx.message.mentions:
        await ctx.send('❌ Buyer ko @mention karo — jaise `!decline Bike @Ram`')
        return
    buyer = ctx.message.mentions[0]
    async with db_pool.acquire() as conn:
        offer_row = await conn.fetchrow(
            "SELECT * FROM offers WHERE item_name = $1 AND buyer_id = $2 AND status = 'pending'",
            item_name, str(buyer.id)
        )
        if not offer_row:
            await ctx.send(f'❌ **{item_name}** pe {buyer.display_name} ka koi pending offer nahi mila.')
            return
        if offer_row['seller_id'] != str(ctx.author.id):
            await ctx.send('❌ Sirf apne item ka offer decline kar sakte ho!')
            return
        await conn.execute(
            "UPDATE offers SET status='declined' WHERE item_name=$1 AND buyer_id=$2",
            item_name, str(buyer.id)
        )
    await ctx.send(f'❌ **{item_name}** pe {buyer.mention} ka offer decline kar diya {ctx.author.mention}.')
    try:
        dm_embed = discord.Embed(
            title='❌ Tumhara Offer Decline Ho Gaya',
            description=f'**{item_name}** pe tumhara ₹{offer_row["offer_price"]} ka offer seller ne decline kar diya.',
            color=discord.Color.red()
        )
        dm_embed.set_footer(text='Alag price pe dobara try karo ya koi aur item dekho!')
        await buyer.send(embed=dm_embed)
    except Exception:
        pass


@client.command()
async def setreportchannel(ctx):
    if not ctx.author.guild_permissions.administrator:
        await ctx.send('❌ Sirf server admins ye command use kar sakte hain.')
        return
    async with db_pool.acquire() as conn:
        await conn.execute(
            "INSERT INTO bot_config (key, value) VALUES ('report_channel_id', $1) ON CONFLICT (key) DO UPDATE SET value=$1",
            str(ctx.channel.id)
        )
    await ctx.send(f'✅ **#{ctx.channel.name}** ab admin reports channel hai. Yahan sab reports aayenge.')


@client.command()
async def setwelcome(ctx):
    if not ctx.author.guild_permissions.administrator:
        await ctx.send('❌ Sirf server admins ye command use kar sakte hain.')
        return
    async with db_pool.acquire() as conn:
        await conn.execute(
            "INSERT INTO bot_config (key, value) VALUES ('welcome_channel_id', $1) ON CONFLICT (key) DO UPDATE SET value=$1",
            str(ctx.channel.id)
        )
    await ctx.send(
        f'✅ **#{ctx.channel.name}** ab welcome/goodbye channel hai!\n'
        f'Jab koi join kare → 🎉 Welcome embed\n'
        f'Jab koi leave kare → 👋 Goodbye embed'
    )


@client.command()
async def report(ctx, *args):
    if not ctx.message.mentions:
        await ctx.send('❌ Sahi format: `!report @user <reason>` — jaise `!report @Ram Fake listing thi`')
        return
    reported_user = ctx.message.mentions[0]
    reason_parts = [w for w in args if not w.startswith('<@')]
    if not reason_parts:
        await ctx.send('❌ Reason batao — jaise `!report @Ram Fake listing thi`')
        return
    reason = ' '.join(reason_parts)
    if reported_user.id == ctx.author.id:
        await ctx.send('❌ Khud ko report nahi kar sakte!')
        return
    async with db_pool.acquire() as conn:
        row = await conn.fetchrow(
            '''INSERT INTO reports (reporter_id, reporter_name, reported_id, reported_name, reason)
               VALUES ($1, $2, $3, $4, $5) RETURNING id''',
            str(ctx.author.id), ctx.author.display_name,
            str(reported_user.id), reported_user.display_name, reason
        )
        report_id = row['id']
        config = await conn.fetchrow("SELECT value FROM bot_config WHERE key = 'report_channel_id'")
    await ctx.send(
        f'🚨 Report #{report_id} submit ho gayi {ctx.author.mention}. Admins dekh lenge.\n'
        f'Reported: **{reported_user.display_name}** | Reason: {reason}'
    )
    if config:
        try:
            channel = client.get_channel(int(config['value']))
            if channel:
                alert = discord.Embed(
                    title=f'🚨 New Report #{report_id}',
                    color=discord.Color.red()
                )
                alert.add_field(name='👤 Reported', value=f'{reported_user.mention} ({reported_user.display_name})', inline=False)
                alert.add_field(name='📝 Reason', value=reason, inline=False)
                alert.add_field(name='👮 Reporter', value=f'{ctx.author.mention} ({ctx.author.display_name})', inline=False)
                alert.add_field(name='📍 Channel', value=ctx.channel.mention, inline=True)
                alert.add_field(name='🕐 Time', value=ctx.message.created_at.strftime('%d %b, %I:%M %p'), inline=True)
                alert.set_footer(text=f'Close karne ke liye: !resolve {report_id}')
                await channel.send(embed=alert)
        except Exception:
            pass


@client.command()
async def reports(ctx):
    if not ctx.author.guild_permissions.administrator:
        await ctx.send('❌ Sirf server admins reports dekh sakte hain.')
        return
    async with db_pool.acquire() as conn:
        rows = await conn.fetch(
            "SELECT * FROM reports WHERE status = 'open' ORDER BY created_at DESC LIMIT 10"
        )
    if not rows:
        await ctx.send('✅ Koi open report nahi hai. Market safe hai!')
        return
    embed = discord.Embed(
        title=f'🚨 Open Reports ({len(rows)})',
        color=discord.Color.red()
    )
    for row in rows:
        embed.add_field(
            name=f'#{row["id"]} — {row["reported_name"]}',
            value=(
                f'📝 {row["reason"]}\n'
                f'👮 Reporter: {row["reporter_name"]} | 🕐 {row["created_at"].strftime("%d %b, %I:%M %p")}\n'
                f'Close: `!resolve {row["id"]}`'
            ),
            inline=False,
        )
    embed.set_footer(text='!resolve <id> se report band karo')
    await ctx.send(embed=embed)


@client.command()
async def resolve(ctx, report_id: int = None):
    if not ctx.author.guild_permissions.administrator:
        await ctx.send('❌ Sirf server admins reports resolve kar sakte hain.')
        return
    if not report_id:
        await ctx.send('❌ Sahi format: `!resolve <id>` — jaise `!resolve 3`')
        return
    async with db_pool.acquire() as conn:
        row = await conn.fetchrow("SELECT * FROM reports WHERE id = $1", report_id)
        if not row:
            await ctx.send(f'❌ Report #{report_id} nahi mila.')
            return
        if row['status'] != 'open':
            await ctx.send(f'⚠️ Report #{report_id} pehle se hi **{row["status"]}** hai.')
            return
        await conn.execute(
            "UPDATE reports SET status = 'resolved' WHERE id = $1", report_id
        )
    embed = discord.Embed(
        title=f'✅ Report #{report_id} Resolved',
        color=discord.Color.green()
    )
    embed.add_field(name='👤 Reported User', value=row['reported_name'], inline=True)
    embed.add_field(name='📝 Reason', value=row['reason'], inline=True)
    embed.add_field(name='👮 Resolved by', value=ctx.author.mention, inline=False)
    await ctx.send(embed=embed)


@client.command()
async def ban(ctx, *args):
    if not ctx.author.guild_permissions.administrator:
        await ctx.send('❌ Sirf server admins ban kar sakte hain.')
        return
    if not ctx.message.mentions:
        await ctx.send('❌ Sahi format: `!ban @user <reason>` — jaise `!ban @Ram Fraud seller hai`')
        return
    target = ctx.message.mentions[0]
    reason_parts = [w for w in args if not w.startswith('<@')]
    reason = ' '.join(reason_parts) if reason_parts else 'Koi reason nahi diya'
    if target.id == ctx.author.id:
        await ctx.send('❌ Khud ko ban nahi kar sakte!')
        return
    async with db_pool.acquire() as conn:
        existing = await conn.fetchrow('SELECT 1 FROM banned_users WHERE user_id = $1', str(target.id))
        if existing:
            await ctx.send(f'⚠️ **{target.display_name}** pehle se ban hai.')
            return
        await conn.execute(
            'INSERT INTO banned_users (user_id, user_name, banned_by, reason) VALUES ($1, $2, $3, $4)',
            str(target.id), target.display_name, ctx.author.display_name, reason
        )
        deleted = await conn.fetch(
            'DELETE FROM market_listings WHERE seller_id = $1 RETURNING item_name', str(target.id)
        )
        await conn.execute(
            "UPDATE offers SET status='declined' WHERE buyer_id=$1 OR seller_id=$1", str(target.id)
        )
    removed_items = [r['item_name'] for r in deleted]
    embed = discord.Embed(
        title=f'🔨 {target.display_name} ko Ban Kar Diya',
        color=discord.Color.dark_red()
    )
    embed.add_field(name='👤 Banned User', value=target.mention, inline=True)
    embed.add_field(name='👮 Banned By', value=ctx.author.mention, inline=True)
    embed.add_field(name='📝 Reason', value=reason, inline=False)
    if removed_items:
        embed.add_field(
            name=f'🗑️ Listings Remove Huyi ({len(removed_items)})',
            value=', '.join(removed_items),
            inline=False
        )
    embed.set_footer(text='Unban ke liye: !unban @user')
    await ctx.send(embed=embed)
    try:
        dm = discord.Embed(
            title='🚫 Grand Market se Ban',
            description=f'Tumhe **Grand Market** se ban kar diya gaya hai.',
            color=discord.Color.dark_red()
        )
        dm.add_field(name='📝 Reason', value=reason, inline=False)
        dm.set_footer(text='Agar galat lage toh server admin se baat karo.')
        await target.send(embed=dm)
    except Exception:
        pass


@client.command()
async def unban(ctx, *args):
    if not ctx.author.guild_permissions.administrator:
        await ctx.send('❌ Sirf server admins unban kar sakte hain.')
        return
    if not ctx.message.mentions:
        await ctx.send('❌ Sahi format: `!unban @user`')
        return
    target = ctx.message.mentions[0]
    async with db_pool.acquire() as conn:
        result = await conn.execute(
            'DELETE FROM banned_users WHERE user_id = $1', str(target.id)
        )
    if result == 'DELETE 1':
        await ctx.send(f'✅ **{target.mention}** ka ban hata diya {ctx.author.mention}. Ab market use kar sakte hain.')
        try:
            await target.send(f'✅ Tumhara **Grand Market** ban hata diya gaya. Ab tum bot use kar sakte ho!')
        except Exception:
            pass
    else:
        await ctx.send(f'❌ **{target.display_name}** ban list mein nahi mila.')


@client.command()
async def banlist(ctx):
    if not ctx.author.guild_permissions.administrator:
        await ctx.send('❌ Sirf server admins ban list dekh sakte hain.')
        return
    async with db_pool.acquire() as conn:
        rows = await conn.fetch('SELECT * FROM banned_users ORDER BY banned_at DESC')
    if not rows:
        await ctx.send('✅ Koi banned user nahi hai. Market clean hai!')
        return
    embed = discord.Embed(
        title=f'🔨 Banned Users ({len(rows)})',
        color=discord.Color.dark_red()
    )
    for row in rows:
        embed.add_field(
            name=f'🚫 {row["user_name"]}',
            value=(
                f'📝 {row["reason"] or "N/A"} | '
                f'👮 By: {row["banned_by"]} | '
                f'🕐 {row["banned_at"].strftime("%d %b")}'
            ),
            inline=False,
        )
    embed.set_footer(text='!unban @user se ban hatao')
    await ctx.send(embed=embed)


@client.command()
async def stats(ctx):
    async with db_pool.acquire() as conn:
        active_count = await conn.fetchval('SELECT COUNT(*) FROM market_listings')
        active_value = await conn.fetchval('SELECT COALESCE(SUM(price), 0) FROM market_listings')
        total_sold = await conn.fetchval('SELECT COUNT(*) FROM sold_history')
        total_earned = await conn.fetchval('SELECT COALESCE(SUM(price), 0) FROM sold_history')
        pending_offers = await conn.fetchval("SELECT COUNT(*) FROM offers WHERE status='pending'")
        total_watchers = await conn.fetchval('SELECT COUNT(*) FROM watchlist')
        total_reports = await conn.fetchval("SELECT COUNT(*) FROM reports WHERE status='open'")
        total_banned = await conn.fetchval('SELECT COUNT(*) FROM banned_users')
        top_item = await conn.fetchrow(
            'SELECT item_name, COUNT(*) as c FROM sold_history GROUP BY item_name ORDER BY c DESC LIMIT 1'
        )
        top_seller = await conn.fetchrow(
            'SELECT seller_name, COUNT(*) as c FROM sold_history GROUP BY seller_name ORDER BY c DESC LIMIT 1'
        )
        most_listed = await conn.fetchrow(
            'SELECT item_name, price FROM market_listings ORDER BY listed_at DESC LIMIT 1'
        )
    embed = discord.Embed(
        title='📊 GRAND MARKET — Server Stats',
        color=discord.Color.blurple()
    )
    embed.add_field(
        name='🏪 Active Market',
        value=f'📦 Listings: **{active_count}**\n💰 Total Value: **₹{active_value:,}**\n🤝 Pending Offers: **{pending_offers}**',
        inline=True
    )
    embed.add_field(
        name='💸 All-Time Sales',
        value=f'✅ Items Sold: **{total_sold}**\n💵 Total Traded: **₹{total_earned:,}**',
        inline=True
    )
    embed.add_field(
        name='👥 Community',
        value=f'🔔 Watchers: **{total_watchers}**\n🚨 Open Reports: **{total_reports}**\n🔨 Banned: **{total_banned}**',
        inline=True
    )
    if top_seller:
        embed.add_field(name='🥇 Top Seller', value=f'**{top_seller["seller_name"]}** — {top_seller["c"]} sales', inline=True)
    if top_item:
        embed.add_field(name='🔥 Most Sold Item', value=f'**{top_item["item_name"]}** — {top_item["c"]} times', inline=True)
    if most_listed:
        embed.add_field(name='🆕 Latest Listing', value=f'**{most_listed["item_name"]}** — ₹{most_listed["price"]}', inline=True)
    embed.set_footer(text='Grand Market — Live stats | Sab real-time database se hai')
    await ctx.send(embed=embed)


@client.command(name='dm')
async def direct_message(ctx, *args):
    if not ctx.message.mentions:
        await ctx.send('❌ Sahi format: `!dm @user <message>` — jaise `!dm @Ram Bike abhi bhi available hai?`')
        return
    target = ctx.message.mentions[0]
    msg_parts = [w for w in args if not w.startswith('<@')]
    if not msg_parts:
        await ctx.send('❌ Message likhna zaroori hai — jaise `!dm @Ram Bike available hai kya?`')
        return
    if target.id == ctx.author.id:
        await ctx.send('❌ Khud ko DM nahi kar sakte!')
        return
    if target.bot:
        await ctx.send('❌ Bots ko DM nahi kar sakte!')
        return
    message_text = ' '.join(msg_parts)
    embed = discord.Embed(
        title='📩 Grand Market — Message Aaya!',
        description=message_text,
        color=discord.Color.blurple()
    )
    embed.add_field(name='👤 From', value=f'{ctx.author.display_name} ({ctx.author.mention})', inline=True)
    embed.add_field(name='📍 Server', value=ctx.guild.name if ctx.guild else 'DM', inline=True)
    embed.set_thumbnail(url=ctx.author.display_avatar.url)
    embed.set_footer(text=f'Reply karne ke liye unhe server mein mention karo | Grand Market')
    try:
        await target.send(embed=embed)
        await ctx.send(f'✅ Message {target.mention} ko bhej diya {ctx.author.mention}! 📩')
    except discord.Forbidden:
        await ctx.send(f'❌ **{target.display_name}** ke DMs band hain. Unhe directly tag karo.')
    except Exception as e:
        await ctx.send(f'❌ Message nahi gaya: {e}')


@client.command()
async def remove(ctx, *args):
    if not args:
        await ctx.send('❌ Sahi format: `!remove <item>` — jaise `!remove Bike`')
        return
    item_name = ' '.join(args)
    async with db_pool.acquire() as conn:
        row = await conn.fetchrow(
            'SELECT seller_id FROM market_listings WHERE item_name = $1', item_name
        )
        if not row:
            await ctx.send(f'❌ **{item_name}** market mein nahi mila.')
            return
        if row['seller_id'] != str(ctx.author.id):
            await ctx.send('❌ Tum sirf apni listings hata sakte ho!')
            return
        await conn.execute('DELETE FROM market_listings WHERE item_name = $1', item_name)
    await ctx.send(f'🗑️ **{item_name}** listing hata di gayi {ctx.author.mention}.')


keep_alive()
client.run(os.getenv('TOKEN'))
