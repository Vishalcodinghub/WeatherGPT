# WeatherGPT — Implementation Plan

## Top-Level Overview

**Goal:** Build a conversational weather application called WeatherGPT where users can search for a city, view current weather, view a daily/hourly forecast, see weather alerts, and ask natural-language questions about weather — all powered by OpenWeatherMap and the Groq LLaMA 3 API.

**Scope (MVP):**
- Single-page Streamlit app
- Default city: New Delhi on first load
- Current weather panel: temperature, feels like, humidity, wind speed, pressure, condition, sunrise, sunset
- Forecast panel: Daily (5-day) and Hourly (next 8 slots, 3-hour intervals) with a toggle
- Weather alerts banner (shown only when alerts exist for the location)
- Conversational AI panel: single-turn Q&A, one question at a time, grounded in real weather data
- Graceful error handling for invalid cities, API failures, and missing alerts
- API keys stored in `.env`, never in source code

**Out of Scope for MVP:**
- Multi-turn chat memory
- User accounts / login
- Historical weather data
- Map view
- Email/push notifications
- Caching

**Key Design Principles:**
- UI code lives only in `app.py`
- API calls live only in `weather_service.py` and `ai_service.py`
- Business logic / data shaping lives in `weather_processor.py`
- Validation and error handling live in `validators.py` and `exceptions.py`
- Helper formatting lives in `utils.py`
- Tests live in `tests/`

---

## Project Structure

```
WeatherGPT/
├── app.py                  # Streamlit entry point — UI only
├── weather_service.py      # OpenWeatherMap API calls
├── ai_service.py           # Groq LLM API calls
├── weather_processor.py    # Data shaping from raw API JSON
├── validators.py           # Input validation functions
├── exceptions.py           # Custom exception classes
├── utils.py                # Formatting helpers (time, emoji, wind, etc.)
├── config.py               # App-level constants (default city, units, etc.)
├── .env                    # API keys — NEVER committed to git
├── .gitignore              # Must include .env
├── requirements.txt        # All pip dependencies
└── tests/
    ├── test_weather_processor.py
    ├── test_validators.py
    ├── test_utils.py
    └── test_ai_service.py
```

---

## Data Models

All data models are plain Python dataclasses with type hints. No database or ORM needed.

### `CurrentWeather`
Fields: `city_name`, `country`, `temperature`, `feels_like`, `humidity`, `pressure`, `wind_speed`, `condition`, `condition_icon_url`, `sunrise`, `sunset`, `timestamp`

### `ForecastDay`
Fields: `date_label`, `temp_min`, `temp_max`, `condition`, `condition_icon_url`, `humidity`, `wind_speed`

### `ForecastHour`
Fields: `time_label`, `temperature`, `condition`, `condition_icon_url`, `humidity`, `wind_speed`

### `WeatherAlert`
Fields: `event`, `description`, `start_time`, `end_time`, `sender`

### `WeatherData` (aggregate — holds all of the above)
Fields: `current`, `daily_forecast` (list of ForecastDay), `hourly_forecast` (list of ForecastHour), `alerts` (list of WeatherAlert, may be empty)

---

## Custom Exceptions

Defined in `exceptions.py`:

- `CityNotFoundError` — raised when the API returns 404 for a city
- `APIKeyError` — raised when the API returns 401 (invalid key)
- `APIRateLimitError` — raised when the API returns 429
- `APIConnectionError` — raised on network timeout or connection failure
- `APIResponseError` — raised for any unexpected non-200 response
- `InvalidCityInputError` — raised by the validator before any API call is made
- `LLMUnavailableError` — raised when the Groq API call fails

---

## Phase 1 — Project Setup

**Intent:** Create the project skeleton, install dependencies, and verify the environment is ready to build.

**Expected Outcomes:**
- All files and folders exist with empty/stub content
- `requirements.txt` lists all needed packages
- `.env` exists locally with placeholder key names
- `.gitignore` is correctly configured
- Running `streamlit run app.py` shows a blank Streamlit page without errors

