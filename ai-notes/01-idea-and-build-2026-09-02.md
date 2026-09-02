# websight — idea, build log, and how it works

Live: https://rt567.github.io/websight/  ·  Repo: github.com/RT567/websight (branch `master`, GitHub Pages serves the repo root)

## The idea

A pun on "website" = "web sight". The page is what it looks like to *look at a spider web*: a plain blue background, a deliberately bad hand-drawn-looking SVG spider web, and every so often the viewer's point of view **blinks** (black eyelids close over the screen). Extra gags layered on later:

- a little lightbulb hanging in the top right; hovering it shows the label **"high light"**
- a fly buzzes across the screen now and then, and while it does a toast in the bottom left says **"⚠ Bug occurring"**

Owner's brief, verbatim spirit: "super low effort, no UI". Keep it that way. One file, no build, no dependencies, no framework.

## Timeline (all 2026-09-02, one session)

| when | what changed | why |
|---|---|---|
| start | `index.html`: blue bg, 8-spoke SVG web, eyelid divs that snap shut ~120ms every 2–7s | initial idea |
| +1 | lightbulb SVG top right with native `<title>` tooltip "high light" | owner request |
| +2 | eyelids rounded with border-radius, blink interval 5–14s | "a little bit of roundness and a bit less common" |
| +3 | background `#2a6fd6` → `#4a8de6` | "slightly lighter blue" |
| +4 | SVG `preserveAspectRatio="none"` so web fills viewport; spokes run off every edge; gaps in rings, stray threads, debris dots | "web should touch the edge, more imperfections like a real web" |
| +5 | ring segments changed from bulging outward to sagging **inward** toward centre (quadratic curves with control point pulled toward centre) | "the curve is the wrong way" — real silk sags between spokes |
| +6 | native tooltip replaced by CSS `#bulb:hover+#tip{display:block}` label | native title tooltips have a ~1s delay; owner wanted instant |
| +7 | web regenerated programmatically: 9 spoke angles, ring vertices placed *on* the spokes with radial jitter only | hand-placed ring points no longer met the spokes after the edge stretch ("spokes don't align with the points") |
| +8 | thin 9th spoke moved from 268° (same as the −92° top spoke, invisible) to 205° | bug spotted in screenshot |
| +9 | eyelids became **concave**: each lid is a black div with a transparent ellipse cut out of its inner edge via `radial-gradient`, so the corners close first and the middle last | "top eyelid edges should be lower than the middle" |
| +10 | shut height 62% → 100% so the two lids overlap and the blink is fully black | at 62% a lens of blue stayed open |
| +11 | fly SVG + `#err` toast; first fly at 6s, then every 15–40s; crossing took 3–5s | owner request |
| +12 | fly crossing 7–10s, toast 18px with thicker yellow stripe | "move a bit slower, warning a little bigger" |

## How the file is organised (`index.html`, ~60 lines)

- **CSS**: `html,body` blue + `overflow:hidden`; `.lid` fixed full-width divs, `height:0`, transition 70ms; `#top` / `#bot` backgrounds are `radial-gradient(ellipse 60% 45% at 50% 100%|0%, transparent 99.5%, #000 100%)` (that gradient *is* the concave curve — change the ellipse radii to change the eyelid shape); `.shut .lid{height:100%}`; `#bulb`, `#tip`, `#fly`, `#err` positioning.
- **SVG web**: `viewBox 0 0 800 600`, centre (400,300), `preserveAspectRatio="none"` (stretches with the window on purpose). Contents are *generated*, see below. Don't hand-edit ring coordinates.
- **Bulb SVG** + `<span id="tip">high light</span>` sibling.
- **Fly SVG** (`#fly`) + `<div id="err">`.
- **Lids** `<div class="lid" id="top">` / `#bot`.
- **JS**: `blink()` adds `.shut` to body for 120ms, reschedules itself 5–14s later, 15% chance of a quick double blink 250ms later; first blink at 1.5s. `fly()` animates `#fly` with `requestAnimationFrame` across the viewport at a random height with two summed sine wobbles, shows `#err` for the duration, reschedules 15–40s later; first at 6s.

