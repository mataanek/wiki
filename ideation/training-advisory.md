# Ideation: Personalized Training Advisory

**Goal:** Build a training advisory system that ingests real-time (or near real-time) health/fitness data from Garmin, Apple Health, and other sources, combines it with the latest scientific/fitness findings, and provides personalized workout recommendations, adjustments, and feedback to the user.

## Why
Training plans that adapt to an individual's current physiological state (HRV, sleep, stress, recent load) are more effective and reduce injury risk. Combining wearable data with evidence-based guidelines can create a dynamic coach that evolves with the user.

## Data Sources
- **Garmin Connect** – heart rate, HRV, stress score, sleep, steps, VO2 max estimate, training load, etc.
  - Access via Garmin Connect API or CSV export; token handling previously problematic.
- **Apple Health** – similar metrics (if user has iPhone/Apple Watch); accessed via HealthKit on macOS or via exported XML/JSON.
- **Other Sources** – manual logs (RPE, nutrition), third-party apps (Strava, TrainingPeaks), lab tests (if available), user-reported fatigue/soreness.
- **Scientific Knowledge Base** – latest research on periodization, intensity distribution, recovery, nutrition, etc. (could be stored as markdown or a small database of evidence summaries).

## Components
1. **Data Ingestion Layer**
   - Scripts to pull data from Garmin (handle OAuth/token refresh), Apple Health (if on macOS), and other sources.
   - Normalize to a common schema (timestamp, metric type, value, source).
   - Store raw and processed data in a local database (e.g., SQLite) or time-series friendly store.

2. **Analysis & State Estimation**
   - Compute readiness scores (e.g., based on HRV, sleep, recent training load).
   - Detect signs of overreaching, illness, or exceptional readiness.
   - Use rolling windows and baselines personalized to the user.

3. **Knowledge Integration**
   - Maintain a curated set of training principles (e.g., polarized training, 80/20 rule, deload frequency) linked to conditions.
   - When state indicates fatigue, suggest easier workouts or extra rest; when ready, suggest harder intervals or volume increase.

4. **Recommendation Engine**
   - Generate daily or weekly workout suggestions: type (endurance, strength, intervals), duration, intensity zones.
   - Provide alternatives based on time constraints or equipment.
   - Explain reasoning (which data points drove the suggestion).

5. **Feedback Loop**
   - After each workout, collect user feedback (RPE, perceived difficulty, completion) and actual data (from wearable).
   - Update models/baselines and refine future recommendations.

6. **Presentation**
   - Output via terminal, wiki log, or simple web dashboard.
   - Optionally send reminders via Telegram or email.

## Challenges & Notes
- **Garmin Token Handling:** Previous attempts failed due to complexities in OAuth2 token refresh and session management. Need a robust solution (maybe using `garminconnect` Python library with proper token persistence).
- **Apple Health Access:** On Linux, access is limited; may require macOS bridge or reliance on exported data.
- **Privacy:** Store data locally; encrypt if desired.
- **Scientific Currency:** Periodically update knowledge base with new findings (e.g., monthly review of recent meta-analyses).

## Next Steps (when revisiting)
1. Re-evaluate Garmin API access: look for updated Python libraries, document token storage strategy.
2. Prototype data pipeline for one source (e.g., Garmin) to fetch and store HRV and sleep.
3. Define a simple readiness score formula and test with historical data.
4. Link readiness to a basic rule-based advisory (e.g., if HRV low + poor sleep → easy day).
5. Gradually incorporate more sources and refine recommendations.
6. Consider creating a Hermes skill or standalone Python module that can be invoked via CLI or cron.

## Related
- Wiki Index: `wiki/whisky_wiki/index.md`
- Existing notes on memory: `wux/MEMORY.md` and `hex/MEMORY.md` (may contain relevant user preferences or past attempts).
- Ideation directory: `/home/mataanek/.hermes/wiki/ideation/` (this file).