**Todo List:**
1. Create the top-level `WeatherGPT/` folder
2. Create empty files: `app.py`, `weather_service.py`, `ai_service.py`, `weather_processor.py`, `validators.py`, `exceptions.py`, `utils.py`, `config.py`
3. Create the `tests/` folder and empty test files
4. Write `requirements.txt` with: `streamlit`, `requests`, `python-dotenv`, `groq`, `pytest`
5. Create `.env` with keys: `OPENWEATHERMAP_API_KEY=your_key_here` and `GROQ_API_KEY=your_key_here`
6. Create `.gitignore` with `.env` included
7. Add a one-line `st.title("WeatherGPT")` in `app.py` to confirm Streamlit runs

**Files Changed:** All files (initial creation)

**Why It Is Needed:** A clean skeleton prevents "where does this code go?" confusion for every phase that follows.

**How to Test:** Run `streamlit run app.py` — a browser tab opens showing "WeatherGPT".

**Status:** [ ] pending

---

## Phase 2 — Weather API Integration

**Intent:** Build `weather_service.py` to fetch raw JSON from OpenWeatherMap and return it to the caller. No data shaping here — just raw responses.

**Expected Outcomes:**
- `weather_service.py` can fetch current weather, forecast, and alerts for any city
- All HTTP errors are caught and re-raised as custom exceptions from `exceptions.py`
- API keys are loaded from `.env` using `python-dotenv` — never hardcoded

**Todo List:**
1. In `config.py`, define constants: `DEFAULT_CITY = "New Delhi"`, `UNITS = "metric"`, `FORECAST_DAYS = 5`, `HOURLY_SLOTS = 8`
2. In `exceptions.py`, define all custom exception classes listed in the Data Models section above
3. In `weather_service.py`, write a `WeatherService` class with:
   - `__init__`: loads `OPENWEATHERMAP_API_KEY` from environment
   - `fetch_current_weather(city: str) -> dict`: calls `/weather` endpoint
   - `fetch_forecast(city: str) -> dict`: calls `/forecast` endpoint
   - `fetch_alerts(lat: float, lon: float) -> dict`: calls `/onecall` endpoint with `exclude` param
   - A private `_handle_response(response)` method that checks status code and raises the correct custom exception
4. All methods return raw parsed JSON (`dict`) — no processing yet

**Files Changed:** `weather_service.py`, `exceptions.py`, `config.py`

**Why It Is Needed:** Isolating API calls means the rest of the app never needs to know about HTTP, URLs, or status codes.

**How to Test:** Temporarily add a `if __name__ == "__main__":` block in `weather_service.py`, create a `WeatherService()` instance, call `fetch_current_weather("New Delhi")`, and `print()` the result. Verify a JSON dict is printed.

**Status:** [ ] pending

---

## Phase 3 — Weather Data Processing

**Intent:** Build `weather_processor.py` to convert raw API JSON into typed dataclasses, and `utils.py` to provide formatting helpers used during that conversion.

**Expected Outcomes:**
- Given raw JSON from the API, `WeatherProcessor` returns clean, typed `WeatherData` objects
- All timestamp conversions, unit labels, and icon URLs are handled here — not in the UI
- `utils.py` contains small pure functions (no side effects) that are independently testable

**Todo List:**
1. In `weather_processor.py`, define dataclasses: `CurrentWeather`, `ForecastDay`, `ForecastHour`, `WeatherAlert`, `WeatherData` (all listed in the Data Models section)
2. Write a `WeatherProcessor` class with:
   - `build_current_weather(raw: dict) -> CurrentWeather`
   - `build_daily_forecast(raw: dict) -> list[ForecastDay]`
   - `build_hourly_forecast(raw: dict) -> list[ForecastHour]`
   - `build_alerts(raw: dict) -> list[WeatherAlert]`
   - `build_weather_data(current_raw, forecast_raw, alerts_raw) -> WeatherData`
