# Breaking News — JerseyCTF Writeup

**Challenge:** Breaking News  
**Category:** OSINT  
**Difficulty:** Easy  
**Points:** 372  
**Solves:** 81  
**Challenge Author:** Madeline B  
**Writeup by:** zham  
**Flag:** `jctf{Izvestiya}`

---

## Description

> After my time in space, I prepared for reentry. I remembered my friend who crashed after his space voyage, and for a moment, I thought I might face the same fate. Luckily, I had a smooth landing and was found by two civilians in the area. Soon after, I was sent congratulations from a newspaper correspondent.
>
> Which newspaper did the correspondent write for?

---

## Background Knowledge (Read This First!)

### Who was Yuri Gagarin?

**Yuri Gagarin** was a Soviet cosmonaut and the first human to travel into outer space. On **April 12, 1961**, he orbited Earth aboard **Vostok 1** for 108 minutes. Rather than landing inside the capsule, Gagarin ejected at ~7,000 meters and parachuted down separately — a fact the Soviet Union kept secret for years to preserve FAI world records.

### Who was Vladimir Komarov?

**Vladimir Komarov** was a Soviet cosmonaut and one of Gagarin's closest friends. He commanded **Soyuz 1** in April 1967 — but the mission was plagued with technical failures. During reentry, his parachute failed to deploy correctly. The capsule crashed into the ground at high speed and Komarov was killed, becoming the **first human to die in a spaceflight incident**.

### What is Izvestiya?

**Izvestiya** (Russian: Известия, meaning *"News"*) is one of Russia's oldest and most prominent newspapers, founded in 1917. During the Soviet era it served as an official state publication. Its correspondents were frequently present at major state events — including the recovery of cosmonauts after landing.

### What is the Wilson Center Digital Archive?

The **Wilson Center Digital Archive** is a declassified primary source repository hosting translated Soviet and Cold War documents. It includes Gagarin's own **Top Secret Report to the State Commission**, filed on April 13, 1961 — the day after his flight — in which he describes in detail everything that happened from launch to recovery.

---

## Solution — Step by Step

### Step 1 — Identify the narrator

The challenge is written in the **first person**. The key clues are:

- "After my time in **space**" → the narrator is a cosmonaut or astronaut
- "A smooth landing and was found by **two civilians**" → a very specific historical detail
- "My **friend** who **crashed** after his space voyage" → a friend who died in a spaceflight accident

The combination of a smooth landing, found by two civilians, and a friend who crashed in space points to one person: **Yuri Gagarin**.

After Vostok 1, Gagarin parachuted into a field near Engels, Russia, where a kolkhoz woman and her granddaughter were the first people to find him. His close friend Vladimir Komarov later died when Soyuz 1 crashed on reentry in 1967.

### Step 2 — Research the landing in detail

A search for Gagarin's landing confirms the "two civilians" clue precisely:

> *"A kolkhoz woman Annihayat Nurskanova and her granddaughter, Rita, observed the strange scene of a figure in a bright orange suit with a large white helmet landing near them by parachute."*
> — Wikipedia, Vostok 1

This matches the challenge description perfectly. Gagarin himself recalled:

> *"When they saw me in my space suit and the parachute dragging alongside as I walked, they started to back away in fear. I told them, don't be afraid, I am a Soviet citizen like you, who has descended from space and I must find a telephone to call Moscow!"*

### Step 3 — Find the primary source

The question asks specifically which newspaper the correspondent wrote for. Rather than relying on secondary sources, we go straight to Gagarin's own declassified report.

The **Wilson Center Digital Archive** hosts the full translated text of Gagarin's Top Secret Report to the State Commission (April 13, 1961), filed in his own words:

> *"Then there were congratulations from the correspondent of Pravda [5], the correspondent of Izvestiya [6], and Comrade IL'ICHEV, the chief agitator and propagandist."*

The footnotes identify both correspondents by name:

- **[5]** — Nikolay Nikolayevich Denisov (1909–1983), editor of the military department of **Pravda**
- **[6]** — Georgiy Nikolayevich Ostroumov (1919–2001), correspondent of **Izvestiya**

### Step 4 — Determine the correct newspaper

The challenge says "a newspaper correspondent" (singular), so we need to determine which one the challenge author intended.

- **Pravda** was tried first — incorrect.
- **Izvestiya** was tried second — **correct**.

The flag is based on the **Izvestiya** correspondent, Georgiy Ostroumov, footnote [6] in Gagarin's own report.

---

## Flag

```
jctf{Izvestiya}
```

---

## Tools Used

| Tool | Purpose | Level |
|------|---------|-------|
| Google / web search | Initial research on cosmonaut landings | ⭐ Easy |
| Wikipedia (Vostok 1) | Confirmed the two civilians who found Gagarin | ⭐ Easy |
| Wilson Center Digital Archive | Located Gagarin's primary source declassified report | ⭐⭐ Medium |

---

## Key Takeaways

- **Read carefully** — the challenge is written from the narrator's perspective. Identifying *who* is speaking is the first and most important step.
- **Two civilians is a very specific clue** — this detail narrows the field to Gagarin's 1961 landing almost immediately. Not many space landings in history involved being found by a woman and her granddaughter.
- **Primary sources beat secondary sources** — Wikipedia pointed to Pravda, which was wrong. Gagarin's own declassified report revealed *two* correspondents were present, and the correct answer was the second one: Izvestiya.
- **Don't stop at the first answer** — always verify against the most authoritative source available. The Wilson Center's translation of Gagarin's top-secret report was the definitive reference that cracked this challenge.
