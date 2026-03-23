# CardVault - Product Requirements Document

## 1. Product Overview

### 1.1 Vision
CardVault is a personal card & subscription management PWA that helps users track membership cards, class passes, and subscription services in one beautiful, customizable interface inspired by Apple Wallet.

### 1.2 Target Users
- Primary: Young professionals who manage multiple fitness/dance memberships and digital subscriptions
- Secondary: Anyone with multiple loyalty cards, class passes, or recurring services to track

### 1.3 Core Value Proposition
- **Visual delight**: Customizable card faces with cute/aesthetic designs
- **Never miss a deadline**: Smart expiration reminders
- **Data safety**: Dual-layer storage (localStorage + Google Sheets backup)
- **AI-powered queries**: Natural language card information lookup via DeepSeek API

---

## 2. Feature Specifications

### 2.1 Card Management

#### 2.1.1 Card Types
| Type | Fields | Example |
|------|--------|---------|
| Class Pass | remaining_count, total_count, expiry_date | Dance class: 8/12 classes, expires Apr 15 |
| Membership | status, expiry_date, renewal_cost | Gym membership, active until Dec 31 |
| Subscription | renewal_date, monthly_cost, auto_renew | ChatGPT Plus, $20/mo, renews May 1 |

#### 2.1.2 Card Data Model
```typescript
interface Card {
  id: string;                    // UUID
  type: 'class_pass' | 'membership' | 'subscription';
  name: string;                  // Display name
  category?: string;             // User-defined category (fitness, tech, etc.)
  
  // Visual
  cardImage?: string;            // Base64 or URL of custom card face
  backgroundColor?: string;      // Fallback if no image
  accentColor?: string;          // For UI elements
  
  // Type-specific data
  remainingCount?: number;       // For class passes
  totalCount?: number;           // For class passes
  expiryDate?: string;           // ISO date string
  renewalDate?: string;          // For subscriptions
  cost?: number;                 // Monthly/yearly cost
  currency?: string;             // CNY, USD, HKD, JPY, etc.
  autoRenew?: boolean;           // For subscriptions
  
  // Meta
  notes?: string;
  createdAt: string;
  updatedAt: string;
  isArchived: boolean;
  
  // Sound
  tapSound?: string;             // Sound file reference for tap feedback
}
```

#### 2.1.3 Card Operations
- **Create**: Add new card with type selection → form → image upload
- **Read**: View card stack, tap to expand details
- **Update**: Edit any field, upload new card face
- **Delete**: Soft delete (archive) or hard delete with sync option
- **Restore**: Recover from Google Sheets backup

### 2.2 Visual System

#### 2.2.1 Card Face Customization
- **Image upload**: Accept JPG/PNG/GIF, auto-crop to card aspect ratio (1.6:1)
- **Cropping tool**: Allow user to position/scale image within card frame
- **Fallback**: Solid color + icon if no custom image
- **Border radius**: 16px consistent with Apple Wallet aesthetic

#### 2.2.2 Card Stack Display
```
┌─────────────────────────────────┐
│  [Card 4 - peeking 10px]        │  ← z-index: 1, scale: 0.92, opacity: 0.7
├─────────────────────────────────┤
│  [Card 3 - peeking 40px]        │  ← z-index: 2, scale: 0.96, opacity: 0.85
├─────────────────────────────────┤
│  [Card 2 - peeking 80px]        │  ← z-index: 3, scale: 1, opacity: 1
├─────────────────────────────────┤
│                                 │
│  [Card 1 - Full Display]        │  ← z-index: 4, expanded view
│                                 │
│  Remaining: 8/12                │
│  Expires: Apr 15, 2026          │
│                                 │
└─────────────────────────────────┘
```

#### 2.2.3 Interaction Sounds
- Tap card: Satisfying "click" sound
- Expand card: Soft "whoosh" 
- Add card: Celebration "ding"
- Each card can have custom sound assignment

### 2.3 Data Persistence

