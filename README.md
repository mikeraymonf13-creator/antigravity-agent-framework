
import express from 'express';
import rateLimit from 'express-rate-limit';
import { validateCaptchaToken } from '../captcha-validation.js';

const router = express.Router();

// Limit repeated CAPTCHA attempts.
const tonConnectLimiter = rateLimit({
  windowMs: 60 * 1000,
  limit: 10,
  standardHeaders: 'draft-8',
  legacyHeaders: false,
  message: {
    error: 'Too many requests. Please try again later.',
  },
});

// CAPTCHA verification middleware.
async function requireCaptcha(req, res, next) {
  try {
    const passToken = req.body?.pass_token;

    if (
      typeof passToken !== 'string' ||
      passToken.trim().length === 0
    ) {
      return res.status(400).json({
        success: false,
        error: 'CAPTCHA token is required.',
      });
    }

    const captchaResult = await validateCaptchaToken(passToken);

    if (!captchaResult?.success) {
      return res.status(403).json({
        success: false,
        error: 'CAPTCHA verification failed.',
      });
    }

    next();
  } catch (error) {
    console.error('CAPTCHA verification error:', error.message);

    return res.status(503).json({
      success: false,
      error: 'CAPTCHA verification is temporarily unavailable.',
    });
  }
}

// TON Connect preflight endpoint.
router.post(
  '/ton-connect/preflight',
  tonConnectLimiter,
  requireCaptcha,
  async (req, res) => {
    try {
      return res.status(200).json({
        success: true,
        captchaVerified: true,
        network: 'testnet',
        message: 'CAPTCHA verified. Continue with TON Connect on the client.',
      });
    } catch (error) {
      console.error('TON Connect preflight error:', error.message);

      return res.status(500).json({
        success: false,
        error: 'Unable to prepare TON Connect.',
      });
    }
  }
);

export default router;
