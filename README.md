# booking-watch

Get a **Telegram** ping the moment a specific **movie + theatre** opens booking
on BookMyShow / District. Runs itself every ~10 minutes on **GitHub Actions**,
so there's nothing to keep running on your own machine.

It works by polling a URL you choose and watching for the transition from
"not bookable" to "booking open" for your theatre. Nothing site-specific is
hardcoded, so if BookMyShow/District tweak their pages you just edit
`config.json` — not the code.

---

## 1. Create a Telegram bot (2 minutes)

1. In Telegram, message **@BotFather** → send `/newbot` → follow prompts.
2. It gives you a **bot token** like `123456:ABC-DEF...`. Save it.
3. Open a chat with your new bot and send it any message (e.g. `hi`). This is
   required before it can message you.
4. Get your **chat id**: open this URL in a browser (paste your token):
   ```
   https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates
   ```
   Look for `"chat":{"id":123456789` — that number is your `TELEGRAM_CHAT_ID`.

## 2. Find the URL to watch

You want the URL where showtimes for your movie appear. Two options:

**A. Simple (the showtimes page).** On BookMyShow, open the movie's
"Book tickets" page for your city and copy the address bar URL, e.g.
`https://in.bookmyshow.com/movies/<slug>/buytickets/<code>/<city>`.
This is the easiest and works for most cases.

