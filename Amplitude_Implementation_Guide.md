# 🔍 Amplitude Implementation Guide
## Как отслеживать сегменты и поведение в Amplitude

---

## ЧАСТЬ 1: SETUP EVENTS ДЛЯ SEGMENTATION

### Шаг 1: Создать User Property "user_segment"
В Amplitude → Analytics → User Properties → Create custom property

**Property Name**: `user_segment`  
**Property Type**: String  
**Values**: `scalper`, `swing_trader`, `copier`, `learner`, `passive`

**Логика классификации** (впишите в свой backend код):

```python
def classify_user_segment(user_data):
    """
    На основе первых действий классифицировать пользователя
    Запустить на день 3-5 после регистрации
    """
    
    avg_trades_per_day = user_data['trades_count'] / max(user_data['days_active'], 1)
    session_duration = user_data['avg_session_minutes']
    has_api_calls = user_data['api_requests_count'] > 0
    copy_trading_enabled = user_data['is_copying']
    completed_tutorial = user_data['tutorial_completed']
    
    # SCALPER: много сделок, короткие сеансы, интерес к API
    if avg_trades_per_day > 10 and session_duration <= 60 and has_api_calls:
        return 'scalper'
    
    # SWING: среднее количество сделок, интерес к анализу и новостям
    elif 2 <= avg_trades_per_day <= 10 and session_duration > 45:
        return 'swing_trader'
    
    # COPIER: копирует сделки, активен в чате, мало собственных сделок
    elif copy_trading_enabled and avg_trades_per_day < 2:
        return 'copier'
    
    # LEARNER: завершил обучение, низкая активность, вопросы в чате
    elif completed_tutorial and avg_trades_per_day < 1 and not copy_trading_enabled:
        return 'learner'
    
    # PASSIVE: минимальная активность, редкие визиты
    elif avg_trades_per_day == 0 and session_duration < 10:
        return 'passive'
    
    else:
        return 'undefined'
```

---

### Шаг 2: Отправлять события из вашего приложения

Убедитесь, что ваше приложение отправляет эти события в Amplitude:

#### Основные события (уже есть?):
```
✅ accountCreated
✅ AddFunds
✅ BrokerSelection
✅ ClosePositions
✅ CopyPositions / CopyPositionDetails
✅ copyingStarted / copyingStopped / copyingRestarted
✅ blotterOpened / blotterFilterApplied
✅ chartTypeChanged
✅ chartTutorialStarted / Finished / Skipped
✅ ChartOptionList / ChartDrawingsBottomSheet
✅ Cold start
✅ 15SecInApp
✅ chatOpened / chatMessageSent
✅ AlertsSettings / AssetsSettings / AccountMetricsSettingsPage
```

#### Добавить эти события (НОВЫЕ):
```
❌ user_segment_identified (отправлять на день 3-5)
    {
      "segment": "scalper",
      "confidence": 0.95,
      "reasoning": "20+ trades/day, 45min session, API calls"
    }

❌ segment_migration (когда юзер переходит в другой сегмент)
    {
      "from_segment": "learner",
      "to_segment": "swing_trader",
      "days_in_previous": 47
    }

❌ execution_latency (для scalper)
    {
      "latency_ms": 47,
      "pair": "EUR/USD",
      "order_type": "market"
    }

❌ profitability (ежедневно, для каждого юзера)
    {
      "pnl_usd": 125.50,
      "pnl_percent": 2.5,
      "trades_today": 15,
      "winning_trades": 10
    }

❌ chart_setup_time (сколько минут настраивает график при открытии)
    {
      "time_minutes": 8,
      "indicators_added": 5,
      "custom_drawings": 2
    }

❌ session_quality_score (оценка сеанса)
    {
      "score": 7.5,
      "factors": {
        "profitability": 1.0,
        "risk_management": 0.8,
        "trade_count": 0.9
      }
    }

❌ tutorial_progress (для learner)
    {
      "module": "technical_analysis",
      "progress_percent": 65,
      "time_spent_minutes": 45,
      "completed": false
    }

❌ support_ticket (для learner - высокий volume)
    {
      "category": "how_to_trade",
      "response_time_hours": 2.5,
      "resolved": true
    }

❌ churn_risk_signal (для платформы, определить риск-юзеров)
    {
      "risk_score": 0.85,
      "signals": [
        "no_activity_14_days",
        "last_month_loss_20_percent",
        "decreased_session_duration"
      ]
    }
```

---

## ЧАСТЬ 2: DASHBOARDS В AMPLITUDE

### Dashboard 1: SEGMENT OVERVIEW (Главный дашборд)

**Метрики по сегментам** (матрица):