#### 2.3.1 Dual-Layer Storage Architecture
```
┌─────────────────────────────────────────────────────────┐
│                     CardVault App                        │
├─────────────────────────────────────────────────────────┤
│                                                          │
│   ┌──────────────────┐    ┌──────────────────────────┐  │
│   │   localStorage   │◄──►│   State Management       │  │
│   │   (Primary)      │    │   (In-memory)            │  │
│   └──────────────────┘    └──────────────────────────┘  │
│            │                         │                   │
│            │ Sync on:                │                   │
│            │ - App open              │                   │
│            │ - Manual sync           │                   │
│            │ - Card CRUD             │                   │
│            ▼                         │                   │
│   ┌──────────────────────────────────┘                  │
│   │                                                      │
│   ▼                                                      │
│   ┌──────────────────────────────────────────────────┐  │
│   │           Google Sheets (Backup)                  │  │
│   │   Sheet: CardVault_Data                           │  │
│   │   - Tab: cards                                    │  │
│   │   - Tab: sync_log                                 │  │
│   │   - Tab: deleted_cards                            │  │
│   └──────────────────────────────────────────────────┘  │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

#### 2.3.2 Sync Operations
| Trigger | Action |
|---------|--------|
| App Launch | Pull from Sheets if localStorage empty OR user chooses "Restore" |
| Card Create/Update | Write to localStorage immediately → queue Sheets sync |
| Card Delete | Remove from localStorage → mark deleted in Sheets (unless user opts for full delete) |
| Manual Sync | Full bidirectional sync with conflict resolution (latest wins) |
| Restore | Pull all non-deleted cards from Sheets → overwrite localStorage |

#### 2.3.3 Google Sheets Structure
**Tab: cards**
| Column | Type | Description |
|--------|------|-------------|
| id | string | UUID |
| data | JSON string | Full card object |
| updated_at | timestamp | Last modification |
| is_deleted | boolean | Soft delete flag |

**Tab: sync_log**
| Column | Type |
|--------|------|
| timestamp | datetime |
| action | string |
| card_id | string |
| device_id | string |

### 2.4 Reminder System

#### 2.4.1 Notification Triggers
| Card Type | Trigger | Message |
|-----------|---------|---------|
| Class Pass | 2 classes remaining | "Only 2 dance classes left! Time to renew?" |
| Class Pass | 7 days before expiry | "Your dance pass expires in 7 days" |
| Membership | 14 days before expiry | "Gym membership expires in 2 weeks" |
| Subscription | 3 days before renewal | "ChatGPT Plus renews in 3 days ($20)" |

#### 2.4.2 Implementation
- PWA Push Notifications (requires user permission)
- In-app notification center
- Visual badges on cards approaching expiry

### 2.5 AI Chat Module

#### 2.5.1 Capabilities
- Query card information: "When does my gym membership expire?"
- Aggregate queries: "How much am I spending on subscriptions monthly?"
- Smart suggestions: "What's expiring this month?"
- Natural language card creation: "Add a new dance class pass, 12 classes, expires June 1"

#### 2.5.2 Integration
```javascript
// DeepSeek API Integration
const AI_CONFIG = {
  endpoint: 'https://api.deepseek.com/v1/chat/completions',
  model: 'deepseek-chat',
  systemPrompt: `You are CardVault assistant. You help users manage their cards and subscriptions.
    Current cards data: {cards_json}
    Respond concisely in the user's language (Chinese/English).`
};
```

### 2.6 PWA Requirements

#### 2.6.1 Manifest
```json
{
  "name": "CardVault",
  "short_name": "CardVault",
  "description": "Manage your cards & subscriptions beautifully",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#F8FAF9",
  "theme_color": "#2DD4A8",
  "icons": [
    { "src": "/icons/192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

#### 2.6.2 Service Worker
- Cache app shell for offline access
- Queue sync operations when offline
- Background sync when connection restored

---

## 3. Technical Architecture

### 3.1 Tech Stack
| Layer | Technology | Rationale |
|-------|------------|-----------|
| Framework | React 18 + Vite | Fast dev, good PWA support |
| Styling | Tailwind CSS | Rapid styling, consistent design system |
| State | Zustand | Lightweight, persist middleware |
| Storage | localStorage + Google Sheets API | Dual-layer redundancy |
| AI | DeepSeek API | Cost-effective, good Chinese support |
| Hosting | Vercel | Easy deployment, good PWA support |
| Sounds | Howler.js | Cross-browser audio |

### 3.2 Project Structure
```
cardvault/
├── public/
│   ├── manifest.json
│   ├── sw.js
│   ├── icons/
│   └── sounds/
├── src/
│   ├── components/
│   │   ├── Card/
│   │   │   ├── CardStack.tsx
│   │   │   ├── CardItem.tsx
│   │   │   ├── CardDetail.tsx
│   │   │   └── CardForm.tsx
│   │   ├── AI/
│   │   │   └── ChatPanel.tsx
│   │   ├── Settings/
│   │   │   ├── SettingsPage.tsx
│   │   │   └── SyncSettings.tsx
│   │   └── common/
│   │       ├── ImageCropper.tsx
│   │       ├── BottomNav.tsx
│   │       └── NotificationBadge.tsx
│   ├── hooks/
│   │   ├── useCards.ts
│   │   ├── useSync.ts
│   │   ├── useNotifications.ts
│   │   └── useAI.ts
│   ├── services/
│   │   ├── storage.ts
│   │   ├── googleSheets.ts
│   │   ├── deepseek.ts
│   │   └── sounds.ts
│   ├── store/
│   │   └── cardStore.ts
│   ├── types/
│   │   └── index.ts
│   ├── utils/
│   │   ├── imageUtils.ts
│   │   ├── dateUtils.ts
│   │   └── syncUtils.ts
│   ├── App.tsx
│   └── main.tsx
├── .env.example
├── package.json
├── vite.config.ts
├── tailwind.config.js
└── vercel.json
```

### 3.3 Key Implementation Notes

#### 3.3.1 Image Cropping
```typescript
// Use react-image-crop for card face cropping
// Aspect ratio locked to 1.6:1 (card dimensions)
// Output as base64 for localStorage, URL for Sheets
```

#### 3.3.2 Sound System
```typescript
// sounds.ts
import { Howl } from 'howler';

const sounds = {
  tap: new Howl({ src: ['/sounds/tap.mp3'] }),
  expand: new Howl({ src: ['/sounds/expand.mp3'] }),
  success: new Howl({ src: ['/sounds/success.mp3'] }),
};

export const playSound = (name: keyof typeof sounds) => {
  sounds[name]?.play();
};
```

#### 3.3.3 Google Sheets Setup
1. Create Google Cloud Project
2. Enable Sheets API
3. Create Service Account
4. Share spreadsheet with service account email
5. Store credentials in Vercel env vars

---

## 4. Design System

### 4.1 Color Palette
```css
:root {
  /* Primary */
  --mint: #2DD4A8;
  --mint-light: #F0FDF9;
  
  /* Background */
  --bg-primary: #F8FAF9;
  --bg-secondary: #EDF4F2;
  --bg-card: #FFFFFF;
  
  /* Text */
  --text-primary: #1A1A1A;
  --text-secondary: #666666;
  --text-muted: #999999;
  
  /* Accents */
  --pink: #E91E63;
  --amber: #FFB300;
  --green: #4CAF50;
  --blue: #2196F3;
  
  /* Status */
  --warning: #FF9800;
  --danger: #F44336;
}
```

### 4.2 Typography
```css
font-family: -apple-system, BlinkMacSystemFont, 'SF Pro Display', 
             'PingFang SC', 'Microsoft YaHei', sans-serif;

/* Scale */
--text-xs: 11px;
--text-sm: 13px;
--text-base: 15px;
--text-lg: 18px;
--text-xl: 24px;
--text-2xl: 28px;
```

### 4.3 Spacing
```css
--space-xs: 4px;
--space-sm: 8px;
--space-md: 12px;
--space-lg: 16px;
--space-xl: 20px;
--space-2xl: 24px;
```

### 4.4 Border Radius
```css
--radius-sm: 8px;
--radius-md: 12px;
--radius-lg: 16px;
--radius-xl: 24px;
--radius-full: 9999px;
```

---

## 5. User Flows

### 5.1 First Launch
```
[App Open] → [Welcome Screen] → [Setup Google Sheets?]
                                       │
                    ┌──────────────────┴──────────────────┐
                    │                                      │
                    ▼                                      ▼
            [Yes - OAuth Flow]                    [Skip - Local Only]
                    │                                      │
                    ▼                                      ▼
            [Connected!]                          [Ready to Use]
                    │                                      │
                    └──────────────┬───────────────────────┘
                                   ▼
                            [Empty State]
                            "Add your first card"
```

### 5.2 Add Card
```
[Tap + Button] → [Select Card Type] → [Fill Form] → [Upload Image?]
                                                          │
                                    ┌─────────────────────┴─────────────────────┐
                                    │                                           │
                                    ▼                                           ▼
                            [Yes - Crop Image]                          [No - Pick Color]
                                    │                                           │
                                    └───────────────┬───────────────────────────┘
                                                    ▼
                                              [Preview Card]
                                                    │
                                                    ▼
                                              [Save] → [Sound Effect] → [Card Added to Stack]
```

### 5.3 Delete Card
```
[Long Press Card] → [Delete Option] → [Confirm Dialog]
                                            │
                        ┌───────────────────┴───────────────────┐
                        │                                       │
                        ▼                                       ▼
                 [Archive Only]                        [Delete from Sheets too]
                        │                                       │
                        ▼                                       ▼
              [Moved to Archive]                    [Permanently Deleted]
                        │
                        ▼
              [Can Restore Later]
```

### 5.4 Restore from Backup
```
[Settings] → [Data & Sync] → [Restore from Sheets]
                                    │
                                    ▼
                            [Confirm Dialog]
                      "This will replace local data"
                                    │
                                    ▼
                            [Fetching...] → [X cards restored!]
```

---

## 6. API Specifications

### 6.1 DeepSeek Chat
```typescript
POST https://api.deepseek.com/v1/chat/completions

Headers:
  Authorization: Bearer {DEEPSEEK_API_KEY}
  Content-Type: application/json

Body:
{
  "model": "deepseek-chat",
  "messages": [
    { "role": "system", "content": "..." },
    { "role": "user", "content": "..." }
  ],
  "temperature": 0.7,
  "max_tokens": 500
}
```

### 6.2 Google Sheets
```typescript
// Using googleapis package
import { google } from 'googleapis';

const sheets = google.sheets({ version: 'v4', auth });

// Read all cards
await sheets.spreadsheets.values.get({
  spreadsheetId: SHEET_ID,
  range: 'cards!A:D',
});

// Append card
await sheets.spreadsheets.values.append({
  spreadsheetId: SHEET_ID,
  range: 'cards!A:D',
  valueInputOption: 'RAW',
  requestBody: { values: [[id, JSON.stringify(card), timestamp, false]] },
});
```

---

## 7. Development Phases

### Phase 1: Core MVP (Week 1-2)
- [ ] Project setup (Vite + React + Tailwind)
- [ ] Card data model & localStorage
- [ ] Card stack UI with animations
- [ ] Add/Edit/Delete card flows
- [ ] Image upload & cropping
- [ ] PWA manifest & basic SW

### Phase 2: Enhanced Features (Week 3)
- [ ] Google Sheets integration
- [ ] Sync logic & conflict resolution
- [ ] Restore from backup flow
- [ ] Sound effects system

### Phase 3: Intelligence (Week 4)
- [ ] DeepSeek AI chat integration
- [ ] Notification system
- [ ] Expiry reminders
- [ ] Settings page

### Phase 4: Polish (Week 5)
- [ ] Animation refinements
- [ ] Performance optimization
- [ ] Error handling & edge cases
- [ ] Testing & bug fixes
- [ ] Vercel deployment

---

## 8. Codex Handoff Notes

### 8.1 Git Setup
```bash
# Initialize repo
cd cardvault
git init
git remote add origin <your-github-repo-url>

# Initial commit
git add .
git commit -m "Initial project structure"
git push -u origin main
```

### 8.2 Development Commands
```bash
# Install dependencies
npm install

# Run dev server
npm run dev

# Build for production
npm run build

# Preview production build
npm run preview

# Deploy to Vercel (after linking)
vercel --prod
```

### 8.3 Environment Variables
```bash
# .env.local
VITE_DEEPSEEK_API_KEY=your_api_key
VITE_GOOGLE_SHEETS_ID=your_sheet_id
VITE_GOOGLE_SERVICE_ACCOUNT=base64_encoded_credentials
```

### 8.4 Key Files to Implement First
1. `src/types/index.ts` - Type definitions
2. `src/store/cardStore.ts` - Zustand store
3. `src/services/storage.ts` - localStorage wrapper
4. `src/components/Card/CardStack.tsx` - Main UI
5. `public/manifest.json` - PWA manifest

---

## 9. Success Metrics

| Metric | Target |
|--------|--------|
| Time to add card | < 30 seconds |
| App load time | < 2 seconds |
| Sync reliability | 99.9% |
| PWA install rate | > 50% of users |

---

## 10. Future Considerations
- Multi-device sync via account system
- Widget for iOS/Android home screen
- Share card templates with friends
- Analytics dashboard (spending trends)
- Barcode/QR code scanning for physical cards
