# CompetencyAI: Viva Study Guide

**Project:** Competency Gap Analysis and Training Recommendation Platform
**Problem statement:** Smart India Hackathon 2026, SIH26101, MoSPI (Group G28)

This guide assumes you know nothing about machine learning or web development. Read it once from top to bottom (about 45 minutes), then use sections 11 and 12 to practise. Everything here matches the code in this folder.

---

## 1. The project in 60 seconds

**The problem.** Government officials are told to "upskill", but there are thousands of courses and no way to know which ones fix *their* weak points.

**Our answer.** A website that does what a good teacher would do:

1. **Ask** the person about their skills (a short check, one skill at a time).
2. **Compare** them with what their job role needs.
3. **Find the gaps** and say how big and how sure we are.
4. **Pick a few courses** that close those gaps, in a sensible order, with a week-by-week plan.

**Three small AI models** do the thinking:

| Model | Question it answers | Technique (one line) |
|---|---|---|
| Course tagger | Which skills does each course teach? | Rules plus a keyword-blind logistic regression (weak supervision) |
| Gap model | How far is this person from the job's target, per skill? | Gradient boosting classifier and regressor |
| Ranker | Which courses should they take first? | Gradient boosting that learns to rank (learning to rank) |

Plus a **Quiz maker** that turns any notes or PDF into multiple-choice questions (the problem statement asks for this too). Its score can be fed back as assessment evidence, so quizzes and gap analysis form one loop.

**The one-sentence result.** On 800 unseen simulated learners, our recommender finds 0.475 of the experts' choices in its top 10, while "show the most popular courses" finds 0.008.

---

## 2. What was rebuilt, and what was deliberately kept (say this honestly)

You were right that the previous website had a poor flow. In the old version a person landed on a dashboard with many controls and had to work out what to do. The new version is a **guided path**: *Role, Skill check, Dashboard (gaps and learning path)*.

| Part | What happened | Why |
|---|---|---|
| **Website (frontend)** | Rebuilt from scratch: new design, new flow, new screens, new animations | The flow and look were the main problem |
| **Backend (API server)** | Rebuilt: new `engine.py`, new endpoints, input validation, tests | Cleaner, safer, and supports the new flow |
| **New AI logic** | Added: step-by-step skill check with a live "what the AI thinks" panel, plain-language reasons for each gap, certainty per skill, study-time budget, week-by-week schedule, "I have done this" progress | Needed for a good user journey |
| **The three trained models and the data pipeline** (`ml/` folder) | **Kept**, not rewritten | They were already trained and evaluated with proper train/validation/test splits. Rewriting them would throw away verified results and add risk without making the product better |
| **New experiment** (`ml/08_evaluate_checkup.py`) | Added | We tested whether our "ask the least-sure skill first" idea is better than random order. **It is not, by a meaningful margin.** We report that honestly (section 8) |

If the examiner asks "did you build all the AI yourselves?", answer: *"Yes, the pipeline in the ml folder is ours: cleaning, tagging, simulation, three models and evaluation. In this version we rebuilt the web app around it and added the check-up logic, explanations and scheduling."*

---

## 3. The big picture (architecture)

```
 3 public course datasets (Coursera, edX, Udemy: 7,240 rows)
        |  ml/02_clean_catalogue.py      wash the data
        v
 6,988 clean courses
        |  ml/03_tag_courses.py          AI #1  course tagger
        v
 2,107 relevant courses, each tagged with skills       ml/framework.py = 31 skills + 11 roles
        |                                                   |
        |            ml/04_simulate_learners.py  4,000 simulated people, noisy tests, 3 expert panelists
        |                                                   |
        |            ml/05_train_gap_model.py    AI #2  gap model
        v                                                   v
   ml/06_train_recommender.py  AI #3  ranker  ->  ml/07_evaluate.py  (the exam on 800 unseen people)
                                              ->  ml/08_evaluate_checkup.py (how many questions are enough)

 ----- at run time -----
 Browser (React website)  <--- JSON over HTTP --->  server/app.py (FastAPI)  --->  server/engine.py
                                                                                     loads: ml/models/*.joblib
                                                                                            ml/data/processed/*
```

**Two halves, two languages.**
- **Python** half: all AI and the server (`ml/`, `server/`).
- **JavaScript** half: the website (`web/src/`), built with React.
- They talk using **HTTP requests that carry JSON** (a text format for data, like `{"role_id": "stat_officer"}`).

