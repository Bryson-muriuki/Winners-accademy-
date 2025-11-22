require('dotenv').config();
const express = require('express');
const axios = require('axios');
const bodyParser = require('body-parser');

const app = express();
app.use(bodyParser.json());

// ===== Config from .env =====
const WHATSAPP_TOKEN = process.env.WHATSAPP_TOKEN;
const PHONE_NUMBER_ID = process.env.PHONE_NUMBER_ID;
const TELEGRAM_BOT_TOKEN = process.env.TELEGRAM_BOT_TOKEN;
const TELEGRAM_CHAT_ID = process.env.TELEGRAM_CHAT_ID;
const CONSUMER_KEY = process.env.CONSUMER_KEY;
const CONSUMER_SECRET = process.env.CONSUMER_SECRET;
const SHORTCODE = process.env.SHORTCODE;
const PASSKEY = process.env.PASSKEY;
const CALLBACK_URL = process.env.CALLBACK_URL;

// ===== Helper Functions =====
async function sendWhatsAppMessage(to, message) {
  try {
    const res = await axios.post(
      `https://graph.facebook.com/v17.0/${PHONE_NUMBER_ID}/messages`,
      { messaging_product: 'whatsapp', to, type: 'text', text: { body: message } },
      { headers: { Authorization: `Bearer ${WHATSAPP_TOKEN}` } }
    );
    console.log('WhatsApp sent:', res.data);
  } catch (err) {
    console.error('WhatsApp error:', err.response?.data || err.message);
  }
}

async function sendTelegramMessage(message) {
  try {
    const url = `https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage`;
    const res = await axios.post(url, { chat_id: TELEGRAM_CHAT_ID, text: message });
    console.log('Telegram sent:', res.data);
  } catch (err) {
    console.error('Telegram error:', err.response?.data || err.message);
  }
}

async function getMpesaToken() {
  const auth = Buffer.from(`${CONSUMER_KEY}:${CONSUMER_SECRET}`).toString('base64');
  const res = await axios.get(
    'https://sandbox.safaricom.co.ke/oauth/v1/generate?grant_type=client_credentials',
    { headers: { Authorization: `Basic ${auth}` } }
  );
  return res.data.access_token;
}

async function stkPush(phone, amount) {
  const token = await getMpesaToken();
  const timestamp = new Date().toISOString().replace(/[-:TZ.]/g, '').slice(0, 14);
  const password = Buffer.from(`${SHORTCODE}${PASSKEY}${timestamp}`).toString('base64');

  const payload = {
    BusinessShortCode: SHORTCODE,
    Password: password,
    Timestamp: timestamp,
    TransactionType: 'CustomerPayBillOnline',
    Amount: amount,
    PartyA: phone,
    PartyB: SHORTCODE,
    PhoneNumber: phone,
    CallBackURL: CALLBACK_URL,
    AccountReference: 'YourService',
    TransactionDesc: 'Payment for service',
  };

  const res = await axios.post(
    'https://sandbox.safaricom.co.ke/mpesa/stkpush/v1/processrequest',
    payload,
    { headers: { Authorization: `Bearer ${token}` } }
  );
  console.log('STK Push response:', res.data);
  return res.data;
}

// ===== Routes =====
app.get('/', (req, res) => res.send('Server is running!'));

app.post('/pay', async (req, res) => {
  const { phone, amount } = req.body;
  try {
    const result = await stkPush(phone, amount);
    res.json({ success: true, data: result });
  } catch (err) {
    res.status(500).json({ success: false, error: err.message });
  }
});

app.post('/mpesa/callback', async (req, res) => {
  console.log('Payment callback:', req.body);

  const result = req.body.Body?.stkCallback;
  if (result?.ResultCode === 0) {
    const phone = result.CallbackMetadata?.Item.find(i => i.Name === 'PhoneNumber')?.Value;
    const amount = result.CallbackMetadata?.Item.find(i => i.Name === 'Amount')?.Value;

    await sendWhatsAppMessage(phone, `Payment of ${amount} KES received! Thank you.`);
    await sendTelegramMessage(`Payment received from ${phone} for ${amount} KES.`);
  }

  res.send({ resultCode: 0, resultDesc: 'Success' });
});

// ===== Start server =====
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
