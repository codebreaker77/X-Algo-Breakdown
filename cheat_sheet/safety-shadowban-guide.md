# Safety and Shadowban Survival Guide

The algorithm doesn't just demote bad content based on user reports; it uses Grok LLMs to actively read, classify, and restrict content the moment it is published.

---

## 1. The 7 Deadly Sins (PTOS Policies)

The `SafetyPtosPolicyClassifier` evaluates posts against 7 specific policy categories. Rather than looking for "bad words," it uses an LLM to understand the *intent* of your post.

1.  `ViolentMedia`
2.  `AdultContent`
3.  `Spam`
4.  `IllegalAndRegulatedBehaviors`
5.  `HateOrAbuse`
6.  `ViolentSpeech`
7.  `SuicideOrSelfHarm`

### The "Deluxe Mode" Trap
If your post is ambiguous (e.g., you are a journalist reporting on a conflict, or a gamer posting a clip from a violent video game), the classifier enters **Deluxe Mode**.
*   **How it works:** It uses an advanced reasoning model (`EAPI_REASONING_INTERNAL`) and literally strips away prompt restrictions (`_strip_thinking_restrictions()`) to allow the AI to debate with itself via Chain-of-Thought reasoning.
*   **Example (Flagged):** Posting a video of a street fight with the caption "Crazy hit!" The LLM reasoning trace will conclude this promotes violence and flag it under `ViolentMedia`.
*   **Example (Safe):** Posting the exact same video with the caption, "Local news report: Suspect apprehended after altercation downtown. A prime example of why community policing is needed." The LLM reasoning trace recognizes the journalistic/educational intent and clears the post.

---

## 2. Low-Follower Spam Scrutiny

There is a dedicated classifier literally named `SpamEapiLowFollowerClassifier`. The algorithm mathematically treats new or small accounts as highly suspicious.

### What Triggers the Spam Filter
*   **Link Dropping:** Posting external links (especially affiliate links, Substack links, or crypto links) when you have under 500 followers.
*   **Repetitive Tagging:** Tagging large accounts in posts trying to get their attention.
*   **Reply Spamming:** Copy-pasting the same reply into multiple threads.

**Tactic:** If you are a new account, stick to native, text-only or image-only content. Build your trust score organically before you start routing traffic off-platform with external URLs.

---

## 3. The "Slop" Score

The `BangerInitialScreenClassifier` doesn't just look for quality; it specifically evaluates a `slop_score`. "Slop" is the internal algorithmic term for low-effort, mass-produced AI-generated content.

### How to Avoid Being Flagged as Slop
If the VLM detects that you copy-pasted a response from ChatGPT, your reach will be severely throttled.

*   **Avoid LLM Formatting:** Do not use the classic `Introduction -> Three Bullet Points -> Conclusion` format that raw LLMs output.
*   **Avoid LLM Vocabulary:** Words like "Delve", "Tapestry", "Crucial", "Navigating the digital landscape", and "Synergy" are massive red flags to the classifier.
*   **Avoid Excessive Emojis:** Using an emoji at the start of every single bullet point is a common artifact of zero-shot AI generation.

**Tactic:** Even if you use AI to draft content, rewrite it in your own voice. Make it conversational, slightly asymmetrical, and include specific, personal anecdotes. The classifier is looking for *human authenticity*.
