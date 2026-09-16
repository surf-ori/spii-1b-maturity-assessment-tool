---
name: labeling-ai-generated-content
description: Use when drafting, editing, reviewing, or approving text, images, audio, or video for publication to an EU audience that involved AI generation or AI modification. Determines whether EU AI Act Article 50 disclosure applies, which obligation (provider marking vs deployer labeling) applies, and how to label it correctly.
---

# Labeling AI-Generated Content (EU AI Act Article 50)

## Overview

EU AI Act Article 50 transparency obligations became legally mandatory on **2 August 2026**. If you generate, modify, or publish AI content for an EU audience, you likely owe a disclosure duty, separate from whether you also choose to use the EU's optional labeling icons.

**Core principle:** The disclosure duty is mandatory; the EU icons are optional. "We didn't use the icon" or "someone touched the wording" is not proof of compliance. Check the actual rule below.

## When to Use

- Publishing AI-generated or AI-modified image, audio, or video that resembles a real person, place, or event ("deepfakes"): hero images, social videos, synthetic voiceovers, AI-touched-up photos
- Publishing AI-generated or AI-modified text on matters of public interest (news, policy statements, official announcements)
- Deciding whether an AI tool's output needs a visible label before it ships
- Reviewing someone else's content for publication and it's unclear whether AI was involved or disclosed
- Choosing whether/how to use the official EU AI-content icons

## Two Separate Obligations

| Role | Who | Duty |
|---|---|---|
| **Provider** (Art. 50(2)) | Builds/offers the generative AI system | Mark outputs (audio/image/video/text) in machine-readable format, detectable as AI-generated/manipulated |
| **Deployer** (Art. 50(4)) | Uses an AI system to generate/manipulate content and publishes it | Disclose to the audience that the content is artificially generated/manipulated |

Most communications/editorial work sits in the **deployer** role: you used a tool (ChatGPT, Midjourney, etc.), you didn't build one. This skill focuses on the deployer disclosure duty.

## Decision Checklist (Deployer)

1. **Is it image, audio, or video that resembles a real person/object/place and would appear authentic ("deepfake")?**
   → **Always disclose.** No editorial-review exemption exists for this category. Exceptions only for evidently artistic/satirical/fictional works (still needs a "suitable" disclosure that doesn't spoil the work) and narrow law-enforcement uses.

2. **Is it text on a matter of public interest (news, official statements, policy content)?**
   → Disclosure is required **unless** the text underwent genuine human editorial review **and** a named natural or legal person holds editorial responsibility for it.
   - Genuine review = someone actually checked and took ownership of the content and its claims, not a skim for typos.
   - "I lightly edited the AI draft myself" does not automatically clear this bar. The organization must be able to name who is editorially responsible and show real review happened.
   - If in doubt, disclose.

3. **Neither of the above** (internal draft, non-public-interest text, never published) → no Article 50 deployer duty, though your own editorial policy may still call for disclosure.

## How to Label

Use the **EU AI-content icons** (optional but recommended, free and pre-tested for perceivability):
- **Basic**: AI was involved in creating the content
- **Fully AI-Generated**: content created entirely by AI
- **Partially AI-Modified**: real/human content that AI altered
- Each comes in black/white, full/50%-transparent variants.

Placement rules (Code of Practice), whether you use the icons or a plain-text label:
- Clearly perceivable and distinguishable **at the latest at first exposure**, not buried in a footer, alt text alone, or file metadata
- Embedded directly in the content itself, except for clearly creative/artistic works (suitable alternative applies there)
- Must **travel with the content**: stays visible when reshared, downloaded, or repurposed (social crops, email newsletters, etc.)
- Pair the icon/tag with a **plain-language caption** (e.g., "AI-generated illustration, not a photograph"), not just a symbol

Provider-side machine-readable marking (e.g., watermarking) is the AI tool vendor's job, not yours, but when choosing between tools, prefer ones that support it.

## Common Mistakes

| Mistake | Reality |
|---|---|
| "I edited the AI text a bit, so no disclosure needed" | Only genuine editorial review plus assumed, named editorial responsibility exempts text, and only text. Light copyediting isn't enough. |
| "It's an image, so if I just skip the EU icon, I'm fine" | The icon is optional; the underlying disclosure duty is not. A plain-language caption alone still satisfies it, but something visible is required. |
| "We disclosed it in the image file metadata" | Must be perceivable at first exposure to the audience, not just embedded invisibly. |
| "The caption got cropped when it was reused on social" | Labels must persist through reshares/downloads. That's a Code of Practice placement requirement, not a nice-to-have. |
| "This is just a future rule, not in force yet" | Article 50 became mandatory 2 August 2026. It is in force now. |
| Treating avoidance ("just find a real photo instead") as the compliance strategy | Fine when a real alternative exists, but doesn't cover the case where it doesn't. You still need the labeling path. |

## Sources

- EU icons for labelling AI-generated content: https://digital-strategy.ec.europa.eu/en/policies/eu-icons-labelling-ai-generated-content
- Code of Practice on transparency of AI-generated content: https://digital-strategy.ec.europa.eu/en/policies/code-practice-ai-generated-content
- Legal basis: EU AI Act, Article 50
