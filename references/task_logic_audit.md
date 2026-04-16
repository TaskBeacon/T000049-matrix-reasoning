# Task Logic Audit

## 1. Paradigm Intent

- Task: Matrix Reasoning
- Primary construct: nonverbal abstract reasoning / fluid intelligence
- Manipulated factors: practice versus scored items, relational complexity, deterministic shape-variant selection, and a two-stage response deadline
- Dependent measures: accuracy, response time, timeout rate, and practice-feedback performance
- Key citations:
  - `W2981704638` Chierchia et al., 2019, open-access matrix reasoning item bank
  - `W2167749256` Brouwers et al., 2008, Raven score variation across settings
  - `W2048613949` Kane et al., 2004, reasoning and visuospatial span
  - `W2088055785` Hampshire et al., 2012, human intelligence fractionation
  - `W2158211182` Bunge, 2004, analogical reasoning and prefrontal cortex
  - `W2110233084` Roca et al., 2009, fluid intelligence after frontal lobe lesions

## 2. Block/Trial Workflow

### Block Structure

- Total blocks: 1
- Trials per block: 15
- Randomization/counterbalancing: fixed sequential order; the 3 practice items always appear first, followed by 12 scored items
- Condition weight policy:
  - `task.condition_weights` is `null` in all configs
  - no runtime weighting is needed because the short-form bank is already ordered
- Condition generation method:
  - Built-in `BlockUnit.generate_conditions(...)` is sufficient
  - condition labels are item IDs from the short-form bank, so no custom generator is required
  - the runtime resolves each label to a deterministic item specification through `src/utils.DEFAULT_TRIAL_BANK`
- Runtime-generated trial values:
  - matrix geometry, option card content, correct-option key, and layout-specific symbol placement are generated at runtime
  - generation is deterministic from the task seed, block seed, and condition label
  - the implementation uses an open-access matrix-reasoning proxy rather than the copyrighted Raven stimulus set

### Trial State Machine

1. State name: fixation
   - Onset trigger: `fixation_onset`
   - Stimuli shown: centered fixation cross
   - Valid keys: none
   - Timeout behavior: fixed 500 ms
   - Next state: blank_screen

2. State name: blank_screen
   - Onset trigger: `blank_onset`
   - Stimuli shown: white blank screen
   - Valid keys: none
   - Timeout behavior: fixed 100 ms
   - Next state: matrix_response_open

3. State name: matrix_response_open
   - Onset trigger: `matrix_response_open_onset`
   - Stimuli shown: 3x3 abstract matrix with one missing cell, four answer cards, and the response prompt
   - Valid keys: `1`, `2`, `3`, `4`
   - Timeout behavior: response remains open for 25 s
   - Next state: matrix_response_warning if no response arrives, otherwise practice_feedback or iti

4. State name: matrix_response_warning
   - Onset trigger: `matrix_response_warning_onset`
   - Stimuli shown: the same matrix and answer cards plus a clock-warning banner
   - Valid keys: `1`, `2`, `3`, `4`
   - Timeout behavior: final 5 s of the 30 s item limit
   - Next state: practice_feedback if the trial is a practice item, otherwise iti

5. State name: practice_feedback
   - Onset trigger: `practice_feedback_onset`
   - Stimuli shown: correctness, incorrectness, or timeout feedback for the practice item
   - Valid keys: none
   - Timeout behavior: fixed brief feedback duration
   - Next state: iti

6. State name: iti
   - Onset trigger: `iti_onset`
   - Stimuli shown: centered fixation cross
   - Valid keys: none
   - Timeout behavior: fixed 600 ms
   - Next state: next trial

## 3. Condition Semantics

For each condition token in `task.conditions`:

- `practice_01`
  - Participant-facing meaning: first practice matrix before scored items
  - Concrete stimulus realization: 3x3 abstract matrix, four answer cards, and practice-only feedback if a response is captured
  - Outcome rules: not scored as a main trial; the participant sees correctness feedback

