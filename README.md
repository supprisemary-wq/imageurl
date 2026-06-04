import os
import httpx
from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes

# Retrieve token from Render Environment Settings
TOKEN = os.getenv("TOKEN", "YOUR_BOT_TOKEN")

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Send a friendly welcome message on /start."""
    welcome_text = (
        "🖼️ **Welcome to the Keyless Image-to-URL Bot!**\n\n"
        "Send or forward me any photo, and I will instantly upload it anonymously and give you a direct link. No setup required!"
    )
    await update.message.reply_text(welcome_text, parse_mode="Markdown")

async def handle_image(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Process incoming images, upload anonymously, and return the link."""
    status_message = await update.message.reply_text("⚡ Processing image link... please wait.")

    try:
        # Get the highest resolution version of the photo sent
        photo_file = await update.message.photo[-1].get_file()
        
        async with httpx.AsyncClient() as client:
            # Download image from Telegram's servers
            tg_response = await client.get(photo_file.file_path)
            if tg_response.status_code != 200:
                raise Exception("Failed to get image from Telegram.")
                
            image_bytes = tg_response.content

            # Upload to Telegraph completely keyless
            telegraph_url = "https://telegra.ph/upload"
            files = {"file": ("image.jpg", image_bytes, "image/jpeg")}
            
            response = await client.post(telegraph_url, files=files, timeout=30.0)
            
        if response.status_code == 200:
            data = response.json()
            # Telegraph returns a list containing the source path
            if isinstance(data, list) and len(data) > 0 and "src" in data[0]:
                direct_url = f"https://telegra.ph{data[0]['src']}"
                
                success_report = (
                    "✅ **IMAGE LINK GENERATED!**\n"
                    "━━━━━━━━━━━━━━━━━━━━\n"
                    f"🔗 **Direct URL:**\n`{direct_url}`\n"
                    "━━━━━━━━━━━━━━━━━━━━\n"
                    "💡 _Tip: Tap the link to instantly copy it._"
                )
                await status_message.edit_text(success_report, parse_mode="Markdown")
                return
                
        await status_message.edit_text("❌ Upload failed. Please try a different image.")

    except Exception as e:
        print(f"Error: {str(e)}")
        await status_message.edit_text("❌ An error occurred while generating your link.")

def main():
    """Start the bot application loop."""
    application = Application.builder().token(TOKEN).build()

    # Register Handlers
    application.add_handler(CommandHandler("start", start))
    application.add_handler(MessageHandler(filters.PHOTO, handle_image))

    # Run polling loop
    print("✅ Image-to-URL Bot is running keyless...")
    application.run_polling(drop_pending_updates=True)

if __name__ == '__main__':
    main()