3. In `utils.py`, write:
   - `unix_to_time(ts: int) -> str` — converts Unix timestamp to "06:12 AM"
   - `unix_to_date(ts: int) -> str` — converts Unix timestamp to "Mon, 23 Jun"
   - `get_weather_emoji(condition: str) -> str` — maps condition to emoji
   - `ms_to_kmh(ms: float) -> float` — converts m/s to km/h and rounds to 1 decimal
   - `format_wind(ms: float) -> str` — returns "Calm" if 0, else "X.X km/h"

**Files Changed:** `weather_processor.py`, `utils.py`

**Why It Is Needed:** The UI should receive clean, ready-to-display data — it should never parse JSON or do arithmetic on raw API values.

**How to Test:** Write unit tests in `tests/test_weather_processor.py` and `tests/test_utils.py` using sample JSON fixtures (copy a real API response into a variable). Assert that output dataclass fields have expected values.

**Status:** [ ] pending

---

## Phase 4 — Streamlit UI

**Intent:** Build the full single-page Streamlit UI in `app.py` that displays weather data using the service and processor from Phases 2–3.

**Expected Outcomes:**
- App loads with New Delhi weather by default
- Search bar allows users to type a city and press Enter or click Search
- Current weather section shows all required fields using `st.metric()`
- Forecast section has a Daily / Hourly radio toggle
- Alerts banner appears only when alerts exist (using `st.warning()`)
- Sunrise and sunset displayed under the current weather card
- All error messages displayed with `st.error()` — app never crashes

**Todo List:**
1. In `app.py`, import all services, processor, and validators
2. Initialize session state for `city` (default: `"New Delhi"`) and `weather_data`
3. Build the header: app title, tagline, search bar, and search button as a row
4. On search: call `validators.validate_city_input()` first, then fetch and process data
5. Build `render_current_weather(data: CurrentWeather)`:
   - Use `st.columns()` to lay out metrics side by side
   - Show: temperature, feels like, humidity, wind speed, pressure, condition + emoji
   - Show sunrise and sunset below the metrics
6. Build `render_forecast(data: WeatherData)`:
   - Add a `st.radio("View", ["Daily", "Hourly"])` toggle
   - Daily: render `ForecastDay` cards in `st.columns(5)` with min/max temp, condition, emoji
   - Hourly: render `ForecastHour` cards in `st.columns(8)` with time, temp, condition
7. Build `render_alerts(alerts: list[WeatherAlert])`:
   - If list is empty: render nothing
   - If list has items: render one `st.warning()` block per alert with event name and description
8. Wrap all data-fetching in a try/except block that catches each custom exception and calls `st.error()` with a user-friendly message

**Files Changed:** `app.py`

**Why It Is Needed:** This is the user-facing layer — everything the user sees and interacts with lives here.

**How to Test:** Run the app, search for "Mumbai" — verify all sections render with real data. Search for "xyzinvalidcity" — verify a clean error message appears, not a crash.

**Status:** [ ] pending

---

## Phase 5 — Conversational AI Integration

**Intent:** Build `ai_service.py` to send a weather context summary plus the user's question to Groq's LLaMA 3 model and return a conversational answer. Then add the chat panel to `app.py`.

**Expected Outcomes:**
- `ai_service.py` builds a clear text summary of current weather data to use as LLM context
- The LLM is instructed via system prompt to only answer weather-related questions
- The Groq API call is wrapped in error handling that raises `LLMUnavailableError` on failure
- A chat input box appears in the UI below the forecast
- User types a question, presses Enter, and a plain-English answer appears below the box
- If LLM is unavailable, a fallback message is shown — the rest of the app still works

**Todo List:**
1. In `ai_service.py`, write an `AIService` class with:
   - `__init__`: loads `GROQ_API_KEY` from environment, initializes Groq client
   - `build_weather_context(data: WeatherData) -> str`: converts `WeatherData` to a readable multi-line text block
   - `ask(context: str, question: str) -> str`: sends system prompt + context + question to Groq, returns the answer string
   - The system prompt must instruct the model: "You are a weather assistant. Only answer weather-related questions. Do not answer anything unrelated to weather."
   - Wrap the Groq call in try/except; raise `LLMUnavailableError` on failure
