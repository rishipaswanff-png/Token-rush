# TokenRush Enterprise Specification & Production Deployment Guide

This document contains the complete production-grade engineering blueprints, structures, and schemas designed to scale the TokenRush Rewards Platform to **1 million active users**. This backend specification connects directly with the native Android Kotlin/Compose client implemented in this workspace.

---

## 1. Complete Cross-Platform (Flutter) Project Structure

For long-term multi-platform delivery, are defined standard Flutter structure configurations:

```text
tokenrush_flutter/
├── android/                  # Native Android wrapper configuration
├── ios/                      # iOS deployment signing configurations
├── web/                      # Security headers (COOP, COEP)
├── assets/
│   ├── images/               # Branded coin logs, splash vectors
│   └── fonts/                # SpaceGrotesk, Inter
├── lib/
│   ├── main.dart             # EdgeToEdge launcher, Firebase hook
│   ├── app.dart              # Material 3 dark-theme orchestrator
│   ├── core/
│   │   ├── network/          # JWT client interceptor, retry logic
│   │   ├── security/         # Device check, emulator protection
│   │   ├── theme/            # Color scheme token mapping
│   │   └── utils/            # Haptic feedbacks, validations
│   ├── data/
│   │   ├── datasources/      # Remote API services & local offline SQLite cache
│   │   ├── models/           # Serialization contracts (User, Wallet, TXs)
│   │   └── repositories/     # Repository orchestration
│   ├── domain/               # Pure business policies (Entities, UseCases)
│   └── presentation/         # Bloc / State Management Views
│       ├── splash/
│       ├── login/            # Phone OTP triggers, Google login
│       ├── home/             # AdMob rewarded callbacks
│       ├── wallet/           # Instant transactions ledger
│       ├── rewards/          # UPI checkout / API validation
│       ├── referral/         # Unique invites generator
│       └── admin_panel/      # System settings overrides
└── pubspec.yaml              # AdMob SDK, Bloc, Retrofit, Crypto
```

---

## 2. Distributed Backend Architecture (High-Scale)

Designed for low latency and high scalability, the enterprise backend operates on Google Cloud Platform:

```
                  ┌──────────────────────┐
                  │   Cloudflare Edge    │
                  │ (WAF, DDoS Shield,   │
                  │   Rate-Limit Filters)│
                  └──────────┬───────────┘
                             │ (HTTPS)
                  ┌──────────▼───────────┐
                  │ Google Cloud Armor   │
                  │ (Strict Geo IP India)│
                  └──────────┬───────────┘
                             │
                             ▼
                    [ API Gateway / GCLB ]
                             │
                    ┌────────┴────────┬──────────────────────┐
                    │                 │                      │
         ┌──────────▼──────────┐ ┌────▼─────────────────┐ ┌──▼──────────────────┐
         │ Auth & User Service │ │ Ad Watch & Earning   │ │ Withdrawal & Rewards│
         │   (NodeJS/Express)  │ │   (Go Microservice)  │ │   (NodeJS/Express)  │
         └──────────┬──────────┘ └────┬─────────────────┘ └──┬──────────────────┘
                    │                 │                      │
     ┌──────────────┼─────────────────┼──────────────────────┤
     │              │                 │                      │
┌────▼──────────┐ ┌─▼─────────────┐ ┌─▼──────────────┐ ┌─────▼───────────────┐
│ Redis Cluster │ │ PostgreSQL DB │ │ Firebase Cloud │ │ AdMob SDK Webhook   │
│(Session/Cache)│ │ (Cloud Spanner)││   Messaging    │ │ (Fraud check logic) │
└───────────────┘ └───────────────┘ └────────────────┘ └─────────────────────┘
```

---

## 3. PostgreSQL Schema (Production Schema DDL)

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Table: Users
CREATE TABLE users (
    uid VARCHAR(128) PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    display_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20) UNIQUE NOT NULL,
    referral_code VARCHAR(12) UNIQUE NOT NULL,
    referred_by VARCHAR(128) REFERENCES users(uid) ON DELETE SET NULL,
    is_banned BOOLEAN DEFAULT FALSE,
    device_fingerprint VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_referral ON users(referral_code);

