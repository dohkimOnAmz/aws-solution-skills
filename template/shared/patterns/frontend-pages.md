# Frontend Page Patterns — <Solution Name>

> Replace with concrete page patterns. Stack: React 18 + Vite + Tailwind v3 + shadcn/ui + Radix + lucide-react + recharts + sonner. **Cloudscape is NOT used.**

## Project layout

```
frontend/
├── index.html
├── vite.config.ts
├── tailwind.config.ts
├── postcss.config.js
├── package.json
└── src/
    ├── main.tsx
    ├── App.tsx
    ├── index.css
    ├── lib/utils.ts
    ├── api/{client,auth}.ts
    ├── pages/
    ├── components/
    │   ├── ui/                   ← shadcn primitives
    │   ├── Layout.tsx
    │   ├── PageHeader.tsx
    │   └── StatCard.tsx
    └── hooks/
```

## Required dependencies

```json
{
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.20.0",
    "@radix-ui/react-dialog": "^1.0.5",
    "@radix-ui/react-select": "^2.0.0",
    "@radix-ui/react-tabs": "^1.0.4",
    "class-variance-authority": "^0.7.0",
    "clsx": "^2.1.0",
    "tailwind-merge": "^2.2.0",
    "tailwindcss-animate": "^1.0.7",
    "lucide-react": "^0.469.0",
    "recharts": "^2.10.4",
    "sonner": "^1.4.0",
    "oidc-client-ts": "^3.0.1",
    "react-oidc-context": "^3.1.1"
  },
  "devDependencies": {
    "tailwindcss": "^3.4.0",
    "autoprefixer": "^10.4.16",
    "postcss": "^8.4.32",
    "@vitejs/plugin-react": "^4.2.1",
    "vite": "^5.0.0",
    "typescript": "^5.3.0"
  }
}
```

## Page template

```tsx
// Replace with actual pages this solution requires
```