- `practice_02`
  - Participant-facing meaning: second practice matrix before scored items
  - Concrete stimulus realization: 3x3 abstract matrix, four answer cards, and practice-only feedback if a response is captured
  - Outcome rules: not scored as a main trial; the participant sees correctness feedback

- `practice_03`
  - Participant-facing meaning: third practice matrix before scored items
  - Concrete stimulus realization: 3x3 abstract matrix, four answer cards, and practice-only feedback if a response is captured
  - Outcome rules: not scored as a main trial; the participant sees correctness feedback

- `easy_01` / `easy_02` / `easy_03` / `easy_04`
  - Participant-facing meaning: lower-complexity scored matrix items in the short-form proxy bank
  - Concrete stimulus realization: 3x3 abstract matrix with four answer cards, generated from deterministic shape/count/rotation/layout rules
  - Outcome rules: scored for accuracy and RT only; no feedback screen is shown

- `medium_01` / `medium_02` / `medium_03` / `medium_04`
  - Participant-facing meaning: medium-complexity scored matrix items in the short-form proxy bank
  - Concrete stimulus realization: 3x3 abstract matrix with four answer cards, generated from deterministic shape/count/rotation/layout rules
  - Outcome rules: scored for accuracy and RT only; no feedback screen is shown

- `hard_01` / `hard_02` / `hard_03` / `hard_04`
  - Participant-facing meaning: highest-complexity scored matrix items in the short-form proxy bank
  - Concrete stimulus realization: 3x3 abstract matrix with four answer cards, generated from deterministic shape/count/rotation/layout rules
  - Outcome rules: scored for accuracy and RT only; no feedback screen is shown

Also document where participant-facing condition text/stimuli are defined:

- Participant-facing text source (config stimuli / code formatting / generated assets): `config/*.yaml` for instructions, prompts, feedback, and summaries; `src/utils.DEFAULT_TRIAL_BANK` for deterministic matrix-bank geometry
- Why this source is appropriate for auditability: the participant text remains in config for easy localization, while the nontext matrix geometry is reproducible from explicit item specs in code
- Localization strategy (how language variants are swapped via config without code edits): replace the YAML instruction, prompt, feedback, and summary strings; the runtime item generator and response mapping do not need to change

## 4. Response and Scoring Rules

- Response mapping: `1` = top-left, `2` = top-right, `3` = bottom-left, `4` = bottom-right
- Response key source (config field vs code constant): config-defined `task.response_keys` and `task.key_list`
- If code-defined, why config-driven mapping is not sufficient: not needed; the mapping is fully config-driven
- Missing-response policy: after 30 s the item times out and is recorded as unanswered
- Correctness logic: the selected response key must match the key assigned to the correct answer card
- Reward/penalty updates: none
- Running metrics: accuracy, mean correct RT, timeout count, and practice-feedback counts

## 5. Stimulus Layout Plan

For every screen with multiple simultaneous options/stimuli:

- Screen name: instruction screen
  - Stimulus IDs shown together: `instruction_text`
  - Layout anchors (`pos`): centered at `[0, 12]`
  - Size/spacing (`height`, width, wrap): `height: 28`, `wrapWidth: 1040`
  - Readability/overlap checks: single text block, no competing stimuli
  - Rationale: keep the task introduction compact and easy to read before the matrix trials begin

- Screen name: matrix_response_open
  - Stimulus IDs shown together: matrix panels, answer cards, `response_prompt`
  - Layout anchors (`pos`): 3x3 matrix centered on the upper half of the screen; answer cards in a 2x2 array below; response prompt at `[0, -345]`
  - Size/spacing (`height`, width, wrap): matrix panels `88 px`; answer cards `132 x 92 px`; prompt `height: 24`, `wrapWidth: 980`
  - Readability/overlap checks: matrix stays visually separated from the answer cards; the prompt sits below the answer grid without overlap
  - Rationale: mirror a standard matrix-reasoning layout with the problem above and four choices below

