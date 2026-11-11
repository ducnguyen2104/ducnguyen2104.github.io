---
title: "Scanning Vietnamese ID Cards and Passports with Vision"
date: 2026-11-11
draft: false
tags: ["ios", "swift", "vision", "ocr", "mrz"]
description: "How to read the machine-readable zone of a Vietnamese citizen ID card or passport on-device: a camera feed, VNRecognizeTextRequest tuned for speed, and a small parser that turns the text into fields."
---

Typing an ID number, a full name, a birth date and an expiry date into a phone is slow, and people get it wrong. Most identity documents already carry that data in a form a machine can read: the two or three lines of `<<<` characters at the bottom of a passport photo page, or on the back of a chip-based Vietnamese citizen ID card (CCCD).

This post explains how I read those lines on iOS with no server and no third-party SDK. The flow is three steps:

1. **Open the camera** with `AVCaptureSession`.
2. **Scan the document** with Vision's `VNRecognizeTextRequest`.
3. **Parse the text** with a small parser of my own.

## What is an MRZ?

MRZ stands for **machine-readable zone**. It's defined by ICAO Document 9303, the standard for travel documents, and it looks like this (ICAO's specimen passport, not a real person):

```
P<UTOERIKSSON<<ANNA<MARIA<<<<<<<<<<<<<<<<<<<
L898902C36UTO7408122F1204159ZE184226B<<<<<10
```

It was designed for machines, and that makes the job much easier:

- **Only `A–Z`, `0–9` and `<`.** No lowercase, no punctuation, no diacritics. `<` is the filler and the separator.
- **Fixed positions.** Every field always sits in the same columns.
- **Check digits.** The document number, the birth date, the expiry date and the whole zone each have one, so most reading errors can be detected.
- **A fixed font**, OCR-B, whose characters are easy to tell apart.

There are three layouts. Vietnam uses two of them:

| Format | Lines × characters | Used on |
| --- | --- | --- |
| TD1 | 3 × 30 | ID cards, including the Vietnamese CCCD |
| TD2 | 2 × 36 | Some older ID cards and visas |
| TD3 | 2 × 44 | Passports, including Vietnamese passports |

### Vietnamese specifics

- **The MRZ is on the back.** Both the CCCD and the newer *thẻ căn cước* carry it on the back of the card.
- **The ID number is split.** The TD1 document number field has room for only 9 characters, so it holds the last 9 digits of the 12-digit ID number. The full 12 digits are in the optional data field. Read the ID number from there.
- **Names have no diacritics.** `Nguyễn Văn An` becomes `NGUYEN<<VAN<AN`. If you need the accented name, it has to come from somewhere else, such as the front of the card or the chip.

## The whole pipeline

```
camera frame
   │  AVCaptureSession, late frames dropped
   ▼
text in the MRZ band
   │  VNRecognizeTextRequest: .fast, no language correction
   ▼
candidate MRZ lines
   │  only A–Z 0–9 <, exactly 30 or 44 characters
   ▼
parsed + validated fields
   │  fix look-alike letters, check every check digit
   ▼
same result in 3 frames in a row?
   │  yes → stop the camera, return the result
   ▼
document data
```

Each frame either makes it all the way through or gets dropped. Nothing is kept from a "nearly right" frame. That works because the camera delivers many frames per second, and you only need a few good ones.

## Step 1: Open the camera

The camera part is standard `AVCaptureSession` code: the back camera feeds frames to a delegate on a background queue. Three decisions matter more than the rest:

- **Drop late frames** (`alwaysDiscardsLateVideoFrames = true`). If recognition is still busy, new frames are thrown away instead of queued. You always work on the freshest image, and memory doesn't pile up.
- **Use enough resolution.** MRZ characters are only a few millimetres tall, especially on an ID card. 1080p works for me; at lower resolutions, runs of `<` blur together.
- **Draw a guide frame.** A rectangle on top of the preview tells the user where to hold the document. It also tells you where the MRZ will be in the image, which you'll use in the next step.

## Step 2: Scan the document with Vision

Each frame goes into a `VNRecognizeTextRequest`. These few settings are where most of the speed comes from:

```swift
let request = VNRecognizeTextRequest(completionHandler: handleText)
request.recognitionLevel = .fast
request.usesLanguageCorrection = false
request.regionOfInterest = mrzBand   // the bottom of the guide frame
```

### Why `.fast`

Vision has two recognition levels:

- **`.accurate`** runs a neural network over whole lines. It's better at handwriting, unusual fonts and tilted text, and it's slower.
- **`.fast`** recognizes character by character. It's quicker and lighter on battery.

An MRZ is about the easiest text there is to read: one fixed font, high contrast, no handwriting. `.fast` handles it well and lets you process many more frames per second. That matters more than per-frame accuracy, because you only need **one** good frame, and the check digits tell you when you've got it.

### Why no language correction

Language correction nudges uncertain results towards real words. That helps with sentences. On an MRZ it hurts:

- **There's nothing to correct towards.** `L898902C36UTO7408122F` isn't a word in any language. Correction can only make it worse, for example by turning a `0` into an `O`.
- **It costs time** on every frame, for no benefit.

The MRZ does need correction, just a different kind: one that knows the layout. More on that in Step 4.

### Only look where the MRZ is

`regionOfInterest` tells Vision to ignore the rest of the frame. Limiting it to the band where the MRZ sits means less work per frame, and fewer junk strings from the photo, the hologram or the table the card is lying on.

Two details are easy to get wrong:

