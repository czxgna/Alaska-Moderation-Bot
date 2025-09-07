import discord
from discord.ext import commands
import datetime

intents = discord.Intents.default()
intents.message_content = True
intents.members = True
intents.guilds = True
intents.bans = True

bot = commands.Bot(command_prefix="!", intents=intents)

# customize this to your log channel name or ID
LOG_CHANNEL_NAME = "mod-logs"  # Or set to the channel ID if preferred


# get timestamp
def get_timestamp():
    return datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")


# create embed for ban
def create_ban_embed(target, moderator, reason):
    embed = discord.Embed(
        title="User Banned (SyncMod)",
        color=discord.Color.red(),
        timestamp=datetime.datetime.utcnow()
    )
    embed.add_field(name="User", value=f"{target} (`{target.id}`)", inline=False)
    embed.add_field(name="Moderator", value=f"{moderator.mention}", inline=False)
    embed.add_field(name="Reason", value=reason, inline=False)
    embed.set_footer(text="Ban issued at")
    return embed


# syncban slash command
@bot.command(name="syncban")
@commands.has_permissions(ban_members=True)
async def syncban(ctx, user: discord.User, *, reason: str = "No reason provided"):
    success_servers = 0
    failed_servers = 0

    dm_embed = discord.Embed(
        title="⛔ You have been banned (SyncMod)",
        description=f"You have been banned from multiple servers by `{ctx.guild.name}`'s staff.",
        color=discord.Color.red()
    )
    dm_embed.add_field(name="Reason", value=reason, inline=False)
    dm_embed.set_footer(text=f"Banned at {get_timestamp()}")

    # Attempt to DM user
    try:
        await user.send(embed=dm_embed)
    except Exception:
        pass  # User might have DMs off

    for guild in bot.guilds:
        try:
            await guild.ban(user, reason=reason, delete_message_days=0)
            success_servers += 1

            # Find log channel in this guild
            log_channel = discord.utils.get(guild.text_channels, name=LOG_CHANNEL_NAME)
            if log_channel:
                await log_channel.send(embed=create_ban_embed(user, ctx.author, reason))
        except Exception as e:
            failed_servers += 1
            print(f"[ERROR] Could not ban in {guild.name}: {e}")

    summary = discord.Embed(
        title="SyncBan Summary",
        description=f"{user} was banned from `{success_servers}` server(s). `{failed_servers}` failed.",
        color=discord.Color.green() if failed_servers == 0 else discord.Color.orange()
    )
    summary.add_field(name="Reason", value=reason, inline=False)
    await ctx.send(embed=summary)


# debugging
@bot.event
async def on_command_error(ctx, error):
    if isinstance(error, commands.MissingPermissions):
        await ctx.send(embed=discord.Embed(
            title="🚫 Missing Permissions",
            description="You don't have permission to use this command.",
            color=discord.Color.red()
        ))
    elif isinstance(error, commands.MissingRequiredArgument):
        await ctx.send(embed=discord.Embed(
            title="⚠️ Missing Argument",
            description="Please provide all required arguments.",
            color=discord.Color.orange()
        ))
    elif isinstance(error, commands.BadArgument):
        await ctx.send(embed=discord.Embed(
            title="❗ Bad Argument",
            description="Could not find that user.",
            color=discord.Color.orange()
        ))
    else:
        raise error


# bot working
@bot.event
async def on_ready():
    print(f"Bot is online as {bot.user}")


# Run the bot
bot.run("")