## Regenerating the web

The web geometry was produced by this Python (seed 7), which replaces everything between `<svg viewBox` and `</svg>`. Re-run it if you want to change spoke angles, ring radii, gaps or dangling bits.

```python
import math,random
random.seed(7)
p='index.html';s=open(p).read()
cx,cy=400,300
angs=[math.radians(a) for a in [-92,-48,-3,44,88,131,178,224,205]]  # 9th is the thin spoke
def pt(a,r): return (cx+math.cos(a)*r, cy+math.sin(a)*r)
def f(x,y): return f"{x:.0f} {y:.0f}"
out=['<svg viewBox="0 0 800 600" preserveAspectRatio="none" fill="none" stroke="#fff" stroke-width="2" stroke-linecap="round">',' <g>']
for i,a in enumerate(angs):
    x,y=pt(a,900); w=' stroke-width="1"' if i==8 else ''
    out.append(f'  <path d="M{cx} {cy} L{f(x,y)}"{w}/>')
out.append(' </g>')
order=sorted(range(9),key=lambda i:(angs[i]+2*math.pi)%(2*math.pi))
radii=[45,95,150,210,275,350]
breaks={2:(4,),4:(7,),5:(2,)}   # ring index -> segment indexes to leave broken
verts=[]
for ri,R in enumerate(radii):
    vs=[pt(angs[i],R*random.uniform(0.9,1.08)) for i in order]; verts.append(vs)
    k=0.12+ri*0.03; d="";need=True
    for j in range(len(vs)):
        a,b=vs[j],vs[(j+1)%len(vs)]
        if j in breaks.get(ri,()): need=True; continue
        if need: d+=f"M{f(*a)} "; need=False
        mx,my=(a[0]+b[0])/2,(a[1]+b[1])/2
        d+=f"Q{f(mx+(cx-mx)*k,my+(cy-my)*k)} {f(*b)} "   # sag toward centre
    out.append(f' <path d="{d.strip()}"/>')
def dang(ri,j,dx,dy,dx2,dy2):
    x,y=verts[ri][j]; return f'M{f(x,y)} Q{f(x+dx,y+dy)} {f(x+dx2,y+dy2)}'
out.append(' <path d="'+' '.join([dang(2,4,5,25,-8,40),dang(4,7,-15,15,-8,35),dang(5,2,25,30,-5,50),dang(3,5,-10,25,10,40)])+'" stroke-width="1"/>')
out.append(' <path d="M28 -10 Q50 60 40 120 L20 140 M760 -10 Q740 40 780 80 M336 -10 Q330 50 395 70" stroke-width="1"/>')
x,y=verts[3][3];out.append(f' <circle cx="{x:.0f}" cy="{y:.0f}" r="2" fill="#fff"/>')
x,y=verts[4][6];out.append(f' <circle cx="{x:.0f}" cy="{y:.0f}" r="1.5" fill="#fff"/>')
out.append('</svg>')
a=s.index('<svg viewBox');b=s.index('</svg>',a)+6
open(p,'w').write(s[:a]+"\n".join(out)+s[b:])
```

## Working on it

- Local preview: `python3 -m http.server 8347` in the repo dir → http://localhost:8347/
- To screenshot the *closed* eyelids in a headless/devtools browser, inject `<style>.lid{height:100%!important}</style>` rather than adding `.shut` — the blink timer removes `.shut` within 120ms and your screenshot will miss it.
- To test the fly, call `fly()` from the console rather than waiting.
- Deploy = `git push`. Pages rebuilds in ~30–60s. Verify with `curl -s https://rt567.github.io/websight/ | grep <something you changed>`.
- Owner tweaks arrive as one-liners ("a bit slower", "lighter blue"). Apply, commit, push, confirm in one short message. No plans, no options.

## Things deliberately not done

- No mobile-specific work; the SVG just stretches.
- No sound, no audio blink.
- Beads (`bd`) was initialised in the repo (see CLAUDE.md) but with one issue only; this project is too small to track.
