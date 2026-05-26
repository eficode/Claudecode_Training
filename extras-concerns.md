Juuri kokeilin omaa Bedrock konffia ja on itsellä todella ripeä joten todennäköinen culprit on ACME corp. AWS Accounttiin ja sitä kautta tiettyyn modeliin liittyvä throttle koska kyseessä on kuitenkin defauttina on-demand palvelu. 

Yksi mitä voitte kokeilla on tuunata nykykonffia (todnäk .claude/settings.json file) ja muuttaa jonkun toisen modelin käyttöön ja katsoa nopeutuuko. 

Mulla esmes nyt :

    "ANTHROPIC_MODEL": "global.anthropic.claude-sonnet-4-5-20250929-v1:0",
    "ANTHROPIC_SMALL_FAST_MODEL": "eu.anthropic.claude-3-haiku-20240307-v1:0"

Tässä listaa mitä voi kokeilla 

Global Cross-Region Models (Worldwide)

  - global.anthropic.claude-opus-4-6-v1 - Claude Opus 4.6
  - global.anthropic.claude-opus-4-5-20251101-v1:0 - Claude Opus 4.5
  - global.anthropic.claude-sonnet-4-5-20250929-v1:0 - Claude Sonnet 4.5 ✓ (your current)
  - global.anthropic.claude-haiku-4-5-20251001-v1:0 - Claude Haiku 4.5

  EU Cross-Region Models

  - eu.anthropic.claude-opus-4-6-v1 - Claude Opus 4.6
  - eu.anthropic.claude-opus-4-5-20251101-v1:0 - Claude Opus 4.5
  - eu.anthropic.claude-sonnet-4-5-20250929-v1:0 - Claude Sonnet 4.5
  - eu.anthropic.claude-haiku-4-5-20251001-v1:0 - Claude Haiku 4.5
  - eu.anthropic.claude-sonnet-4-20250514-v1:0 - Claude Sonnet 4
  - eu.anthropic.claude-3-7-sonnet-20250219-v1:0 - Claude 3.7 Sonnet
  - eu.anthropic.claude-3-5-sonnet-20240620-v1:0 - Claude 3.5 Sonnet
  - eu.anthropic.claude-3-sonnet-20240229-v1:0 - Claude 3 Sonnet
  - eu.anthropic.claude-3-haiku-20240307-v1:0 - Claude 3 Haiku