2. In `app.py`, add `render_chat(data: WeatherData)`:
   - A `st.text_input()` for the user's question
   - A "Ask" button
   - On click: call `ai_service.ask()`, display response with `st.info()`
   - If question is empty: show `st.warning("Please type a question first.")`
   - If `LLMUnavailableError`: show `st.error("AI assistant is currently unavailable...")`

**Files Changed:** `ai_service.py`, `app.py`

**Why It Is Needed:** This is the core differentiator of WeatherGPT — making weather information conversational and accessible in plain language.

**How to Test:** Load the app with a city, type "Should I carry an umbrella today?", click Ask. Verify the response mentions actual weather conditions for that city. Then type "Who won the cricket match?" — verify the model refuses to answer.

**Status:** [ ] pending

---

## Phase 6 — Weather Alert Handling

**Intent:** Ensure weather alerts are correctly fetched using coordinates from the current weather response, processed into `WeatherAlert` objects, and displayed only when present.

**Expected Outcomes:**
- `weather_service.py` extracts `lat` and `lon` from the current weather response and uses them to call the alerts endpoint
- If no alerts exist for a location, the `alerts` list in `WeatherData` is empty — no errors thrown
- The UI shows nothing in the alerts section for cities with no active alerts
- For cities with active alerts, the UI shows a yellow warning banner per alert

**Todo List:**
1. Confirm that `fetch_alerts()` in `weather_service.py` uses `lat`/`lon` extracted from the current weather JSON (not a city name)
2. In `WeatherProcessor.build_alerts()`, handle the case where the `alerts` key is absent from the API response — return an empty list, do not raise an exception
3. In `app.py`'s `render_alerts()`, add a guard: `if not alerts: return` — show nothing, no empty box
4. Test with a city that has no current alerts — verify no alert section renders
5. Simulate an alert by temporarily creating a mock `WeatherAlert` object in code — verify the banner renders correctly

**Files Changed:** `weather_service.py`, `weather_processor.py`, `app.py`

**Why It Is Needed:** Missing alerts are normal, not errors. The app must silently skip them rather than crashing or showing confusing empty boxes.

**How to Test:** Run the app with a normal city — alerts section should be invisible. Add a mock alert to the data object manually and re-run — banner should appear in yellow.

**Status:** [ ] pending

---

## Phase 7 — Input Validation and Error Handling

**Intent:** Build `validators.py` to validate user input before any API calls are made, and ensure every possible failure point in the app raises a specific custom exception that is caught and shown as a friendly error message.

**Expected Outcomes:**
- Invalid city input (empty, too short, contains numbers/special characters) is caught before any API call
- Every custom exception from `exceptions.py` is caught somewhere in `app.py` and shown with `st.error()`
- The app never shows a raw Python traceback to the user
- All validation rules are documented as constants in `validators.py`

**Todo List:**
1. In `validators.py`, write `validate_city_input(city: str) -> None`:
   - Strip whitespace; raise `InvalidCityInputError` if the result is empty
   - Raise `InvalidCityInputError` if length is less than 2 characters
   - Raise `InvalidCityInputError` if length exceeds 100 characters
   - Raise `InvalidCityInputError` if the input contains digits or special characters (allow letters, spaces, hyphens, apostrophes, dots — for city names like "St. John's" or "Ooty")
2. In `app.py`, ensure the top-level try/except chain catches: `InvalidCityInputError`, `CityNotFoundError`, `APIKeyError`, `APIRateLimitError`, `APIConnectionError`, `APIResponseError`, `LLMUnavailableError`, and bare `Exception` as a last resort
3. Map each exception to a specific user-friendly `st.error()` message (see the Error Cases section of the system design)
4. In `validators.py`, write `validate_question_input(question: str) -> None`:
   - Raise `ValueError` if question is empty after stripping
   - Raise `ValueError` if question exceeds 500 characters

**Files Changed:** `validators.py`, `app.py`

**Why It Is Needed:** Users will type garbage. APIs will fail. The app must always stay alive and guide the user, never crash.