- Screen name: matrix_response_warning
  - Stimulus IDs shown together: matrix panels, answer cards, `response_prompt`, `clock_warning`
  - Layout anchors (`pos`): warning banner at `[0, 300]`; other elements keep the same positions as the open window
  - Size/spacing (`height`, width, wrap): warning `height: 24`, `wrapWidth: 500`
  - Readability/overlap checks: warning text is placed above the matrix so it does not cover the reasoning grid or answer cards
  - Rationale: preserve the same stimulus layout while clearly marking the late-response phase

- Screen name: practice feedback / block break / goodbye
  - Stimulus IDs shown together: `practice_feedback_*`, `block_break`, `good_bye`
  - Layout anchors (`pos`): centered at `[0, 0]`
  - Size/spacing (`height`, width, wrap): `height: 26-30`, `wrapWidth: 700-1000`
  - Readability/overlap checks: single centered text block, no concurrent visual elements
  - Rationale: keep the feedback and summary states clean and unambiguous

## 6. Trigger Plan

Map each phase/state to trigger code and semantics.

- `exp_onset`: task start
- `exp_end`: task end
- `block_onset`: first and only block begins
- `block_end`: block completion
- `fixation_onset`: fixation cross appears
- `blank_onset`: blank buffer screen appears
- `matrix_response_open_onset`: matrix and answer cards appear, response window opens
- `matrix_response_warning_onset`: clock warning appears for the final 5 s
- `response_1` / `response_2` / `response_3` / `response_4`: participant selected a response card
- `response_timeout`: no response before the response window closed
- `practice_feedback_onset`: practice feedback screen appears
- `iti_onset`: inter-trial fixation appears

## 7. Architecture Decisions (Auditability)

- `main.py` runtime flow style (simple single flow / helper-heavy / why): simple mode-aware human/qa/sim flow with sequential condition ordering so practice items always stay first
- `utils.py` used? (yes/no): yes
- If yes, exact purpose (adaptive controller / sequence generation / asset pool / other): deterministic matrix-bank generation, option positioning, and stimulus assembly for the abstract reasoning screen
- Custom controller used? (yes/no): no
- If yes, why PsyFlow-native path is insufficient: not applicable
- Legacy/backward-compatibility fallback logic required? (yes/no): no
- If yes, scope and removal plan: not applicable; the only compatibility helper is the generic duration normalization for long response windows stored in milliseconds

## 8. Inference Log

List any inferred decisions not directly specified by references:

- Decision: use a 15-item short form with 3 practice items and 12 scored items
  - Why inference was required: the MaRs-IB paper provides an 80-item open-access bank, not the exact short-form battery used here
  - Citation-supported rationale: the paper gives a larger matrix-reasoning item bank and shows it is suitable for computerized research use

- Decision: split the response deadline into a 25 s open phase and a 5 s warning phase
  - Why inference was required: the paper supports a 30 s matrix-reasoning limit, but the software benefits from a distinct late-warning phase for auditability
  - Citation-supported rationale: MaRs-IB emphasizes matrix reasoning with response-time measurement and item-level timing

- Decision: map `easy` / `medium` / `hard` to increasing relational-complexity proxy bands
  - Why inference was required: the paper describes a matrix reasoning bank with variable item difficulty, but it does not define our short-form tier labels
  - Citation-supported rationale: the MaRs-IB paper explicitly discusses abstract reasoning items with differing relational demand and response-time variation

- Decision: synthesize deterministic geometric stimuli from code rather than ship copyrighted matrix items
  - Why inference was required: the task must be open-access and reproducible, but the selected papers do not provide a freely reusable item-by-item image set for this exact short form
  - Citation-supported rationale: the MaRs-IB paper says the items are open access and similar to Raven’s matrices, supporting a software-built proxy

- Decision: keep the 4-choice keyboard response mapping
  - Why inference was required: the paper describes selecting the missing matrix completion from four options, but the exact keyboard mapping is a software decision
  - Citation-supported rationale: the matrix reasoning paper describes four answer options and computerized implementation

## Contract Note

- Participant-facing labels/instructions/options should be config-defined whenever possible.
- `src/run_trial.py` should not hardcode participant-facing text that would require code edits for localization.
