# Timeline Modernization Notes

> Comprehensive guide for modernizing legacy Solid apps (AngularJS → Preact/htm)

## Overview

**Project:** solid-social/timeline
**Date:** January 2026
**Before:** AngularJS 1.x app with bower dependencies
**After:** Preact/htm single-file app with zero build step

## Architecture Decisions

### Why Preact/htm?

| Option | Build Step | Size | Learning Curve |
|--------|-----------|------|----------------|
| React | Yes | 40kb+ | Medium |
| Vue | Yes | 30kb+ | Medium |
| Preact/htm | **No** | **4kb** | Low (JSX-like) |
| Vanilla JS | No | 0kb | High (verbose) |

**Winner: Preact/htm** - React-like DX without build tooling.

### Why solid-oidc?

| Library | Size | Dependencies | Build Step |
|---------|------|--------------|------------|
| @inrupt/solid-client-authn-browser | 200kb+ | Many | Yes |
| solid-oidc | **~600 lines** | **Zero** | **No** |

**Winner: solid-oidc** - Minimal, works with ESM imports.

### ESM Imports (No Build)

```javascript
import { h, render } from 'https://esm.sh/preact@10.19.3'
import { useState, useEffect } from 'https://esm.sh/preact@10.19.3/hooks'
import htm from 'https://esm.sh/htm@3.1.1'
import * as $rdf from 'https://esm.sh/rdflib@2.2.35'
import { Session } from 'https://esm.sh/solid-oidc@0.0.3'
```

## Implementation Details

### 1. Authentication Flow

```javascript
// Polyfill for non-HTTPS (development)
if (!crypto.randomUUID) {
  crypto.randomUUID = function() {
    return '10000000-1000-4000-8000-100000000000'.replace(/[018]/g, c =>
      (+c ^ crypto.getRandomValues(new Uint8Array(1))[0] & 15 >> +c / 4).toString(16)
    )
  }
}

// Create session
const session = new Session({
  onStateChange: (e) => {
    currentWebId = e.detail.isActive ? e.detail.webId : null
    updateFetcher()
  }
})

// Restore on load
await session.restore()
await session.handleRedirectFromLogin()

// Login
await session.login(issuer, window.location.href)

// Authenticated fetch
session.authFetch(url, options)
```

### 2. Profile Discovery

```javascript
async function fetchProfile(webId) {
  const docUrl = webId.split('#')[0]
  const res = await getAuthFetch()(docUrl, {
    headers: { 'Accept': 'text/turtle, application/ld+json, */*' }
  })
  const rawText = await res.text()
  const mimeType = contentType.includes('json') ? 'application/ld+json' : 'text/turtle'
  $rdf.parse(rawText, store, docUrl, mimeType)

  const me = $rdf.sym(webId)
  const storage = store.any(me, PIM('storage'))

  // Derive timeline from storage if not explicit
  let timelineUri = store.any(me, ST('timeline'))?.value
  if (!timelineUri && storage) {
    timelineUri = storage.value.replace(/\/?$/, '/') + 'public/timeline/'
  }
}
```

### 3. Data Storage Pattern

**Daily files instead of LDP containers:**

```
/public/timeline/
├── 2024-01-01.ttl
├── 2024-01-02.ttl
└── 2024-01-03.ttl
```

**Why?** Many servers don't return `ldp:contains` for container listings. Daily files are:
- Predictable URLs (no container enumeration needed)
- Easy to fetch (just check last N days)
- Appendable via SPARQL PATCH

### 4. Creating Posts

```javascript
// First post of day: PUT new file
const turtle = `@prefix sioc: <http://rdfs.org/sioc/ns#>.
@prefix dct: <http://purl.org/dc/terms/>.
@prefix xsd: <http://www.w3.org/2001/XMLSchema#>.

<#post-${Date.now()}> a sioc:Post ;
    sioc:content """${content}""" ;
    dct:created "${new Date().toISOString()}"^^xsd:dateTime ;
    dct:creator <${userWebId}> .`

await authFetch(dayFileUri, {
  method: 'PUT',
  headers: { 'Content-Type': 'text/turtle' },
  body: turtle
})

// Subsequent posts: PATCH to append
const sparql = `INSERT DATA {
  <#post-${Date.now()}> a <http://rdfs.org/sioc/ns#Post> ;
    <http://rdfs.org/sioc/ns#content> """${content}""" ;
    <http://purl.org/dc/terms/created> "${new Date().toISOString()}"^^xsd:dateTime ;
    <http://purl.org/dc/terms/creator> <${userWebId}> .
}`

await authFetch(dayFileUri, {
  method: 'PATCH',
  headers: { 'Content-Type': 'application/sparql-update' },
  body: sparql
})
```

