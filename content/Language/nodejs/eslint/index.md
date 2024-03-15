---
title: Eslint plugin for TypeScript
tags:
  - nodejs
date: '2023-11-04'
summary: ESLint is a valuable tool for improving code quality, maintaining consistency, preventing errors
---

<!-- TypeScript is a strongly typed programming language that builds on JavaScript.
TypeScript adds additional syntax to JavaScript that allows you to declare the shapes of objects and functions in code. It provides a set of language services that allow for running powerful inferences and automations with that type information.

ESLint is an awesome linter for JavaScript code.
ESLint statically analyzes your code to quickly find problems. It allows creating a series of assertions called lint rules around what your code should look or behave like, as well as auto-fixer suggestions to improve your code for you, and loading in lint rules from shared plugins. -->

## why Typescript need ESLint
ESLint is widely used in the JavaScript development ecosystem. The rule can help enforce a consistent coding style across a project or team. ESLint assists in maintaining high code quality standards,catches common programming errors and potential bugs during development.
ESLint is highly configurable. Developers can customize its rules based on the requirements of their projects or teams. This flexibility allows ESLint to adapt to different coding styles and preferences.

ESLint is a valuable tool for improving code quality, maintaining consistency, preventing errors. JavaScript development projects. It helps developers write cleaner, more reliable code.

## mainly eslint plugin package 

1. **@typescript-eslint/eslint-plugin**
    - An ESLint plugin which provides lint rules for TypeScript codebases.

2. **@typescript-eslint/parser**
     - An ESLint parser which leverages TypeScript ESTree to allow for ESLint to lint TypeScript source code.
3. **eslint**:
    - ESLint uses Espree for JavaScript parsing.
    - ESLint uses an AST to evaluate patterns in code.
    - ESLint is completely pluggable, every single rule is a plugin and you can add more at runtime.

4. **@typescript-eslint/eslint-plugin-tslint**
    - ESLint plugin that wraps a TSLint configuration and lints the whole source using TSLint.
5. **eslint-config-prettier**:
   - Turns off all rules that are unnecessary or might conflict with Prettier.
   - This lets you use your favorite shareable config without letting its stylistic choices get in the way when using Prettier.
   - Note that this config only turns rules off, so it only makes sense using it together with some other config.

## others common eslint plugin

 - **eslint-plugin-import**
 - **eslint-plugin-jsdoc**
 - **eslint-plugin-no-null**
 - **eslint-plugin-prefer-arrow**
 - **eslint-plugin-sonarjs**
 - **eslint-plugin-unicorn**
 - **prettier**
 - **sonarjs**
 - **eslint-config-prettier**
 - **eslint-plugin-etc**
 - **eslint-plugin-jsdoc**
 - **eslint-plugin-node**
 - **eslint-plugin-prefer-arro**
 - **eslint-plugin-sonarjs**
 - **eslint-plugin-unicorn**
 - **eslint-config-airbnb**
 - **eslint-import-resolver-typescript**
 - **eslint-plugin-etc**
 - **eslint-plugin-jsx-a11y**
 - **eslint-plugin-node**
 - **eslint-plugin-prefer-arrow**
 - **eslint-plugin-react**
 - **eslint-plugin-react-hooks**

## Typescript/ESLint Link

if you want to know about typescript and eslint rules,please refer below.
- https://typescript-eslint.io/rules/

- https://eslint.org/docs/latest/rules/


