# Weight Loss Timeline Calculator - Product Requirements and Technical Documentation

## 1. Purpose
Build a single-page calculator that estimates
- average daily energy deficit
- estimated days to reach a target weight
- projected weight over time
- contribution of steps, strength training, cardio, and monthly over-limit days

The app supports two calculation modes
- Quick Average - closed-form estimate using average weight
- Weekly Simulation - day-by-day simulation with optional weight-dependent scaling

Everything in UI and copy must be in English.

## 2. Target deliverable
- Output as one self-contained HTML file (index.html)
- Mobile-first responsive UI
- Runs locally in a browser without build steps

## 3. Inputs, variables, and units

### 3.1 Core inputs
C - Daily intake on normal days, kcal/day  
B - Baseline daily burn for sedentary lifestyle, kcal/day  
S - Strength workouts per week, count/week  
K - Cardio workouts per week, count/week  
P - Average steps per day, steps/day  
O - Over-limit days per month, days/month  
W0 - Current weight, kg  
Wt - Target weight, kg  

### 3.2 Advanced inputs and defaults
P0 - Baseline steps already included in B, default 5000 steps/day  
deltaOver - Extra calories on an over-limit day, default 2000 kcal/day  
kcalPerKg - Energy per 1 kg weight change, default 7700 kcal/kg  

Strength session assumptions  
METstr default 5  
MinStr default 60 minutes  

Cardio session assumptions  
METcar default 7  
MinCar default 30 minutes  

Steps coefficient  
stepCoef default 0.375  
Meaning kcal/day = stepCoef * weight(kg) * (stepsDelta/1000)

Calculation method  
Method in { QuickAvg, WeeklySim }

WeeklySim configuration  
MaxDays default 2000  
B behavior
- LockB - keep B constant across time
- ScaleBByWeight - scale B proportionally to current weight, Bscaled = B * (W/W0)
Over-limit modeling
- AverageMonthly - add average over-limit kcal as constant daily add-on
- CalendarDays - model explicit over-limit days in a month

### 3.3 Optional Auto baseline B
If the user does not know B, provide an Auto Baseline option.

Required inputs for Auto Baseline  
Sex in { Male, Female }  
Age, years  
Height, cm  
ActivityBase default 1.2

Mifflin St Jeor
Male BMR = 10*W + 6.25*Height - 5*Age + 5
Female BMR = 10*W + 6.25*Height - 5*Age - 161

Baseline burn
B = ActivityBase * BMR

In QuickAvg use Wavg, in WeeklySim use current W.

## 4. Calculation logic

### 4.1 Derived values
Average weight
Wavg = (W0 + Wt) / 2

Total required deficit
Kneed = kcalPerKg * (W0 - Wt)

Daily average extra intake from over-limit days
Cover = (deltaOver * O) / 30

Average daily intake
Cavg = C + Cover

### 4.2 Steps energy adjustment
Assume B already includes P0 steps.

Esteps = stepCoef * Wref * (P - P0) / 1000

Wref
- QuickAvg -> Wavg
- WeeklySim -> current W

Note
If P < P0, Esteps is negative.

### 4.3 Training energy using MET
Energy for one session
Esess(MET, minutes, Wref) = MET * 3.5 * Wref / 200 * minutes

Daily average from strength
Estr = (S / 7) * Esess(METstr, MinStr, Wref)

Daily average from cardio
Ecar = (K / 7) * Esess(METcar, MinCar, Wref)

### 4.4 Daily burn and deficit
Average burn
TDEEavg = B + Esteps + Estr + Ecar

Average deficit
Def = TDEEavg - Cavg

### 4.5 Days to target, QuickAvg
If Def <= 0
- status NoProgress
- do not return a finite days value
- show actionable suggestions

Else
Days = Kneed / Def
Display as ceiling integer.

### 4.6 WeeklySim
Goal - produce time series and better realism under weight change.

Pseudo
W = W0
day = 0
while W > Wt and day < MaxDays
  determine Bday
    if AutoBaseline enabled -> compute from W and user demographics
    else if LockB -> Bday = B
    else if ScaleBByWeight -> Bday = B * (W / W0)

  Wref = W
  compute Esteps, Estr, Ecar
  compute Cday
    if AverageMonthly -> Cday = C + (deltaOver*O)/30
    if CalendarDays -> Cday = C + deltaOver on selected days else C

  TDEEday = Bday + Esteps + Estr + Ecar
  DefDay = TDEEday - Cday
  dW = DefDay / kcalPerKg
  W = W - dW
  record series
  day += 1