-- Table: Wallet balance (Real-time ledger)
CREATE TABLE wallets (
    id SERIAL PRIMARY KEY,
    user_id VARCHAR(128) UNIQUE REFERENCES users(uid) ON DELETE CASCADE,
    total_tokens INTEGER DEFAULT 0 CHECK (total_tokens >= 0),
    available_balance INTEGER DEFAULT 0 CHECK (available_balance >= 0),
    lifetime_earnings INTEGER DEFAULT 0 CHECK (lifetime_earnings >= 0),
    referral_earnings INTEGER DEFAULT 0 CHECK (referral_earnings >= 0),
    last_updated TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Table: Ledger Transactions
CREATE TABLE transactions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id VARCHAR(128) REFERENCES users(uid) ON DELETE CASCADE,
    amount_tokens INTEGER NOT NULL,
    amount_inr DECIMAL(10,2) NOT NULL,
    tx_type VARCHAR(50) NOT NULL, -- REWARD_AD, REFERRAL_BONUS, DAILY_BONUS, WITHDRAWAL_UPI, REDEMPTION_REFUND
    description VARCHAR(255) NOT NULL,
    status VARCHAR(20) DEFAULT 'COMPLETED', -- COMPLETED, PENDING, FAILED
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_tx_user_time ON transactions(user_id, created_at DESC);

-- Table: Withdrawals Requests
CREATE TABLE withdrawals (
    id SERIAL PRIMARY KEY,
    user_id VARCHAR(128) REFERENCES users(uid) ON DELETE CASCADE,
    type VARCHAR(50) NOT NULL, -- UPI, AMAZON, FLIPKART, GOOGLE_PLAY
    amount_tokens INTEGER NOT NULL,
    amount_inr DECIMAL(10,2) NOT NULL,
    destination VARCHAR(255) NOT NULL, -- UPI ID or User email format
    gift_card_code VARCHAR(255) DEFAULT NULL,
    status VARCHAR(20) DEFAULT 'PENDING', -- PENDING, APPROVED, REJECTED
    remarks VARCHAR(255) DEFAULT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Table: Daily Ad Logs (limits reset)
CREATE TABLE daily_ad_logs (
    id SERIAL PRIMARY KEY,
    user_id VARCHAR(128) REFERENCES users(uid) ON DELETE CASCADE,
    log_date DATE NOT NULL DEFAULT CURRENT_DATE,
    count INTEGER DEFAULT 1,
    UNIQUE (user_id, log_date)
);

-- Table: Anti-Fraud Threat Logs
CREATE TABLE fraud_logs (
    id SERIAL PRIMARY KEY,
    user_id VARCHAR(128) REFERENCES users(uid) ON DELETE SET NULL,
    threat_type VARCHAR(50) NOT NULL, -- EMULATOR, VPN_PROXY, BOT_SPEED, RATE_LIMIT
    description TEXT NOT NULL,
    device_fingerprint VARCHAR(255) NOT NULL,
    ip_address VARCHAR(45) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Table: System Configurations
CREATE TABLE system_settings (
    id INTEGER PRIMARY KEY CHECK (id = 1),
    reward_per_ad INTEGER NOT NULL DEFAULT 10,
    daily_limit INTEGER NOT NULL DEFAULT 10,
    tokens_per_inr_rate DECIMAL(10,2) NOT NULL DEFAULT 10.00,
    referral_bonus INTEGER NOT NULL DEFAULT 100,
    min_withdrawal_inr INTEGER NOT NULL DEFAULT 5,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Seed System Configuration
INSERT INTO system_settings (id, reward_per_ad, daily_limit, tokens_per_inr_rate, referral_bonus, min_withdrawal_inr) 
VALUES (1, 10, 10, 10.00, 100, 5) ON CONFLICT DO NOTHING;
```

---

## 4. API Endpoints Documentation

### Authentication `/api/v1/auth`
*   `POST /login` -> Verifies Firebase JWT, creates wallet, and yields session key.
*   `POST /otp/send` -> Triggers OTP SMS gateway to Indian phone.
*   `POST /otp/verify` -> Checks code and initializes onboarding.

### Ad Watch Engine `/api/v1/ads`
*   `POST /claim-reward` -> Secures ad completions.
    *   **Body:** `{ "adUnitId": "ca-app-pub...", "integrityToken": "SHA256..." }`
    *   **Anti-Fraud:** Validates IP (not VPN), checks velocity, increases SQL count.

### Rewards & Redemptions `/api/v1/withdrawals`
*   `POST /request` -> Locks balances, creates pending payouts.
    *   **Body:** `{ "channel": "UPI", "upiId": "test@axl", "tokens": 100 }`
*   `GET /history` -> Yields user withdrawal and voucher history.

---

## 5. Security & Anti-Fraud Algorithms

### 5.1 VPN Detection Logic (NodeJS Proxy Shield)
```javascript
const dns = require('dns').promises;

async function checkVPN(ip) {
    // 1. IP Whitelist check or standard Geolocation API check
    const isHostingProvider = await queryIpIntelligence(ip); 
    if (isHostingProvider) {
        throw new Error('VPN_OR_PROXY_NOT_ALLOWED');
    }
    return true;
}
```

### 5.2 Device Fingerprinting Strategy
We enforce native Play Integrity checks on the payload to prevent bot farm scripts from spoofing device parameters. Hardware backing validation ensures one wallet per physics device.

---

## 6. AdMob Rewarded Video Integration Loop (Client Production Hooks)

```kotlin
import com.google.android.gms.ads.AdRequest
import com.google.android.gms.ads.rewarded.RewardedAd
import com.google.android.gms.ads.rewarded.RewardedAdLoadCallback

class RewardAdManager(private val context: Context) {
    private var rewardedAd: RewardedAd? = null

    fun loadAd(adUnitId: String, onLoaded: () -> Unit) {
        val adRequest = AdRequest.Builder().build()
        RewardedAd.load(context, adUnitId, adRequest, object : RewardedAdLoadCallback() {
            override fun onAdLoaded(ad: RewardedAd) {
                rewardedAd = ad
                onLoaded()
            }
        })
    }

    fun showAd(activity: Activity, onRewardEarned: (rewardAmount: Int) -> Unit) {
        rewardedAd?.show(activity) { rewardItem ->
            onRewardEarned(rewardItem.amount)
        }
    }
}
```

---

## 7. Production Deployment & Scalability Playbook

1.  **Database Scalability**: Deploy Cloud Spanner across Indian multi-region nodes (Mumbai first, with backups in Singapore) for ACID compliance at extreme scale.
2.  **Edge Accelerations**: Cloudflare Edge Workers perform JSON web token (JWT) signatures validations directly on the CDN routing, reducing database lookups.
3.  **Ad Fraud Controls**: Correlate Google Play Integrity hardware keys. Any user completing reward triggers at speeds faster than 5 seconds will trigger automatic lockout locks.