**How to Test:** In the search bar, type: empty string, "A", "New Delhi 123", a 200-character string. Each should show a different clean error message without crashing the app.

**Status:** [ ] pending

---

## Phase 8 — Testing

**Intent:** Write a focused suite of pytest unit tests covering the data processing, validation, formatting, and AI service logic. The goal is to catch regressions during future changes.

**Expected Outcomes:**
- All tests in `tests/` pass with `pytest` from the command line
- Tests use mock/fixture data — no real API calls in tests
- Each function in `utils.py`, `validators.py`, and `weather_processor.py` has at least one test
- Edge cases (empty input, missing JSON keys, zero wind speed, alerts absent) are tested

**Todo List:**

### `tests/test_utils.py`
1. Test `unix_to_time()` with a known Unix timestamp — assert expected "HH:MM AM/PM" string
2. Test `unix_to_date()` — assert expected "Day, DD Mon" string
3. Test `get_weather_emoji()` with "Rain", "Clear", "Snow", "Thunderstorm", unknown condition
4. Test `ms_to_kmh()` with 0, 5.5, 10.0 — assert rounded km/h values
5. Test `format_wind()` with 0 (expect "Calm") and non-zero value

### `tests/test_validators.py`
6. Test `validate_city_input()` with empty string — assert `InvalidCityInputError` raised
7. Test with single character — assert `InvalidCityInputError` raised
8. Test with "New Delhi" — assert no exception raised
9. Test with "123Mumbai" — assert `InvalidCityInputError` raised
10. Test with a 101-character string — assert `InvalidCityInputError` raised
11. Test with valid edge cases: "St. John's", "Ooty", "New York"

### `tests/test_weather_processor.py`
12. Test `build_current_weather()` with a sample current weather JSON fixture — assert all fields populated correctly
13. Test `build_daily_forecast()` with a sample forecast JSON fixture — assert returns list of 5 `ForecastDay` objects
14. Test `build_hourly_forecast()` with a sample forecast JSON fixture — assert returns list of 8 `ForecastHour` objects
15. Test `build_alerts()` with a JSON fixture that has no `alerts` key — assert returns empty list
16. Test `build_alerts()` with a fixture that has one alert — assert returns list of one `WeatherAlert`

### `tests/test_ai_service.py`
17. Test `build_weather_context()` with a mock `WeatherData` object — assert the returned string contains city name, temperature, and humidity
18. Test `ask()` with a mocked Groq client (use `unittest.mock.patch`) — assert it returns a string

**Files Changed:** `tests/test_utils.py`, `tests/test_validators.py`, `tests/test_weather_processor.py`, `tests/test_ai_service.py`

**Why It Is Needed:** Tests catch bugs introduced during refactoring and give confidence when preparing for SIH demos.

**How to Test:** Run `pytest tests/ -v` from the project root. All 18+ tests should pass with green output.

**Status:** [ ] pending

---

## Phase 9 — Deployment

**Intent:** Deploy the app to Streamlit Community Cloud so it is accessible from any browser without installing anything.

**Expected Outcomes:**
- App is hosted at a public URL (e.g., `https://weathergpt-yourname.streamlit.app`)
- API keys are stored as Streamlit Cloud secrets — not in any file
- App loads correctly with the New Delhi default and all features work
- `.env` file is never pushed to GitHub

