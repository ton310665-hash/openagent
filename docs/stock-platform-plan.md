# AI Stock Analysis Platform — ผลสำรวจและแผนพัฒนา

สถานะ: **ยังไม่เริ่มเขียนโค้ด** เพราะ sandbox ที่ใช้พัฒนาถูกบล็อกเครือข่าย (ดูหัวข้อ "ปัญหาที่ติดอยู่")

## 1. ผลสำรวจโปรเจกต์เดิม (openagent)

| ส่วน | ที่มีอยู่แล้ว | นำมาใช้ต่อได้ไหม |
|---|---|---|
| Backend | Go + Beego, รูปแบบ `object/` → `controllers/` → `routers/router.go` | ใช้ต่อ เพิ่ม entity หุ้นตามรูปแบบเดิม |
| Database | xorm, รองรับ MySQL / PostgreSQL (`lib/pq`), migrate อัตโนมัติใน `object/adapter.go` | ใช้ต่อ |
| ระบบผู้ใช้ | Casdoor (OAuth), `RequireSignedIn()`, `IsAdmin()` | ใช้ต่อ ไม่ต้องสร้างระบบ login ใหม่ |
| AI | `model/` มี provider ~30 ราย (Claude, OpenAI, Gemini, DeepSeek ฯลฯ) จัดการผ่านหน้า Provider, API key อยู่ฝั่ง server | ใช้ต่อสำหรับ AI Analysis |
| Frontend | React 18 (JS ไม่ใช่ TS), Ant Design 6, ECharts 5, i18next (en/zh) | ใช้ต่อ — ECharts มี candlestick ในตัว |
| Usage/ค่าใช้จ่าย | มีระบบ usage และราคาโมเดล | ใช้จำกัดค่า AI ได้ |

**ยังไม่มี:** ข้อมูลตลาด, ตัวชี้วัดทางเทคนิค, watchlist, portfolio, risk calculator, screener, alerts, backtest, ภาษาไทย

**ข้อเสนอ:** สร้างเป็นโมดูล "Stocks" ในแอปเดิม แทนการเขียนแอปใหม่ ใช้ JS ตามโค้ดเดิม (การย้ายไป TypeScript ทั้งโปรเจกต์ใหญ่เกินกว่าคุ้ม)

## 2. ปัญหาที่ติดอยู่

| สิ่งที่ต้องใช้ | Host | สถานะ |
|---|---|---|
| Go toolchain 1.25.8 + Go modules | `proxy.golang.org` | DNS ไม่ resolve / 403 (เครื่องมี Go 1.24.7 แต่ go.mod ต้องการ 1.25) |
| npm packages (frontend) | `registry.npmjs.org` | DNS ไม่ resolve |
| ข้อมูลหุ้น | `query1/query2.finance.yahoo.com`, `stooq.com`, `alphavantage.co`, `finnhub.io` | 403 จาก network policy |

ผลคือ build backend, build frontend และดึงข้อมูลตลาดจริงไม่ได้ ตามสเปกข้อ 6 และ 10 จึงไม่ควรเขียนโค้ดที่ทดสอบไม่ได้หรือใช้ข้อมูลจำลอง

## 3. แหล่งข้อมูลตลาด (ต้องเลือก)

| ผู้ให้บริการ | หุ้นไทย (SET) | หุ้นสหรัฐฯ | ค่าใช้จ่าย | หมายเหตุ |
|---|---|---|---|---|
| Yahoo Finance (ไม่เป็นทางการ) | ✅ (`PTT.BK`) | ✅ | ฟรี | ไม่มีเอกสาร/SLA, ข้อมูล delayed, ข้อกำหนดการใช้งานไม่ชัดสำหรับเชิงพาณิชย์ |
| Twelve Data | ✅ (แพ็กเกจจ่ายเงิน) | ✅ | ฟรี 800 req/วัน, จ่ายเงินสำหรับหลายตลาด | มีเอกสารชัดเจน |
| Finnhub | จำกัด | ✅ | ฟรี 60 req/นาที | ข่าวและงบการเงินดี |
| Alpha Vantage | จำกัด | ✅ | ฟรี 25 req/วัน | น้อยเกินไปสำหรับ screener |

แนะนำ: ออกแบบชั้น `MarketDataProvider` ให้สลับผู้ให้บริการได้ เริ่มจาก **Twelve Data หรือ Finnhub** (มีเอกสารและ API key) และใช้ Yahoo เป็นตัวเลือกสำหรับใช้ส่วนตัวเท่านั้น

## 4. แผน Phase 1 (MVP)

Backend (Go)
1. `stock/` — interface `MarketDataProvider` (search, quote, candles, news) + adapter ผู้ให้บริการที่เลือก, cache ใน DB, จัดการ rate limit, ติดป้าย Real-time / Delayed / EOD และเวลาอัปเดต
2. `stock/indicator/` — SMA, EMA, RSI (Wilder), MACD, Bollinger, ATR, volume ratio, แนวรับ/แนวต้าน (pivot) พร้อม unit test เทียบค่าอ้างอิง
3. `object/stock_watchlist.go` — watchlist แยกตามผู้ใช้ (owner = user)
4. `object/stock_analysis.go` — เก็บรายงาน AI: ส่ง quote + indicator ที่คำนวณแล้วให้โมเดลผ่าน provider เดิม, บังคับ JSON (fact vs AI opinion, Bullish/Neutral/Bearish, "ประเมินไม่ได้" ถ้าข้อมูลไม่พอ), จำกัดจำนวนครั้งต่อผู้ใช้ต่อวัน
5. Risk calculator — คำนวณฝั่ง server + unit test (position size = risk budget ÷ |entry − stop|, R:R, % ของทุน)

Frontend (React)
6. เมนู Stocks: Dashboard, ค้นหา, หน้าหุ้น (candlestick + volume + indicators ด้วย ECharts), Watchlist, Risk Calculator, AI report
7. locale ภาษาไทย `web/src/locales/th/data.json`, คำอธิบายศัพท์สำหรับมือใหม่, loading/empty/error state ทุกหน้า, responsive

Phase 2–3 ตามสเปก (Portfolio, News/Fundamentals, Alerts, Screener, Backtest) ทำหลัง MVP ผ่านการทดสอบ

## 5. สิ่งที่ต้องได้จากเจ้าของโปรเจกต์

1. เปิด Network access ให้ host: `proxy.golang.org`, `sum.golang.org`, `registry.npmjs.org` และ host ของผู้ให้บริการข้อมูลที่เลือก
2. เลือกผู้ให้บริการข้อมูลตลาด และตลาดที่ต้องการ (ไทย / สหรัฐฯ)
3. API key ของผู้ให้บริการข้อมูล (ตั้งเป็น secret ของ environment ไม่ใส่ในโค้ด)
4. AI provider ที่จะใช้ (ตั้งค่าผ่านหน้า Provider ที่มีอยู่แล้ว)
