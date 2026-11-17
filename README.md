# Approved Documents Archive (React Native + Expo, with a Python server)

Scan or pick a document approved by the President. The system reads it, suggests the type
(procurement, financial, general internal, general external), the approval date, the offices
and the people involved. A staff member confirms, and the PDF is filed as:

```
Archive/
  2026/
    11-November/
      Procurement/        2026-11-05_purchase-of-laptops_a1b2c3.pdf
      Financial/
      General-Internal/
      General-External/
    10-October/
      ...
```

Everything is searchable by date, type, office, person, or any word in the document.
The model rationale (for your instructor) is in **NARRATIVE.md**.

## Two parts
- **app** (this folder): the phone/browser app with three tabs: Add, Search, Browse.
- **server/**: reads documents, suggests fields, files them, and runs search. Your model plugs in here.

You need both running. The app works with the keyword baseline on day one.

## 1. Start the server
Needs **Python 3.10+**.

```
cd server
setup.bat            (Windows)        ./setup.sh            (Mac/Linux)
```
Then start it (Windows):
```
.venv\Scripts\activate
uvicorn app:app --host 0.0.0.0 --port 8000
```
Mac/Linux: `. .venv/bin/activate` and the same `uvicorn` line.

Digital PDFs work immediately. For **photos and scanned PDFs** also install OCR (once):
```
pip install -r requirements-ocr.txt
```
Files are stored in `server/data/archive/`, and the search index in `server/data/archive.db`.

## 2. Start the app
Needs **Node.js 22 LTS** (20.19+ also works). In the project folder (not `server`):
```
setup.bat            (Windows)        ./setup.sh            (Mac/Linux)
npx expo start
```
Press `w` to open in the browser, or scan the QR code with Expo Go on your phone.

**Expo Go version matching.** Expo Go only opens the SDK it currently supports. If it says the
project is incompatible, run `setup.bat 57` (or the number the message names). Browser mode
(`w`) works with any SDK.

**Phone on Wi-Fi:** edit `.env` (created by setup) and set
`EXPO_PUBLIC_API_URL=http://<your-computer-ip>:8000`, then restart Expo.
Your phone and computer must be on the same network, and the firewall must allow port 8000.

## Using the app
1. **Add:** choose a PDF, or scan pages with the camera (add each page in order). Tap *Read document*.
2. Review the suggestions. Fix anything wrong, switch on *approved and signed by the President*,
   and tap *Save to archive*. The app shows exactly which folder it went to.
3. **Search:** type any words (a name, an office, a topic). Filter by type, year or office.
4. **Browse:** walk the folders Year > Month > Type, the same way they are on disk.

The same file cannot be archived twice (the server detects duplicates).

## Plug in your own model
1. Collect confirmed documents by using the app, then export them:
   `http://localhost:8000/export/training.csv`
   (or start from `server/sample_training.csv` to see the format).
2. Train:
   ```
   pip install -r requirements-ml.txt
   python train_classifier.py my_documents.csv
   ```
   It prints macro-F1 and a confusion matrix, and saves `models/doc-classifier`.
3. Serve it:
   ```
   MODEL_ID=./models/doc-classifier uvicorn app:app --host 0.0.0.0 --port 8000
   ```
   (Windows PowerShell: `$env:MODEL_ID="./models/doc-classifier"` first.)
   The app header changes from "keyword baseline" to "your model".

The model's labels must be `procurement`, `financial`, `general_internal`, `general_external`.

## Where files are stored, and moving to the cloud
All file access goes through `server/storage.py`. Today it uses the local disk. Moving later
means adding one class there, not changing the app. Recommended path:

1. **Now:** local server, with a scheduled backup copy of `server/data/` to an external drive.
2. **Next, if the college already uses Google Workspace:** a Google Drive *Shared Drive* is the
   easiest cloud step (`GoogleDriveStorage` is stubbed in `storage.py`).
3. **Otherwise:** S3-compatible storage. You can run MinIO on your own server first and move
   to a cloud provider later without code changes (`S3Storage` is stubbed).
4. **Domain:** put the server behind HTTPS (for example Caddy or nginx) on a college domain.
   Keep the search index (SQLite) local, and move to PostgreSQL if many staff use it at once.

Because these are confidential records, I would keep the originals on a college-controlled
server until the college decides on its cloud and records policy.

## Security and privacy (please read before using real documents)
- Set a password-like key on the server so only your app can read the archive:
  `ARCHIVE_API_KEY=some-long-random-text` (server) and `EXPO_PUBLIC_API_KEY=` the same text (`.env`).
  Without it, anyone on your network who finds the server can read every document.
- This is a shared key, not user accounts. Before real deployment add staff logins and an
  audit trail of who opened or filed what.
- Use HTTPS when the server is reachable beyond your own computer.
- Documents may contain personal data (RA 10173). Get the office's approval first, and check
  the college records office about retention rules.
- The app records that a staff member confirmed approval. It cannot verify signatures.

## Honest limits
- Suggestions for dates, titles, offices and names come from simple rules and are often right but
  not always. That is why a person confirms before filing. A trained name-recognition model
  could be added later if names are often missed.
- Stamps, signatures and crooked photos reduce OCR quality. Flat, bright, sharp photos work best.
- iPhone HEIC photos are not supported. Use "Most Compatible" in camera settings, or scan to PDF.
- The first release has no editing or deleting of archived documents.
- Verified on Windows: Expo Doctor, TypeScript, the web export, API analysis/save/search/browse/PDF
  retrieval, and photo OCR all passed. Camera permissions and Expo Go behavior still need checking
  on the target phones.

## Troubleshooting
- **"Could not reach ..."**: the server is not running, or the API URL is wrong for your device.
- **"Missing or wrong API key"**: the key in `.env` does not match `ARCHIVE_API_KEY`.
- **"Project is incompatible with this version of Expo Go"**: run setup again with the SDK number it names.
- **Scanned PDF or photo reads nothing**: install OCR (`pip install -r requirements-ocr.txt`) and restart the server.
- **Server will not start with a SQLite/FTS5 error**: install a newer Python from python.org.

## Layout
```
App.tsx, index.js            app shell and tabs
src/api.ts                   talks to the server
src/screens/                 AddScreen, SearchScreen, BrowseScreen
src/components/              Chip, DocumentRow, DocumentDetail
server/app.py                API: analyze, save, search, tree, files, export
server/extract.py            PDF text, OCR, suggested date/title/offices/names
server/classifier.py         keyword baseline, or your model via MODEL_ID
server/storage.py            local disk now; Drive/S3 stubs
server/db.py                 SQLite + full-text search
server/train_classifier.py   fine-tune DistilBERT on your documents
server/config/               document types and the office list (edit offices.json)
scripts/setup.js             installs the app for your Expo SDK
NARRATIVE.md                 model selection narrative
```
