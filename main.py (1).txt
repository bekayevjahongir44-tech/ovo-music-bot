
import os
import logging
import re
from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes
import yt_dlp

logging.basicConfig(level=logging.INFO)
TOKEN = os.getenv("BOT_TOKEN")
CHANNEL = os.getenv("CHANNEL_LINK", "@ovo_bot")  # kanalingni yoz

# /start komandasi
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        f"🎵 *OVO Sound Music ga xush kelibsiz!*\n\n"
        f"Musiqa nomini yozing, men topib beraman!\n"
        f"Masalan: `Miyagi - Minor` yoki `Morgenshtern`\n\n"
        f"📢 Kanalimiz: {CHANNEL}",
        parse_mode="Markdown"
    )

# Musiqa qidirish funksiyasi
def search_youtube(query):
    ydl_opts = {
        'format': 'bestaudio/best',
        'noplaylist': True,
        'quiet': True,
        'default_search': 'ytsearch1',
        'extract_flat': False,
    }
    with yt_dlp.YoutubeDL(ydl_opts) as ydl:
        info = ydl.extract_info(f"ytsearch1:{query}", download=False)
        if 'entries' in info and len(info['entries']) > 0:
            return info['entries'][0]
        return info

# Har qanday text yozganda musiqa chiqarish
async def handle_music(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.message.text.strip()
    if query.startswith('/'):
        return
    
    # Juda qisqa so'zlarni ignore
    if len(query) < 2:
        return

    status_msg = await update.message.reply_text(f"🔍 `{query}` qidirilmoqda...", parse_mode="Markdown")
    
    try:
        # 1. YouTube dan topish
        video = search_youtube(query + " audio")
        if not video:
            await status_msg.edit_text("😕 Musiqa topilmadi, boshqa nom bilan urinib ko'ring!")
            return

        title = video.get('title', query)
        webpage_url = video.get('webpage_url')
        duration = video.get('duration', 0)
        thumbnail = video.get('thumbnail')
        
        # 2. Audio ni yuklab olish va yuborish
        await status_msg.edit_text(f"⬇️ *{title}* yuklanmoqda...", parse_mode="Markdown")
        
        ydl_opts = {
            'format': 'bestaudio/best',
            'outtmpl': f'/tmp/%(id)s.%(ext)s',
            'postprocessors': [{
                'key': 'FFmpegExtractAudio',
                'preferredcodec': 'mp3',
                'preferredquality': '192',
            }],
            'quiet': True,
        }
        
        file_path = None
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(webpage_url, download=True)
            file_path = ydl.prepare_filename(info)
            # mp3 ga aylangandan keyin
            file_path = os.path.splitext(file_path)[0] + ".mp3"

        # 3. Telegram ga MP3 qilib yuborish
        if os.path.exists(file_path):
            await update.message.reply_audio(
                audio=open(file_path, 'rb'),
                title=title[:64],
                performer="OVO Sound Music",
                caption=f"🎵 *{title}*\n\n📢 {CHANNEL} - obuna bo'ling!\n🔗 {webpage_url}",
                parse_mode="Markdown"
            )
            os.remove(file_path)
            await status_msg.delete()
        else:
            # Agar yuklab bo'lmasa link berish
            await status_msg.edit_text(
                f"🎵 *{title}* topildi!\n\n🔗 {webpage_url}\n\n📢 {CHANNEL}",
                parse_mode="Markdown"
            )

    except Exception as e:
        logging.error(f"Error: {e}")
        try:
            await status_msg.edit_text(f"❌ Xatolik: {str(e)[:200]}\nQaytadan urinib ko'ring!")
        except:
            await update.message.reply_text("❌ Xatolik yuz berdi, qaytadan urinib ko'ring!")

def main():
    if not TOKEN:
        print("BOT_TOKEN yo'q! Render da Environment ga qo'ying")
        return
    
    app = Application.builder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_music))
    
    print("OVO Sound Music bot ishga tushdi...")
    app.run_polling()

if __name__ == "__main__":
    main()
