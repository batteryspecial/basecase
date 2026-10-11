# Hack the North 2026

Physical AI is limited by how its hardware was programmed, not by the hardware itself. One webcam, driven by agents, becomes a people counter, a follow-cam, a security guard, a privacy camera, a traffic counter, or a typing guide. Every switch happens from one sentence, with no retraining. Applies to other pieces of hardware as well.

## Components

| Component | Port | What it does |
| --- | --- | --- |
| Camera service | 8003 | Reads the webcam and serves raw frames. Later drives the stepper via Arduino |
| Perception (reflex layer) | 8001 | Runs every frame: detect, track, check attributes, run behaviors, draw overlays, emit events |
| Orchestrator (agent layer) | 8000 | LLM agent with tools. Turns instructions into behaviors and queries; reacts to events |
| Frontend | served by 8000 | Enriched live feed, chat with image upload, agent trace, events, behaviors, switch timer |
