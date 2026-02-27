# Default Language Configuration Specification

## Overview
Add the ability to specify a default language (e.g., `defaultLang: "sk"`) for the Cookie Consent Banner. This language will be used if `langAutoSelect` is disabled, or if the auto-detected language does not match any available languages in the `texts` array.

## Problem Statement
Currently, the banner either auto-detects the browser language (if `langAutoSelect` is true) or falls back to the first language defined in the `texts` array (which is usually `cs`). When deploying the banner on a specifically Slovak or English website, the developer might want to force the default language to "sk" or "en" regardless of the array order, while still keeping other languages available in the dropdown.

## Background
The widget determines the initial language in the `PrivacyGenerator` constructor. If `langAutoSelect` fails or is disabled, it falls back to `this.textsArray[0].lang`. We need to introduce a new configuration option `defaultLang` that takes precedence over the array order.

### Justification
This feature gives developers explicit control over the default language per deployment, which is crucial when reusing the same script on different localized domains (e.g., `.cz`, `.sk`, `.com`).

## Acceptance Criteria
- [ ] A new configuration option `defaultLang: "cs"` is available in the `options` object.
- [ ] If `defaultLang` is provided and matches an available language, it is used as the fallback instead of the first element in the `texts` array.
- [ ] If `langAutoSelect` is true and a supported browser language is detected, it still takes precedence over `defaultLang`.
- [ ] The logic falls back to `textsArray[0].lang` ONLY if `defaultLang` is not provided or invalid.

### Design Goals
- Minimal impact on existing behavior (backwards compatible).
- Simple string configuration.

## Affected Code

### Current Implementation
In `PrivacyGenerator.prototype.init`:
```javascript
		// Determine which language to use FIRST
		if (this.textsArray.length > 0) {
			// Auto-select language based on browser settings (only if multilingual is enabled)
			if (this.multilingual && this.langAutoSelect) {
				this.currentLanguage = this.detectBrowserLanguage();
				if (this.currentLanguage) {
					this.log("DEBUG", "Auto-detected language: " + this.currentLanguage);
				}
			}

			// Fallback to first language if no match or if multilingual is disabled
			if (!this.currentLanguage) {
				this.currentLanguage = this.textsArray[0].lang;
				this.log("DEBUG", "Using default language: " + this.currentLanguage);
			}
		}
```

### Required Code Changes
1. Add `defaultLang: "cs"` to the user configuration `options` object.
2. Store `this.defaultLang` in the PrivacyGenerator constructor.
3. Update the fallback logic:
```javascript
			// Fallback to explicitly configured defaultLang, then to first language in array
			if (!this.currentLanguage) {
				var configuredDefault = this.defaultLang;
				// Validate that configured default actually exists in textsArray
				var defaultExists = false;
				for (var i = 0; i < this.textsArray.length; i++) {
					if (this.textsArray[i].lang === configuredDefault) {
						defaultExists = true;
						break;
					}
				}
				
				if (configuredDefault && defaultExists) {
					this.currentLanguage = configuredDefault;
				} else {
					this.currentLanguage = this.textsArray[0].lang;
				}
				this.log("DEBUG", "Using default language: " + this.currentLanguage);
			}
```

## Implementation Plan

### Executable Steps
- [ ] Step 1: Add `defaultLang: "cs"` to the default `options` object at the bottom of `modal-cookiebar-v2.html`.
- [ ] Step 2: Initialize `this.defaultLang = options.defaultLang || "";` in the initialization section (around line 669).
- [ ] Step 3: Implement the new fallback logic in the `PrivacyGenerator` constructor (around line 692).

## Test-Driven Development

### Verification Criteria
1. When `langAutoSelect` is false and `defaultLang` is "sk", the banner loads in Slovak.
2. When `langAutoSelect` is false and `defaultLang` is invalid (e.g., "fr"), it falls back to the first item (CS).

### Test Cases
- [ ] Test 1: Set `langAutoSelect: false` and `defaultLang: "sk"`. Open browser. Banner should be in SK.
- [ ] Test 2: Set `langAutoSelect: false` and `defaultLang: "en"`. Open browser. Banner should be in EN.

## AI-Human Collaboration
- Human to review spec.md and approve the approach.