---

## 4. How to run it

You need Python 3.10 or newer and Node.js 18 or newer.

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt

python -m uvicorn server.app:app --port 8000
```
Open **http://localhost:8000**. The website is already built (`web/dist`) and served by the same server, so one command runs everything.

**To edit the website and see changes live**
```bash
# terminal 1: python -m uvicorn server.app:app --port 8000
# terminal 2:
cd web && npm install && npm run dev      # opens http://localhost:5173
```
Rebuild for production with `npm run build`.

**To run the tests:** `python -m pytest -q` (6 tests, all should pass).
**To retrain everything (about 4 minutes):** `cd ml && python run_all.py`. Random seeds are fixed, so numbers repeat.

---

## 5. The website: what happens on each screen

| Screen | What the person does | What the code does |
|---|---|---|
| **Your role** (`Start.jsx`) | Picks a role, years of experience, education. Sees what the role needs. Or opens a sample profile. | Calls `GET /api/meta` for roles and `GET /api/samples` for demo people |
| **Skill check** (`Checkup.jsx`) | Rates themselves from 1 to 5 on one skill at a time (keys 1 to 5 work). Can add a test score or manager rating. Watches the right panel update. | After every answer, calls `POST /api/checkup/next`. The server returns the next skill, the AI's overall certainty and its current belief for every skill |
| **Dashboard, Skill gaps** (`Results.jsx`) | Sees "You are 64% of the way to Statistical Officer", then each skill as a **ladder** and opens a row to see why | Calls `POST /api/analyze` |
| **Dashboard, Learning path** | Sees ordered courses with weeks, hours, a reason, and "readiness after this course". Changes courses, hours per week or total time. Marks courses done. Downloads the plan as CSV or prints it. | Same call, re-run when settings change |
| **Courses**, **Quiz maker**, **How it works** | Browse the catalogue; make a quiz from notes or a PDF and **add the quiz score as assessment evidence** for a skill (it becomes a test score the gap model uses); read the evidence | `/api/courses`, `/api/quiz`, `/api/reports` |

**The ladder (the signature visual).** A bar from level 0 to 5. The coloured part is your estimated level. The hatched part is the gap. The black tick is the level the role needs. One glance tells you how far you are from the target.

**Flow design decisions you can mention:**
- One question per screen, with plain-language level descriptions, so people do not need to understand "proficiency levels".
- You can stop after 4 answers; the AI estimates the rest (and tells you it did).
- Progress is saved in the browser, so a refresh loses nothing.
- Colours come from the subject: deep peacock teal ink, a saffron accent for the "required level" mark, and plain red/amber/green only for gap severity. No gradients or decorative blobs.
- Every screen has loading, error and empty states. Keyboard focus is visible. Animation respects "reduced motion".

**Frontend files**

| File | Job |
|---|---|
| `main.jsx` | Starts the app |
| `App.jsx` | Top bar, step guard (you cannot open Results before answering), picks the screen from the URL `#/...` |
| `store.jsx` | Everything the person has told us, saved in `localStorage` |
| `api.js` | The only file that talks to the server |
| `components.jsx` | Ladder, tags, count-up number |
| `pages/*.jsx` | One file per screen |
| `styles.css` | The whole design system (colours, spacing, motion) |

**React in 5 lines (examiners may ask).**
- A *component* is a function that returns a piece of screen.
- *State* (`useState`) is data that, when changed, makes the screen redraw.
- `useEffect` runs code after the screen draws, for example "call the server when answers change".
- *Props* are inputs passed to a component.
- Vite is the tool that bundles the code into the `dist` folder.

---

## 6. The backend: what each file does

| File | Job |
|---|---|
| `server/app.py` | The web routes (FastAPI). Checks input, calls the engine, returns JSON |
| `server/engine.py` | **The brain.** Loads the three models once and does gap prediction, next-question choice, course ranking, scheduling |
| `server/quizgen.py` | Makes multiple-choice questions from text |
| `tests/test_api.py` | 6 automatic tests |

**Endpoints**

| URL | Does |
|---|---|
| `GET /api/health` | Is the server alive |
| `GET /api/meta` | Roles, skills, level names |
| `GET /api/samples` | Six ready-made learners |
| `POST /api/checkup/next` | Answers so far, gives next skill, certainty, per-skill beliefs |
| `POST /api/analyze` | Answers, gives gaps, learning path, schedule |
| `GET /api/courses` | Search the catalogue |
| `GET /api/reports` | All evaluation numbers |
| `POST /api/quiz` | Text or file, gives MCQs |

