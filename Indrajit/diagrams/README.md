# Diagrams

Place exported PNG/SVG files next to the `.mmd` sources.

## Files

| Source | Intended image | Meaning |
|--------|----------------|---------|
| `dev-only-agent-promotion.mmd` | `dev-only-agent-promotion.png` | Deck claim: agents in Dev → Git → CI/CD → Prod artifacts |
| `e2e-build-and-runtime.mmd` | `e2e-build-and-runtime.png` | Full build phases + runtime |

## Generate locally

```bash
cd Indrajit/diagrams

# Deck-style promotion diagram
npx -y @mermaid-js/mermaid-cli@latest \
  -i dev-only-agent-promotion.mmd \
  -o dev-only-agent-promotion.png

# Full E2E flow
npx -y @mermaid-js/mermaid-cli@latest \
  -i e2e-build-and-runtime.mmd \
  -o e2e-build-and-runtime.png
```

Or paste either `.mmd` into https://mermaid.live and Export PNG.