|  | Scalper | Swing | Copier | Learner | Passive |
|--|---------|-------|--------|---------|---------|
| **Users** | # | # | # | # | # |
| **% of Total** | 15% | 25% | 30% | 20% | 10% |
| **DAU** | # | # | # | # | # |
| **MAU** | # | # | # | # | # |
| **Day 1 Retention** | % | % | % | % | % |
| **Day 7 Retention** | % | % | % | % | % |
| **Day 30 Retention** | % | % | % | % | % |
| **Avg Trades/Day** | # | # | # | # | # |
| **Avg Session Min** | # | # | # | # | # |
| **Monthly LTV** | $ | $ | $ | $ | $ |
| **Churn Rate** | % | % | % | % | % |

**Как создать в Amplitude**:
1. Retention → Event Segmentation
2. Slice by: `user_segment`
3. Выбрать события: `accountCreated`, `15SecInApp`, etc.

---

### Dashboard 2: SCALPER PERFORMANCE

**Цель**: Убедиться, что scalper-опыт оптимален

```
Row 1:
  [Execution Latency]  [Spread Average]  [Daily Trade Volume]  [Platform Uptime]
  47ms                 0.8 pips          2.5M trades/day       99.87%

Row 2: Time series
  [Execution Latency Trend (30d)]
  Line chart: latency over time (должна быть ≤ 50ms)

Row 3: Time series
  [Trades Closed per Hour]
  Bar chart: распределение сделок по часам (пики активности)

Row 4:
  [Top Scalpers by Volume]  [Top Scalpers by Profit]
  Table: юзеры, кол-во сделок  Table: юзеры, прибыль
```

**Events to track**:
- `execution_latency`
- `profitability`
- `daily_trades`
- `spread_feedback` (если scalper оставляет отзыв о спредах)

---

### Dashboard 3: SWING TRADER ENGAGEMENT

**Цель**: Убедиться, что контент и анализ на уровне

```
Row 1:
  [Avg Chart Setup Time]  [Indicators Used]  [Webinar Attendance]  [Analysis Time %]
  8 min                   4.2                67%                   35%

Row 2: Time series
  [Chart Analysis Time Trend (30d)]
  Area chart: время на анализ (должно расти или быть стабильным)

Row 3: Segmentation
  [Sentiment: Trading Ideas]
  Pie chart: юзеры, которые скопировали идею из feed (%)

Row 4:
  [Top Performers This Month]
  Table: swing traders с лучшей прибылью (для motivation)
```

**Events to track**:
- `chartTutorialStarted / Finished / Skipped`
- `blotterOpened`
- `CopyPositionDetails` (смотрел идеи других)
- `profitability`
- `webinar_attended`

---

### Dashboard 4: COPIER SOCIAL SIGNALS

**Цель**: Убедиться, что копировщики находят хороших сигнальщиков

```
Row 1:
  [Top Signal Providers]    [New Copiers This Week]  [Avg Copiers per Signal]  [Churn After Loss]
  [Ranked list, 10]         # (trend)                #                        %

Row 2: Time series
  [Signal Provider Attrition]
  Line chart: % юзеров, покидающих сигнальщика в неделю (после убытков)

Row 3: Funnel
  [Copy Trading Funnel]
  Browse → Click → Copy → Hold Position (min 1 week) → Success/Churn
  [48%] → [35%] → [20%] → [12%] → [7%]

Row 4:
  [Chat Activity]           [Community Growth]
  # of messages/week        # of active users
```

**Events to track**:
- `CopyPositionDetails`
- `copyingStarted`
- `copyingStopped / copyingRestarted`
- `chatMessageSent`
- `profitability`
- `time_to_signal_change` (как долго держит одного сигнальщика?)

---

### Dashboard 5: LEARNER PROGRESSION

**Цель**: Отслеживать обучение и dropout

```
Row 1:
  [Tutorial Completion %]  [Demo Trades Avg]  [Time to 1st Real Trade]  [Support Tickets/User]
  42%                      45                 12 days                   8.3

Row 2: Funnel
  [Learner Progression]
  Registered → Completed Tutorial → Demo Trading → 1st Real Trade → Retention D30
  [100%] → [42%] → [28%] → [8%] → [3%]
  
  (Bottleneck: только 28% пробуют демо!)

Row 3: Cohort
  [Cohort Analysis: Tutorial Completion]
  Week 1: 40% completions
  Week 2: 35% completions
  Week 3: 32% completions
  (Trend: улучшается?)

Row 4:
  [Support Response Time]  [Mentorship Signups]
  Avg: 2.5 hours          # of learners with mentor
```

**Events to track**:
- `chartTutorialStarted / Finished / Skipped`
- `tutorial_progress` (по модулям)
- `AddFunds` (первое пополнение)
- `ClosePositions` (первая реальная позиция)
- `chatOpened` / `support_ticket`
- `mentorship_signup`

---

### Dashboard 6: PASSIVE REACTIVATION

