---
title: "Interactive Spanish Learning Chatbot"
excerpt: "As my capstone internship project for my M.S. in Computational Linguistics, I developed a web‑based Spanish‑learning chatbot using Flask and spaCy. The application delivers scaffolded lessons for beginner learners (multiple‑choice → open‑response → review) and provides real‑time, rule‑based feedback on grammar, conjugation, and spelling. Lesson content is modular and easily expandable, making the tool adaptable for broader language‑learning contexts."
collection: portfolio
---

## Project Overview
As my capstone project for the M.S. in Human Language Technology (HLT) program, I developed a web‑based Spanish‑learning chatbot designed to help beginner learners practice vocabulary and sentence construction. The chatbot combines a Flask back‑end with a lightweight JavaScript front‑end and uses spaCy for linguistic analysis. The application delivers scaffolded lessons, moving the learner from multiple‑choice recognition to open‑response production and finally to targeted review of their mistakes. It provides real‑time, rule‑based feedback on grammar, conjugation, and spelling. Additionally, it uses modular, JSON‑based lesson content so new topics can be added without altering the application code.

This project allowed me to bring together skills learned throughout the HLT program. From morphology and parsing to Python programming and corpus structuring, it reflects how coursework translated into an applied solution. It also helped me gain new experience in full‑stack development, state management, and user‑oriented application design. While currently lightweight in scope, this project was designed as a scalable foundation for a more advanced, feature‑rich system that could later incorporate a myriad of lessons, speech recognition, adaptive review scheduling, or multimedia interaction. It serves both as an academic exercise in applying linguistic and computational concepts, and as a starting point for future development.

**Technologies used:** Flask, spaCy, JSON, JavaScript, HTML/CSS 

---

<img width="600" alt="image" src="https://github.com/user-attachments/assets/5816fad3-38bd-44ea-9704-320fbf5de54f" />

## System Design
The system consists of three primary layers: a back‑end written in Flask, a front‑end built with HTML/CSS and JavaScript, and lesson content stored as structured JSON. The back‑end handles session management, lesson progression, and language error analysis. The front‑end dynamically displays lessons, choices, and progress updates. Lessons themselves are completely decoupled from logic, making the content easily editable or expandable.

For example, here is a snippet from one of the simplest lessons, greetings:
```json
{
  "flashcards": [
    { "question": "How do you say 'hello' in Spanish?", "answer": "hola" },
    { "question": "How do you say 'good morning' in Spanish?", "answer": "buenos días" }
  ],
  "sentences": [
    { "question": "Translate: 'Hello, good morning!'", "answer": "¡Hola, buenos días!" }
  ]
}
```

## Morphological Error Analysis
One of the most linguistically interesting parts of the project is the error analysis system. Rather than relying on a black‑box model, I used spaCy’s es_core_news_sm pipeline to analyze user responses for morphological agreement and grammatical correctness. This method produces clear, interpretable feedback that users can act on.

The system uses multiple strategies for analysis. First, a similarity check using difflib detects spelling and accent issues:
```
seq = difflib.SequenceMatcher(None, normalize_text(user_answer), normalize_text(correct_answer))
if seq.ratio() < 0.9:
    error_details.append({"type": "accent", "msg": "check your spelling or accents"})
```
Then, it compares verb lemmas and morphological features (person, number, tense) between the user’s answer and the expected answer:
```
for c_verb in correct_verbs:
    for u_verb in user_verbs:
        if u_verb.lemma_.lower() == c_verb.lemma_.lower():
            if (u_verb.morph.get("Person") != c_verb.morph.get("Person")
                or u_verb.morph.get("Number") != c_verb.morph.get("Number")):
                subject = correct_pronouns[0].text if correct_pronouns else "the subject"
                error_details.append({
                    "type": "verb_conjugation",
                    "msg": f"the verb '{u_verb.text}' doesn’t match '{subject}'"
                })
```
It also checks for gender and number agreement in nouns and adjectives:
```
for u_tok, c_tok in zip(user_tokens, correct_tokens):
    if u_tok.pos_ in ("NOUN", "ADJ") and c_tok.pos_ in ("NOUN", "ADJ"):
        if u_tok.morph.get("Gender") != c_tok.morph.get("Gender"):
            error_details.append({"type": "gender", "msg": f"check gender for '{u_tok.text}'"})
```
<img width="600" alt="image" src="https://github.com/user-attachments/assets/3aee6ded-41da-4535-8dd6-2c502d34a922" />