**Why FastAPI?** It is fast, validates input automatically (Pydantic models reject a rating of 9 with a clear error), and generates documentation at `http://localhost:8000/docs`.

**Why is `features.py` shared?** `ml/features.py` turns a person into numbers. The *same file* is used in training and in the live server, so the two can never disagree.

---

## 7. The data

| File | Courses | Real or simulated |
|---|---|---|
| `coursera.csv` | 3,522 | Real, public |
| `edx.csv` | 720 | Real, public |
| `udemy.csv` | 3,678 | Real, public (includes subscriber counts) |
| Competency framework (31 skills, 11 roles) | | Written by us in the shape of iGOT Karmayogi (Domain, Functional, Behavioural) |
| **Learners, assessments, expert paths** | 4,000 people | **Simulated.** Real government assessment data is not public |

**Cleaning (7,240 rows to 6,988 courses):** unified three different layouts, dropped 680 off-topic music courses, removed duplicates in three steps (same link, same title, near-identical title using TF-IDF similarity of 0.92 or more), removed non-English rows, stripped HTML and coupon junk, mapped messy difficulty labels to Beginner, Intermediate, Advanced. **Rule we followed: never silently invent data.** Every guess carries a flag, and the site shows "(estimated)" for guessed course hours.

**Relevance filter:** after tagging, only courses that teach at least one of our 31 skills stay: 2,107.

---

## 8. The three AI models, explained simply

### AI #1: the course tagger (`ml/03_tag_courses.py`)

**Problem:** nobody has labelled "this course teaches SQL". Labelling 7,000 courses by hand is not realistic.

**Idea: weak supervision.** Get rough labels automatically, then train a model to generalise them.

1. **Four rules vote** (title keyword, skills text, description keywords, TF-IDF similarity). Votes combine with a **noisy-OR**: `score = 1 - (1-a)(1-b)(1-c)(1-d)`. It is high if *any* rule is confident and higher if several agree.
2. **A logistic regression** is trained to predict those labels from text **with all keywords deleted**. To succeed it must learn *surrounding* words ("cells, formulas, worksheet" means spreadsheets).
3. **Blend:** `0.6 x rules + 0.4 x context model`.

**Honest check:** with 5-fold out-of-fold prediction the keyword-blind model re-discovers the rule labels at F1 = 0.60. If it had only memorised keywords it would score near 0.

### AI #2: the gap model (`ml/features.py`, `ml/05_train_gap_model.py`)

**Task:** for each person and each skill their role needs, predict **no gap, moderate gap or critical gap**, and the gap size in levels.

**Inputs (23 numbers per person-skill pair):** the level the role needs; self-rating, test score, manager rating (any may be missing); averages of the *other* skills in the same group (this is how it estimates untested skills); a correction for people who over-rate themselves (Dunning-Kruger); how many related courses they finished.

**Missing values stay missing** (NaN). Gradient boosting handles this natively.

**Models compared on validation data:** Logistic Regression 0.716, Random Forest 0.730, **Gradient Boosting 0.743** (winner). Chosen on validation, tested once.

**Test results (11,739 unseen rows):**

| System | Macro-F1 |
|---|---|
| Always guess the most common class | 0.209 |
| Trust self-rating | 0.535 |
| Quiz score, else self-rating (rule baseline) | 0.627 |
| **Our gap model** | **0.745** |

For skills with **no test at all**: 0.698 against 0.536 for the rule. This proves the model really infers from related skills.

**Ablation (what is each kind of evidence worth?):** role only 0.355; plus experience and education 0.383; plus self-rating 0.653; plus manager 0.691; plus quiz 0.741; plus course history 0.740 (no gain, an honest result).

**Certainty.** For every skill the model gives three probabilities (no, moderate, critical). The **certainty** we show is the largest of the three (0.33 means coin toss, 1.0 means sure). The "AI certainty" meter is the importance-weighted average over the role's core skills.

### AI #3: the recommender (`ml/recommender.py`, `ml/06_train_recommender.py`)