- **Coordinates start at the bottom left**, not the top left like UIKit, and run from 0 to 1.
- **Orientation.** The back camera delivers landscape images. When the phone is held upright, pass `.right` as the orientation to the request handler. With the wrong orientation, Vision reads the image sideways and finds nothing.

## Step 3: Pick out the MRZ lines

Vision returns every piece of text it found, not just the MRZ. Before parsing, keep only lines that could be MRZ lines:

1. **Sort them top to bottom.** Line order matters in an MRZ.
2. **Uppercase them and remove spaces.** Vision sometimes splits a line into words, especially around runs of `<`.
3. **Keep only lines made of `A–Z`, `0–9` and `<`.**
4. **Look for the right shape:** three lines of exactly 30 characters (ID card) or two lines of exactly 44 (passport). Anything else, skip the frame.

A line of 29 or 31 characters isn't worth trying to repair. The next frame is only a few dozen milliseconds away.

## Step 4: Parse the lines

You don't need a library for this. Because every field has a fixed position, parsing is cutting each line at known offsets. Here's the layout of an ID card, using ICAO's specimen card:

```
Line 1:  I<  UTO  D23145890  7  <<<<<<<<<<<<<<<
         │   │    │          │  └ optional data (CCCD: full 12-digit ID number)
         │   │    │          └ check digit
         │   │    └ document number
         │   └ issuing country
         └ document type

Line 2:  740812  2  F  120415  9  UTO  <<<<<<<<<<<  6
         │       │  │  │       │  │    │            └ composite check digit
         │       │  │  │       │  │    └ optional data 2
         │       │  │  │       │  └ nationality
         │       │  │  │       └ check digit
         │       │  │  └ expiry date (YYMMDD)
         │       │  └ sex
         │       └ check digit
         └ birth date (YYMMDD)

Line 3:  ERIKSSON<<ANNA<MARIA<<<<<<<<<<
         surname, then <<, then given names separated by <
```

A passport is the same idea with a different layout: the name is on line 1, and everything else is on line 2.

The interesting parts are what makes the parser trustworthy: check digits, and fixing OCR mistakes before checking them.

### Check digits

The algorithm fits in three steps:

1. Turn each character into a number: `0–9` stay as they are, `A` is 10 … `Z` is 35, and `<` is 0.
2. Multiply them by the weights 7, 3, 1, 7, 3, 1, … in turn.
3. Add everything up. The check digit is the sum modulo 10.

For the specimen passport number `L898902C3`:

```
characters  L   8  9  8  9  0  2  C   3
values      21  8  9  8  9  0  2  12  3
weights     7   3  1  7  3  1  7  3   1
products    147 24 9  56 27 0  14 36  3   → sum 316 → check digit 6
```

The MRZ says `L898902C36`, so the number was read correctly. The parser checks **every** check digit, including the composite one at the end, which covers several fields at once. If any of them fails, the whole frame is dropped.

### Fixing what OCR gets wrong

Even on a clear image, OCR mixes up look-alike characters: `0` and `O`, `1` and `I`, `5` and `S`, `8` and `B`. You can't fix that everywhere. A Vietnamese passport number starts with a letter, so an `O` there could be correct.

But dates and check digits can **only** be digits. At those positions, an `O` must really be a `0`. So before checking anything, the parser replaces look-alike letters at the positions that must hold a digit, and nowhere else.

This is the correction an MRZ benefits from: one that knows the layout. It's why Vision's language correction can stay off.

### A few small rules

- **Names:** split at the first `<<`. Before it is the surname, after it the given names, with each single `<` becoming a space. Vietnamese names are written surname first anyway, so `NGUYEN<<VAN<AN` gives `NGUYEN` and `VAN AN`.
- **Dates** are `YYMMDD`, so the century has to be guessed. A birth year "after" this year must be in the 1900s; an expiry date is always in the 2000s. I keep dates as `DateComponents`, not `Date`: a birth date is a calendar day, and a `Date` can shift it by a day across time zones.
- **Test it with ICAO's specimen documents.** They belong to no real person, so they're safe to put in a unit test. Add a misread (`O` in a date) that should still parse, and a wrong digit that should fail.

## Step 5: Wait for a stable result

Check digits protect the document number and the dates. They **don't** protect the names. If Vision reads `NGUYEN` as `NGUYFN` in one frame, the result is still "valid".

So the scanner only accepts a result once it has seen the **same** result in three frames in a row. A misread letter rarely repeats exactly across three frames, so what you hand back is almost always right. Because `.fast` processes many frames per second, the extra frames cost the user only a fraction of a second.

## Things that still go wrong

- **Glare.** The CCCD is laminated, and a ceiling light right over the MRZ wipes out a few characters. Tilting the card slightly fixes it. A hint in the UI ("tilt to avoid reflections") helps more than any code change.
- **Focus.** Users hold the card too close. A guide frame sized for a sensible distance, or restricting auto-focus to the near range, helps.

## A note on privacy

Everything above runs on the device. Vision is local, and the parser is plain Swift. The ID data never leaves the phone unless your app sends it somewhere, which is a good property to keep for something as sensitive as a national ID.

## Wrapping up

Reading an MRZ on iOS takes three pieces, none of them a dependency:

1. **`AVCaptureSession`** delivers frames, dropping late ones.
2. **`VNRecognizeTextRequest`** with `.fast` and no language correction reads the text quickly, because an MRZ isn't natural language and doesn't need the slow path.
3. **A small parser** fixes look-alike letters where only digits are allowed, verifies every check digit and cuts the lines into fields. Waiting for the same result in a few frames covers the names the check digits miss.

The same MRZ data is also the key for the next step: reading the chip in the passport or CCCD over NFC. That's a topic for another post.
