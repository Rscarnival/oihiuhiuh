import discord
from google.oauth2.service_account import Credentials
from googleapiclient.discovery import build
from oauth2client.service_account import ServiceAccountCredentials

# Set up credentials to access Google Sheets API
scope = ['https://www.googleapis.com/auth/spreadsheets', 'https://www.googleapis.com/auth/drive']
credentials = ServiceAccountCredentials.from_json_keyfile_name("client_secret.json", scope)

sheet_id = ''
sheet_name = ''

# Set up Discord client and channel ID
intents = discord.Intents().all()  # specify all intents
client = discord.Client(intents=intents)
channel_id = 1079315161413853235

# Define function to fetch data from Google Sheet and format it as a Discord embed message
async def send_sheet_embed(rows):
    # Format data as a Discord embed message
    embed = discord.Embed(title='Google Sheet Table', color=discord.Color.blue())
    for row in rows:
        row_string = ''
        for i, cell in enumerate(row[1:], start=1):
            row_string += cell + '\t\t'
            if i % 4 == 0:  # Add new line after every 4th cell
                row_string += '\n'
        embed.add_field(name=row[0], value=row_string, inline=False)
    embed.set_footer(text=f'Total Rows: {len(rows)}')

    # Send the embed message to Discord channel
    channel = client.get_channel(channel_id)
    await channel.send(embed=embed)

# Run the bot and send the embed message on startup
@client.event
async def on_ready():
    # Fetch data from Google Sheet using Sheets API
    service = build('sheets', 'v4', credentials=credentials)
    sheet = service.spreadsheets()
    range_name = f"{sheet_name}!A1:N502" # select range of the sheet
    result = sheet.values().get(spreadsheetId=sheet_id, range=range_name).execute()
    rows = result.get('values', [])

    # Group rows by value in column M and send each group as a separate Discord embed message
    current_group = None
    group_rows = []
    for row in rows:
        # Ignore empty rows or rows with missing data
        if len(row) < 14:
            continue
        # Hide column M by removing it from the row
        row.pop(12)
        # Check if the current row belongs to a new group
        if current_group != row[-1]:
            # Send the current group as a Discord embed message
            if group_rows:
                await send_sheet_embed(group_rows)
            # Start a new group with the current row
            current_group = row[-1]
            group_rows = [row]
        else:
            # Add the current row to the current group
            group_rows.append(row)
    # Send the final group as a Discord embed message
    if group_rows:
        await send_sheet_embed(group_rows)

    print(f'Logged in as {client.user}')

client.run('')