1. **Coverage score.** For each course: how much of the person's gap it closes (right skill and right level). Courses that start too high are skipped.
2. **Shortlist** the top 80.
3. **Learned ranker.** 15 numbers per person-course pair (coverage, level fit, rating, length, popularity and so on). A gradient-boosting regressor learns to predict how many of 3 experts would pick the course. This is *pointwise learning to rank*.
4. **Diversity.** After choosing a course for a skill, the next course for the same skill loses score, so you do not get eight SQL courses.
5. **Order as a path.** Biggest gap first. Inside one skill, beginner before advanced.
6. **No cheating.** The ranker is trained on *estimated* gaps produced out-of-fold (each person predicted by a model that never saw them), exactly the situation at run time.

**Results on 800 unseen learners**

| System | Recall@10 | NDCG@10 | Real gaps covered |
|---|---|---|---|
| Random | 0.006 | 0.007 | 0.146 |
| **Most popular courses (baseline)** | 0.008 | 0.015 | 0.272 |
| Ours, rules only | 0.256 | 0.276 | 0.746 |
| **Ours, learned ranker (final)** | **0.475** | **0.537** | 0.737 |
| Reference: same ranker given the *true* gaps | 0.583 | 0.650 | 0.766 |

The gap between 0.475 and 0.583 is the cost of having to *estimate* gaps. Improvement over popularity: +0.468 recall@10 (95% interval 0.454 to 0.483).

### New in this version: the skill check and the plan

**Skill check logic (`engine.checkup_state`).**
- After each answer, the gap model re-predicts every skill. The next question is the skill whose prediction is **least certain**.
- You can finish after 4 answers.

**We tested whether that ordering helps (`ml/08_evaluate_checkup.py`, 300 unseen people, self-ratings only, macro-F1 against the true gap class):**

| Skills answered | 2 | 4 | 6 | 8 | 10 |
|---|---|---|---|---|---|
| Least-sure first | 0.438 | 0.478 | 0.506 | 0.538 | 0.554 |
| Random order | 0.429 | 0.469 | 0.513 | 0.529 | 0.544 |

**Finding:** order makes almost no difference (differences are within noise). The *number of answers* matters: accuracy rises with every answer. This is why the site lets you stop early but nudges you to answer more. We tried three other orderings too (hardest first, one per skill group, biggest predicted gap first) and none clearly beat random. Reporting a negative result honestly is a strength in a viva.

**Plain-language reasons (`engine._reasons`).** Built only from the numbers the model saw, for example "You rated yourself 2; your role needs 4", or "Not rated yet. Inferred from your other ratings in this area". No invented explanation.

**Readiness percentage.** `100 x (1 - total estimated gap / total possible gap)` over the role's core skills.

**Path trajectory.** After each course the remaining gap is reduced by what that course can teach (right skill, enough coverage), giving "Readiness after this course".

**Schedule.** `hours_per_week` and each course's hours give start and end weeks. A "total time you can spare" budget trims the list.

### Quiz maker (`server/quizgen.py`)

No paid AI. Classic, explainable NLP: spaCy splits sentences and tags parts of speech; **TextRank** (PageRank for sentences) finds the important sentences; key terms are noun phrases ranked by TF-IDF; a sentence becomes a definition question or a fill-in-the-blank; wrong options come from other key terms that appear in similar sentences. **Limit:** questions are recall-style, and short texts give weaker wrong options.

---

## 9. Honest limitations (say these before you are asked)

1. **Learners and expert paths are simulated.** Real iGOT data is not public; the problem statement itself says "student-sourced public data". The numbers prove the pipeline works and show which design choices help. They are **not** a promise of accuracy on real officials.
2. **Partly circular.** We built the simulator and the models. We reduced this: experts see the *true* gaps and strict rule tags; the system sees only *noisy* evidence and *learned* tags. The "true gaps" reference row shows the remaining difference.
3. **Course tags are weak labels.** `ml/data/processed/gold_review_sample.csv` has 80 courses to hand-label to measure true tagger accuracy.
4. **The catalogue is general-purpose**, thin on some skills (survey design, quality standards, GIS).
5. **Level descriptions in the skill check are generic**, not a validated test. Self-rating is a weak signal; this is why test scores and manager ratings can be added.
6. **No live iGOT connection.** There is no public API. The integration point is replacing the catalogue and framework files.

---

## 10. A five-minute demo script