**Цель**: Минимизировать churn, отслеживать переход в active

```
Row 1:
  [Active Users]           [Churned (30d)]      [Reactivated (30d)]    [LTV]
  20%                      75%                  5%                     $1.50

Row 2: Time series
  [Inactivity Curve]
  % of users active by days since last login (30d lookback)

Row 3: Reactivation Funnel
  Sent Email → Opened → Clicked → Logged In → Made Deposit → Trading
  [100%] → [22%] → [8%] → [5%] → [1%] → [0.2%]

Row 4:
  [Referral Program Performance]
  Referrals sent: 2.1k
  Conversions: 8%
  Revenue from referrals: $5,200
```

**Events to track**:
- `Cold start` (время последнего визита)
- `15SecInApp` (сессии)
- `reactivation_email_sent` (ваш маркетинг)
- `AddFunds`
- `referral_link_clicked`
- `referral_signup`

---

## ЧАСТЬ 3: ALERTS И MONITORING

### Установить alerts в Amplitude для экстренных ситуаций:

```
Alert 1: SCALPER EXECUTION LATENCY
  Condition: execution_latency > 100ms for 60 minutes
  Action: Notify engineering team (критично!)
  
Alert 2: COPIER CHURN SPIKE
  Condition: copyingStopped rate > 15% daily
  Action: Notify product team
  Alert text: "High copier churn detected - check signal provider performance"
  
Alert 3: LEARNER DROPOUT
  Condition: tutorial_completed < 35% in rolling 7 days
  Action: Notify education team
  
Alert 4: PASSIVE INACTIVE
  Condition: % of users with 0 activity last 30d > 60%
  Action: Notify retention team
  Alert text: "Passive segment churn accelerating - trigger reactivation campaign"
```

---

## ЧАСТЬ 4: WEEKLY REPORTING TEMPLATE

Каждый понедельник отправлять отчет маркетолога со следующими метриками:

