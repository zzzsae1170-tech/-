worker: python bot.pypython-telegram-bot==20.3
aiohttpFROM python:3.11-slim

RUN apt-get update && apt-get install -y ffmpeg gcc build-essential && apt-get clean && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "bot.py"]import os
import asyncio
import shutil
import tempfile
import subprocess
from pathlib import Path
import aiohttp
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, ContextTypes, filters, CallbackQueryHandler

BOT_TOKEN = os.environ.get("BOT_TOKEN")
TELEGRAM_MAX_BYTES = int(os.environ.get("TELEGRAM_MAX_BYTES", 50 * 1024 * 1024))
DEFAULT_CRF = 28

if not BOT_TOKEN:
    raise RuntimeError("مطلوب: إعداد متغير البيئة BOT_TOKEN")

QUALITY_OPTIONS = {
    "high": 23,
    "medium": 28,
    "low": 35
}

user_files = {}

async def start_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "مرحباً! أرسل لي فيديو لأقوم بضغطه.\n"
        "بعد الإرسال، اختر جودة الضغط من الأزرار."
    )

async def handle_video(update: Update, context: ContextTypes.DEFAULT_TYPE):
    msg = update.message
    file_obj = msg.video or (msg.document if msg.document and msg.document.mime_type.startswith("video") else None)
    if not file_obj:
        await msg.reply_text("يرجى إرسال فيديو أو ملف فيديو.")
        return

    tmp = tempfile.mkdtemp(prefix="tgcomp_")
    input_path = Path(tmp) / (file_obj.file_name or "input.mp4")

    f = await context.bot.get_file(file_obj.file_id)
    await f.download_to_drive(str(input_path))

    user_files[msg.from_user.id] = {"input_path": input_path, "tmp_dir": tmp}

    keyboard = [
        [
            InlineKeyboardButton("جودة عالية", callback_data="compress_high"),
            InlineKeyboardButton("جودة متوسطة", callback_data="compress_medium"),
            InlineKeyboardButton("جودة منخفضة", callback_data="compress_low"),
        ]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    await msg.reply_text("اختر جودة الضغط:", reply_markup=reply_markup)

async def compress_video(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()

    user_id = query.from_user.id
    if user_id not in user_files:
        await query.edit_message_text("عذراً، لم يتم العثور على فيديو للضغط. أرسل فيديو جديد.")
        return

    quality_key = query.data.replace("compress_", "")
    crf = QUALITY_OPTIONS.get(quality_key, DEFAULT_CRF)

    data = user_files[user_id]
    input_path = data["input_path"]
    tmp_dir = data["tmp_dir"]
    output_path = Path(tmp_dir) / "output.mp4"

    await query.edit_message_text("⏳ جاري الضغط...")

    def ffmpeg_compress():
        cmd = [
            "ffmpeg", "-y", "-i", str(input_path),
            "-vcodec", "libx264", "-crf", str(crf),
            "-preset", "fast",
            "-movflags", "+faststart",
            str(output_path)
        ]
        subprocess.run(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)

    loop = asyncio.get_event_loop()
    await loop.run_in_executor(None, ffmpeg_compress)

    size = output_path.stat().st_size
    if size <= TELEGRAM_MAX_BYTES:
        await query.message.reply_video(video=open(output_path, "rb"), caption=f"تم ضغط الفيديو بجودة {quality_key}.")
    else:
        async with aiohttp.ClientSession() as sess:
            with open(output_path, "rb") as fh:
                data = fh.read()
            url = f"https://transfer.sh/{output_path.name}"
            async with sess.put(url, data=data) as resp:
                text = await resp.text()
                if resp.status in (200, 201):
                    await query.message.reply_text(f"حجم الفيديو بعد الضغط كبير جداً، يمكنك تحميله من هنا:\n{text.strip()}")
                else:
                    await query.message.reply_text("فشل رفع الفيديو. حاول مرة أخرى لاحقاً.")

    shutil.rmtree(tmp_dir, ignore_errors=True)
    user_files.pop(user_id, None)

def main():
    application = ApplicationBuilder().token(BOT_TOKEN).build()
    application.add_handler(CommandHandler("start", start_cmd))
    application.add_handler(MessageHandler(filters.VIDEO | filters.Document.VIDEO, handle_video))
    application.add_handler(CallbackQueryHandler(compress_video, pattern=r"compress_.*"))
    application.run_polling()

if __name__ == "__main__":
    main()# -
لتقليل حجم الفيديوهات 
