# Next.js ISR revalidation external-memory retention repro

Minimal reproduction for a memory retention in Next.js App Router ISR revalidation:
RSC segment data produced by each re-render is kept alive through the module-scope
`CachedParams` WeakMap in `next/dist/server/request/params.js` (the cached promise
captures the render's AsyncLocalStorage context). The effect is strongly amplified
on Node 22 compared with Node 24.

## Layout

- `app/posts/[slug]/page.tsx` - one ISR route (`revalidate = 3600`), 200 params, ~100 KB rendered body
- `instrumentation.ts` - logs `process.memoryUsage()` every 5s (`[mem]` lines)
- `drive.mjs` - forces ISR re-renders of all 200 routes per round via the `x-prerender-revalidate` header and prints the server's memory line after each round

## Run

```bash
npm install
npm run build
npm run start > server.log 2>&1 &
node drive.mjs server.log 25
```

Run once under Node 22.x and once under Node 24.x (fresh server each time).

## Results observed (this machine, Linux x64)

- **Node 22.22.3**: `arrayBuffers` climbs to a 75-85 MB plateau sustained for 14
  consecutive rounds (rounds 8-21) before a single full GC drops it to 27 MB; RSS drifts 271 -> 327 MB.
  Raw numbers: `results-node22b.txt`.
- **Node 24.18.0**: collected continuously; `arrayBuffers` oscillates 24-42 MB, RSS flat ~375 MB.
  Raw numbers: `results-node24.txt`.

In a real application (785 ISR routes, ~100 KB RSC payloads) the same A/B held
190-240 MB (Node 22) vs 72-85 MB (Node 24), and in production it accumulated to
1.16 GB of arrayBuffers over ~4 days with a steady ~230 MB JS heap.
