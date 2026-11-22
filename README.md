CompetencyAI (version 2)
Competency Gap Analysis and Training Recommendation Platform. Smart India Hackathon 2026 · PS SIH26101 · MoSPI · Group G28.

Start here: docs/VIVA_GUIDE.md explains every part in plain language and has viva questions with answers.

Run it
python -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python -m uvicorn server.app:app --port 8000
Open http://localhost:8000. The website is already built in web/dist.

Develop the website
cd web && npm install && npm run dev     # http://localhost:5173 (needs the server above running)
npm run build                            # rebuild web/dist
Test, retrain
python -m pytest -q                      # 6 API tests
cd ml && python run_all.py               # retrain all AI and rewrite ml/reports (about 4 minutes)
Folders
ml/       all the AI: data cleaning, course tagger, simulator, gap model, ranker, evaluation
server/   FastAPI backend (app.py = routes, engine.py = logic, quizgen.py = quiz maker)
web/      React website (src/pages = screens, src/styles.css = design system)
tests/    automatic API tests
docs/     VIVA_GUIDE.md (+ PDF)
Honest note
The 6,988 courses are real public data. Learners, assessments and expert paths are simulated, because real iGOT data is not public. Results show the pipeline works; they are not accuracy claims for real officials.