1. **(30 s) Problem.** "A huge catalogue, no way to know which course closes *my* gap."
2. **(45 s) Role.** Open the site, pick *Statistical Officer*. "Each role has target levels. Here is what this one needs."
3. **(60 s) Skill check.** Start. Answer 5 skills with keys 1 to 5. Point at the right panel: "The AI already estimates all 17 skills. Solid dots are what I answered, the rest are inferred. The certainty meter rises as I answer."
4. **(60 s) Results.** Show Results. "64% ready. Each ladder shows my level against the target." Open one gap: "These are the reasons and the probabilities."
5. **(60 s) Path.** Learning path tab. "Ordered biggest gap first. Change study time to 10 hours and the weeks update. Mark one course done and the plan recalculates."
6. **(30 s) Evidence.** *How it works* page: "Recall@10 of 0.475 against 0.008 for popularity."
7. **(30 s) Honesty.** "Learners are simulated. With real data we replace the data files and re-run."

Practise this twice before the viva. Have the sample profile *Ananya* ready as a backup if typing is slow.

---

## 11. Likely viva questions and model answers

### About the project
**Q1. What problem does it solve?** Officials face a huge course catalogue and no way to know which courses close their own gaps. We infer each person's gaps from their role and assessment evidence and recommend a short, ordered path.

**Q2. What is a competency?** A skill needed for a job, for example SQL or survey design. We use 31, grouped as Domain, Functional and Behavioural, like iGOT Karmayogi.

**Q3. What is a "gap"?** The difference between the level the role needs and the person's level, per skill. We label it no, moderate (0.5 to 1.5 levels) or critical (1.5 levels or more).

**Q4. Why simulated learners? Is that fake?** Real assessment data belongs to iGOT and is not public. The courses are real. The simulator is explicit rules, kept separate from the models, and the same code runs on real data when available. We say clearly that results show the pipeline works, not real-world accuracy.

**Q5. Isn't it circular, since you built the experts and the model?** Partly, and we say so. We reduced it: experts use the true hidden gaps and strict rules; the system sees noisy evidence and learned tags. The "ranker with true gaps" row measures what imperfect inference costs (0.475 against 0.583).

**Q6. How does it connect to iGOT Karmayogi?** There is no public API, so no live link. Our framework follows iGOT's structure, and the integration point is replacing the catalogue and framework files with iGOT's.

**Q6b. How do the quizzes connect to the gap analysis?** After finishing a generated quiz you pick the skill it tested. The score (for example 6 of 8) is converted to 1 to 5 (here 4.0) and stored as that skill's test score, which the gap model already accepts as its strongest evidence.

**Q7. What did you improve over the first version?** A guided flow instead of a dashboard, a rebuilt backend with validation and tests, live belief panel during the skill check, plain-language reasons, per-skill certainty, a study schedule and progress tracking. The trained models were kept because they were already properly evaluated.

### About machine learning
**Q8. What is machine learning?** A program that learns patterns from examples instead of being given explicit rules.

**Q9. What is supervised learning?** Learning from examples that have correct answers. Our gap model learns from people whose true gap class is known (in the simulator).

**Q10. What is weak supervision?** Training on rough, automatically made labels when perfect hand labels do not exist. Our course tags come from rules plus a context model.

**Q11. Why gradient boosting?** It won on validation (0.743 against 0.730 for random forest and 0.716 for logistic regression), handles missing values natively, and captures interactions such as "high quiz score but very high self-rating".

**Q12. What is gradient boosting?** Many small decision trees built one after another, each correcting the mistakes of the previous ones.

**Q13. What is overfitting, and how did you avoid it?** Memorising training data instead of learning general patterns. We split people 60/20/20 into train/validation/test (nobody appears in two splits), chose models on validation only, used the test set once, and used out-of-fold predictions for the recommender.

**Q14. What is data leakage?** When information from the answer sneaks into the inputs. Examples we guarded against: splitting by person, excluding the current skill when averaging related skills, and out-of-fold gap estimates.

**Q15. Why F1 and not accuracy?** The "critical gap" class is rare (14%). Macro-F1 averages F1 per class so the rare class counts equally.

**Q16. Explain precision, recall, F1.** Precision: of what we flagged, how much was right. Recall: of what was truly there, how much we found. F1 is their harmonic mean.

**Q17. What are Recall@K and NDCG@K?** Recall@10: the experts chose N courses; how many appear in our top 10. NDCG@10: like recall but rewards putting courses that more experts agreed on higher in the list.

**Q18. What is "gap coverage"?** Of the person's real gap, how much has at least one useful course (right skill and level) in our top 10. It does not depend on the simulated experts.

**Q19. What is TF-IDF?** A number showing how special a word is to one text: frequent in this text, rare in others.