### 5. Fetching Posts

```javascript
async function fetchTimeline(timelineUri) {
  // Check last 30 days
  for (let i = 0; i < 30; i++) {
    const date = new Date()
    date.setDate(date.getDate() - i)
    const dayStr = date.toISOString().slice(0, 10)
    const dayFileUri = `${timelineUri}${dayStr}.ttl`

    const head = await authFetch(dayFileUri, { method: 'HEAD' })
    if (!head.ok) continue

    await fetcher.load(dayFileUri)
  }

  // Query all posts from store
  const posts = store.each(null, RDF('type'), SIOC('Post'))
}
```

### 6. Container Creation

```javascript
async function ensureContainer(uri) {
  const head = await authFetch(uri, { method: 'HEAD' })
  if (head.ok) return true

  // Create container
  await authFetch(uri, {
    method: 'PUT',
    headers: {
      'Content-Type': 'text/turtle',
      'Link': '<http://www.w3.org/ns/ldp#BasicContainer>; rel="type"'
    }
  })
}
```

## RDF Vocabularies Used

```javascript
const FOAF = $rdf.Namespace('http://xmlns.com/foaf/0.1/')      // Profiles
const SIOC = $rdf.Namespace('http://rdfs.org/sioc/ns#')        // Posts
const DCT = $rdf.Namespace('http://purl.org/dc/terms/')        // Metadata
const PIM = $rdf.Namespace('http://www.w3.org/ns/pim/space#')  // Storage
const ST = $rdf.Namespace('http://www.w3.org/ns/solid/terms#') // Solid
const LDP = $rdf.Namespace('http://www.w3.org/ns/ldp#')        // Containers
const LIKE = $rdf.Namespace('http://ontologi.es/like#')        // Likes
```

## Gotchas & Solutions

### 1. crypto.randomUUID not available on HTTP

```javascript
// Polyfill before any imports
if (!crypto.randomUUID) {
  crypto.randomUUID = function() { /* ... */ }
}
```

### 2. solid-oidc uses authFetch not fetch

```javascript
// Wrong
session.fetch(url)

// Right
session.authFetch(url, options)
```

### 3. rdflib fetcher doesn't parse JSON-LD well

```javascript
// Manual parse instead of fetcher.load()
const res = await fetch(url)
const text = await res.text()
$rdf.parse(text, store, url, 'text/turtle')
```

### 4. Servers without ldp:contains

Don't rely on container listings. Use predictable URL patterns:
- `/timeline/2024-01-03.ttl` ✓
- `/timeline/2024-01-03/` with ldp:contains ✗

### 5. GitHub Pages Jekyll build fails

Add `.nojekyll` file to bypass Jekyll processing.

## File Structure

```
timeline/
├── index.html          # Main app (single file)
├── index-legacy.html   # Original AngularJS (preserved)
├── .nojekyll           # Bypass Jekyll
├── README.md           # Documentation
├── MODERNIZATION.md    # This file
├── LICENSE
└── images/
    └── og-image.svg    # Social preview
```

## Deployment

```bash
# Both branches should be in sync
git checkout gh-pages
git merge master
git push

# GitHub Pages serves from gh-pages branch
```

## Checklist for Modernizing Other Apps

- [ ] Preserve original as `index-legacy.html`
- [ ] Set up Preact/htm with ESM imports
- [ ] Implement solid-oidc auth (with crypto polyfill)
- [ ] Create RDF store with rdflib
- [ ] Manual profile parsing (don't rely on fetcher for JSON-LD)
- [ ] Use daily .ttl files instead of LDP containers
- [ ] PUT for new files, PATCH for appending
- [ ] Add OGP meta tags
- [ ] Add .nojekyll for GitHub Pages
- [ ] Write comprehensive README
- [ ] Enable issues on repo

## Stats

| Metric | Before | After |
|--------|--------|-------|
| Framework | AngularJS 1.x | Preact/htm |
| Build step | Bower + Grunt | None |
| Auth library | Various | solid-oidc |
| Bundle size | ~500kb | ~50kb |
| Files | Many | 1 HTML file |
| Rating | N/A | 62/100 |

---

*Use this guide when modernizing linkeddata/cimba and other legacy Solid apps.*
