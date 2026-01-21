const express = require('express');
const axios = require('axios');
const admin = require('firebase-admin');
const cors = require('cors');
require('dotenv').config();

const app = express();
app.use(cors());
app.use(express.json());

// 1. Initialize Firebase Admin (Using Service Account)
const serviceAccount = require("./serviceAccountKey.json");
admin.initializeApp({
  credential: admin.credential.cert(serviceAccount),
  databaseURL: "https://your-project-id.firebaseio.com"
});
const db = admin.firestore();

// 2. NagorikPay Configuration
const NAGORIKPAY_CONFIG = {
  BRAND_KEY: "yWFzmSyGYyY7EBEWTmh34dDcAhXyayB1LY2C3XQISk4OEI93eB",
  DEVICE_KEY: "PecCkQwN5lFyekZZEtrcoSANhleAh8ww7wNmeuwV",
  CREATE_URL: "https://secure-pay.nagorikpay.com/api/payment/create",
  VERIFY_URL: "https://secure-pay.nagorikpay.com/api/payment/verify",
  FIXED_AMOUNT: 500 // BDT - Fixed on backend for security
};

// --- ROUTE: INITIATE PAYMENT ---
app.post('/api/pay/initiate', async (req, res) => {
  const { userId, customerName, customerEmail, customerPhone } = req.body;

  if (!userId) return res.status(400).json({ error: "User ID required" });

  try {
    const payload = {
      amount: NAGORIKPAY_CONFIG.FIXED_AMOUNT,
      customer_name: customerName || "User",
      customer_email: customerEmail || "user@taskripple.com",
      customer_phone: customerPhone || "01700000000",
      order_id: `ACT_${userId}_${Date.now()}`, // Unique order ID
      success_url: `${process.env.FRONTEND_URL}/#/activation?status=SUCCESS`,
      cancel_url: `${process.env.FRONTEND_URL}/#/activation?status=CANCEL`,
      type: 'checkout'
    };

    const response = await axios.post(NAGORIKPAY_CONFIG.CREATE_URL, payload, {
      headers: { 
        'api-key': NAGORIKPAY_CONFIG.BRAND_KEY,
        'device-key': NAGORIKPAY_CONFIG.DEVICE_KEY, // If required by NagorikPay
        'Content-Type': 'application/json' 
      }
    });

    if (response.data.status) {
      res.json({ payment_url: response.data.payment_url });
    } else {
      res.status(400).json({ message: response.data.message || "Gateway Error" });
    }
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// --- ROUTE: VERIFY PAYMENT & ACTIVATE ---
app.post('/api/pay/verify', async (req, res) => {
  const { transactionId, userId } = req.body;

  try {
    // 1. Verify with NagorikPay
    const verifyRes = await axios.post(NAGORIKPAY_CONFIG.VERIFY_URL, { transactionId }, {
      headers: { 'api-key': NAGORIKPAY_CONFIG.BRAND_KEY }
    });

    const data = verifyRes.data;

    // 2. Security Checks
    if (data.status === "COMPLETED") {
      
      const paymentRef = db.collection('payments').doc(transactionId);
      const paymentDoc = await paymentRef.get();

      // Prevent duplicate activation
      if (paymentDoc.exists) {
        return res.status(400).json({ message: "Transaction already processed." });
      }

      // 3. Atomically Update User and Log Payment
      const batch = db.batch();
      const userRef = db.collection('users').doc(userId);

      batch.update(userRef, {
        status: 'ACTIVE',
        isVerified: true,
        activatedAt: admin.firestore.FieldValue.serverTimestamp()
      });

      batch.set(paymentRef, {
        userId,
        transactionId,
        amount: data.amount,
        method: data.payment_method,
        status: 'COMPLETED',
        createdAt: admin.firestore.FieldValue.serverTimestamp()
      });

      await batch.commit();
      res.json({ success: true, message: "Account activated!" });
    } else {
      res.status(400).json({ success: false, message: "Payment not completed." });
    }
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// --- ROUTE: WEBHOOK (Optional for async updates) ---
app.post('/api/pay/webhook', async (req, res) => {
    // Handle async logic if NagorikPay hits this IP
    console.log("Webhook received:", req.body);
    res.sendStatus(200);
});

app.listen(5000, () => console.log('Payment Server running on port 5000'));