## Lesson Flow
The chatbot uses a scaffolded approach to learning. Lessons begin with a multiple‑choice phase to help learners recognize key vocabulary. They then transition to a sentence‑building phase that requires typing spanish answers in full. Finally, any missed items are added to a review phase for reinforcement.

The progression between these stages is handled by a state machine in Flask sessions:
```
def get_next_step(feedback=None):
    steps = get_steps()
    idx = session.get("lesson_step", 0)
    phase = session.get("lesson_phase")

    if idx < len(steps):
        step = steps[idx]
        reply = f"{feedback}\n\n{step['question']}" if feedback else step['question']
        resp = {"reply": reply, "progress": {"current": idx + 1, "total": len(steps)}}
        if phase == "flashcards":
            resp["choices"] = generate_choices(step["answer"], [s["answer"] for s in steps])
        return jsonify(resp)

    if phase == "flashcards":
        session.update({"lesson_step": 0, "lesson_phase": "sentences"})
        return get_next_step("great work! now let's practice full sentences.")
    if phase == "sentences" and session["review_list"]:
        session.update({"lesson_step": 0, "lesson_phase": "review"})
        random.shuffle(session["review_list"])
        return get_next_step("let's review the ones you missed!")
    return end_lesson()
```

<img width="600" alt="image" src="https://github.com/user-attachments/assets/ff5978cb-de25-4111-86e9-3acc8b7d7a95" />

## Tracking and Logging
To make the tool useful for reflection and potential research, I implemented performance tracking and detailed logging. The chatbot tracks first‑try accuracy and builds a list of items for review. It also logs every mistake to a daily JSON file for later analysis:
```
def log_mistake(topic, user_answer, correct_answer, feedback, error_type):
    os.makedirs(LOG_DIR, exist_ok=True)
    log_file = os.path.join(LOG_DIR, f"mistakes_{datetime.now().date()}.json")
    entry = {
        "timestamp": datetime.now().isoformat(),
        "topic": topic,
        "user_answer": user_answer,
        "correct_answer": correct_answer,
        "feedback": feedback,
        "error_type": error_type
    }
    if not os.path.exists(log_file):
        with open(log_file, "w") as f: json.dump([], f)
    with open(log_file, "r+") as f:
        data = json.load(f)
        data.append(entry)
        f.seek(0); json.dump(data, f, indent=2)
```
This functionality could eventually support features like progress dashboards or analytics on common learner errors.

## What I Learned
This project gave me the opportunity to apply core HLT concepts in a real‑world context. I used morphological parsing and rule‑based NLP to build a feedback engine that is interpretable and educationally meaningful. I structured lesson data like a corpus, a skill honed in our corpus linguistics courses. I also improved my Python and Flask skills by designing a state‑driven web application.

Beyond applying what I already knew, I learned a great deal about front‑end development and user experience. Building a system that is intuitive for non‑technical users required me to think about interface design and progression pacing. I also became more comfortable working across the stack, integrating back‑end linguistic processing with a browser‑based interface.

## Outcomes and Future Work
The end result is a functional, modular Spanish‑learning chatbot that can easily support new lessons or even other languages. In the future, I would like to add speech recognition for pronunciation practice, integrate spaced‑repetition techniques for review scheduling, and build an analytics dashboard to help learners track their progress.

## Conclusion
This capstone internship project brought together linguistic theory, rule‑based natural language processing, and practical software engineering. It demonstrates my ability to design and build tools that are educationally grounded, technically robust, and interpretable. It also represents an important step toward my goal of developing applications that make language learning more accessible and effective.
