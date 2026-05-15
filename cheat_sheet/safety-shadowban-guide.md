# Safety and Shadowban Survival Guide

The algorithm doesn't just demote bad content; it uses Grok LLMs to actively classify and restrict it.

## 1. The 7 Deadly Sins (PTOS Policies)
The `SafetyPtosPolicyClassifier` evaluates posts against 7 specific policy categories:
- `ViolentMedia`
- `AdultContent`
- `Spam`
- `IllegalAndRegulatedBehaviors`
- `HateOrAbuse`
- `ViolentSpeech`
- `SuicideOrSelfHarm`

*Tactic*: If you are walking the line on any of these (e.g., posting about true crime or war), the system may use "Deluxe Mode" which enables a chain-of-thought reasoning model (`EAPI_REASONING_INTERNAL`) to debate whether your post violates TOS. Be extremely clear about your educational/journalistic intent in the text to pass this LLM screen.

## 2. Low-Follower Spam Scrutiny
There is a dedicated classifier literally named `SpamEapiLowFollowerClassifier`.
*Tactic*: If you have a low follower count, you are under significantly higher algorithmic scrutiny for spam behavior. Avoid using multiple links, repetitive text, or aggressive tagging until your follower count is established.

## 3. The "Slop" Score
The `BangerInitialScreenClassifier` doesn't just look for quality; it specifically evaluates a `slop_score`.
*Tactic*: "Slop" refers to low-effort, mass-produced AI-generated content. If you are using LLMs to write your posts, heavily edit them to sound human. If the system flags a high `slop_score`, your reach will be severely throttled.