**Todo List:**
1. Confirm `.gitignore` includes: `.env`, `__pycache__/`, `*.pyc`, `.pytest_cache/`
2. Push the project to a public or private GitHub repository
3. Go to [share.streamlit.io](https://share.streamlit.io), connect your GitHub account, and select the repo
4. In Streamlit Cloud settings → Secrets, add:
   ```
   OPENWEATHERMAP_API_KEY = "your_real_key"
   GROQ_API_KEY = "your_real_key"
   ```
5. Update `weather_service.py` and `ai_service.py` to use `os.environ.get()` which works for both `.env` (local) and Streamlit Cloud secrets (production)
6. Trigger a deploy and verify the app loads at the public URL
7. Test all features (search, forecast toggle, alerts, AI chat) from the deployed URL

**Files Changed:** `.gitignore`, `weather_service.py`, `ai_service.py` (minor env loading check)

**Why It Is Needed:** A deployed URL is essential for SIH presentations — judges will want to see a live demo, not a local laptop app.

**How to Test:** Open the public URL in a browser on a different device (e.g., a phone). Search for a city and use all features. The app should work identically to local.

**Status:** [ ] pending

---

## Phase 10 — SIH Presentation and Demo Preparation

**Intent:** Polish the app's appearance, prepare a compelling demo flow, and ensure the app is stable and impressive for an SIH-level panel.

**Expected Outcomes:**
- App has a clean title, tagline, and brief description in the sidebar or header
- A curated list of 3–5 demo cities with interesting weather (e.g., Mumbai during monsoon, Delhi in winter, Chennai coastal)
- AI responses are tested for quality and accuracy on likely demo questions
- README.md is complete: project description, setup instructions, API key setup, and run command
- All edge cases tested: invalid city, API failure simulation, empty question

**Todo List:**
1. Add `st.set_page_config(page_title="WeatherGPT", page_icon="🌤️", layout="wide")` at the top of `app.py`
2. Add a sidebar with: project name, one-line description, "About" section explaining the tech stack
3. Pre-test these demo questions for quality:
   - "Should I carry an umbrella today?"
   - "Is it a good day for outdoor exercise?"
   - "What should I wear today?"
   - "Is the air quality going to be a problem today?" (LLM should note it doesn't have AQI data)
4. Write `README.md` covering: what the app does, how to set up `.env`, how to install requirements, how to run locally, how to deploy
5. Run `pytest tests/ -v` one final time — all tests must pass
6. Load the deployed URL and do a full end-to-end walkthrough: default city → search → forecast toggle → AI question → invalid city → alert (if available)

**Files Changed:** `app.py` (page config, sidebar), `README.md` (new file)

**Why It Is Needed:** SIH judges evaluate polish, clarity, and robustness — not just functionality. A well-prepared demo is as important as the code.

**How to Test:** Do a full rehearsal of the demo flow on the deployed URL, timing it to under 3 minutes.

**Status:** [ ] pending

---

## Future Improvements (Post-MVP)

These are explicitly out of scope for the MVP but documented here so the project can grow:

1. **Multi-turn chat** — maintain a conversation history list in session state and pass it to the LLM
2. **Unit toggle** — add °C / °F switch; pass `units` parameter to OpenWeatherMap
3. **AQI (Air Quality Index)** — OpenWeatherMap has a free `/air_pollution` endpoint
4. **Map view** — use `st.map()` with the city's lat/lon
5. **Historical climate** — use OpenWeatherMap's paid history endpoint or a separate climate dataset
6. **IMD data integration** — connect to India Meteorological Department open data for India-specific accuracy
7. **Multi-language support** — OpenWeatherMap supports `lang` param; Groq can respond in other languages
8. **Voice input** — use `streamlit-audiorecorder` + Whisper API for voice-to-text questions
9. **Data caching** — add `@st.cache_data(ttl=600)` to weather service calls to reduce API usage
10. **Responsive charts** — use `plotly` or `altair` to show temperature trend lines over the forecast period

---

## Error Message Reference

| Exception | User-Facing Message |
|---|---|
| `InvalidCityInputError` | "Please enter a valid city name (letters only, 2–100 characters)." |
| `CityNotFoundError` | "City not found. Please check the spelling or try a nearby city." |
| `APIKeyError` | "Service configuration error. Please contact the administrator." |
| `APIRateLimitError` | "Too many requests. Please wait a moment and try again." |
| `APIConnectionError` | "Unable to connect to weather service. Please check your internet connection." |
| `APIResponseError` | "Unexpected error from weather service. Please try again." |
| `LLMUnavailableError` | "AI assistant is currently unavailable. The weather data above is still accurate." |

---

## Dependencies Reference

```
streamlit
requests
python-dotenv
groq
pytest
```

All installable with: `pip install -r requirements.txt`
