# Contributing

Thanks for your interest in model-router.

This project is maintained by [Dan Runion](https://github.com/runiondd). It's MIT-licensed, so you're free to fork it and adapt it for your own setup. Changes to *this* repo go through review.

## How to propose a change

1. **Fork** the repo and create a branch for your change.
2. Make your edit. Keep it focused — one logical change per pull request.
3. Open a **pull request** against `main` with a short description of *what* and *why*.
4. I review every PR and either merge, request changes, or decline it. Direct pushes to `main` are disabled — everything lands through a reviewed PR.

## What makes a good PR

- A clear reason. The skill is opinionated on purpose (the break-even gate, the no-recursion stance); if you're changing that, say why.
- Small and self-contained beats large and sweeping.
- If you're tuning a number (e.g. the ~1.5k-token threshold), explain the case that motivated it.

## Ideas welcome

Open an [issue](https://github.com/runiondd/model-router/issues) if you'd rather discuss before writing code — a routing edge case, a cleaner mechanism, or the SDK-loop main-thread variant.
