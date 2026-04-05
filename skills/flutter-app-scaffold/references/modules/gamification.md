# Gamification Module

Points, achievements, and streaks to drive user engagement. The specific achievements and point values should be tailored to the app's core actions.

## Files to Create

```
lib/features/gamification/
├── data/
│   └── repositories/
│       └── gamification_repository.dart
├── domain/
│   └── models/
│       ├── user_stats.dart           # + generated files
│       ├── achievement.dart          # + generated files
│       └── point_transaction.dart    # + generated files
└── presentation/
    ├── providers/
    │   └── gamification_providers.dart
    ├── screens/
    │   └── achievements_screen.dart
    └── widgets/
        ├── points_display.dart
        ├── achievement_card.dart
        ├── streak_indicator.dart
        └── level_progress_bar.dart
```

## Database Tables

### user_stats
```sql
CREATE TABLE public.user_stats (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE UNIQUE NOT NULL,
    total_points INTEGER DEFAULT 0 NOT NULL,
    level INTEGER DEFAULT 1 NOT NULL,
    current_streak INTEGER DEFAULT 0 NOT NULL,
    longest_streak INTEGER DEFAULT 0 NOT NULL,
    last_active_date DATE,
    created_at TIMESTAMPTZ DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT NOW() NOT NULL
);
```

### point_transactions
```sql
CREATE TABLE public.point_transactions (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    points INTEGER NOT NULL,
    action TEXT NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW() NOT NULL
);
```

### achievements (reference table + user junction)
```sql
CREATE TABLE public.achievements (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT NOT NULL,
    icon TEXT NOT NULL,           -- emoji or icon name
    points_required INTEGER,     -- points needed to unlock (null if action-based)
    action_type TEXT,            -- action that unlocks this (null if points-based)
    action_count INTEGER,        -- how many times the action must be performed
    tier TEXT DEFAULT 'bronze'   -- bronze, silver, gold, platinum
);

CREATE TABLE public.user_achievements (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
    achievement_id UUID REFERENCES public.achievements(id) NOT NULL,
    unlocked_at TIMESTAMPTZ DEFAULT NOW() NOT NULL,
    UNIQUE(user_id, achievement_id)
);
```

## Designing Points & Achievements

During implementation, design the point system around the app's core actions. The goal is to reward behaviors that make the app more valuable to the user.

### Point Value Guidelines

| Action Frequency | Point Value | Examples |
|-----------------|-------------|---------|
| Multiple times daily | 5-10 pts | Logging an item, completing a task |
| Daily | 15-25 pts | Daily check-in, streak maintenance |
| Weekly | 50-100 pts | Completing a weekly goal, sharing content |
| One-time | 100-500 pts | Completing onboarding, first purchase, inviting a friend |

### Level Curve

```dart
// Points needed for each level — exponential curve
static int pointsForLevel(int level) => (level * level * 100);

// Level 1: 100 pts, Level 2: 400 pts, Level 5: 2500 pts, Level 10: 10000 pts
```

### Achievement Categories

Design 15-25 achievements across these categories:
- **Getting started**: First actions (first item created, first share, profile completed)
- **Consistency**: Streak-based (3-day streak, 7-day, 30-day)
- **Volume**: Quantity-based (10 items, 50 items, 100 items)
- **Social**: Community actions (first invite, 5 friends joined)
- **Mastery**: Advanced features (used every feature, power user)

## Integration Pattern

The gamification controller should be called from other features when point-worthy actions occur:

```dart
// In any feature's controller, after a successful action:
ref.read(gamificationControllerProvider.notifier).awardPoints(
  action: 'create_item',
  points: 10,
  description: 'Created a new item',
);
```

### RPC Function for Awarding Points

```sql
CREATE OR REPLACE FUNCTION award_points(
    p_action TEXT,
    p_points INTEGER,
    p_description TEXT DEFAULT NULL
)
RETURNS JSONB AS $$
DECLARE
    v_user_id UUID := auth.uid();
    v_new_total INTEGER;
    v_new_level INTEGER;
    v_new_achievements JSONB := '[]'::jsonb;
BEGIN
    -- Record transaction
    INSERT INTO point_transactions (user_id, points, action, description)
    VALUES (v_user_id, p_points, p_action, p_description);

    -- Update total points
    UPDATE user_stats
    SET total_points = total_points + p_points,
        updated_at = NOW()
    WHERE user_id = v_user_id
    RETURNING total_points INTO v_new_total;

    -- Calculate new level
    v_new_level := floor(sqrt(v_new_total / 100.0))::integer;
    IF v_new_level < 1 THEN v_new_level := 1; END IF;

    UPDATE user_stats SET level = v_new_level WHERE user_id = v_user_id;

    -- Check for new achievements (action-based)
    -- ... check and unlock achievements based on action counts

    RETURN jsonb_build_object(
        'new_total', v_new_total,
        'new_level', v_new_level,
        'new_achievements', v_new_achievements
    );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

## Streak Logic

Update streaks on daily app usage:

```sql
CREATE OR REPLACE FUNCTION update_daily_streak()
RETURNS VOID AS $$
DECLARE
    v_last_active DATE;
    v_today DATE := CURRENT_DATE;
BEGIN
    SELECT last_active_date INTO v_last_active
    FROM user_stats WHERE user_id = auth.uid();

    IF v_last_active = v_today THEN
        RETURN; -- Already counted today
    ELSIF v_last_active = v_today - INTERVAL '1 day' THEN
        -- Consecutive day: increment streak
        UPDATE user_stats SET
            current_streak = current_streak + 1,
            longest_streak = GREATEST(longest_streak, current_streak + 1),
            last_active_date = v_today
        WHERE user_id = auth.uid();
    ELSE
        -- Streak broken: reset to 1
        UPDATE user_stats SET
            current_streak = 1,
            last_active_date = v_today
        WHERE user_id = auth.uid();
    END IF;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```
