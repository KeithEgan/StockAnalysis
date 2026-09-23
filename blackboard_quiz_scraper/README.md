# Blackboard quiz question harvester

Keeps taking a Blackboard (Ultra) self-test that draws random questions from a question bank. Each unique question goes into a Word document once.

For each attempt the script does this: **Start attempt** → scroll to the bottom → type `0` in the last text box → **Submit** → **Submit** (confirm) → **Close** → repeat.

## Setup (Windows)

```
pip install -r requirements.txt
playwright install chromium
```

## Run

```
python scrape_quiz.py --title "Fluid Mechanics 2 - Self-test 1"
```

1. A browser window opens. Log in to Blackboard, then open the quiz's **Assessment Details** page (the page with the "Start attempt" button).
2. Switch back to the terminal and press **Enter**.
3. Leave it running. It stops on its own after 15 attempts in a row with no new questions. You can also press Ctrl+C at any time.

Output goes in `quiz_output/`:

- `questions.docx`: every unique question, as text plus a screenshot of the question (keeps diagrams, subscripts and equations).
- `questions.json` and `screenshots/`: saved progress. If you re-run, the script continues and never adds a question twice.

Useful options:

| Option | Meaning |
|---|---|
| `--url URL` | Open this page first instead of navigating yourself |
| `--max-attempts N` | Hard limit on attempts (default 100) |
| `--stop-after N` | Stop after N attempts in a row with nothing new (default 15) |
| `--settle SECONDS` | Extra wait after page loads; increase it if Blackboard is slow (default 2) |
| `--browser msedge` | Use your installed Microsoft Edge instead of Playwright's Chromium |
| `--no-screenshots` | Text only |
| `--out DIR` | Output folder (default `quiz_output`) |

Your login is kept in `.bb_browser_profile/`, so you normally only log in once.

## If something goes wrong

When a step fails, the script saves `error_attempt_N.png` and `error_attempt_N.html` to the output folder. It then goes back to the quiz page and tries again. The HTML file shows which button or element it could not find.

## Notes

- Two questions count as duplicates only when their text and images match. If the bank uses calculated questions with random numbers, each number variation is saved as a separate question.
- Each run submits real attempts, and they appear in your attempt history. Only use the script on quizzes that allow unlimited, ungraded attempts, like formative self-tests. Check that your institution allows this kind of automation.