**Q20. What is learning to rank?** Teaching a model to put the best items at the top of a list. Ours predicts the number of expert votes (0 to 3) for each person-course pair.

**Q21. What is out-of-fold prediction?** Predicting each item with a model that never saw it, by training on the other folds. It gives honest estimates for training data.

**Q22. What is a bootstrap confidence interval?** Re-sample the 800 test people many times to see how much a result could wobble. We report 95% intervals.

**Q23. What is an ablation?** Adding or removing one ingredient to measure its value. Self-rating is the cheapest big gain (0.383 to 0.653).

**Q24. What is the Dunning-Kruger effect and how do you use it?** People who know less tend to over-rate themselves. When test scores exist, the model measures each person's over-confidence on other skills and corrects their self-rating.

**Q25. Your check-up asks the least-sure skill first. Does it help?** We measured it: no meaningful gain over random order. What helps is the number of answers. We kept the ordering (it is no worse) but do not claim it is better.

**Q26. What does "AI certainty 63%" mean?** The importance-weighted average of the model's highest class probability across the role's core skills. It is the model's own confidence, not a guarantee of accuracy.

### About the code
**Q27. What is an API?** A set of URLs a program can call to get data. The website calls our Python server's API.

**Q28. What is JSON?** A simple text format for structured data.

**Q29. What does FastAPI do here?** Defines the routes, validates input with Pydantic and returns JSON.

**Q30. Why are the models loaded once?** Loading takes time. `get_engine()` is cached, so the models load at start-up and each request is fast (about 10 to 20 milliseconds in our tests).

**Q31. How does the website avoid losing data on refresh?** `store.jsx` saves the answers to `localStorage` after every change.

**Q32. What happens when you click an answer in the skill check?** The store updates, a `useEffect` calls `POST /api/checkup/next`, the server rebuilds features, runs the gap model, and returns the next skill and all beliefs. React redraws the ladders.

**Q33. How do you handle server errors?** `api.js` converts failures into readable messages; each screen shows an error state with a retry button.

**Q34. How is the quiz built?** spaCy, TextRank, TF-IDF key terms, template questions, co-occurrence distractors (section 8).

**Q35. What would you do with real data?** Replace `catalogue_tagged.csv` with the real catalogue, `learners.pkl` with real assessments and mentor-approved paths, then run `python run_all.py`.

**Q36. What are your next steps?** Real iGOT data, hand-labelled gold set for the tagger, sentence-embedding models for tagging, validated skill tests, per-user accounts and progress history.

---

## 12. Numbers to memorise

| Fact | Value |
|---|---|
| Raw courses, clean, relevant | 7,240, 6,988, 2,107 |
| Skills, roles | 31, 11 |
| Simulated learners (train / validation / test) | 4,000 (60% / 20% / 20%) |
| Gap model macro-F1 (ours / rule baseline) | 0.745 / 0.627 |
| Gap model without any quiz | 0.698 against 0.536 |
| Gap size error (levels) | 0.24 against 0.37 for the rule |
| Recommender Recall@10 (ours / popularity) | 0.475 / 0.008 |
| Gap coverage@10 (ours / popularity) | 0.737 / 0.272 |
| Tagger context model F1 | 0.60 |
| Check-up: macro-F1 at 4 and 10 answers | 0.48 and 0.55 |

---

## 13. Glossary

- **Competency**: a skill needed for a job.
- **Gap**: how far you are below the level the job needs.
- **Feature**: one number describing something about a person or course.
- **Training / validation / test**: study material / practice exam used to pick the best model / final exam used once.
- **Overfitting**: memorising instead of understanding.
- **Out-of-fold**: predicted by a model that never saw that item.
- **TF-IDF**: how special a word is to one text.
- **Logistic regression**: draws a boundary between yes and no and gives a probability.
- **Gradient boosting**: many small trees, each fixing the previous one's mistakes.
- **Weak supervision**: training on rough automatic labels.
- **Learning to rank**: teaching a model to put the best items first.
- **Ablation**: removing or adding one ingredient to see its value.
- **Bootstrap**: re-sampling to see how much a result could vary.
- **Macro-F1**: F1 averaged over classes, so rare classes count equally.
- **Certainty**: the highest class probability the model gives.
- **API / JSON / endpoint**: how the website and server talk.
- **React / component / state**: the library for the website, a piece of screen, and data that redraws the screen when it changes.
