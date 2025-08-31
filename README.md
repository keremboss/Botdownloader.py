# Botdownloader.py
import telebot
from telebot import types
import yt_dlp
import os
import re

# የቦትዎን ቶከን እዚህ ያስገቡ
TOKEN = '1998034266:AAErPZkifx7LrylziAc4z7ATTRMUHS1j83M' 
bot = telebot.TeleBot(TOKEN)

video_url = ""

# ፕሮክሲን ለመጠቀም ከፈለጉ እዚህ ቦታ ላይ ፕሮክሲ አድራሻዎን ያስገቡ።
# ምሳሌ: 'socks5://user:pass@host:port'
# የ PythonAnywhere ነፃ ፕሮክሲ ካለ, እሱን መጠቀም ይችላሉ።
# ከሌለዎት, መስመሩን ባዶ ይተዉት።
PROXY = None

@bot.message_handler(commands=['start', 'help'])
def send_welcome(message):
    bot.reply_to(message, "ሰላም! እኔ ከዩቲዩብ፣ ከፌስቡክ እና ከቲክቶክ ቪዲዮዎችን ማውረድ እችላለሁ። የሚፈልጉትን ቪዲዮ ሊንክ ይላኩልኝ።")

@bot.message_handler(func=lambda message: re.match(r'^(https?://)?(www\.)?(youtube\.com|youtu\.be|facebook\.com|fb\.watch|tiktok\.com|vm\.tiktok\.com|www\.tiktok\.com)/.+', message.text))
def handle_link(message):
    global video_url
    video_url = message.text
    chat_id = message.chat.id
    
    bot.send_message(chat_id, "የቪዲዮውን መረጃ እያገኘሁ ነው። እባክዎ ትንሽ ይጠብቁ...")

    try:
        ydl_opts = {'quiet': True, 'no_warnings': True}
        if PROXY:
            ydl_opts['proxy'] = PROXY
            
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(video_url, download=False)
            
        video_title = info.get('title', 'ያልተሰየመ ቪዲዮ')
        
        markup = types.InlineKeyboardMarkup(row_width=2)
        
        available_formats = set()
        for f in info.get('formats', []):
            if f.get('ext') == 'mp4' and 'vbr' in f and f.get('height') is not None:
                available_formats.add(f.get('height'))
        
        sorted_formats = sorted(list(available_formats), reverse=True)
        
        if not sorted_formats:
            bot.send_message(chat_id, "ለዚህ ቪዲዮ የሚገኙ የጥራት አማራጮች የሉም።")
            return

        for res in sorted_formats:
            markup.add(types.InlineKeyboardButton(f"{res}p", callback_data=f"{res}p"))
        
        markup.add(types.InlineKeyboardButton("ኦዲዮ ብቻ", callback_data='audio'))
        
        bot.send_message(chat_id, f"የ'**{video_title}**' ቪዲዮን ለማውረድ የጥራት ደረጃ ይምረጡ።", reply_markup=markup, parse_mode='Markdown')

    except Exception as e:
        bot.send_message(chat_id, f"ቪዲዮውን ማግኘት አልተቻለም። ሊንኩን ደግመው ያረጋግጡ።\nስህተት: {e}")

@bot.callback_query_handler(func=lambda call: True)
def callback_handler(call):
    quality = call.data
    chat_id = call.message.chat.id
    
    if not video_url:
        bot.send_message(chat_id, "እባክዎ መጀመሪያ የቪዲዮውን ሊንክ ይላኩልኝ።")
        return
        
    bot.send_message(chat_id, "ማውረድ እየተጀመረ ነው... እባክዎ ይጠብቁ።")
    bot.send_chat_action(chat_id, 'upload_document')

    try:
        if quality == 'audio':
            ydl_opts = {
                'format': 'bestaudio/best',
                'outtmpl': '%(title)s.%(ext)s',
                'postprocessors': [{
                    'key': 'FFmpegExtractAudio',
                    'preferredcodec': 'mp3',
                    'preferredquality': '192',
                }],
            }
        else:
            ydl_opts = {
                'format': f'bestvideo[height<={quality[:-1]}]',
                'outtmpl': '%(title)s.%(ext)s',
            }

        if PROXY:
            ydl_opts['proxy'] = PROXY
            
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(video_url, download=True)
            file_path = ydl.prepare_filename(info)

        with open(file_path, 'rb') as f:
            if quality == 'audio':
                bot.send_audio(chat_id, f)
            else:
                bot.send_video(chat_id, f)

        os.remove(file_path)
        
    except Exception as e:
        bot.send_message(chat_id, f"ፋይሉን በማውረድ ጊዜ ስህተት አጋጥሟል።\nስህተት: {e}")

bot.polling()
