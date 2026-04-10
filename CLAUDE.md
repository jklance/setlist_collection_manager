# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status

This project is in the **design phase** — no code has been written yet. The README.md contains the full product concept and data model.

## What This Is

An event planner app for tracking attendance at live performances (concerts, festivals, plays, comedy shows, etc.). Users maintain accounts to document past attendance and plan future events, with social features for coordinating with friends.

## Data Model

The core entities and their relationships (defined in README.md):

- **Venues** — locations where events happen; every Event requires a Venue
- **Artists** — performers (bands, comedians, troupes, etc.)
- **Show Runs** — named series of shows (e.g., a tour); an Artist can have many Show Runs
- **Performances** — a single exhibition at one Venue on one date; belongs to at most one Show Run
- **Events** — a collection of one or more Performances across one or more dates at a Venue
- **Notes** — user-created notes on Events, Performances, Artists, or Venues with privacy levels (Public, Friends, Private)
- **Users** — accounts with optional Setlist FM API key
- **Relationships** — user-to-user connections (following or blocked); mutual follows = friends
- **Attendance** — links Users to Performances (past and future)

## Tech Stack

Python-based (per .gitignore). No framework or tooling has been chosen yet.

## External Integrations

- **Setlist FM API** (api.setlist.fm) — for linking performance data to setlists; users store their own API key
- **Calendar export** — planned feature for Google Calendar or similar