Outputs
- Days = day
- WeightSeries by day
- TDEESeries by day
- IntakeSeries by day
- DefSeries by day

Stop conditions
- day reaches MaxDays -> status MaxDaysReached

## 5. Validation and guardrails
Hard validation errors
- Wt must be less than W0
- Required numeric fields must be valid numbers and within plausible bounds

Suggested bounds
C 0..10000
B 0..10000
S 0..14
K 0..14
P 0..40000
O 0..30
W0 30..300
Wt 30..300

Warnings (non-blocking)
- Def > 1500 kcal/day -> High deficit warning
- C < 1500 kcal/day -> Very low intake warning
- Def <= 0 -> No progress warning

## 6. User interface requirements

### 6.1 Mobile-first layout
- Single-column layout on small screens
- Sticky primary action button Calculate on mobile
- Use large touch targets and input affordances
- Use responsive typography and spacing
- On larger screens, use a two-column layout
  - left inputs
  - right results and charts

### 6.2 Input section
Core inputs
- Daily intake on normal days C
- Baseline daily burn B with toggle Auto Baseline
- Strength sessions per week S
- Cardio sessions per week K
- Steps per day P
- Over-limit days per month O
- Current weight W0
- Target weight Wt
- Method toggle QuickAvg vs WeeklySim
- Calculate button

Auto Baseline fields when enabled
- Sex
- Age
- Height
- ActivityBase

Advanced section collapsible
- P0, deltaOver, kcalPerKg
- MET and minutes for strength and cardio
- stepCoef
- WeeklySim options
  - MaxDays
  - B behavior toggle LockB vs ScaleBByWeight
  - Over-limit modeling toggle AverageMonthly vs CalendarDays
  - Calendar day picker when CalendarDays selected

### 6.3 Results section
Must show
- Estimated days to target
- Estimated end date (today + days) using the browser locale
- Average deficit Def
- Average burn TDEE
- Average intake Cavg
- Factor contributions
  - baseline B
  - steps Esteps
  - strength Estr
  - cardio Ecar
  - over-limit penalty Cover

Statuses
- NoProgress
- MaxDaysReached
- OK

### 6.4 What-if panel
Interactive controls to adjust
- C slider step 50 kcal
- P slider step 500 steps
- S stepper
- K stepper
- O slider
Show
- updated days to target
- delta vs baseline scenario

## 7. Charts and visualizations

### 7.1 Weight vs time
- Line chart
- X axis days
- Y axis kg
For QuickAvg
- build a linear series from Def and kcalPerKg
For WeeklySim
- use simulated WeightSeries
Mark
- vertical line at target day
- horizontal line at target weight

### 7.2 Energy balance
- Line chart with two series
  - Burn (TDEE)
  - Intake (Cavg or Cday)
- Optional shaded area showing deficit

### 7.3 Factor contribution
- Bar chart or waterfall chart
  - baseline B
  - steps Esteps
  - strength Estr
  - cardio Ecar
  - minus over-limit Cover
  - resulting Def

### 7.4 Sensitivity summary
- Small table showing marginal impact examples
  - +2000 steps changes days by X
  - -200 kcal changes days by Y
  - -1 over-limit day changes days by Z

## 8. Export requirements
Provide buttons
- Export scenario and results as JSON
- Export time series as CSV
CSV columns
day, weight_kg, burn_kcal, intake_kcal, deficit_kcal

## 9. Accessibility and quality
- Use semantic HTML
- Labels for all inputs
- Keyboard navigable
- Color is not the only encoding, include numeric values
- Show units next to fields and in tooltips
- All text in English

## 10. Acceptance criteria
- Given valid inputs, the app returns days to target and intermediate values matching formulas
- WeeklySim produces a weight series that monotonically approaches the target under positive deficit
- When Def <= 0 the UI shows NoProgress and actionable suggestions
- Charts render correctly on mobile and desktop
- Exported JSON and CSV contain correct data
- Everything works offline except external CDN assets if used

## 11. Non-functional requirements
- Fast recalculation
  - QuickAvg in under 50 ms
  - WeeklySim under 300 ms for MaxDays up to 2000
- No backend required
- Data stays in browser
- Optional localStorage save and load scenarios