**B. Most reliable (the internal API).** These sites load showtimes via a
background request. To capture it:
1. Open the showtimes page in Chrome, press **F12** → **Network** tab.
2. Filter by **Fetch/XHR**, reload the page.
3. Find the request whose response contains venue/showtime data (search the
   responses for your theatre's name).
4. Right-click it → **Copy** → **Copy as cURL**, and copy the request URL.
5. If it needs headers/cookies, add them as a JSON object in the
   `HEADERS_JSON` secret (see below).

> Note: these internal APIs aren't official and may change or require cookies.
> If option B stops working, fall back to option A. Keep the poll interval
> polite (the default 10 min is fine).

## 3. Configure

```bash
cp config.example.json config.json
```

Edit `config.json`:
- `movie`   — the movie title as it appears on the page (helps avoid false hits).
- `theatre` — the theatre name exactly as shown on the site.
- `target_url` — the URL from step 2.

Matching is case-insensitive and whitespace-tolerant. Booking counts as "open"
when the theatre name **and** a booking signal (`book tickets`, `showtime`, ...)
are both present, and it's not showing only `notify me` / `coming soon`.

## 4. Test locally (optional but recommended)

```bash
pip install -r requirements.txt

export TELEGRAM_BOT_TOKEN=123456:ABC...
export TELEGRAM_CHAT_ID=123456789
python poller.py
```

It prints `available=True/False`. To confirm Telegram works, temporarily point
`target_url` at a page you know lists the theatre with booking open — you
should get a message.

## 5. Deploy on GitHub Actions

1. Create a **new GitHub repo** and push these files to it:
   ```bash
   git init && git add . && git commit -m "init booking-watch"
   git branch -M main
   git remote add origin https://github.com/<you>/booking-watch.git
   git push -u origin main
   ```
2. In the repo: **Settings → Secrets and variables → Actions → New repository secret**
   and add:
   - `TELEGRAM_BOT_TOKEN`
   - `TELEGRAM_CHAT_ID`
   - `HEADERS_JSON` *(optional — only for API URLs that need headers/cookies)*
3. The workflow (`.github/workflows/booking-watch.yml`) runs every 10 minutes
   automatically. Trigger a test run anytime from the **Actions** tab →
   *booking-watch* → **Run workflow**.

When booking opens you get one Telegram message. State is saved in `state.json`
(committed back by the workflow) so you aren't re-notified every run.

### Public vs private repo

- **Public repo:** GitHub Actions minutes are **unlimited & free**. Recommended.
  Nothing sensitive is exposed — your token/chat id live in encrypted Secrets,
  only the movie/theatre/URL are in `config.json`.
- **Private repo:** the free tier gives ~2000 Actions minutes/month. Running
  every 10 min (~4300 runs/mo) would exceed it — bump the cron in the workflow
  to `*/30 * * * *` (every 30 min) to stay comfortably under.

## Adjusting

- **Interval:** edit the `cron` line in `.github/workflows/booking-watch.yml`
  (`*/10 * * * *` = every 10 min; GitHub's minimum is 5 and runs may lag a few
  minutes).
- **Multiple movies/theatres:** duplicate the folder into separate repos, or
  extend `config.json` into a list and loop in `poller.py` (ask and I'll wire
  it up).
- **False positives/negatives:** tune `open_signals` / `closed_signals` in
  `config.json`.

## Files

| File | Purpose |
|------|---------|
| `poller.py` | Fetch URL, detect availability, send Telegram alert |
| `config.example.json` | Copy to `config.json` and fill in |
| `.github/workflows/booking-watch.yml` | Scheduled always-on runner |
| `requirements.txt` | Python deps (`requests`) |
| `state.json` | Auto-managed; tracks last-seen availability |

https://in.bookmyshow.com/movies/chennai/the-odyssey/buytickets/ET00480917/20260719
curl 'https://in.bookmyshow.com/api/seo/v1/footer?url=%2Fmovies%2Fchennai%2Fthe-odyssey%2Fbuytickets%2FET00480917%2F20260719' \
  -H 'x-app-code: WEB' \
  -H 'sec-ch-ua-platform: "Windows"' \
  -H 'x-geohash: tf3' \
  -H 'sec-ch-ua: "Not;A=Brand";v="8", "Chromium";v="150", "Microsoft Edge";v="150"' \
  -H 'sec-ch-ua-mobile: ?0' \
  -H 'x-latitude: 13.056' \
  -H 'baggage: sentry-environment=production,sentry-release=release_460,sentry-public_key=4d17a59c2597410e714ab31d421148d9,sentry-trace_id=e9ab6daf4eed4411ac2b6e7003266f56,sentry-transaction=%2Fmovies%2F%3AregionNameSlug%2F%3AmovieNameSlug%2Fbuytickets%2F%3AeventCode%2F%3AshowDate%3F,sentry-sampled=false,sentry-sample_rand=0.7556345325745871,sentry-sample_rate=0.001' \
  -H 'sentry-trace: e9ab6daf4eed4411ac2b6e7003266f56-9ddaa22289392f8b-0' \
  -H 'x-region-slug: chennai' \
  -H 'Accept: application/json, text/plain, */*' \
  -H 'x-platform-code: WEB' \
  -H 'x-longitude: 80.206' \
  -H 'x-platform: WEB' \
  -H 'Referer: https://in.bookmyshow.com/movies/chennai/the-odyssey/buytickets/ET00480917/20260719' \
  -H 'x-segments;' \
  -H 'true-client-ip: 157.51.138.216' \
  -H 'x-bms-id: 1.114112958.1784204635776' \
  -H 'x-advertiser-id: 1606277553033370021' \
  -H 'User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/150.0.0.0 Safari/537.36 Edg/150.0.0.0' \
  -H 'x-location-selection: manual' \
  -H 'x-region-code: CHEN'

url=%2Fmovies%2Fchennai%2Fthe-odyssey%2Fbuytickets%2FET00480917%2F20260719

  {
    "header": {
        "canonical": "/movies/chennai/the-odyssey/buytickets/ET00480917/20260719",
        "meta_description": "Online movie ticket booking for a Action, Adventure, Drama, Fantasy film The Odyssey with release date, show timings, cinemas & theaters on BookMyShow.",
        "meta_keywords": "The Odyssey movie, movie show timings, online tickets, movies, movies in Chennai",
        "title": "The Odyssey Movie Showtimes in Chennai & Online Ticket Booking"
    },
    "accordionTitle": "Know more about BookMyShow",
    "footer": {
        "content": [],
        "links": [
            {
                "uuid": "38fe8172-00af-3d2e-9524-cb6eb9c43dcb",
                "heading": "Movies Now Showing",
                "items": [
                    {
                        "uuid": "f33914e4-6a87-387a-8297-5cfea09131fd",
                        "label": "Dhamaal 4",
                        "link": "/movies/dhamaal-4/ET00452553"
                    },
                    {
                        "uuid": "1b7443e7-f4ad-3e93-a78d-5c88de973da1",
                        "label": "The Odyssey",
                        "link": "/movies/the-odyssey/ET00452034"
                    },
                    {
                        "uuid": "d6120127-a9b7-3d6c-a5fd-0ed240d3ade5",
                        "label": "Lenin",
                        "link": "/movies/lenin/ET00441159"
                    },
                    {
                        "uuid": "0b2d978e-8d6d-3bae-9977-3496cffe61a3",
                        "label": "Evil Dead Burn",
                        "link": "/movies/evil-dead-burn-tamil/ET00496607"
                    },
                    {
                        "uuid": "ca8ac283-a0c8-3d04-ae84-b74e41a04c36",
                        "label": "Idhayam Murali",
                        "link": "/movies/idhayam-murali/ET00442409"
                    },
                    {
                        "uuid": "dc85d7da-ee38-31be-b843-d5ef46f51dc1",
                        "label": "Anbe Diana",
                        "link": "/movies/anbe-diana/ET00504562"
                    },
                    {
                        "uuid": "4ee197a9-9f8b-3626-b12f-725d81bc3214",
                        "label": "Alpha",
                        "link": "/movies/alpha/ET00403805"
                    },
                    {
                        "uuid": "8ca8b659-0059-3eb4-9145-6747dad9963d",
                        "label": "Oh..! Sukumari",
                        "link": "/movies/oh-sukumari-tamil/ET00503395"
                    },
                    {
                        "uuid": "35848831-d5dc-382b-8bcb-fddece065d73",
                        "label": "Gatta Kusthi 2",
                        "link": "/movies/gatta-kusthi-2/ET00502802"
                    },
                    {
                        "uuid": "50cab869-5c16-3ff4-aaf5-852f5dc383f4",
                        "label": "Welcome To The Jungle",
                        "link": "/movies/welcome-to-the-jungle/ET00369379"
                    }
                ]
            },
            {
                "uuid": "6b562030-ad95-3d6b-b640-3d5c944a4bbe",
                "heading": "Top Cinemas in Chennai",
                "items": [
                    {
                        "uuid": "0605505e-9c9f-3eb5-b6d8-999e5596ac2a",
                        "label": "PVR: VR Chennai, Anna Nagar",
                        "link": "https://in.bookmyshow.com/cinemas/chennai/pvr-vr-chennai-anna-nagar/PCAN"
                    },
                    {
                        "uuid": "e6bc6436-1a3e-305a-8402-71c6739340c8",
                        "label": "Medavakkam Kumaran Cinemas RGB LASER Dolby Atmos",
                        "link": "https://in.bookmyshow.com/cinemas/chennai/medavakkam-kumaran-cinemas-rgb-laser-dolby-atmos/MMKC"
                    },
                    {
                        "uuid": "1a02bbbc-8d97-3ab0-8028-d7cfa4836170",
                        "label": "MovieMax: PR Mall, Wall Tax Road, Chennai",
                        "link": "https://in.bookmyshow.com/cinemas/chennai/moviemax-pr-mall-wall-tax-road-chennai/MMPR"
                    },
                    {
                        "uuid": "567876e5-4f8a-3031-adde-f483ac918dc8",
                        "label": "Jothi Theatre 4K A/c DTS: ST Thomas Mount",
                        "link": "https://in.bookmyshow.com/cinemas/chennai/jothi-theatre-4k-ac-dts-st-thomas-mount/JOTG"
                    },
                    {
                        "uuid": "b745b880-1f75-34f4-9660-a5418aa1552d",
                        "label": "PVR: SKLS Galaxy Mall, Red Hills Chennai",
                        "link": "https://in.bookmyshow.com/cinemas/chennai/pvr-skls-galaxy-mall-red-hills-chennai/PSKL"
                    },
                    {
                        "uuid": "caec8224-9b59-3d8d-a598-9b50f0e9e25e",
                        "label": "PVR: Heritage RSL ECR, Chennai",
                        "link": "https://in.bookmyshow.com/cinemas/chennai/pvr-heritage-rsl-ecr-chennai/PVHR"
                    },
                    {
                        "uuid": "e33441a9-9fa7-322c-b9a1-7a195982c375",
                        "label": "AGS Cinemas: T. Nagar",
                        "link": "https://in.bookmyshow.com/cinemas/chennai/ags-cinemas-t-nagar/ACTN"
                    },
                    {
                        "uuid": "f8adebc8-3b39-3dbb-b4fe-d0ce00ab1ecd",
                        "label": "Vels Theatres: Chennai",
                        "link": "https://in.bookmyshow.com/cinemas/chennai/vels-theatres-chennai/VVGT"
                    },
                    {
                        "uuid": "29daf2d0-3fa4-37fe-8240-da3db320d9eb",
                        "label": "Rakki Cinemas: OMR, Kelambakkam",
                        "link": "https://in.bookmyshow.com/cinemas/chennai/rakki-cinemas-omr-kelambakkam/RAKK"
                    },
                    {
                        "uuid": "010bbf98-7fef-3eb0-a694-bd27e4f5811a",
                        "label": "GanapathyRam Theatre 4K Dolby 7.1: Chennai",
                        "link": "https://in.bookmyshow.com/cinemas/chennai/ganapathyram-theatre-4k-dolby-71-chennai/GRTC"
                    }
                ]
            },
            {
                "uuid": "22c5bf30-e6f4-3d70-98c6-9f0a043c7329",
                "heading": "Top Cinemas Chains in India",
                "items": [
                    {
                        "uuid": "39de66b4-b1a3-38e2-b7f8-8c1344304651",
                        "label": "Justickets",
                        "link": "https://in.bookmyshow.com/cinemas-list/JTAP/all-regions/JTAP"
                    },
                    {
                        "uuid": "d060c7c5-0972-34eb-8d6f-a4eefff5866f",
                        "label": "PVR",
                        "link": "https://in.bookmyshow.com/cinemas-list/PVR/all-regions/PVR"
                    },
                    {
                        "uuid": "c22db47c-ba42-3726-bc15-519a6be8c077",
                        "label": "INOX",
                        "link": "https://in.bookmyshow.com/cinemas-list/INOX/all-regions/INOX"
                    },
                    {
                        "uuid": "0fdb8a75-2565-3ecb-af84-0727b90f02dc",
                        "label": "Miraj Cinemas",
                        "link": "https://in.bookmyshow.com/cinemas-list/MIRC/all-regions/MIRC"
                    },
                    {
                        "uuid": "c920d0be-b4c2-3cd8-8a69-b129fa905a30",
                        "label": "Cinepolis",
                        "link": "https://in.bookmyshow.com/cinemas-list/CNPL/all-regions/CNPL"
                    },
                    {
                        "uuid": "fb7d5712-2f71-3ba4-a2f4-94fb01a7c4f0",
                        "label": "K Sera Sera Box Office Pvt. Ltd.",
                        "link": "https://in.bookmyshow.com/cinemas-list/KSSB/all-regions/KSSB"
                    },
                    {
                        "uuid": "18ea902c-7b27-3f92-b959-a50c8728e512",
                        "label": "Connplex Cinemas Ltd",
                        "link": "https://in.bookmyshow.com/cinemas-list/CNML/all-regions/CNML"
                    },
                    {
                        "uuid": "853d4600-0165-32c3-b864-32602001d551",
                        "label": "Asian Cinemas",
                        "link": "https://in.bookmyshow.com/cinemas-list/ASCI/all-regions/ASCI"
                    },
                    {
                        "uuid": "fa13be04-02ec-33b8-9013-bca5d27e744a",
                        "label": "Primeshow Films",
                        "link": "https://in.bookmyshow.com/cinemas-list/PRME/all-regions/PRME"
                    },
                    {
                        "uuid": "72b0751e-910f-30f3-b804-882efe8974d2",
                        "label": "Gold Cinema",
                        "link": "https://in.bookmyshow.com/cinemas-list/SNGL/all-regions/SNGL"
                    }
                ]
            },
            {
                "uuid": "6eb80777-3da6-345d-aec5-9b114442ca5f",
                "heading": "Movies Now Showing in Chennai",
                "items": [
                    {
                        "uuid": "1b7443e7-f4ad-3e93-a78d-5c88de973da1",
                        "label": "The Odyssey",
                        "link": "/movies/the-odyssey/ET00452034"
                    },
                    {
                        "uuid": "ca8ac283-a0c8-3d04-ae84-b74e41a04c36",
                        "label": "Idhayam Murali",
                        "link": "/movies/idhayam-murali/ET00442409"
                    },
                    {
                        "uuid": "dc85d7da-ee38-31be-b843-d5ef46f51dc1",
                        "label": "Anbe Diana",
                        "link": "/movies/anbe-diana/ET00504562"
                    },
                    {
                        "uuid": "35848831-d5dc-382b-8bcb-fddece065d73",
                        "label": "Gatta Kusthi 2",
                        "link": "/movies/gatta-kusthi-2/ET00502802"
                    },
                    {
                        "uuid": "3c7470ba-b85c-34e3-ae77-7934c8446941",
                        "label": "Arulvaan",
                        "link": "/movies/arulvaan/ET00507507"
                    },
                    {
                        "uuid": "0b2d978e-8d6d-3bae-9977-3496cffe61a3",
                        "label": "Evil Dead Burn",
                        "link": "/movies/evil-dead-burn-tamil/ET00496607"
                    },
                    {
                        "uuid": "f33914e4-6a87-387a-8297-5cfea09131fd",
                        "label": "Dhamaal 4",
                        "link": "/movies/dhamaal-4/ET00452553"
                    },
                    {
                        "uuid": "8ca8b659-0059-3eb4-9145-6747dad9963d",
                        "label": "Oh..! Sukumari",
                        "link": "/movies/oh-sukumari-tamil/ET00503395"
                    },
                    {
                        "uuid": "d6120127-a9b7-3d6c-a5fd-0ed240d3ade5",
                        "label": "Lenin",
                        "link": "/movies/lenin/ET00441159"
                    },
                    {
                        "uuid": "278f2dec-692f-3d13-83ff-03167851c576",
                        "label": "Spider-Man: Brand New Day",
                        "link": "/movies/spiderman-brand-new-day-epiq-3d/ET00505581"
                    }
                ]
            },
            {
                "uuid": "c0cb31ac-0032-344d-b867-050a1494535d",
                "heading": "Upcoming Movies in Chennai",
                "items": [
                    {
                        "uuid": "ade4c03d-0c27-3796-8a83-89571fd5a913",
                        "label": "Mor Wali Alag He",
                        "link": "/movies/mor-wali-alag-he/ET00507334"
                    },
                    {
                        "uuid": "f857d77e-62fc-3865-bc1b-2ead57adf9a6",
                        "label": "Vadala",
                        "link": "/movies/vadala/ET00495930"
                    },
                    {
                        "uuid": "3d9483fc-ba71-3427-b932-d15b8b3b1ff1",
                        "label": "Antappante Athbudha Pravarthikal",
                        "link": "/movies/antappante-athbudha-pravarthikal/ET00507419"
                    },
                    {
                        "uuid": "dbae90b3-d77a-3b62-8c47-59d0d32301af",
                        "label": "Venkatramaiah Gari Taluka",
                        "link": "/movies/venkatramaiah-gari-taluka/ET00507247"
                    },
                    {
                        "uuid": "2772e2fd-bf34-3d60-b8c9-c944eb5d0bec",
                        "label": "Shatir",
                        "link": "/movies/shatir/ET00505234"
                    },
                    {
                        "uuid": "0f2b0016-944b-37ef-b1c5-c03f15a9b94e",
                        "label": "Bagavan",
                        "link": "/movies/bagavan/ET00504892"
                    },
                    {
                        "uuid": "8355114e-5da9-36f1-b829-1ee11cbc28eb",
                        "label": "Rahun Main Tere Rubaru",
                        "link": "/movies/rahun-main-tere-rubaru/ET00487417"
                    },
                    {
                        "uuid": "3ffed01a-bfeb-3c12-a282-66619dda9ea3",
                        "label": "Oka Court Case",
                        "link": "/movies/oka-court-case/ET00503835"
                    },
                    {
                        "uuid": "ae045d36-092d-3a4f-9035-081c9bae63ff",
                        "label": "Raja The Raja",
                        "link": "/movies/raja-the-raja/ET00505584"
                    },
                    {
                        "uuid": "6af7ac4a-668b-3528-8df1-ed36466cf14b",
                        "label": "Father's Day",
                        "link": "/movies/fathers-day/ET00454710"
                    },
                    {
                        "uuid": "f9a0762f-95a1-30eb-b4ca-acc600a87ed5",
                        "label": "Operation Aruna Reddy",
                        "link": "/movies/operation-aruna-reddy/ET00506635"
                    },
                    {
                        "uuid": "25801110-ebce-3ca1-9ade-b2ac425594bc",
                        "label": "Devi",
                        "link": "/movies/devi/ET00507835"
                    },
                    {
                        "uuid": "6a16a37b-9481-3b73-8d64-bb4e38fa5855",
                        "label": "Black Gold",
                        "link": "/movies/black-gold/ET00504975"
                    },
                    {
                        "uuid": "6180b35a-631c-3dd7-a782-25c1c13d3269",
                        "label": "Bol Bol Rani",
                        "link": "/movies/bol-bol-rani/ET00505203"
                    },
                    {
                        "uuid": "37d4d2df-ecd8-3508-bdb1-3e9a68611347",
                        "label": "Dastaar",
                        "link": "/movies/dastaar/ET00499898"
                    },
                    {
                        "uuid": "6537c615-8fa8-38d8-af1d-914af8959533",
                        "label": "Mastini Pathshala 2",
                        "link": "/movies/mastini-pathshala-2/ET00504624"
                    },
                    {
                        "uuid": "b3ff909f-4ad0-3ec0-91bf-ae5f6f9d08d5",
                        "label": "Arjunan Per Paththu",
                        "link": "/movies/arjunan-per-paththu/ET00507074"
                    },
                    {
                        "uuid": "b4354fd9-b9e5-3c6c-af0d-7650fb8bb27d",
                        "label": "Sarvantaryami",
                        "link": "/movies/sarvantaryami/ET00505647"
                    },
                    {
                        "uuid": "59168d5f-2881-336b-ae03-0a86b2deaa53",
                        "label": "Achyuta Avataaram",
                        "link": "/movies/achyuta-avataaram/ET00506996"
                    },
                    {
                        "uuid": "fd05a00a-5393-3fe7-8e37-c535c2bd5c78",
                        "label": "Brahmarshi Patriji",
                        "link": "/movies/brahmarshi-patriji/ET00507554"
                    }
                ]
            },
            {
                "uuid": "4b438731-0a4e-3d0f-9db7-f03479f2f3b4",
                "heading": "Movies By Genre",
                "items": [
                    {
                        "uuid": "433d8c19-c7c9-3d5a-837e-a91bd535c177",
                        "label": "Drama Movies",
                        "link": "/explore/drama-movies-chennai"
                    },
                    {
                        "uuid": "3308b9f2-7aa4-3553-8b83-11ba1cc44ed9",
                        "label": "Comedy Movies",
                        "link": "/explore/comedy-movies-chennai"
                    },
                    {
                        "uuid": "4c4f0e1b-a63f-37e4-9c84-5bb048d07375",
                        "label": "Action Movies",
                        "link": "/explore/action-movies-chennai"
                    },
                    {
                        "uuid": "1414a60b-db37-391f-ba8f-9fe7e2ee270c",
                        "label": "Thriller Movies",
                        "link": "/explore/thriller-movies-chennai"
                    },
                    {
                        "uuid": "8576262c-5df1-37b1-a1ba-04c89e2d2627",
                        "label": "Romantic Movies",
                        "link": "/explore/romantic-movies-chennai"
                    },
                    {
                        "uuid": "e8e45a80-ffad-35bd-bc1c-07ef49b76eee",
                        "label": "Adventure Movies",
                        "link": "/explore/adventure-movies-chennai"
                    },
                    {
                        "uuid": "1d5313c2-4ee6-3e91-9819-ff1f8dfa38e6",
                        "label": "Family Movies",
                        "link": "/explore/family-movies-chennai"
                    },
                    {
                        "uuid": "86b43fab-8f4f-37fa-a8f6-9a2fed3eeabc",
                        "label": "Horror Movies",
                        "link": "/explore/horror-movies-chennai"
                    },
                    {
                        "uuid": "51ac1255-171e-36f2-b6bb-05ee1a5e04e0",
                        "label": "Animation Movies",
                        "link": "/explore/animation-movies-chennai"
                    },
                    {
                        "uuid": "9f32f82f-672d-3f2e-b65e-83f3f69f1e3a",
                        "label": "Mystery Movies",
                        "link": "/explore/mystery-movies-chennai"
                    },
                    {
                        "uuid": "c0119feb-c6d9-34da-84d7-0d37af8dc82e",
                        "label": "Crime Movies",
                        "link": "/explore/crime-movies-chennai"
                    },
                    {
                        "uuid": "f7863dde-f306-37e2-8b62-4daee5d78004",
                        "label": "Fantasy Movies",
                        "link": "/explore/fantasy-movies-chennai"
                    },
                    {
                        "uuid": "397312b6-db59-3dae-9e9a-f5c6ff7dab73",
                        "label": "Psychological Movies",
                        "link": "/explore/psychological-movies-chennai"
                    },
                    {
                        "uuid": "5048350b-65e0-3514-8dcf-d7000720d8a8",
                        "label": "Adult Movies",
                        "link": "/explore/adult-movies-chennai"
                    },
                    {
                        "uuid": "3026f4c8-9000-3c19-8965-56a502847a77",
                        "label": "Adaptation Movies",
                        "link": "/explore/adaptation-movies-chennai"
                    },
                    {
                        "uuid": "f55c5b48-9fd2-30f8-a88d-e1f17b44fa83",
                        "label": "Heist Movies",
                        "link": "/explore/heist-movies-chennai"
                    },
                    {
                        "uuid": "e1517a9c-b87e-3bc0-bf4d-969dbef42537",
                        "label": "Musical Movies",
                        "link": "/explore/musical-movies-chennai"
                    },
                    {
                        "uuid": "13a4d64d-9fc0-3d83-aa74-4a2162ae0d0d",
                        "label": "War Movies",
                        "link": "/explore/war-movies-chennai"
                    },
                    {
                        "uuid": "f05facdb-ac7f-37d8-8a01-2627edc8d2ce",
                        "label": "Anime Movies",
                        "link": "/explore/anime-movies-chennai"
                    },
                    {
                        "uuid": "6472d8f8-3168-3641-9b2b-f10f8c825789",
                        "label": "Devotional Movies",
                        "link": "/explore/devotional-movies-chennai"
                    },
                    {
                        "uuid": "da29511f-88d8-3305-9ac9-5bbe8c072e19",
                        "label": "Political Movies",
                        "link": "/explore/political-movies-chennai"
                    },
                    {
                        "uuid": "c1ff6f76-a645-3838-a169-ce6e6b933375",
                        "label": "Screening Movies",
                        "link": "/explore/Screening-movies-chennai"
                    },
                    {
                        "uuid": "4415929c-6ddb-305b-8406-543612799a5a",
                        "label": "Sports Movies",
                        "link": "/explore/sports-movies-chennai"
                    },
                    {
                        "uuid": "91a23a23-aded-3c64-b58b-edfa7a87325e",
                        "label": "Biography Movies",
                        "link": "/explore/biography-movies-chennai"
                    },
                    {
                        "uuid": "6d36358c-0af7-312b-a303-2e4386a5ae7a",
                        "label": "Noir Movies",
                        "link": "/explore/noir-movies-chennai"
                    },
                    {
                        "uuid": "83ceccff-7f14-39ba-a969-85df1743aa66",
                        "label": "Tragedy Movies",
                        "link": "/explore/tragedy-movies-chennai"
                    },
                    {
                        "uuid": "366a5b47-7da7-32b1-a244-42e8e6bcd6c8",
                        "label": "Bollywood Movies",
                        "link": "/explore/bollywood-movies-chennai"
                    },
                    {
                        "uuid": "9cb80d30-feaa-3885-9b7c-61da094d0668",
                        "label": "Sci-Fi Movies",
                        "link": "/explore/sci-fi-movies-chennai"
                    },
                    {
                        "uuid": "1cd5b949-1958-394e-8854-55e35b999a07",
                        "label": "Suspense Movies",
                        "link": "/explore/suspense-movies-chennai"
                    },
                    {
                        "uuid": "402f479d-fa8c-3aa5-9b92-280b8383b59d",
                        "label": "Classic Movies",
                        "link": "/explore/classic-movies-chennai"
                    },
                    {
                        "uuid": "28f629b9-22e3-37db-b6bd-ced14045b6a3",
                        "label": "Historical Movies",
                        "link": "/explore/historical-movies-chennai"
                    },
                    {
                        "uuid": "a7efe4d6-2ff3-36e5-bacc-fd9754a50275",
                        "label": "Supernatural Movies",
                        "link": "/explore/supernatural-movies-chennai"
                    },
                    {
                        "uuid": "3a8385d2-f3ec-34d2-9130-9eacc9f17ecd",
                        "label": "Period Movies",
                        "link": "/explore/period-movies-chennai"
                    },
                    {
                        "uuid": "31b16d2c-b311-3062-96c6-7b232e4ac3c5",
                        "label": "Mythological Movies",
                        "link": "/explore/mythological-movies-chennai"
                    }
                ]
            },
            {
                "uuid": "a99c45c9-01b1-3930-b5dc-5c7d70fc59b0",
                "heading": "Movies By Language",
                "items": [
                    {
                        "uuid": "805f443d-4e95-3371-a2d0-b79468ef3257",
                        "label": "Movies in English",
                        "link": "/explore/english-movies-chennai"
                    },
                    {
                        "uuid": "481797de-b291-3439-9cc1-124278e71602",
                        "label": "Movies in Tamil",
                        "link": "/explore/tamil-movies-chennai"
                    },
                    {
                        "uuid": "f614045f-3165-3159-9ad5-af07b734584f",
                        "label": "Movies in Hindi",
                        "link": "/explore/hindi-movies-chennai"
                    },
                    {
                        "uuid": "1e9592ab-4ac9-3615-8f7d-941129198d4f",
                        "label": "Movies in Telugu",
                        "link": "/explore/telugu-movies-chennai"
                    },
                    {
                        "uuid": "76168ea7-023e-3048-b6e4-4c80a65aa82f",
                        "label": "Movies in Malayalam",
                        "link": "/explore/malayalam-movies-chennai"
                    },
                    {
                        "uuid": "57c136ba-2dc4-3ff1-bec9-54acca218410",
                        "label": "Movies in Kannada",
                        "link": "/explore/kannada-movies-chennai"
                    },
                    {
                        "uuid": "db52ae0e-960e-3d53-abf7-ff1dfd509807",
                        "label": "Movies in Sindhi",
                        "link": "/explore/sindhi-movies-chennai"
                    },
                    {
                        "uuid": "c7228fb2-9c2b-3a82-83f5-f072f977996c",
                        "label": "Movies in Bengali",
                        "link": "/explore/bengali-movies-chennai"
                    },
                    {
                        "uuid": "c456db45-eecb-3226-9e8a-2201d9a38b5a",
                        "label": "Movies in Flemish",
                        "link": "/explore/flemish-movies-chennai"
                    },
                    {
                        "uuid": "6e77864e-619d-3dd8-98ee-501ee599f605",
                        "label": "Movies in Chattisgarhi",
                        "link": "/explore/chattisgarhi-movies-chennai"
                    },
                    {
                        "uuid": "58d62f79-f414-3fd3-9d90-7b6af63f0c96",
                        "label": "Movies in French",
                        "link": "/explore/french-movies-chennai"
                    },
                    {
                        "uuid": "426f4474-17e7-3946-984e-95b1174c8357",
                        "label": "Movies in Portuguese",
                        "link": "/explore/portuguese-movies-chennai"
                    },
                    {
                        "uuid": "52472d3e-46fe-338d-9255-07dc866a8d51",
                        "label": "Movies in Magahi",
                        "link": "/explore/magahi-movies-chennai"
                    },
                    {
                        "uuid": "dc206fa4-b5b3-3fad-80b6-14ff26ef9752",
                        "label": "Movies in Assamese",
                        "link": "/explore/assamese-movies-chennai"
                    },
                    {
                        "uuid": "6304e288-b110-358e-a5c5-60ed73d819c9",
                        "label": "Movies in Gujarati",
                        "link": "/explore/gujarati-movies-chennai"
                    },
                    {
                        "uuid": "c709819b-e79a-37c1-85d2-2828540505d8",
                        "label": "Movies in Rajasthani",
                        "link": "/explore/rajasthani-movies-chennai"
                    },
                    {
                        "uuid": "5181ca21-50dc-36aa-bc4e-c431f674f015",
                        "label": "Movies in Konkani",
                        "link": "/explore/konkani-movies-chennai"
                    },
                    {
                        "uuid": "db008807-a5ec-3777-9f42-7200c8ddc03a",
                        "label": "Movies in Afrikaans",
                        "link": "/explore/afrikaans-movies-chennai"
                    },
                    {
                        "uuid": "b07a7085-46df-35db-8dcd-c7f9c93defaa",
                        "label": "Movies in Khasi",
                        "link": "/explore/khasi-movies-chennai"
                    },
                    {
                        "uuid": "7aa1fcb8-33b4-393c-a0a0-fa6726c44e73",
                        "label": "Movies in Odia",
                        "link": "/explore/oriya-movies-chennai"
                    },
                    {
                        "uuid": "ddcb06d0-c452-3d27-baed-6b594f9aa4d7",
                        "label": "Movies in Tulu",
                        "link": "/explore/tulu-movies-chennai"
                    },
                    {
                        "uuid": "4db9ef5d-aa0d-3829-b230-26a1299dc7bc",
                        "label": "Movies in Japanese",
                        "link": "/explore/japanese-movies-chennai"
                    },
                    {
                        "uuid": "0242c0c2-4b9b-3339-8b03-43487a042ea5",
                        "label": "Movies in Multi Language",
                        "link": "/explore/multi-language-movies-chennai"
                    },
                    {
                        "uuid": "875aa868-1c23-37bf-afcc-451ce569ac3b",
                        "label": "Movies in English 7D",
                        "link": "/explore/english-7d-movies-chennai"
                    },
                    {
                        "uuid": "e1bc2486-f6f6-3c94-9e47-158f499f3ebf",
                        "label": "Movies in Bhojpuri",
                        "link": "/explore/bhojpuri-movies-chennai"
                    },
                    {
                        "uuid": "f495109f-2bfd-3d69-846c-88eed379ee53",
                        "label": "Movies in Nepali",
                        "link": "/explore/nepali-movies-chennai"
                    },
                    {
                        "uuid": "6ba187bd-fe78-3f43-9232-6ccf00e51c3f",
                        "label": "Movies in Marathi",
                        "link": "/explore/marathi-movies-chennai"
                    },
                    {
                        "uuid": "60949ed5-03bf-3bb6-ac95-aa65796e0254",
                        "label": "Movies in Haryanavi",
                        "link": "/explore/haryanavi-movies-chennai"
                    },
                    {
                        "uuid": "91210e9c-32cd-321b-ac0d-a2492abf811b",
                        "label": "Movies in Punjabi",
                        "link": "/explore/punjabi-movies-chennai"
                    },
                    {
                        "uuid": "08e60d9b-fbea-3898-b461-bb921f8f572c",
                        "label": "Movies in Urdu",
                        "link": "/explore/urdu-movies-chennai"
                    }
                ]
            },
            {
                "uuid": "04579e21-1630-3395-b125-60b37e21c561",
                "heading": "English Movies By Genre",
                "items": [
                    {
                        "uuid": "759c3837-c060-3206-ae85-07615059257c",
                        "label": "English Adult Movies",
                        "link": "/explore/english-adult-movies-chennai"
                    },
                    {
                        "uuid": "fd3d7951-467d-3afb-a924-3c7513417c83",
                        "label": "English Adaptation Movies",
                        "link": "/explore/english-adaptation-movies-chennai"
                    },
                    {
                        "uuid": "b81c9a99-09e7-37b1-a996-79c7184c1d51",
                        "label": "English Heist Movies",
                        "link": "/explore/english-heist-movies-chennai"
                    },
                    {
                        "uuid": "ddb6829f-42a9-3a29-9d61-821faeb7b32d",
                        "label": "English Musical Movies",
                        "link": "/explore/english-musical-movies-chennai"
                    },
                    {
                        "uuid": "b335a7d2-7714-32ce-8fbe-9d81f00cf3bf",
                        "label": "English Horror Movies",
                        "link": "/explore/english-horror-movies-chennai"
                    },
                    {
                        "uuid": "acc7354c-74ab-3349-a27f-5bf50a525734",
                        "label": "English Mystery Movies",
                        "link": "/explore/english-mystery-movies-chennai"
                    },
                    {
                        "uuid": "cd830dd5-fba6-3021-aaa9-0fe91ddfe5e7",
                        "label": "English War Movies",
                        "link": "/explore/english-war-movies-chennai"
                    },
                    {
                        "uuid": "dffe52d7-e750-3e33-88e1-18ea999b4817",
                        "label": "English Anime Movies",
                        "link": "/explore/english-anime-movies-chennai"
                    },
                    {
                        "uuid": "3372aaee-ed81-3d5e-9922-9f665033bc23",
                        "label": "English Devotional Movies",
                        "link": "/explore/english-devotional-movies-chennai"
                    },
                    {
                        "uuid": "c0098621-8a01-352d-bc3b-2cf222b49899",
                        "label": "English Family Movies",
                        "link": "/explore/english-family-movies-chennai"
                    },
                    {
                        "uuid": "3f973d29-5c69-31e4-8dd9-7be0dbb79961",
                        "label": "English Comedy Movies",
                        "link": "/explore/english-comedy-movies-chennai"
                    },
                    {
                        "uuid": "bf41b9fb-ac8e-3840-9c82-08db2c93039a",
                        "label": "English Crime Movies",
                        "link": "/explore/english-crime-movies-chennai"
                    },
                    {
                        "uuid": "7fa961c5-7ad5-31f0-8d7d-e475e39af873",
                        "label": "English Political Movies",
                        "link": "/explore/english-political-movies-chennai"
                    },
                    {
                        "uuid": "e58f9917-5557-3ce0-878b-ed6f6dd8d5d3",
                        "label": "English Screening Movies",
                        "link": "/explore/english-Screening-movies-chennai"
                    },
                    {
                        "uuid": "8dd21f5c-40d6-3341-b47f-d1028e947617",
                        "label": "English Sports Movies",
                        "link": "/explore/english-sports-movies-chennai"
                    },
                    {
                        "uuid": "1af27263-e108-3af1-a1cf-b91899072325",
                        "label": "English Animation Movies",
                        "link": "/explore/english-animation-movies-chennai"
                    },
                    {
                        "uuid": "9c232368-b24c-3d26-aee6-ef628c33b9c8",
                        "label": "English Biography Movies",
                        "link": "/explore/english-biography-movies-chennai"
                    },
                    {
                        "uuid": "10acc8d9-adce-3afc-82fe-2a460d7850f8",
                        "label": "English Fantasy Movies",
                        "link": "/explore/english-fantasy-movies-chennai"
                    },
                    {
                        "uuid": "1a2cca52-0ba8-3ebf-a9cb-c8e4261e85aa",
                        "label": "English Action Movies",
                        "link": "/explore/english-action-movies-chennai"
                    },
                    {
                        "uuid": "7c01a59c-de64-3704-9332-20c7e0c6757e",
                        "label": "English Psychological Movies",
                        "link": "/explore/english-psychological-movies-chennai"
                    },
                    {
                        "uuid": "140031d8-7742-33a2-ad41-5d92aad8315d",
                        "label": "English Bollywood Movies",
                        "link": "/explore/english-bollywood-movies-chennai"
                    },
                    {
                        "uuid": "0a1b0dc2-3941-308d-bf56-c2e0e9dd4c56",
                        "label": "English Sci-Fi Movies",
                        "link": "/explore/english-sci-fi-movies-chennai"
                    },
                    {
                        "uuid": "dac73ee2-9936-3a1f-bdab-d785f3a98444",
                        "label": "English Suspense Movies",
                        "link": "/explore/english-suspense-movies-chennai"
                    },
                    {
                        "uuid": "7080bf8a-f9c7-3995-9521-ce1d802eca63",
                        "label": "English Adventure Movies",
                        "link": "/explore/english-adventure-movies-chennai"
                    },
                    {
                        "uuid": "fdafb023-9a22-3df5-9ce5-bcc8b3a1152f",
                        "label": "English Classic Movies",
                        "link": "/explore/english-classic-movies-chennai"
                    },
                    {
                        "uuid": "38ec9967-11da-3bb6-a8c1-c355a180f816",
                        "label": "English Historical Movies",
                        "link": "/explore/english-historical-movies-chennai"
                    },
                    {
                        "uuid": "bf25ec5b-c063-331c-819f-c831d56e8c78",
                        "label": "English Supernatural Movies",
                        "link": "/explore/english-supernatural-movies-chennai"
                    },
                    {
                        "uuid": "1a876996-3060-38fb-93a7-982adc55caa6",
                        "label": "English Period Movies",
                        "link": "/explore/english-period-movies-chennai"
                    },
                    {
                        "uuid": "c847e976-894e-3ef0-82ca-25b639242d2a",
                        "label": "English Mythological Movies",
                        "link": "/explore/english-mythological-movies-chennai"
                    },
                    {
                        "uuid": "ae7d7e9b-0a9e-37ab-b0d7-9b7dd6501617",
                        "label": "English Romantic Movies",
                        "link": "/explore/english-romantic-movies-chennai"
                    },
                    {
                        "uuid": "b2cee95b-4ddd-35c4-942a-a8bcf5d0eaa0",
                        "label": "English Noir Movies",
                        "link": "/explore/english-noir-movies-chennai"
                    },
                    {
                        "uuid": "b2525999-43e8-3673-9641-cef493df5cdb",
                        "label": "English Tragedy Movies",
                        "link": "/explore/english-tragedy-movies-chennai"
                    },
                    {
                        "uuid": "7193238f-a3c6-3e02-9196-32d481ae1eab",
                        "label": "English Drama Movies",
                        "link": "/explore/english-drama-movies-chennai"
                    },
                    {
                        "uuid": "fa2bb277-f91d-3de4-af6b-951bd48823cf",
                        "label": "English Thriller Movies",
                        "link": "/explore/english-thriller-movies-chennai"
                    }
                ]
            },
            {
                "uuid": "1f686e5e-8810-3781-beb1-3e9dbb6e5c38",
                "heading": "Upcoming Movies By Genre",
                "items": [
                    {
                        "uuid": "938f8637-f296-36fc-a663-1ebbca2cb63a",
                        "label": "Upcoming Drama Movies",
                        "link": "/explore/drama-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "afa108d3-7041-360a-b326-4fc1b4a16644",
                        "label": "Upcoming Thriller Movies",
                        "link": "/explore/thriller-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "43e24093-a817-3e1b-bc6d-d41413c59dee",
                        "label": "Upcoming Action Movies",
                        "link": "/explore/action-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "91531869-f008-3135-a10f-ca452875dc46",
                        "label": "Upcoming Comedy Movies",
                        "link": "/explore/comedy-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "b0bc0bd9-bf92-312f-b391-2a08d05cccf7",
                        "label": "Upcoming Romantic Movies",
                        "link": "/explore/romantic-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "8424e12a-2695-3115-8c71-16040f040e66",
                        "label": "Upcoming Crime Movies",
                        "link": "/explore/crime-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "61c10463-c516-3d8d-90a1-eab1266fb4e3",
                        "label": "Upcoming Adventure Movies",
                        "link": "/explore/adventure-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "dd60de7e-fca2-3ad2-9eb0-82b7f4cb62e9",
                        "label": "Upcoming Horror Movies",
                        "link": "/explore/horror-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "0329f705-c025-3348-b657-b3bb5c5fbbbf",
                        "label": "Upcoming Family Movies",
                        "link": "/explore/family-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "d0bd811f-ef7e-38e0-bae6-66192a2880b6",
                        "label": "Upcoming Fantasy Movies",
                        "link": "/explore/fantasy-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "b2413b96-c9a5-3316-979c-b015fbfe3687",
                        "label": "Upcoming Mystery Movies",
                        "link": "/explore/mystery-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "8b3bedf1-cb15-3f23-baca-f261ef1a965d",
                        "label": "Upcoming Suspense Movies",
                        "link": "/explore/suspense-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "9e86ac4a-951d-3709-aa91-48591a9769c7",
                        "label": "Upcoming Biography Movies",
                        "link": "/explore/biography-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "929b0e51-bc1a-3576-84fe-164bf1dd7022",
                        "label": "Upcoming Period Movies",
                        "link": "/explore/period-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "c9c9e0d2-2892-35d4-9488-c7a2b8c409cd",
                        "label": "Upcoming Sci-Fi Movies",
                        "link": "/explore/sci-fi-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "53bcdee8-fd2e-35ed-ad4f-5226c198fad8",
                        "label": "Upcoming Historical Movies",
                        "link": "/explore/historical-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "0f19242c-336c-3c40-9a98-fa7328360c3d",
                        "label": "Upcoming Animation Movies",
                        "link": "/explore/animation-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "f36e08df-0c6f-3e31-ad37-e9c609c48fa7",
                        "label": "Upcoming Mythological Movies",
                        "link": "/explore/mythological-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "9b3be55c-c577-31dd-b427-5d59fa58bd97",
                        "label": "Upcoming Psychological Movies",
                        "link": "/explore/psychological-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "1be11997-2181-33b1-858c-b2314fe31c0f",
                        "label": "Upcoming Musical Movies",
                        "link": "/explore/musical-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "fbdb6592-25cd-3e86-93c5-c373fcef4e0c",
                        "label": "Upcoming Political Movies",
                        "link": "/explore/political-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "29827b17-7893-35f6-90f1-afa02ac85572",
                        "label": "Upcoming Sports Movies",
                        "link": "/explore/sports-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "8d7940f0-7fa9-3393-a6ce-35b53a247620",
                        "label": "Upcoming Supernatural Movies",
                        "link": "/explore/supernatural-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "60a4898d-b286-317b-9158-5bce6bebc4bb",
                        "label": "Upcoming War Movies",
                        "link": "/explore/war-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "1d2bf813-d8a7-3c4b-945b-5dfc25737f54",
                        "label": "Upcoming Adult Movies",
                        "link": "/explore/adult-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "f5ad4222-f56f-3edf-a7ce-9ad5d90180a2",
                        "label": "Upcoming Adaptation Movies",
                        "link": "/explore/adaptation-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "e78f9e28-7b2f-3115-ac58-14dc8b1d34fc",
                        "label": "Upcoming Heist Movies",
                        "link": "/explore/heist-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "f5138e57-5606-3e85-92a2-10dc0ee18601",
                        "label": "Upcoming Anime Movies",
                        "link": "/explore/anime-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "451500ca-7d62-3344-a897-e97219282df2",
                        "label": "Upcoming Devotional Movies",
                        "link": "/explore/devotional-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "cc048288-43a9-3532-87d6-0cff5ccdfe8c",
                        "label": "Upcoming Screening Movies",
                        "link": "/explore/Screening-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "afaede9a-6624-3aab-b1ce-cea2eff0672d",
                        "label": "Upcoming Bollywood Movies",
                        "link": "/explore/bollywood-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "d8227035-63f3-3c76-8e6c-23de44fe0c39",
                        "label": "Upcoming Classic Movies",
                        "link": "/explore/classic-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "7a4b8c8f-a691-37db-9cf0-dff3d9102d57",
                        "label": "Upcoming Noir Movies",
                        "link": "/explore/noir-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "e4a937fa-4f29-34af-8149-deef014f0c1e",
                        "label": "Upcoming Tragedy Movies",
                        "link": "/explore/tragedy-upcoming-movies-chennai"
                    }
                ]
            },
            {
                "uuid": "b91ff705-119d-3ec8-b454-76d07f044b64",
                "heading": "Upcoming Movies By Language",
                "items": [
                    {
                        "uuid": "df359bff-dfbb-3777-8a2f-d7f0d9b87a47",
                        "label": " Upcoming Movies in Hindi",
                        "link": "/explore/hindi-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "0b04e322-ecf1-3acf-8e47-3f819abc29cc",
                        "label": " Upcoming Movies in Telugu",
                        "link": "/explore/telugu-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "c781dfcd-8bf5-3263-ae75-7b57c7700408",
                        "label": " Upcoming Movies in Tamil",
                        "link": "/explore/tamil-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "e42f3728-830a-3737-b30e-2ea5a1b6fecb",
                        "label": " Upcoming Movies in Kannada",
                        "link": "/explore/kannada-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "e6d2d5eb-c8c7-3d3f-99bc-283321302ffb",
                        "label": " Upcoming Movies in Malayalam",
                        "link": "/explore/malayalam-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "9154b61f-b33f-3197-be91-d737aabe5ad0",
                        "label": " Upcoming Movies in English",
                        "link": "/explore/english-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "fff99cb3-cc45-391b-bd9a-09db3dbc913b",
                        "label": " Upcoming Movies in Bengali",
                        "link": "/explore/bengali-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "8691db0f-4307-3ed2-a829-9e2417a18ab5",
                        "label": " Upcoming Movies in Marathi",
                        "link": "/explore/marathi-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "f9e9d649-65ba-3004-97ac-d927e832a57b",
                        "label": " Upcoming Movies in Punjabi",
                        "link": "/explore/punjabi-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "e5d1e25c-f05b-3b09-b42d-b6e2893abfee",
                        "label": " Upcoming Movies in Gujarati",
                        "link": "/explore/gujarati-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "f0f7e445-b6b7-3e76-8007-cbf6a36a6ed3",
                        "label": " Upcoming Movies in Odia",
                        "link": "/explore/oriya-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "8f97aece-6d15-38a5-b961-8f31b3010d54",
                        "label": " Upcoming Movies in Bhojpuri",
                        "link": "/explore/bhojpuri-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "cafc2dd7-a060-3bfc-9520-570ed80f9313",
                        "label": " Upcoming Movies in Khasi",
                        "link": "/explore/khasi-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "bdc53cc7-76b2-3ffa-bb91-f6022db02af2",
                        "label": " Upcoming Movies in Japanese",
                        "link": "/explore/japanese-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "97f87158-6b80-3220-b50f-102ff9491a4c",
                        "label": " Upcoming Movies in Tulu",
                        "link": "/explore/tulu-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "3a70a4f4-f70a-37da-8451-94b2fedf8fab",
                        "label": " Upcoming Movies in Chattisgarhi",
                        "link": "/explore/chattisgarhi-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "14a5e2ea-c56f-37d3-8d43-2a564cc941df",
                        "label": " Upcoming Movies in Magahi",
                        "link": "/explore/magahi-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "b67683dd-cc9a-35b7-a484-e37ad53b1ae7",
                        "label": " Upcoming Movies in Assamese",
                        "link": "/explore/assamese-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "18e2880b-b934-3dbd-bc5d-ad212d9572a6",
                        "label": " Upcoming Movies in Rajasthani",
                        "link": "/explore/rajasthani-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "627c08fe-2a69-35b7-8369-4076a679734b",
                        "label": " Upcoming Movies in Konkani",
                        "link": "/explore/konkani-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "ac4bd261-c317-3598-86c3-d951eb2ae573",
                        "label": " Upcoming Movies in Afrikaans",
                        "link": "/explore/afrikaans-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "87839693-db0a-39f0-a906-62fe168ee38d",
                        "label": " Upcoming Movies in Sindhi",
                        "link": "/explore/sindhi-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "50dd5cbc-18c8-32ed-ad27-1ceb775b5cd1",
                        "label": " Upcoming Movies in Flemish",
                        "link": "/explore/flemish-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "e7647858-32a0-3a98-a40c-25ddd5dd53bc",
                        "label": " Upcoming Movies in French",
                        "link": "/explore/french-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "04249e05-c3b0-3d52-8447-9210c503588f",
                        "label": " Upcoming Movies in Portuguese",
                        "link": "/explore/portuguese-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "80fad6f8-8699-3305-8341-df9d2e10ef1a",
                        "label": " Upcoming Movies in Multi Language",
                        "link": "/explore/multi-language-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "64904408-3ebe-3385-937a-2f1e0f4e3ece",
                        "label": " Upcoming Movies in English 7D",
                        "link": "/explore/english-7d-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "120e2b4b-4f1d-3726-9584-19dc1a9e688e",
                        "label": " Upcoming Movies in Nepali",
                        "link": "/explore/nepali-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "fff47e85-bf33-346f-8d70-7a513019843e",
                        "label": " Upcoming Movies in Haryanavi",
                        "link": "/explore/haryanavi-upcoming-movies-chennai"
                    },
                    {
                        "uuid": "66e0eaf5-820e-31c3-9f23-efba8825f41b",
                        "label": " Upcoming Movies in Urdu",
                        "link": "/explore/urdu-upcoming-movies-chennai"
                    }
                ]
            },
            {
                "uuid": "2a755117-ca2e-35e5-bd43-7f1bbbc5591b",
                "heading": "Movies By Format",
                "items": [
                    {
                        "uuid": "4af1366f-5303-3b66-8762-d84967ad3f1d",
                        "label": "Movies in 2D",
                        "link": "/explore/2d-movies-chennai"
                    },
                    {
                        "uuid": "73ef3dac-e24c-3e44-b638-f7d0d1d1dc90",
                        "label": "Movies in 3D",
                        "link": "/explore/3d-movies-chennai"
                    },
                    {
                        "uuid": "f2ad8934-82d7-3c3c-9734-0b9f3d866ab7",
                        "label": "Movies in 4DX",
                        "link": "/explore/4dx-movies-chennai"
                    },
                    {
                        "uuid": "57f495fb-431a-38ed-b5e0-44f0a012a9fa",
                        "label": "Movies in MX4D 3D",
                        "link": "/explore/mx4d-3d-movies-chennai"
                    },
                    {
                        "uuid": "82cfbb71-3176-3f89-8217-417b3f84a8c5",
                        "label": "Movies in IMAX 2D",
                        "link": "/explore/imax-2d-movies-chennai"
                    },
                    {
                        "uuid": "6e17e7ba-140b-39a8-98e2-73f0d824d2d6",
                        "label": "Movies in 4DX 3D",
                        "link": "/explore/4dx-3d-movies-chennai"
                    },
                    {
                        "uuid": "cd3f66ea-6221-38b2-86c4-efa1eeb9e6d3",
                        "label": "Movies in DOLBY CINEMA 2D",
                        "link": "/explore/dolby-cinema-2d-movies-chennai"
                    },
                    {
                        "uuid": "7665fb59-f093-3c5a-b757-541601801ec6",
                        "label": "Movies in 3D SCREEN X",
                        "link": "/explore/3d-screen-x-movies-chennai"
                    },
                    {
                        "uuid": "36170edb-a6e3-34a6-89c5-3f1db25b13d8",
                        "label": "Movies in 2D SCREEN X",
                        "link": "/explore/2d-screen-x-movies-chennai"
                    },
                    {
                        "uuid": "fa2b4be5-8b6d-3433-b35e-48bac420e5ce",
                        "label": "Movies in 7D",
                        "link": "/explore/7d-movies-chennai"
                    },
                    {
                        "uuid": "77c50e99-eb03-3a2b-a2c3-3b64bdfca199",
                        "label": "Movies in ICE",
                        "link": "/explore/ice-movies-chennai"
                    },
                    {
                        "uuid": "9f437074-31bd-3d36-86a4-0a4d4176f7a6",
                        "label": "Movies in HOUSEFULL 5A",
                        "link": "/explore/housefull-5a-movies-chennai"
                    },
                    {
                        "uuid": "59a3c166-91b3-30cd-ad4a-0bd973c641ee",
                        "label": "Movies in Full Dome",
                        "link": "/explore/full-dome-movies-chennai"
                    },
                    {
                        "uuid": "1934758e-12f3-3f19-a69b-3af09ba522f6",
                        "label": "Movies in IMAX 3D",
                        "link": "/explore/imax-3d-movies-chennai"
                    },
                    {
                        "uuid": "e80cfc13-44e7-3efc-bb93-df4d1fd494f9",
                        "label": "Movies in DOLBY CINEMA 3D",
                        "link": "/explore/dolby-cinema-3d-movies-chennai"
                    },
                    {
                        "uuid": "32658c17-5f04-3340-b3a9-155bfd80a70f",
                        "label": "Movies in HOUSEFULL 5B",
                        "link": "/explore/housefull-5b-movies-chennai"
                    },
                    {
                        "uuid": "4df9f58e-c215-3c21-bfe0-a678ab39f7eb",
                        "label": "Movies in MX4D",
                        "link": "/explore/mx4d-movies-chennai"
                    }
                ]
            },
            {
                "uuid": "bdf1b6ca-522c-3eea-ac0b-539bcdcad76d",
                "heading": "COUNTRIES",
                "items": [
                    {
                        "uuid": "9739b97c-dece-3d25-8237-7a8098b9dbe6",
                        "label": "Indonesia",
                        "link": "https://id.bookmyshow.com/"
                    },
                    {
                        "uuid": "d5816dfa-5f90-3f41-bd82-de0880b7c15b",
                        "label": "Singapore",
                        "link": "https://sg.bookmyshow.com/"
                    },
                    {
                        "uuid": "be74fb7d-c9e8-3de7-9ee6-ef9dcc2d51ff",
                        "label": "Sri Lanka",
                        "link": "https://lk.bookmyshow.com/"
                    },
                    {
                        "uuid": "415c802e-3be6-3be6-91d2-268165272ad2",
                        "label": "West Indies",
                        "link": "https://wi.bookmyshow.com/"
                    }
                ]
            },
            {
                "uuid": "dd419bfa-b7b3-3741-84af-fc17d0a09889",
                "heading": "HELP",
                "items": [
                    {
                        "uuid": "47c9b84a-a12a-3a28-a451-5f69b5ddb2ee",
                        "label": "About Us",
                        "link": "/aboutus"
                    },
                    {
                        "uuid": "4dcbf350-ad02-3650-864a-2410689661e5",
                        "label": "Contact Us",
                        "link": "/contactus"
                    },
                    {
                        "uuid": "3b2747e2-b607-35e4-9903-06820557056e",
                        "label": "Current Opening",
                        "link": "/careers/"
                    },
                    {
                        "uuid": "010c4ffb-da52-3a7f-b2e0-e82cc0413856",
                        "label": "Press Release",
                        "link": "/press-release"
                    },
                    {
                        "uuid": "93f28833-4d19-34fd-ae73-8cb0fd99deee",
                        "label": "Press Coverage",
                        "link": "/press-coverage"
                    },
                    {
                        "uuid": "7120da49-fc0e-3b5c-9d6f-0c8f1339ae86",
                        "label": "FAQs",
                        "link": "/faq"
                    },
                    {
                        "uuid": "fc8f8f72-57a2-395a-a8e8-79b0714f3ae9",
                        "label": "Terms and Conditions",
                        "link": "/terms-and-conditions"
                    },
                    {
                        "uuid": "7e8dc23b-5b2c-3e65-b0c5-1fb86d3149c6",
                        "label": "Privacy Policy",
                        "link": "/privacy"
                    }
                ]
            },
            {
                "uuid": "e33b664d-859c-3a47-97c2-a86e60a84300",
                "heading": "BOOKMYSHOW EXCLUSIVES",
                "items": [
                    {
                        "uuid": "210c2e7c-be6b-3548-897b-3e6aa213aeaf",
                        "label": "Lollapalooza India",
                        "link": "https://lollaindia.com/"
                    },
                    {
                        "uuid": "802685e4-a433-30e1-bd1e-7e19e9859475",
                        "label": "BookAChange",
                        "link": "/donation"
                    },
                    {
                        "uuid": "f362cc21-7508-37f1-92d8-4d0efaa64b58",
                        "label": "Corporate Vouchers",
                        "link": "/voucher"
                    },
                    {
                        "uuid": "85568560-46ac-3a2c-974c-14b4561566e7",
                        "label": "Gift Cards",
                        "link": "/giftcards"
                    },
                    {
                        "uuid": "acbceadf-3fcd-3c77-a87e-cbf1a6e8cfec",
                        "label": "List My Show",
                        "link": "/s/list-your-show/"
                    },
                    {
                        "uuid": "06ff9f25-0451-3185-ad96-a7ba36da6c03",
                        "label": "Offers",
                        "link": "/offers"
                    },
                    {
                        "uuid": "ee0036a5-18c8-3dc6-9b6d-114d43fcc38d",
                        "label": "Stream",
                        "link": "/explore/c/stream"
                    },
                    {
                        "uuid": "ea834d9a-9152-3444-8945-013425669a7b",
                        "label": "Movie Trailers",
                        "link": "/explore/movie-trailers"
                    },
                    {
                        "uuid": "a14a2923-aa9d-395c-881b-a7ef9b11e0bc",
                        "label": "Summer Activities",
                        "link": "/explore/c/summer-activities"
                    }
                ]
            },
            {
                "uuid": "b1509be8-34e2-3730-963b-ed5e1f6b7ea3",
                "heading": "New Year Eve & Christmas Celebration",
                "items": [
                    {
                        "uuid": "74b64cf0-0011-3f21-b689-33a099d10265",
                        "label": "New Year Parties",
                        "link": "/explore/new-year-parties"
                    },
                    {
                        "uuid": "d90e1b4b-8f1c-3f1f-a8da-426af44c5736",
                        "label": "Christmas",
                        "link": "/explore/christmas-celebrations"
                    },
                    {
                        "uuid": "c0df9b00-5a4d-3c3f-8519-824492a570fb",
                        "label": "Dinner Experience",
                        "link": "/explore/nye-dinner-experience"
                    },
                    {
                        "uuid": "ba3e0842-3f0e-37ad-ac45-24a7361b9eba",
                        "label": "Live Performances",
                        "link": "/explore/nye-live-performances"
                    },
                    {
                        "uuid": "0b2f6da6-fc15-34ff-9ad3-97f2753c49cf",
                        "label": "Nature Trails",
                        "link": "/explore/nye-nature-trails"
                    },
                    {
                        "uuid": "8511e380-a3b9-37fb-8f00-23c7f24c8319",
                        "label": "Staycation",
                        "link": "/explore/nye-staycation"
                    },
                    {
                        "uuid": "cb223e69-1b6d-3ed3-b130-6aea1aeeda2c",
                        "label": "Unique Experiences",
                        "link": "/explore/nye-unique-experiences"
                    }
                ]
            }
        ],
        "faq": []
    },
    "hiddenTags": [],
    "breadcrumb": [
        {
            "uuid": "0ba79464-28ff-34c2-ae88-3893b7a40610",
            "cta": "/explore/home/chennai",
            "item": "Home"
        },
        {
            "uuid": "88689e25-cbf2-3687-9e49-dfcb7cb5105d",
            "cta": "/explore/movies-chennai",
            "item": "Movies in Chennai"
        },
        {
            "uuid": "3eaab8ac-b41a-3781-b954-7ac3d4a6a274",
            "cta": "/explore/hindi-movies",
            "item": "Hindi Movies"
        },
        {
            "uuid": "074cdc6d-1702-3a62-a957-e30dc87b28fe",
            "cta": "/movies/chennai/the-odyssey/buytickets/ET00480917/20260719",
            "item": "The Odyssey"
        }
    ],
    "noFollow": false
}
