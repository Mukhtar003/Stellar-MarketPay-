# Storybook bundler

The Next.js application is built by Next.js and reads `next.config.mjs`.
Storybook is the only frontend tool that uses Vite here: `.storybook/main.ts`
selects `@storybook/react-vite`. The removed root `vite.config.ts` was not
loaded by Next.js, Storybook, or the test scripts.
