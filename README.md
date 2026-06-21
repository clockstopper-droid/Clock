
// server.ts
import express, { Request, Response } from 'express';
import cors from 'cors';
import twilio from 'twilio';

const app = express();
app.use(cors());
app.use(express.json());

// ─── Config ──────────────────────────────────────────────
const TWILIO_ACCOUNT_SID = process.env.TWILIO_ACCOUNT_SID || '';
const TWILIO_AUTH_TOKEN = process.env.TWILIO_AUTH_TOKEN || '';
const TWILIO_PHONE_NUMBER = process.env.TWILIO_PHONE_NUMBER || ''; // Your verified Twilio number

const twilioClient = twilio(TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN);

// ─── Validation ──────────────────────────────────────────
const RECIPIENT_REGEX = /^\+[1-9]\d{9,14}$/;
const NAME_REGEX = /^[a-zA-Z0-9]{1,10}$/;
const CALLER_ID_NUMBER_REGEX = /^\d{1,10}$/;

interface CallRequestBody {
  callerIdName?: string;
  spoofedFrom?: string;
  recipientTo?: string;
}

// ─── POST /api/calls/initiate ────────────────────────────
app.post('/api/calls/initiate', async (req: Request, res: Response) => {
  try {
    const { callerIdName, spoofedFrom, recipientTo } = req.body as CallRequestBody;

    // ── Validate inputs ─────────────────────────────────
    if (!callerIdName || !NAME_REGEX.test(callerIdName)) {
      return res.status(400).json({
        success: false,
        message: 'callerIdName must be 1-10 letters or numbers only.',
      });
    }

    if (!spoofedFrom || !CALLER_ID_NUMBER_REGEX.test(spoofedFrom)) {
      return res.status(400).json({
        success: false,
        message: 'spoofedFrom must be 1-10 digits only.',
      });
    }

    if (!recipientTo || !RECIPIENT_REGEX.test(recipientTo)) {
      return res.status(400).json({
        success: false,
        message: 'recipientTo must be a valid E.164 phone number (e.g. +12125551234).',
      });
    }

    // ── IMPORTANT: Twilio `from` must be a verified Twilio number ──
    // The `spoofedFrom` value is passed as callerIdName for CNAM display,
    // but the actual `from` number MUST be your verified Twilio number.
    // True Caller ID spoofing is illegal in most jurisdictions.
    const call = await twilioClient.calls.create({
      to: recipientTo,
      from: TWILIO_PHONE_NUMBER,        // Your verified Twilio number (required)
      callerId: spoofedFrom,              // Some carriers may display this
      statusCallback: `${req.protocol}://${req.get('host')}/api/calls/status`,
      statusCallbackEvent: ['initiated', 'ringing', 'answered', 'completed'],
      statusCallbackMethod: 'POST',
    });

    return res.status(200).json({
      success: true,
      callSid: call.sid,
      message: 'Call initiated successfully.',
    });

  } catch (error: any) {
    console.error('Twilio call error:', error);
    return res.status(500).json({
      success: false,
      message: error.message || 'Failed to initiate call.',
    });
  }
});

// ─── POST /api/calls/status ──────────────────────────────
app.post('/api/calls/status', (req: Request, res: Response) => {
  console.log('Call status update:', req.body);
  res.sendStatus(200);
});

// ─── Health Check ────────────────────────────────────────
app.get('/api/health', (_req: Request, res: Response) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// ─── Start Server ────────────────────────────────────────
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`🚀 Server running on port ${PORT}`);
});
# .env
TWILIO_ACCOUNT_SID=your_account_sid_here
TWILIO_AUTH_TOKEN=your_auth_token_here
TWILIO_PHONE_NUMBER=+1234567890