```
📊 FOREX APP - WEEKLY MARKETING REPORT
Date: April 7-13, 2026
Prepared by: [Name]

═══════════════════════════════════════════════════════════════

📈 KEY METRICS (Week vs Previous Week)
   Total Users:           12,450 (+3% WoW)
   New Signups:           850 (+12% WoW)
   DAU:                   4,200 (-2% WoW)
   Day-7 Retention:       42% (-1pp WoW)
   Day-30 Retention:      28% (stable)
   Churn Rate (Weekly):   8.5% (+0.3pp WoW)

═══════════════════════════════════════════════════════════════

🎯 SEGMENT PERFORMANCE

SCALPER (15% of users, 42% of revenue):
  ├─ Users: 1,867 (+2% WoW)
  ├─ DAU: 1,850 (99% engaged!)
  ├─ Avg Trades/Day: 28 (+1 WoW)
  ├─ Execution Latency: 47ms (target: <50ms) ✅
  ├─ Churn Rate: 1.2% (healthy)
  ├─ LTV (MTD): $2,450
  └─ Note: Stable, no issues

SWING TRADER (25% of users, 35% of revenue):
  ├─ Users: 3,112 (+4% WoW)
  ├─ DAU: 2,200 (71% engaged)
  ├─ Avg Trades/Day: 3.5 (stable)
  ├─ Webinar Attendance: 67% (last week: 65%)
  ├─ Chart Analysis Time: 38 min avg
  ├─ Churn Rate: 6.2% (-0.5pp, improving!)
  ├─ LTV (MTD): $450
  └─ Note: Strong engagement with educational content

COPIER (30% of users, 20% of revenue):
  ├─ Users: 3,735 (+5% WoW)
  ├─ DAU: 450 (12% engaged - low!)
  ├─ Avg Copiers per Signal Provider: 2.1
  ├─ Copy → Hold > 1 week: 12% (target: 20%)
  ├─ Chat Activity: 2.3k messages (stable)
  ├─ Churn Rate: 18% (+2pp, degrading)
  ├─ LTV (MTD): $120
  └─ Note: ⚠️ CHURN RISING - check top signal provider performance

LEARNER (20% of users, 3% of revenue):
  ├─ Users: 2,490 (+2% WoW)
  ├─ DAU: 220 (9% engaged - very low)
  ├─ Tutorial Completion: 42% (target: >50%)
  ├─ Demo to Real Trade: 8% conversion
  ├─ Support Tickets: 1,250 (avg response: 2.2h)
  ├─ Churn Rate: 42% (expected for beginners)
  ├─ LTV (MTD): $28
  └─ Note: Good tutorial engagement, but dropout after tutorial

PASSIVE (10% of users, <1% of revenue):
  ├─ Users: 1,246 (-1% WoW)
  ├─ DAU: 80 (6% engaged)
  ├─ Session Duration: 6 min avg
  ├─ Activities: Almost none
  ├─ Reactivation Email CTR: 3.2%
  ├─ Churn Rate: 65% (expected)
  ├─ LTV (MTD): $2
  └─ Note: As expected, minimal activity

═══════════════════════════════════════════════════════════════

⚡ CRITICAL ALERTS THIS WEEK:
  🔴 COPIER CHURN SPIKE: +2pp week-over-week
     Action: Check top 5 signal providers' last week performance
     Hypothesis: May be market volatility (FOMC) affecting results
     
  🟡 LEARNER TUTORIAL DROPOUT: 58% don't complete
     Action: Review tutorial for complexity / drop off points
     Next: A/B test shorter tutorial (15 min vs 25 min)

═══════════════════════════════════════════════════════════════

📱 MARKETING CAMPAIGNS THIS WEEK:

Campaign 1: "Best Execution for Scalpers" (Paid)
  ├─ Spend: $2,500
  ├─ Impressions: 450k
  ├─ Clicks: 12.3k (CTR: 2.7%)
  ├─ Conversions: 340 new scalpers
  ├─ CPA: $7.35
  └─ Status: ✅ Profitable (LTV $2450 >> CPA $7)

Campaign 2: "Copy Trading Made Easy" (Organic)
  ├─ Reach: 85k (Instagram, TikTok)
  ├─ Engagements: 12.4k
  ├─ Conversions: 85 new copiers
  ├─ CPA: $0 (organic)
  └─ Status: ✅ Best ROI

Campaign 3: "Forex for Beginners" (YouTube)
  ├─ Views: 25k (3 videos)
  ├─ Watch time: 12.5h
  ├─ Conversions: 120 new learners
  ├─ CPA: ~$0
  └─ Status: ✅ Soft acquisition

═══════════════════════════════════════════════════════════════

🎬 RECOMMENDATIONS NEXT WEEK:

1. URGENT: Investigate Copier Churn
   → Check signal providers' ROI last 7 days
   → If lower, send retention email: "Try different signal provider"
   
2. OPTIMIZE: Learner Tutorial
   → A/B test: Current 25-min version vs New 15-min version
   → Metric: % completing tutorial
   → Target: Increase from 42% → 55%
   
3. SCALE: Scalper Campaigns
   → CPA is $7, LTV is $2,450 = 350x ROAS
   → Increase spend from $2.5k → $5k/week
   
4. TEST: Passive Reactivation
   → Send 2 cohorts different emails:
   → A: "Get 5% interest on balance" 
   → B: "Invite friend, both get $50"
   → Track: Click rate, deposit rate, ROI

═══════════════════════════════════════════════════════════════

Budget Allocation Recommendation (Next Month):
  Scalpers: 50% ($50k/month) - Best ROI
  Swing Traders: 25% ($25k/month) - Educational content
  Copiers: 15% ($15k/month) - Social media
  Learners: 8% ($8k/month) - YouTube, tutorials
  Passive: 2% ($2k/month) - Minimal

═══════════════════════════════════════════════════════════════

Questions / Issues:
- Why DAU dropped 2% despite +3% new users? (Investigation needed)
- Is execution latency (47ms) competitive vs. [Competitor]? (Check weekly)
```

---

## ЧАСТЬ 5: QUICK START CHECKLIST

Что нужно сделать в следующие 7 дней:

- [ ] **День 1**: Встреча с техническим директором
  - Обсудить, какие события уже отправляются в Amplitude
  - Договориться о добавлении новых событий (execution_latency, profitability, etc.)

- [ ] **День 2-3**: Настроить в Amplitude
  - Создать user property `user_segment` (enum: scalper, swing, copier, learner, passive)
  - Создать 6 dashboards (один на сегмент)
  - Настроить alerts (copier churn, execution latency)

- [ ] **День 4-5**: Внедрить logic классификации сегментов
  - На день 3-5 после регистрации, отправить event `user_segment_identified`
  - Проверить, что правильно классифицирует юзеров

- [ ] **День 6**: Pullить первые данные
  - Посмотреть, сколько юзеров в каждом сегменте
  - Проверить, что числа близко к 15%, 25%, 30%, 20%, 10%

- [ ] **День 7**: Первый отчет
  - Подготовить недельный отчет по шаблону выше
  - Отправить стейкхолдерам

---

## Итог

С этой setup-ой вы сможете:
1. ✅ Отслеживать поведение каждого сегмента в реальном времени
2. ✅ Идентифицировать, когда юзер "требует внимания" (риск churn)
3. ✅ Измерять ROI маркетинга по сегментам
4. ✅ Оптимизировать messaging для каждого сегмента
5. ✅ Растить LTV каждого сегмента отдельно

**Последний совет**: Используйте эти данные еженедельно. Не собирайте отчеты один раз в месяц - это слишком поздно для оптимизации.

Happy tracking! 🚀

